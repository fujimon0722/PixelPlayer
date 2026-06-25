# streaming-qqmusic.md

> QQ Music の楽曲同期 (Repository) とストリーミングプロキシ (StreamProxy) 仕様。
> 暗号化・署名レイヤー (WebView JS 経由) を含む。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/qqmusic/
├─ QqMusicRepository.kt                 // 806 lines
└─ QqMusicStreamProxy.kt                // 62 lines

app/src/main/java/com/theveloper/pixelplay/data/remote/qqmusic/
├─ QQMusicEncryptInterceptor.kt         // 91 lines  [OkHttp Interceptor]
├─ QQMusicSecurity.kt                   // 61 lines  [AES-GCM + XOR utility]
└─ QQSignGenerator.kt                   // 275 lines [WebView-based sign]
```

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `presentation/qqmusic/*`, `data/repository/MusicRepositoryImpl.kt` |
| 下流 (依存先) | `data/network/qqmusic/QqMusicApiService`, `data/remote/qqmusic/{QQSignGenerator, QQMusicSecurity, QQMusicEncryptInterceptor}`, `data/database/{QqMusicDao, MusicDao}`, `data/preferences/PlaylistPreferencesRepository`, `data/stream/CloudMusicUtils`, `androidx.security.crypto.EncryptedSharedPreferences`, `android.webkit.WebView`, `android.util.Base64` |

---

## 1. `QqMusicRepository` (`@Singleton`)

### 依存 (`@Inject constructor`)

| 依存 | 用途 |
| --- | --- |
| `QqMusicApiService` | API 呼び出し |
| `QqMusicDao` (Room) | QQ Music 専用 DAO |
| `MusicDao` | 統合 DAO |
| `PlaylistPreferencesRepository` | アプリ内プレイリスト |
| `@ApplicationContext Context` | EncryptedSharedPreferences |

### 内部データクラス

```kotlin
data class BulkSyncResult(
    val playlistCount: Int,
    val syncedSongCount: Int,
    val failedPlaylistCount: Int
)

enum class PlaylistSyncType {
    CREATED,  // 用户创建
    COLLECTED, // 用户收藏 (默认)
    ALL
}
```

### 定数

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `QQ_MUSIC_SONG_ID_OFFSET` | `6_000_000_000_000L` | 統合 Song ID |
| `QQ_MUSIC_ALBUM_ID_OFFSET` | `7_000_000_000_000L` | 統合 Album ID |
| `QQ_MUSIC_ARTIST_ID_OFFSET` | `8_000_000_000_000L` | 統合 Artist ID |
| `QQ_MUSIC_PARENT_DIRECTORY` | `"/Cloud/QQMusic"` | |
| `QQ_MUSIC_GENRE` | `"QQ Music"` | |
| `QQ_MUSIC_PLAYLIST_PREFIX` | `"qqmusic_playlist:"` | |
| `QQ_USER_PLAYLIST_PAGE_SIZE` | `100` | |
| `QQ_PLAYLIST_SONG_PAGE_SIZE` | `1000` | |
| `QQ_MAX_PLAYLIST_PAGES` | `200` | |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `prefs` | `SharedPreferences` | EncryptedSharedPreferences (qqmusic_cookies JSON) |
| `_isLoggedInFlow` | `MutableStateFlow<Boolean>` | |
| `isLoggedInFlow` | `StateFlow<Boolean>` | |
| `inFlightSongUrlRequests` | `ConcurrentHashMap<String, CompletableDeferred<Result<String>>>` | 同一 songMid の多重リクエスト集約 |
| `lastSongUrlAttemptAtMs` | `ConcurrentHashMap<String, Long>` | songMid 単位の最終試行時刻 |
| `songUrlRequestCooldownMs` | `1500L` | |
| `qqSongUrlRequestMutex` | `Mutex` | |
| `lastGlobalSongUrlRequestAtMs` | `var Long` | |
| `globalSongUrlRequestIntervalMs` | `1100L` | |

### 派生プロパティ

| プロパティ | 型 | 説明 |
| --- | --- | --- |
| `isLoggedIn` | `Boolean` | `api.hasLogin()` |
| `userNickname` | `String?` | ログイン後にメモ |
| `userAvatarUrl` | `String?` | ログイン後にメモ |

### 認証情報管理

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getPlaylists` | `fun getPlaylists(): Flow<List<QqMusicPlaylistEntity>>` | DAO 監視 |
| `initFromSavedCookies` | `fun initFromSavedCookies()` | prefs から復元し `api.setPersistedCookies` |
| `loginWithCookies` | `suspend fun loginWithCookies(cookieJson): Result<String>` | Cookie JSON → `api.setPersistedCookies` → `api.getUserCreatedPlaylists(size=1)` でユーザ検証 → nickname/avatar 取得 → DB 初期化 → flow true |
| `logout` | `suspend fun logout()` | DB + アプリプレイリスト + prefs + `api.logout` |
| `deletePlaylistById` | `suspend fun deletePlaylistById(playlistId: Long)` | DB + アプリプレイリスト削除 |

### 同期

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `syncUserPlaylists` | `suspend fun syncUserPlaylists(syncType: PlaylistSyncType = ALL): Result<List<QqMusicPlaylistEntity>>` | 作成 (`disslist`) と 收藏 (`cdlist`) を並行取得し、和集合 + 重複排除。 |
| `syncPlaylistSongs` | `suspend fun syncPlaylistSongs(playlistId): Result<Int>` | `api.getPlaylistDetail` を 1000 件ずつページング。 |
| `syncAllPlaylistsAndSongs` | `suspend fun syncAllPlaylistsAndSongs(syncType): Result<BulkSyncResult>` | 全プレイリスト + 各楽曲順次。 |

### `syncUserPlaylists` のロジック

```
1. createdStart=0 から QQ_USER_PLAYLIST_PAGE_SIZE ずつ:
   val raw = api.getUserCreatedPlaylists(start = createdStart, size = ...)
   val disslist = data.disslist (id = tid, name = diss_name)
   entitiesById に upsert
2. CD リストも同様のページング
3. 重複は entitiesById (LinkedHashMap) で吸収
4. 最終的に entitiesById.values を返す
5. ローカルにあってリモートに無い playlist を stale として削除
```

### `syncPlaylistSongs` のページング

```
1. songBegin=0 から QQ_PLAYLIST_SONG_PAGE_SIZE ずつ:
   val raw = api.getPlaylistDetail(dissid, songBegin, songNum, ownerUin, uin)
   val songlist = cdlist[0].songlist
   parseTrackToEntity で順次パース
2. 期待曲数 (`song_cnt`) と `songBegin` 比較で打ち切り判定
```

### クエリ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getSongUrl` | `suspend fun getSongUrl(songMid: String): Result<String>` | 多重防止 + 全体スロットル + デフォルト→ MP3 フォールバック。 |
| `requestPurl` | `private suspend fun requestPurl(songMid, filename?): String` | `api.getSongDownloadUrl` を 1.1 秒間隔スロットルで実行。 |

### `getSongUrl` の特殊ロジック

```
1. songMid 単位で 1.5 秒クールダウン + 多重集約
2. api.getSongDownloadUrl(songMid) → defaultPurl (M800 or M500)
3. mediaMid = defaultPurl.substringBefore("?").drop(4).substringBefore(".")
4. mp3Filename = "M500${mediaMid}.mp3"   // MP3 fallback
5. mp3Purl = api.getSongDownloadUrl(songMid, filename = mp3Filename)
6. 戻り値: defaultUrl が http 始まりなら defaultUrl、なければ mp3Purl
```

### 内部ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `parseTrackToEntity` | `private fun parseTrackToEntity(track: JSONObject, playlistId: Long): QqMusicSongEntity` | songmid / songname (Base64 decode) / singer / albumname / albummid / interval。 |
| `decodeBase64IfNeeded` | `private fun decodeBase64IfNeeded(input: String): String` | `^[A-Za-z0-9+/=]+$` で Base64 判定しデコード、失敗時は原文。 |
| `jsonToMap` | `private fun jsonToMap(json: String): Map<String, String>` | JSON 文字列 → Map |

### 統合レイヤー

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `syncUnifiedLibrarySongsFromQqMusic` | `private suspend fun syncUnifiedLibrarySongsFromQqMusic()` | `QqMusicSongEntity` → 統合 Song/Album/Artist upsert |
| `parseArtistNames` | `private fun parseArtistNames(rawArtist): List<String>` | 区切り文字分割 |
| `toUnifiedSongId` | `private fun toUnifiedSongId(mid: String): Long` | `QQ_MUSIC_SONG_ID_OFFSET + mid.hashCode().absoluteValue` |
| `toUnifiedAlbumId` | `private fun toUnifiedAlbumId(albumMid: String, albumName): Long` | albumMid が非空なら hashCode、なければ albumName.lowercase().hashCode() |
| `toUnifiedArtistId` | `private fun toUnifiedArtistId(artistName): Long` | |
| `getAppPlaylistIdForQqMusic` | `private suspend fun getAppPlaylistIdForQqMusic(qqPlaylistId): String` | `"qqmusic_playlist:$id"` |
| `updateAppPlaylistForQqMusicPlaylist` | `private suspend fun updateAppPlaylistForQqMusicPlaylist(...)` | アプリ内プレイリスト upsert |
| `deleteAppPlaylistForQqMusicPlaylist` | `private suspend fun deleteAppPlaylistForQqMusicPlaylist(...)` | 削除 |

---

## 2. `QqMusicStreamProxy` (`@Singleton`)

`CloudStreamProxy<String>` の実装。

### 抽象メンバー実装

| メンバー | 値 | 説明 |
| --- | --- | --- |
| `allowedHostSuffixes` | `setOf("qq.com", "gtimg.com", "music.qq.com", "y.qq.com", "dl.stream.qqmusic.qq.com", "stream.qqmusic.qq.com")` | QQ Music 配信ホスト群 |
| `cacheExpirationMs` | `10L * 60 * 1000` (10 分) | URL キャッシュ TTL |
| `proxyTag` | `"QqMusicStreamProxy"` | |
| `routePath` | `"/qqmusic/{songMid}"` | |
| `routeParamName` | `"songMid"` | |
| `uriScheme` | `"qqmusic"` | |
| `routePrefix` | `"/qqmusic"` | |

### 抽象メソッド実装

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `parseRouteParam` | `override fun parseRouteParam(value: String): String?` | songMid をそのまま。 |
| `validateId` | `override fun validateId(id: String): Boolean` | `CloudStreamSecurity.validateQqMusicSongMid` (regex)。 |
| `formatIdForUrl` | `override fun formatIdForUrl(id: String): String` | |
| `resolveStreamUrl` | `override suspend fun resolveStreamUrl(id: String): String?` | `repository.getSongUrl(id)` |

### 独自メソッド

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `extractIdFromUri` | `override fun extractIdFromUri(uri: Uri): String?` | `uri.host` を songMid として抽出。 |
| `resolveQqMusicUri` | `fun resolveQqMusicUri(uriString: String): String?` | `resolveUri(uriString)` ラッパ。 |
| `warmUpStreamUrl` | `suspend fun warmUpStreamUrl(uriString: String)` | URL 解決 + キャッシュ温め。 |

---

## 3. `QQMusicEncryptInterceptor` (OkHttp `Interceptor`)

### 役割

`https://...musics.fcg` へのリクエストを自動的に:
1. `sign` クエリ付与
2. リクエストボディを AES-128-GCM で暗号化 + Base64
3. レスポンスを VM (vm_new.js) → XOR fallback で復号

### コンストラクタ

```kotlin
class QQMusicEncryptInterceptor(
    private val signGenerator: QQSignGenerator
) : Interceptor
```

### `intercept(chain: Interceptor.Chain): Response` のフロー

```
1. url に "musics.fcg" が無ければスキップ
2. request body を Buffer に展開 → plaintextJson
3. sign = signGenerator.generateSign(plaintextJson)
4. encryptedBase64Body = signGenerator.encryptRequestWithVm(plaintextJson)
                       ?: QQMusicSecurity.encryptRequest(plaintextJson)
5. newUrl = url.newBuilder().addQueryParameter("encoding", "ag-1").addQueryParameter("sign", sign)
6. newRequest = request.newBuilder()
                     .url(newUrl)
                     .post(encryptedBase64Body.toRequestBody("text/plain"))
                     .header("accept", "application/octet-stream")
                     .header("content-type", "text/plain")
7. response = chain.proceed(newRequest)
8. response.body.bytes()
9. vmDecrypted = signGenerator.decryptResponseWithVm(responseBytes)
10. decryptedJson = vmDecrypted ?: QQMusicSecurity.decryptResponse(responseBytes)
11. return response.newBuilder().body(decryptedJson.toResponseBody("application/json")).build()
```

---

## 4. `QQMusicSecurity` (`object`)

### 役割

`musics.fcg` のフォールバック暗号化 (WebView JS が利用できない場合の pure Kotlin 実装)。

### 定数

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `AES_KEY_HEX` | `"bd035870d5afa4133454af0836f5e1cf"` | AES-128 鍵 |
| `AES_KEY_BYTES` | 16 bytes | 上記 hex を展開 |
| `XOR_KEY` | 21 bytes (固定) | レスポンス復号用 |

### `encryptRequest`

```
1. SecureRandom で 12 byte nonce
2. Cipher.getInstance("AES/GCM/NoPadding") + GCMParameterSpec(128, nonce)
3. plaintext UTF-8 bytes を暗号化 (末尾に GCM tag 16 bytes 付与)
4. ByteBuffer に nonce + ciphertext を連結
5. Base64.encodeToString(..., NO_WRAP)
```

出力フォーマット: `Base64(Nonce[12] + Ciphertext + Tag[16])`

### `decryptResponse`

```
1. for i in 0 until size:
     decrypted[i] = encryptedData[i] xor XOR_KEY[i % 21]
2. String(decrypted, UTF-8)
```

---

## 5. `QQSignGenerator`

### 役割

QQ Music の **JavaScript で書かれた署名アルゴリズム** を Android WebView 上で実行し `zzb...` 形式の署名を取得。

### 依存

- `Context` (applicationContext 使用)
- `assets/qq_sign.js` (sign ロジック本体)
- `assets/vm_new.js` (暗号化/復号 VM)

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `appContext` | `Context` | |
| `mainHandler` | `Handler` | UI thread |
| `signLock` | `Any` | `synchronized` 用 |
| `webView` | `@Volatile WebView?` | 遅延初期化 |
| `webViewReady` | `@Volatile Boolean` | |
| `encryptLatch` | `@Volatile CountDownLatch?` | 暗号化 Promise 結果待ち |
| `encryptResultRef` | `@Volatile AtomicReference<String?>?` | |

### 内部クラス `JsBridge`

`@JavascriptInterface` で `onEncryptResult(value: String?)` を JS から呼び出すと `encryptLatch.countDown()` で通知。

### 遅延ロード

```kotlin
private val jsContent: String by lazy {
    appContext.assets.open("qq_sign.js").use { InputStreamReader(it).readText() }
}

private val vmDecryptContent: String? by lazy {
    runCatching {
        appContext.assets.open("vm_new.js").use { InputStreamReader(it).readText() }
    }.getOrNull()
}
```

### `ensureWebView(): WebView`

- Main thread 上で `WebView(appContext)` を作成。
- `WebSettings.javaScriptEnabled = true`, `domStorageEnabled = true`, `allowFileAccess = false`, `mixedContentMode = NEVER_ALLOW`, `safeBrowsingEnabled = true`。
- `webViewClient.onPageFinished` で `webViewReady = true`。
- `addJavascriptInterface(JsBridge(), "AndroidBridge")`。
- `loadUrl("https://y.qq.com/")` で HTTPS 起点 (Web Crypto API 利用前提)。

### 公開 API

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `generateSign` | `fun generateSign(jsonData: String): String?` | `(function(){<jsContent>; return getSign(<quotedJson>);})()` を `evaluateJavascript` で実行。3 秒タイムアウト。 |
| `encryptRequestWithVm` | `fun encryptRequestWithVm(plaintext: String): String?` | `oe.__cgiEncrypt(payload)` を呼び Promise 結果を `AndroidBridge.onEncryptResult` で受け取る。 |
| `decryptResponseWithVm` | `fun decryptResponseWithVm(encryptedData: ByteArray): String?` | Base64 → Uint8Array → `oe.__cgiDecrypt(bytes.buffer)` を呼び結果をデコード。 |

### スレッドセーフ

```kotlin
synchronized(signLock) {
    if (Looper.myLooper() == Looper.getMainLooper()) {
        Timber.e("generateSign should not run on main thread")
        return null
    }
    ...
}
```

`signLock` で連続呼び出しを直列化、WebView 状態破壊を防止。

---

## 6. 内部実装メモ

### 全体スロットル 1.1 秒 + 曲単位 1.5 秒

Netease と同じパターン。`requestPurl` 内で `globalSongUrlRequestIntervalMs` 待機。

### ファイル名による品質切替

`getSongDownloadUrl(songMid, filename = "M500<mediaMid>.mp3")` で MP3 フォールバック。M800 は FLAC 系 (高品质)。

### Cookie の診断

`logCookieKeyDiagnostics(stage)` で必須キー (`uin`, `qm_keyst`, `euin`, `psrf_qqaccess_token`) をログにダンプ (欠落時に警告)。

### WebView コスト

`signLock` による直列化で WebView 競合状態を回避しているが、初回ロード時に `https://y.qq.com/` のロードが必要 (約 1〜2 秒)。`generateSign` の初回呼び出しが UI に出ないよう Repository 側で warm-up。

### Base64 タイトル

QQ Music は曲名・アルバム名を Base64 エンコードして返すことがある (`decodeBase64IfNeeded`)。

---

## 7. 関連ファイル

- 低レベル API: [`network-qqmusic.md`](./network-qqmusic.md)
- Stream Proxy 抽象基底: [`streaming-cloud.md`](./streaming-cloud.md) §1
- セキュリティ: [`streaming-cloud.md`](./streaming-cloud.md) §2
- DB: [`../01-data-foundation/database.md`](../01-data-foundation/database.md)
