# streaming-jellyfin.md

> Jellyfin の **楽曲同期 (Repository)** と **ストリーミングプロキシ (StreamProxy)** 仕様。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/jellyfin/
├─ JellyfinRepository.kt          // 662 lines
├─ JellyfinStreamProxy.kt         // 62 lines
└─ model/
    ├─ JellyfinAlbum.kt           // 31 lines
    ├─ JellyfinArtist.kt          // 21 lines
    ├─ JellyfinCredentials.kt     // 63 lines
    ├─ JellyfinPlaylist.kt        // 27 lines
    └─ JellyfinSong.kt            // 53 lines
```

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `presentation/jellyfin/*`, `data/repository/MusicRepositoryImpl.kt`, `data/image/JellyfinCoilFetcher.kt`, `data/worker/SyncWorker.kt` |
| 下流 (依存先) | `data/network/jellyfin/{JellyfinApiService, JellyfinResponseParser}`, `data/database/{JellyfinDao, MusicDao}`, `data/preferences/PlaylistPreferencesRepository`, `data/stream/CloudMusicUtils`, `androidx.security.crypto.EncryptedSharedPreferences` |

---

## 1. `JellyfinRepository` (`@Singleton`)

### 依存 (`@Inject constructor`)

| 依存 | 用途 |
| --- | --- |
| `JellyfinApiService` | Jellyfin REST 呼び出し |
| `JellyfinDao` (Room) | Jellyfin 専用エンティティ DAO |
| `MusicDao` | 統合 Song/Album/Artist DAO |
| `PlaylistPreferencesRepository` | アプリ内プレイリスト統合 |
| `@ApplicationContext Context` | `EncryptedSharedPreferences` のキー保存 |

### 定数

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `TAG` | `"JellyfinRepo"` | ログ |
| `PREFS_NAME` | `"jellyfin_prefs"` | EncryptedSharedPreferences 名 |
| `KEY_SERVER_URL` | `"server_url"` | URL 永続化 |
| `KEY_USERNAME` | `"username"` | ユーザ名 |
| `KEY_ACCESS_TOKEN` | `"access_token"` | アクセストークン |
| `KEY_USER_ID` | `"user_id"` | ユーザ ID |
| `JELLYFIN_SONG_ID_OFFSET` | `12_000_000_000_000L` | 統合 Song ID オフセット |
| `JELLYFIN_ALBUM_ID_OFFSET` | `13_000_000_000_000L` | 統合 Album ID |
| `JELLYFIN_ARTIST_ID_OFFSET` | `14_000_000_000_000L` | 統合 Artist ID |
| `JELLYFIN_PARENT_DIRECTORY` | `"/Cloud/Jellyfin"` | 統合 Song.parentDirectory |
| `JELLYFIN_GENRE` | `"Jellyfin"` | 統合 Song.genre |
| `JELLYFIN_PLAYLIST_PREFIX` | `"jellyfin_playlist:"` | アプリ内プレイリスト ID プレフィクス |
| `LIBRARY_PLAYLIST_ID` | `"__library__"` | 全楽曲仮想プレイリスト ID |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `prefs` | `SharedPreferences` | EncryptedSharedPreferences (取得失敗時は通常 prefs) |
| `_isLoggedInFlow` | `MutableStateFlow<Boolean>` | ログイン状態 |
| `isLoggedInFlow` | `StateFlow<Boolean>` | 公開 |

### 認証情報 / ライフサイクル

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `initFromSavedCredentials` | `private fun initFromSavedCredentials()` | 起動時に prefs から復元し `api.setCredentials` を呼ぶ。 |
| `isLoggedIn` | `val isLoggedIn: Boolean` | 派生プロパティ。 |
| `serverUrl` | `val serverUrl: String?` | `api.getServerUrl()` のエイリアス。 |
| `username` | `val username: String?` | 永続ユーザ名。 |
| `getAuthorizationHeader` | `fun getAuthorizationHeader(): String? = api.getAuthorizationHeader()` | StreamProxy / Coil 用。 |
| `login` | `suspend fun login(serverUrl, username, password): Result<String>` | `api.authenticateByName` → `api.setCredentials` → prefs 永続化 → `dao.clearAll` → `_isLoggedInFlow.value = true`。 |
| `logout` | `suspend fun logout()` | `dao.clearAll` で全 Jellyfin データ削除 + アプリ内プレイリスト削除 + prefs 消去 + `_isLoggedInFlow.value = false`。 |

### 同期

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `syncPlaylists` | `suspend fun syncPlaylists(): Result<List<JellyfinPlaylistEntity>>` | `/Playlists` を取得 → パース → DB upsert → ローカルから消えたリモートは削除。 |
| `syncPlaylistSongs` | `suspend fun syncPlaylistSongs(playlistId): Result<Int>` | `/Playlists/{id}/Items` → 楽曲エンティティ upsert → アプリ内プレイリスト作成/更新。 |
| `syncLibrarySongs` | `suspend fun syncLibrarySongs(): Result<Int>` | `/Items?IncludeItemTypes=Audio` を 500 件ページング → 楽曲 upsert。 |
| `syncAllPlaylistsAndSongs` | `suspend fun syncAllPlaylistsAndSongs(): Result<BulkSyncResult>` | ライブラリ → プレイリスト → 各プレイリスト楽曲の順で実行。 |

### クエリ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getPlaylists` | `fun getPlaylists(): Flow<List<JellyfinPlaylistEntity>>` | DAO 監視。 |
| `getPlaylistSongs` | `fun getPlaylistSongs(playlistId): Flow<List<Song>>` | `JellyfinSongEntity.toSong()` 変換。 |
| `getAllSongs` | `fun getAllSongs(): Flow<List<Song>>` | 全曲。 |
| `searchSongs` | `suspend fun searchSongs(query, limit = 30): Result<List<Song>>` | `api.searchSongs` → パース → Song 変換。 |
| `searchLocalSongs` | `fun searchLocalSongs(query): Flow<List<Song>>` | DB ローカル LIKE 検索。 |
| `deletePlaylist` | `suspend fun deletePlaylist(playlistId)` | DB とアプリ内プレイリスト両方削除。 |
| `getStreamUrl` | `fun getStreamUrl(songId, maxBitRate = 0): String` | `api.getStreamUrl` (StreamProxy から呼ばれる)。 |
| `getImageUrl` | `fun getImageUrl(itemId?, size = 500): String?` | `api.getImageUrl`。null 時は Coil がフォールバック。 |
| `getLyrics` | `suspend fun getLyrics(songId): Result<String>` | `api.getLyrics` → `\n` 連結。 |

### 統合レイヤー

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `syncUnifiedLibrarySongsFromJellyfin` | `suspend fun syncUnifiedLibrarySongsFromJellyfin()` | `JellyfinSongEntity` を `SongEntity/AlbumEntity/ArtistEntity` に変換して `musicDao` に upsert。 |
| `parseArtistNames` | `private fun parseArtistNames(rawArtist): List<String>` | 区切り文字 (`&`, `,`, `;`, `feat.`, `ft.`) で分割。 |
| `toUnifiedSongId` | `private fun toUnifiedSongId(jellyfinId): Long` | `JELLYFIN_SONG_ID_OFFSET + jellyfinId.hashCode().absoluteValue`。 |
| `toUnifiedAlbumId` | `private fun toUnifiedAlbumId(albumId?, albumName): Long` | albumId があれば hash、なければ albumName.lowercase().hashCode() の絶対値 + offset。 |
| `toUnifiedArtistId` | `private fun toUnifiedArtistId(artistName): Long` | `JELLYFIN_ARTIST_ID_OFFSET + artistName.lowercase().hashCode().absoluteValue`。 |
| `updateAppPlaylistForJellyfinPlaylist` | `private suspend fun updateAppPlaylistForJellyfinPlaylist(jellyfinPlaylistId, songs)` | アプリ内プレイリスト `jellyfin_playlist:<id>` を作成/更新。 |
| `deleteAppPlaylistForJellyfinPlaylist` | `private suspend fun deleteAppPlaylistForJellyfinPlaylist(jellyfinPlaylistId)` | アプリ内プレイリスト削除。 |
| `JellyfinSong.toDisplaySong()` | `private fun JellyfinSong.toDisplaySong(): Song` | Song ドメイン変換。 |

---

## 2. `JellyfinStreamProxy` (`@Singleton`)

`CloudStreamProxy<String>` の実装。Ktor CIO で `127.0.0.1:<port>/jellyfin/{itemId}` を待ち受ける。

### 抽象メンバー実装

| メンバー | 値 | 説明 |
| --- | --- | --- |
| `allowedHostSuffixes` | (実装) | Jellyfin サーバホスト (`serverUrl.host`) の派生 suffix 群。 |
| `cacheExpirationMs` | `30L * 60 * 1000` (30 分) | 内部 URL キャッシュ TTL。 |
| `proxyTag` | `"JellyfinStreamProxy"` | ログタグ |
| `routePath` | `"/jellyfin/{itemId}"` | Ktor ルート |
| `routeParamName` | `"itemId"` | ルートパラメータ名 |
| `uriScheme` | `"jellyfin"` | URI scheme |
| `routePrefix` | `"/jellyfin"` | プロキシの URL プレフィクス |

### 抽象メソッド実装

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `parseRouteParam` | `override fun parseRouteParam(value: String): String?` | itemId をそのまま返す (Jellyfin の Item ID は GUID)。 |
| `validateId` | `override fun validateId(id: String): Boolean` | `CloudStreamSecurity.validateJellyfinItemId` (regex)。 |
| `formatIdForUrl` | `override fun formatIdForUrl(id: String): String` | `id` をそのまま。 |
| `resolveStreamUrl` | `override suspend fun resolveStreamUrl(id: String): String?` | `repository.getStreamUrl(id)` の戻り値。 |

### 独自メソッド

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `extractIdFromUri` | `override fun extractIdFromUri(uri: Uri): String?` | `uri.host` を itemId として抽出。 |
| `resolveJellyfinUri` | `fun resolveJellyfinUri(uriString: String): String?` | `resolveUri(uriString)` の薄いラッパ。 |
| `warmUpStreamUrl` | `suspend fun warmUpStreamUrl(uriString: String)` | `resolveJellyfinUri` を呼んでキャッシュを温める。 |

### プロキシ URL 例

```
jellyfin://<itemId>
↓ resolveJellyfinUri
http://127.0.0.1:<actualPort>/jellyfin/<itemId>
```

---

## 3. `data/jellyfin/model/*`

### `JellyfinCredentials`

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `serverUrl` | `String` | 入力された URL |
| `username` | `String` | |
| `password` | `String` | ログイン後メモリから破棄想定 |
| `accessToken` | `String?` | ログイン後取得 |
| `userId` | `String?` | ログイン後取得 |

派生:
- `isValid: Boolean` — `serverUrl.isNotBlank() && username.isNotBlank()`
- `hasToken: Boolean` — `accessToken.isNullOrBlank() == false`
- `normalizedHttpUrlOrNull: HttpUrl?` — `okhttp3.HttpUrl` パース (失敗時 null)
- `normalizedServerUrl: String` — `trim().trimEnd('/')` + `https?://` を自動補完
- `connectionValidationError(): String?` — `HttpUrl` パース + 必須スキーム検証

### `JellyfinSong` / `JellyfinAlbum` / `JellyfinArtist` / `JellyfinPlaylist`

`@Parcelize @Immutable data class`。詳細は [`network-jellyfin.md`](./network-jellyfin.md) §3 参照。

`JellyfinSong.resolvedMimeType` は `containerToMimeType` を呼んだ結果 (Media3 の ContentType 判定用)。

---

## 4. 内部実装メモ

### 暗号化 (EncryptedSharedPreferences)

```
MasterKey.Builder(context).setKeyScheme(AES256_GCM)
  → EncryptedSharedPreferences.create(
      context, "jellyfin_prefs", masterKey,
      AES256_SIV, AES256_GCM
    )
```
失敗時は通常 `SharedPreferences` にフォールバック (try/catch)。

### 統合 Song ID 計算

```kotlin
private fun toUnifiedSongId(jellyfinId: String): Long =
    JELLYFIN_SONG_ID_OFFSET + jellyfinId.hashCode().absoluteValue.toLong()
```

`hashCode()` は String 内容で決定的だが負値になり得るため `.absoluteValue` で正規化。

### 統合アルバム/アーティスト ID

```kotlin
// albumId が GUID の場合
val normalized = if (!albumId.isNullOrBlank()) {
    albumId.hashCode().absoluteValue.toLong()
} else {
    albumName.lowercase().hashCode().absoluteValue.toLong()
}
JELLYFIN_ALBUM_ID_OFFSET + normalized
```

### アプリ内プレイリスト同期

Jellyfin のプレイリスト ID を `jellyfin_playlist:<jellyfinId>` として `PlaylistPreferencesRepository` に登録 (楽曲 ID は統合 ID を使用)。

---

## 5. 関連ファイル

- 低レベル API: [`network-jellyfin.md`](./network-jellyfin.md)
- Stream Proxy 抽象基底: [`streaming-cloud.md`](./streaming-cloud.md) §1
- セキュリティ: [`streaming-cloud.md`](./streaming-cloud.md) §2
- 認証情報モデル: `data/jellyfin/model/JellyfinCredentials.kt`
- 画像取得: [`image-fetchers.md`](./image-fetchers.md) §2
- Coil: [`image-fetchers.md`](./image-fetchers.md)
- DB: [`../01-data-foundation/database.md`](../01-data-foundation/database.md)
