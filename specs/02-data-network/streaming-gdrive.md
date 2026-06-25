# streaming-gdrive.md

> Google Drive の OAuth 連携 + 楽曲同期 + ストリーミングプロキシ仕様。
> プロキシは **Ktor CIO で独自実装** (CloudStreamProxy を継承しない)。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/gdrive/
├─ GDriveConstants.kt          // 18 lines (定数群)
├─ GDriveApiService.kt         // 207 lines (OAuth + REST ラッパ)
├─ GDriveRepository.kt         // 545 lines (同期 + 認証)
└─ GDriveStreamProxy.kt        // 273 lines (Ktor CIO プロキシ)
```

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `presentation/gdrive/*`, `data/repository/MusicRepositoryImpl.kt` |
| 下流 (依存先) | `data/database/{GDriveDao, MusicDao}`, `data/preferences/PlaylistPreferencesRepository`, `data/stream/CloudMusicUtils`, `data/stream/CloudStreamSecurity`, `androidx.security.crypto.EncryptedSharedPreferences`, `io.ktor:ktor-server-cio`, `okhttp3.OkHttpClient`, `com.google.api.services.drive` 系 API は使わず自前 OkHttp。 |

---

## 1. `GDriveConstants` (`object`)

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `WEB_CLIENT_ID` | `"YOUR_WEB_CLIENT_ID.apps.googleusercontent.com"` | **プレースホルダ** (要差し替え) |
| `SCOPE_DRIVE_READONLY` | `"https://www.googleapis.com/auth/drive.readonly"` | OAuth scope |
| `TOKEN_ENDPOINT` | `"https://oauth2.googleapis.com/token"` | |
| `DRIVE_API_BASE` | `"https://www.googleapis.com/drive/v3"` | |
| `AUDIO_MIME_TYPES` | `setOf("audio/mpeg", "audio/mp3", "audio/wav", "audio/x-wav", "audio/flac", "audio/x-flac", "audio/mp4", "audio/x-m4a", "audio/aac", "audio/ogg", "audio/x-vorbis+ogg", "audio/webm")` | 検索クエリ用 |

> ⚠️ `WEB_CLIENT_ID` は **要差し替え**。現状のままだと OAuth 交換時に失敗する。

---

## 2. `GDriveApiService`

### 依存 (`@Inject constructor`)

| 依存 | 用途 |
| --- | --- |
| `OkHttpClient` | ベース |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `accessToken` | `var String?` | 現在の access token (refresh で更新) |

### 公開 API

#### トークン管理

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `setAccessToken` | `fun setAccessToken(token: String)` | メモリに反映。 |
| `clearAccessToken` | `fun clearAccessToken()` | メモリから破棄。 |
| `hasToken` | `fun hasToken(): Boolean` | `accessToken` 非空。 |
| `getAuthHeader` | `fun getAuthHeader(): String` | `"Bearer <accessToken>"` (空でも prefix は付与)。 |

#### ファイル操作

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getStreamUrl` | `fun getStreamUrl(fileId): String` | `"https://www.googleapis.com/drive/v3/files/$fileId?alt=media"` (token はヘッダで送る)。 |
| `listAudioFiles` | `suspend fun listAudioFiles(folderId, pageToken?): String` | `/files?q='<folderId>' in parents and (<audio mime or>) and trashed=false&fields=...&pageToken=...`。 |
| `listFolders` | `suspend fun listFolders(parentId = "root", pageToken?): String` | `/files?q='<parentId>' in parents and mimeType='application/vnd.google-apps.folder' and trashed=false`。 |
| `getFileMetadata` | `suspend fun getFileMetadata(fileId): String` | `/files/{id}?fields=...`。 |
| `createFolder` | `suspend fun createFolder(name, parentId = "root"): String` | `POST /files` で `mimeType=application/vnd.google-apps.folder`。 |
| `getUserInfo` | `suspend fun getUserInfo(): String` | `userinfo?alt=json` を `oauth2` 経由で。 |

#### OAuth Token

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `exchangeAuthCode` | `suspend fun exchangeAuthCode(code, clientId, clientSecret, redirectUri): String` | `POST /token` (application/x-www-form-urlencoded) `code=...&client_id=...&client_secret=...&redirect_uri=...&grant_type=authorization_code`。 |
| `refreshToken` | `suspend fun refreshToken(refreshToken, clientId, clientSecret): String` | `grant_type=refresh_token` で更新。 |

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `executeGet` | `private suspend fun executeGet(url): String` | GET 共通。`Authorization: Bearer` を付与。 |

### 内部実装メモ

- **手動 OkHttp**: `com.google.api.services.drive` は使わず、軽量に `OkHttpClient` のみで叩く。
- **ページング**: `pageToken` 引数で繰り返し呼び出し可能。

---

## 3. `GDriveRepository` (`@Singleton`)

### 依存 (`@Inject constructor`)

| 依存 | 用途 |
| --- | --- |
| `GDriveApiService` | OAuth + REST |
| `GDriveDao` | GDrive 専用 DAO |
| `MusicDao` | 統合 DAO |
| `@ApplicationContext Context` | EncryptedSharedPreferences |

### 内部データクラス

```kotlin
data class BulkSyncResult(
    val folderCount: Int,
    val syncedSongCount: Int,
    val failedFolderCount: Int
)

data class DriveFolder(val id: String, val name: String)
```

### 定数

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `GDRIVE_SONG_ID_OFFSET` | `6_000_000_000_000L` | 統合 Song ID |
| `GDRIVE_ALBUM_ID_OFFSET` | `7_000_000_000_000L` | 統合 Album ID |
| `GDRIVE_ARTIST_ID_OFFSET` | `8_000_000_000_000L` | 統合 Artist ID |
| `GDRIVE_PARENT_DIRECTORY` | `"/Cloud/GoogleDrive"` | |
| `GDRIVE_GENRE` | `"Google Drive"` | |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `prefs` | `SharedPreferences` | EncryptedSharedPreferences (`gdrive_access_token`, `gdrive_refresh_token`, `gdrive_token_expires_at`, `gdrive_email`, `gdrive_display_name`, `gdrive_avatar`) |
| `_isLoggedInFlow` | `MutableStateFlow<Boolean>` | |
| `isLoggedInFlow` | `StateFlow<Boolean>` | |

### 派生プロパティ

| プロパティ | 型 | 説明 |
| --- | --- | --- |
| `isLoggedIn` | `Boolean` | `api.hasToken()` |
| `userEmail` | `String?` | prefs の `gdrive_email` |
| `userDisplayName` | `String?` | prefs の `gdrive_display_name` |
| `userAvatar` | `String?` | prefs の `gdrive_avatar` |

### 認証情報管理

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `initFromSavedTokens` | `private fun initFromSavedTokens()` | prefs から復元 + `api.setAccessToken` |
| `loginWithCredential` | `suspend fun loginWithCredential(authCode, clientId, clientSecret, redirectUri): Result<String>` | `api.exchangeAuthCode` → ユーザ情報取得 → prefs 永続化 → `dao.clearAll` |
| `refreshAccessToken` | `suspend fun refreshAccessToken(): Result<String>` | `api.refreshToken` でトークン更新 |
| `ensureValidToken` | `private suspend fun ensureValidToken()` | 有効期限チェック (60 秒前) → 必要なら refresh |
| `logout` | `suspend fun logout()` | DB + prefs + `api.clearAccessToken` + flow false |
| `saveTokens` | `private fun saveTokens(accessToken, refreshToken?, expiresIn)` | prefs に保存 |

### フォルダ操作

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getFolders` | `fun getFolders(): Flow<List<GDriveFolderEntity>>` | DAO 監視 |
| `listDriveFolders` | `suspend fun listDriveFolders(parentId = "root"): Result<List<DriveFolder>>` | `api.listFolders` をページング |
| `createMusicFolder` | `suspend fun createMusicFolder(parentId = "root"): Result<DriveFolder>` | `"PixelPlay Music"` フォルダ作成 |
| `addFolder` | `suspend fun addFolder(folderId, name)` | DB に登録 |
| `removeFolder` | `suspend fun removeFolder(folderId)` | DB から削除 |

### 同期

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `syncFolderSongs` | `suspend fun syncFolderSongs(folderId): Result<Int>` | `api.listAudioFiles` をページング → パース → DB upsert |
| `syncAllFoldersAndSongs` | `suspend fun syncAllFoldersAndSongs(): Result<BulkSyncResult>` | 全フォルダ順次 |

### クエリ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getAllSongs` | `fun getAllSongs(): Flow<List<Song>>` | |
| `getFolderSongs` | `fun getFolderSongs(folderId): Flow<List<Song>>` | |
| `getStreamUrl` | `fun getStreamUrl(fileId): String` | `api.getStreamUrl` |
| `getAuthHeader` | `fun getAuthHeader(): String` | Stream Proxy / Refresh 用 |

### ファイル名パース

`parseFileToEntity` で `"<artist> - <title>.mp3"` 形式を想定。

```kotlin
val nameWithoutExt = fileName.substringBeforeLast(".")
val parts = nameWithoutExt.split(" - ", limit = 2)
val artist = if (parts.size == 2) parts[0] else "Unknown Artist"
val title = if (parts.size == 2) parts[1] else nameWithoutExt
val album = fileName.substringBeforeLast(".").take(50)  // 暫定
```

### 統合レイヤー

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `syncUnifiedLibrarySongsFromGDrive` | `private suspend fun syncUnifiedLibrarySongsFromGDrive()` | `GDriveSongEntity` → 統合 Song/Album/Artist upsert |
| `parseFileToEntity` | `private fun parseFileToEntity(file: JSONObject, folderId): GDriveSongEntity` | ファイルメタデータ → エンティティ |
| `parseArtistNames` | `private fun parseArtistNames(rawArtist): List<String>` | 区切り文字分割 |
| `toUnifiedSongId` | `private fun toUnifiedSongId(driveFileId): Long` | `GDRIVE_SONG_ID_OFFSET + driveFileId.hashCode().absoluteValue` |
| `toUnifiedAlbumId` | `private fun toUnifiedAlbumId(albumName): Long` | `GDRIVE_ALBUM_ID_OFFSET + albumName.lowercase().hashCode().absoluteValue` |
| `toUnifiedArtistId` | `private fun toUnifiedArtistId(artistName): Long` | |

---

## 4. `GDriveStreamProxy` (`@Singleton`)

**`CloudStreamProxy` を継承せず**、独自 Ktor CIO サーバを起動する。理由: OAuth トークンの動的更新が必要なため、基底のキャッシュ抽象と相性が悪い。

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `ALLOWED_REMOTE_HOST_SUFFIXES` | `setOf("googleapis.com", "googleusercontent.com", "gstatic.com")` | ホワイトリスト |
| `server` | `var EmbeddedServer<CIOApplicationEngine, CIOApplicationEngine.Configuration>?` | Ktor サーバ |
| `actualPort` | `var Int` | バインド済みポート |
| `proxyScope` | `CoroutineScope` | `SupervisorJob + Dispatchers.IO` |
| `startJob` | `var Job?` | サーバ起動ジョブ |

### 公開 API

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `isReady` | `fun isReady(): Boolean` | `actualPort > 0` |
| `startIfNeeded` | `fun startIfNeeded()` | 既に起動中なら no-op |
| `awaitReady` | `suspend fun awaitReady(timeoutMs = 10_000L): Boolean` | 50ms ポーリング |
| `ensureReady` | `suspend fun ensureReady(timeoutMs = 10_000L): Boolean` | 未起動なら起動 + 待機 |
| `getProxyUrl` | `fun getProxyUrl(fileId): String` | `"http://127.0.0.1:<port>/gdrive/<fileId>"` |
| `resolveGDriveUri` | `fun resolveGDriveUri(uriString: String): String?` | URI scheme `gdrive://<fileId>` からプロキシ URL を組み立て |
| `start` | `fun start()` | `ServerSocket(0)` で空きポート取得 → Ktor 起動 |
| `stop` | `fun stop()` | サーバ停止 + スコープ cancel |

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `createServer` | `private fun createServer(port): EmbeddedServer<...>` | Ktor ルーティング定義。`GET /gdrive/{fileId}` を処理。 |

### Ktor ルート内部

```
GET /gdrive/{fileId}
1. rangeValidation = CloudStreamSecurity.validateRangeHeader(call.request.headers["Range"])
2. streamUrl = repository.getStreamUrl(fileId)
3. authHeader = repository.getAuthHeader()
4. requestBuilder = Request.Builder().url(streamUrl)
   .header("Authorization", authHeader)
5. upstream = withContext(Dispatchers.IO) {
   okHttpClient.newCall(requestBuilder.build()).execute()
}
6. if (upstream.code == 401):
   refreshResult = repository.refreshAccessToken()
   newAuthHeader = repository.getAuthHeader()
   upstream = retry with new auth header
7. respondBytesWriter で Content-Type / Content-Range / Accept-Ranges を維持しつつ転送
8. 64 KB buffer で chunk transfer
```

### Ktor の選定理由

- **シンプルな HTTP サーバ**: OkHttp で upstream fetch + Ktor で localhost serve。
- **Range 制御**: Media3 (ExoPlayer) の seek に必須。
- **Range 検証**: `CloudStreamSecurity.validateRangeHeader` で SSRF・DoS 対策。

---

## 5. 内部実装メモ

### OAuth 2.0 暗黙フロー

```
1. ユーザを Google OAuth エンドポイントへ誘導 (Authorization Code Flow)
2. リダイレクトで code を取得
3. exchangeAuthCode(code, ...) で access_token + refresh_token を取得
4. access_token をメモリに、refresh_token を EncryptedSharedPreferences に保存
5. 有効期限前に refresh
```

### 有効期限管理

```kotlin
private suspend fun ensureValidToken() {
    val expiresAt = prefs.getLong("gdrive_token_expires_at", 0L)
    if (System.currentTimeMillis() + 60_000L >= expiresAt) {
        refreshAccessToken()
    }
}
```

### ファイル名パースの限界

`"Artist - Title.mp3"` 形式以外はすべて "Unknown Artist"。ID3 タグ読込は未実装。

### Ktor 接続プール

`OkHttpClient` を 1 つ使い回すため upstream fetch はコネクションプール効率が良い。Ktor CIO は軽量 (coroutine ベース)。

---

## 6. 関連ファイル

- 認証 UI: `presentation/gdrive/auth/GDriveLoginViewModel.kt`
- ダッシュボード UI: `presentation/gdrive/dashboard/GDriveDashboardViewModel.kt`
- DB: [`../01-data-foundation/database.md`](../01-data-foundation/database.md)
- セキュリティ: [`streaming-cloud.md`](./streaming-cloud.md) §2
- Cloud Stream 抽象 (基底未継承): [`streaming-cloud.md`](./streaming-cloud.md) §1
