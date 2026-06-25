# streaming-cloud.md

> クラウド音源再生の **横断基盤** 仕様。
> 抽象基底クラス `CloudStreamProxy<K>`、URL キャッシュ、`Range` / SSRF 検証 `CloudStreamSecurity`、共通 JSON ユーティリティ `CloudMusicUtils`。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/stream/
├─ CloudMusicUtils.kt       // 40 lines
├─ CloudStreamProxy.kt      // 296 lines (abstract class)
└─ CloudStreamSecurity.kt   // 241 lines
```

## 役割分担

| クラス | 役割 |
| --- | --- |
| `CloudMusicUtils` | クロスプロバイダ共通ヘルパ (Cookie JSON 復元、アーティスト名分割)。 |
| `CloudStreamProxy<K>` | **抽象基底**。Ktor CIO で localhost HTTP サーバを立ち上げ、URL キャッシュ + Range / Host 検証 + 認証ヘッダ付与 + chunked 転送の標準実装。 |
| `CloudStreamSecurity` | **横断セキュリティ**。Range ヘッダ検証、SSRF 対策 (private/CGNAT/IPv6 ローカル)、MIME / Content-Length 検証、ホスト suffix マッチ。 |

---

## 1. `CloudStreamUtils` (`object`)

### `data class BulkSyncResult`

```kotlin
data class BulkSyncResult(
    val playlistCount: Int,
    val syncedSongCount: Int,
    val failedPlaylistCount: Int
)
```

> 各 Repository (`JellyfinRepository`, `NavidromeRepository`, `NeteaseRepository`, `QqMusicRepository`) はローカルに同名の `BulkSyncResult` を再宣言する場合あり (フィールド差異)。

### 公開 API

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `jsonToMap` | `fun jsonToMap(json: String): Map<String, String>` | `JSONObject(json)` を keys() 走査して Map に展開。QQ Music / Netease のクッキー永続化で使用。 |
| `parseArtistNames` | `fun parseArtistNames(rawArtist: String): List<String>` | `utils.splitArtistsByDelimiters(rawArtist)` で分割 (区切り: `&`, `,`, `;`, `feat.`, `ft.`)。 |

### 利用例

```kotlin
// Netease / QQ の cookie 復元
val map = CloudMusicUtils.jsonToMap(prefs.getString("netease_cookies", "{}"))

// 統合 Song/Artist 変換
val names = CloudMusicUtils.parseArtistNames(song.artist)
```

---

## 2. `CloudStreamProxy<K>` (`abstract class`)

### ジェネリクス

`K : Any` — 楽曲 ID の型。プロバイダ毎に:
- `String` (Jellyfin, Navidrome, QQ Music)
- `Long` (Netease)

### 依存 (`constructor`)

| 依存 | 用途 |
| --- | --- |
| `OkHttpClient` | upstream fetch 用 (ベース DI) |

### 抽象メンバー (要サブクラス実装)

| メンバー | 型 | 目的 |
| --- | --- | --- |
| `allowedHostSuffixes` | `Set<String>` | ホワイトリスト suffix (例: `["y.qq.com", "qq.com"]`)。 |
| `cacheExpirationMs` | `Long` | URL キャッシュ TTL。 |
| `proxyTag` | `String` | ログタグ。 |
| `routePath` | `String` | Ktor ルート (例: `"/jellyfin/{itemId}"`)。 |
| `routeParamName` | `String` | Ktor パラメータ名。 |
| `uriScheme` | `String` | URI scheme (例: `"jellyfin"`)。 |
| `routePrefix` | `String` | URL プレフィクス (例: `"/jellyfin"`)。 |
| `parseRouteParam` | `(String) → K?` | Ktor パラメータ文字列 → ID。 |
| `validateId` | `(K) → Boolean` | ID のバリデーション。 |
| `formatIdForUrl` | `(K) → String` | URL 埋め込み用文字列。 |
| `resolveStreamUrl` | `suspend (K) → String?` | クラウド側 URL の解決。 |
| `extractIdFromUri` (open, default: `uri.host`) | `(Uri) → String?` | URI から ID 抽出。 |

### 内部状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `server` | `var EmbeddedServer<CIOApplicationEngine, CIOApplicationEngine.Configuration>?` | Ktor サーバ |
| `actualPort` | `var Int` | バインド済みポート |
| `proxyScope` | `CoroutineScope` | `SupervisorJob + Dispatchers.IO` |
| `startJob` | `var Job?` | 起動ジョブ |
| `urlCache` | `ConcurrentHashMap<K, CachedUrl>` | (id → url) + TTL |

### 内部データクラス

```kotlin
private data class CachedUrl(val url: String, val timestamp: Long, val expirationMs: Long) {
    fun isExpired(): Boolean = System.currentTimeMillis() - timestamp > expirationMs
}
```

### 公開 API

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `isReady` | `fun isReady(): Boolean` | `actualPort > 0` |
| `startIfNeeded` | `fun startIfNeeded()` | 既に起動中なら no-op |
| `awaitReady` | `suspend fun awaitReady(timeoutMs = 10_000L): Boolean` | 50ms ポーリング |
| `ensureReady` | `suspend fun ensureReady(timeoutMs = 10_000L): Boolean` | 未起動なら起動 + 待機 |
| `getProxyUrl` | `fun getProxyUrl(id: K): String` | `"http://127.0.0.1:<port><routePrefix>/<formatId>"` |
| `resolveUri` | `fun resolveUri(uriString: String): String?` | `<uriScheme>://<id>` → プロキシ URL |
| `start` | `fun start()` | `ServerSocket(0)` で空きポート取得 + Ktor 起動 |
| `stop` | `fun stop()` | サーバ停止 + スコープ cancel |

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getOrFetchStreamUrl` | `protected suspend fun getOrFetchStreamUrl(id: K): String?` | URL キャッシュ確認 → 必要なら `resolveStreamUrl` → キャッシュ保存。 |
| `createServer` | `private fun createServer(port): EmbeddedServer<...>` | Ktor ルート定義。 |

### Ktor ルーティング

```
GET <routePath>
例: GET /jellyfin/{itemId}
1. rawParam = call.parameters[routeParamName]
2. id = parseRouteParam(rawParam)
3. validateId(id) で 400 BadRequest
4. rangeValidation = CloudStreamSecurity.validateRangeHeader(call.request.headers["Range"])
5. streamUrl = getOrFetchStreamUrl(id)
6. streamUrl が null なら 404
7. streamUrl を toHttpUrlOrNull() パース
8. isSafeRemoteStreamUrl(streamUrl, allowedHostSuffixes) で 403 Forbidden
9. requestBuilder = Request.Builder().url(streamUrl)
   .header("Range", rangeValidation.normalizedHeader)  // 任意
10. upstream = okHttpClient.newCall(...).execute() (withContext IO)
11. upstream が失敗 (5xx, 401, etc.) なら対応コード返却
12. respondBytesWriter で:
    - contentTypeHeader, contentLengthHeader, contentRangeHeader, acceptRangesHeader を維持
    - 64 KB buffer で chunk 転送
    - 失敗時は接続を閉じてクライアントに通知
```

### サブクラス実装例 (Jellyfin)

```kotlin
@Singleton
class JellyfinStreamProxy @Inject constructor(
    private val repository: JellyfinRepository,
    okHttpClient: OkHttpClient
) : CloudStreamProxy<String>(okHttpClient) {

    override val allowedHostSuffixes = run {
        val host = repository.serverUrl?.toHttpUrlOrNull()?.host
        if (host != null) setOf(host) else emptySet()
    }
    override val cacheExpirationMs = 30L * 60 * 1000
    override val proxyTag = "JellyfinStreamProxy"
    override val routePath = "/jellyfin/{itemId}"
    override val routeParamName = "itemId"
    override val uriScheme = "jellyfin"
    override val routePrefix = "/jellyfin"

    override fun parseRouteParam(value: String): String? = value
    override fun validateId(id: String): Boolean = CloudStreamSecurity.validateJellyfinItemId(id)
    override fun formatIdForUrl(id: String): String = id
    override suspend fun resolveStreamUrl(id: String): String? = repository.getStreamUrl(id)
}
```

---

## 3. `CloudStreamSecurity` (`object`)

### 定数

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `MAX_STREAM_CONTENT_LENGTH_BYTES` | `2L * 1024 * 1024 * 1024` (2 GB) | Content-Length 上限 |
| `MAX_RANGE_HEADER_LENGTH` | `64` | Range 文字列長の上限 |
| `MAX_RANGE_VALUE_BYTES` | `8L * 1024 * 1024 * 1024` (8 GB) | Range 値上限 |
| `GDRIVE_FILE_ID_REGEX` | `^[A-Za-z0-9_-]{10,200}$` | GDrive ID 検証 |
| `QQMUSIC_SONG_MID_REGEX` | `^[A-Za-z0-9_-]{6,50}$` | QQ Music mid 検証 |
| `NAVIDROME_SONG_ID_REGEX` | `^[A-Za-z0-9_-]{1,100}$` | Navidrome ID 検証 |
| `JELLYFIN_ITEM_ID_REGEX` | `^[A-Za-z0-9]{1,100}$` | Jellyfin item ID 検証 |
| `FORBIDDEN_HOSTS` | `["localhost", "127.0.0.1", "0.0.0.0", "::1", "[::1]"]` | 明示禁止ホスト |
| `LOCAL_DNS_SUFFIXES` | `[".local", ".lan", ".home", ".internal", ".home.arpa"]` | ローカル DNS suffix |
| `EXTRA_ALLOWED_AUDIO_TYPES` | `setOf("application/octet-stream", "audio/", "video/", "image/")` | MIME 許可リスト |

### データクラス

```kotlin
data class RangeHeaderValidation(
    val isValid: Boolean,
    val normalizedHeader: String? = null,
    val startInclusive: Long? = null,
    val endInclusive: Long? = null,
    val isSuffixRange: Boolean = false
)
```

### 公開 API

#### ID 検証

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `validateTelegramFileId` | `fun validateTelegramFileId(fileId: Int): Boolean` | `fileId > 0` |
| `validateNeteaseSongId` | `fun validateNeteaseSongId(songId: Long): Boolean` | `songId > 0L` |
| `validateGDriveFileId` | `fun validateGDriveFileId(fileId: String): Boolean` | regex マッチ |
| `validateQqMusicSongMid` | `fun validateQqMusicSongMid(songMid: String): Boolean` | regex マッチ |
| `validateNavidromeSongId` | `fun validateNavidromeSongId(songId: String): Boolean` | regex マッチ |
| `validateJellyfinItemId` | `fun validateJellyfinItemId(itemId: String): Boolean` | regex マッチ |

#### Range ヘッダ検証

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `validateRangeHeader` | `fun validateRangeHeader(rawHeader: String?): RangeHeaderValidation` | `bytes=start-end` / `bytes=start-` / `bytes=-suffix` をパース。 |

#### コンテンツ検証

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `isSupportedAudioContentType` | `fun isSupportedAudioContentType(contentTypeHeader: String?): Boolean` | MIME プレフィクス一致 (`audio/`, `video/`, `image/`, `application/octet-stream`) |
| `isAcceptableContentLength` | `fun isAcceptableContentLength(contentLengthHeader: String?): Boolean` | `0 < length <= 2GB` |

#### URL / ホスト検証

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `isSafeRemoteStreamUrl` | `fun isSafeRemoteStreamUrl(url: String, allowedHostSuffixes: Set<String>): Boolean` | URL パース + ホスト検証 (後述)。 |
| `mapUpstreamStatusToProxyStatus` | `fun mapUpstreamStatusToProxyStatus(code: Int): HttpStatusCode` | upstream ステータスを Ktor の `HttpStatusCode` に変換。 |

#### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `hostMatchesAllowedSuffix` | `private fun hostMatchesAllowedSuffix(host, allowedHostSuffixes): Boolean` | 大文字小文字無視で suffix マッチ |
| `isLocalOrPrivateHost` | `internal fun isLocalOrPrivateHost(host): Boolean` | `localhost` 系 + IPv4 private + IPv6 private + CGNAT + ローカル DNS suffix |
| `isPrivateIpv4Literal` | `internal fun isPrivateIpv4Literal(host): Boolean` | `10/8`, `172.16/12`, `192.168/16`, `127/8`, `169.254/16`, `0/8` |
| `isCgnatIpv4Literal` | `private fun isCgnatIpv4Literal(host): Boolean` | `100.64/10` (RFC 6598) |
| `isPrivateIpv6Literal` | `private fun isPrivateIpv6Literal(host): Boolean` | `::/128`, `fc00::/7`, `fe80::/10`, `::ffff:` マップ |

### `isSafeRemoteStreamUrl` の判定ロジック

```
1. url を HttpUrl にパース。失敗 → false
2. host = httpUrl.host.lowercase()
3. FORBIDDEN_HOSTS にあれば false
4. isLocalOrPrivateHost(host) → false
5. Navidrome/Jellyfin 内部ストリームパス (`stream.view`, `Audio/.../universal`) は allowedHostSuffixes 必須
6. hostMatchesAllowedSuffix(host, allowedHostSuffixes) → true
7. 上記すべてパスなら true
```

---

## 4. 内部実装メモ

### ポート競合回避

```kotlin
val freePort = ServerSocket(0).use { it.localPort }
val createdServer = createServer(freePort)
```

`0` を指定すると OS が空きポートを割り当て。`actualPort` フィールドに確定値を保持。

### Range ヘッダのサフィックス表記

`bytes=-500` (末尾 500 バイト) を `RangeHeaderValidation(isSuffixRange=true, endInclusive=500)` に変換。

### URL キャッシュのヒット率改善

`ConcurrentHashMap<K, CachedUrl>` でスレッドセーフ。`isExpired()` はキャッシュ取得時に都度評価。

### 起動の冪等性

`startIfNeeded()` + `ensureReady()` の二段構え:
- 既に起動中 → 既存 `actualPort` を使用。
- 未起動 → `start()` で空きポート取得 → Ktor 起動 → `actualPort` 更新。
- レース対策: `startJob` を保持し多重起動を抑制。

### Media3 (ExoPlayer) 互換性

`Content-Type`, `Content-Length`, `Content-Range`, `Accept-Ranges` ヘッダをそのまま上流からコピーすることで、ExoPlayer のキャッシュ戦略を尊重。

---

## 5. 関連ファイル

- 各プロバイダのプロキシ実装:
  - [`streaming-jellyfin.md`](./streaming-jellyfin.md) §2
  - [`streaming-navidrome.md`](./streaming-navidrome.md) §2
  - [`streaming-netease.md`](./streaming-netease.md) §2
  - [`streaming-qqmusic.md`](./streaming-qqmusic.md) §2
- 独自実装 (基底を継承しない): [`streaming-gdrive.md`](./streaming-gdrive.md) §4, [`streaming-telegram.md`](./streaming-telegram.md) §4
- 統合 Song/Album/Artist ID: README.md §4
