# network-jellyfin.md

> Jellyfin REST API クライアント (`@Singleton`) と JSON→ドメインモデル変換器。
> 完全な API リファレンスは https://api.jellyfin.org/ を参照。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/network/jellyfin/
├─ JellyfinApiService.kt        // OkHttp ベースの低レベル HTTP クライアント (357 lines)
└─ JellyfinResponseParser.kt    // JSONObject -> JellyfinSong/Album/Artist/Playlist (164 lines)
```

## ファイル概要

| ファイル | 行数 | 概要 |
| --- | ---: | --- |
| `JellyfinApiService.kt` | 357 | OkHttp ベース。`MediaBrowser Client=...` 認証ヘッダ・`/Users/AuthenticateByName`・`/Items` 等を生 HTTP で叩く。 |
| `JellyfinResponseParser.kt` | 164 | 生 JSON から `JellyfinSong/Album/Artist/Playlist` を構築する pure 関数群。 |

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `data/jellyfin/JellyfinRepository.kt`, `data/jellyfin/JellyfinStreamProxy.kt`, `data/image/JellyfinCoilFetcher.kt` |
| 下流 (依存先) | `data/jellyfin/model/JellyfinCredentials.kt`, `org.json.JSONObject`, `okhttp3` |

## 1. `JellyfinApiService`

### 定数 (`companion object`)

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `TAG` | `"JellyfinApi"` | Timber ログタグ |
| `CLIENT_NAME` | `"PixelPlayer"` | Authorization ヘッダ |
| `CLIENT_VERSION` | `"1.0"` | Authorization ヘッダ |
| `DEVICE_NAME` | `"Android"` | Authorization ヘッダ |
| `DEVICE_ID` | `"PixelPlayer-Android"` | Authorization ヘッダ |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `credentials` | `@Volatile JellyfinCredentials?` | 現在の認証情報 |
| `okHttpClient` | `OkHttpClient` | `baseOkHttpClient.newBuilder()` で 15s/30s/15s タイムアウト |

### 公開 API メソッド

| メソッド | シグネチャ | 目的 / 戻り値 | 呼び出し元 (相対) |
| --- | --- | --- | --- |
| `setCredentials` | `fun setCredentials(credentials: JellyfinCredentials)` | 認証情報を差し替える。 | `JellyfinRepository.login()` |
| `clearCredentials` | `fun clearCredentials()` | 認証情報を破棄 (ログアウト時の状態初期化)。 | `JellyfinRepository.logout()` |
| `hasCredentials` | `fun hasCredentials(): Boolean` | `credentials?.hasToken == true` を返す。 | Repository 判定 |
| `getServerUrl` | `fun getServerUrl(): String?` | 正規化済み URL を返す。 | UI 表示 |
| `getAuthorizationHeader` | `fun getAuthorizationHeader(): String?` | `MediaBrowser Client=..., Token="..."` 文字列。 | Stream Proxy・Coil Fetcher |
| `authenticateByName` | `suspend fun authenticateByName(serverUrl, username, password): Result<Pair<String, String>>` | `/Users/AuthenticateByName` を叩いて `AccessToken` と `User.Id` を返す。 | `JellyfinRepository.login()` |
| `ping` | `suspend fun ping(): Result<Boolean>` | `/System/Ping` で接続性確認。 | Login VM |
| `getMusicItems` | `suspend fun getMusicItems(parentId, startIndex, limit, searchTerm, sortBy, sortOrder): Result<List<JSONObject>>` | `/Items` の汎用ラッパ。`IncludeItemTypes=Audio` で楽曲一覧取得。 | Repository ライブラリ同期 |
| `getAlbums` | `suspend fun getAlbums(parentId, startIndex, limit): Result<List<JSONObject>>` | `IncludeItemTypes=MusicAlbum`。 | Repository |
| `getAlbumItems` | `suspend fun getAlbumItems(albumId): Result<List<JSONObject>>` | `/Items?ParentId=albumId` でアルバム内楽曲。 | Repository |
| `getArtists` | `suspend fun getArtists(): Result<List<JSONObject>>` | `IncludeItemTypes=MusicArtist`。 | Repository |
| `getPlaylists` | `suspend fun getPlaylists(): Result<List<JSONObject>>` | `/Playlists`。 | Repository |
| `getPlaylistItems` | `suspend fun getPlaylistItems(playlistId): Result<List<JSONObject>>` | `/Playlists/{id}/Items`。 | Repository |
| `searchSongs` | `suspend fun searchSongs(query, limit = 30): Result<List<JSONObject>>` | `/Items?searchTerm=...&IncludeItemTypes=Audio`。 | Repository 検索 |
| `getStreamUrl` | `fun getStreamUrl(itemId, maxBitRate = 0): String` | `/Audio/{itemId}/universal?deviceId=...&api_key=...` を組み立てる (クエリ文字列 `api_key` を埋め込む旧式互換)。 | `JellyfinRepository.getStreamUrl()`, `JellyfinStreamProxy` |
| `getImageUrl` | `fun getImageUrl(itemId, imageType = "Primary", maxWidth = 500): String` | `/Items/{id}/Images/{type}?maxWidth=...`。 | `JellyfinCoilFetcher`, Repository |
| `getLyrics` | `suspend fun getLyrics(itemId): Result<String>` | `/Items/{id}/Lyrics` を取得し、`Lyrics[].Text` を `\n` 連結で返す (LRC 形式)。 | `JellyfinRepository.getLyrics()` |

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `buildAuthorizationHeader` | `private fun buildAuthorizationHeader(): String` | `accessToken` を `Token="..."` トークンとして連結。 |
| `request` | `private suspend fun request(path, params): Result<String>` | OkHttp 共通 GET。`credentials` 未設定は `IllegalStateException` を `Result.failure` 化。レスポンスコードごとに成否判定。 |
| `requestJson` | `private suspend fun requestJson(path, params): Result<JSONObject>` | `request` を JSONObject までパース。 |

### 設計上のポイント

- **DI**: `@Inject constructor(baseOkHttpClient: OkHttpClient)` — シングルトン OkHttp の接続プールを再利用し、メモリ削減 (コメント `P1-3` で言及)。
- **認証ヘッダ**: Jellyfin は Authorization ヘッダに `MediaBrowser Client=..., Device=..., DeviceId=..., Version=...` を要求し、`accessToken` があれば `Token="..."` を追加する独自形式。Basic でも Bearer でもない。
- **ストリーム URL の `api_key` パラメータ**: `getStreamUrl` は `api_key=<accessToken>` を URL クエリに埋め込む旧互換を維持 (Jellyfin サーバは token クエリでも受け付ける)。

## 2. `JellyfinResponseParser` (`object`)

### 定数

| 定数 | 値 |
| --- | --- |
| `TAG` | `"JellyfinParser"` |

### 公開関数

| 関数 | シグネチャ | 目的 |
| --- | --- | --- |
| `parseSong` | `fun parseSong(json: JSONObject): JellyfinSong` | `ArtistItems`/`AlbumArtist`/`Album` 等の任意ネストからタイトル/アーティスト/アルバム/ID/長さ/トラック/ジャンル/ビットレートを抽出。 |
| `parseSongs` | `fun parseSongs(jsonArray: List<JSONObject>): List<JellyfinSong>` | リスト変換。 |
| `parseAlbum` | `fun parseAlbum(json: JSONObject): JellyfinAlbum` | `Name`/`AlbumArtist`/`ProductionYear`/`Genres` から構築。 |
| `parseAlbums` | `fun parseAlbums(jsonArray: List<JSONObject>): List<JellyfinAlbum>` | リスト変換。 |
| `parseArtist` | `fun parseArtist(json: JSONObject): JellyfinArtist` | `Name` + `AlbumCount` 派生。 |
| `parseArtists` | `fun parseArtists(jsonArray: List<JSONObject>): List<JellyfinArtist>` | リスト変換。 |
| `parsePlaylist` | `fun parsePlaylist(json: JSONObject): JellyfinPlaylist` | `Name`/`ChildCount`/`CumulativeRunTimeTicks`/`DateCreated`/`DateModified`。 |
| `parsePlaylists` | `fun parsePlaylists(jsonArray: List<JSONObject>): List<JellyfinPlaylist>` | リスト変換。 |

### 非公開ヘルパ

| 関数 | シグネチャ | 目的 |
| --- | --- | --- |
| `containerToMimeType` | `private fun containerToMimeType(container: String?): String?` | `flac`→`audio/flac`, `mp3`→`audio/mpeg`, `m4a`/`aac`→`audio/mp4`, `ogg`/`vorbis`→`audio/ogg`, `opus`→`audio/opus`, `webm`→`audio/webm` をマップ。 |
| `parseTimestamp` | `private fun parseTimestamp(timestamp: String?): Long` | ISO-8601 文字列を epoch ms に変換 (失敗時 0L)。 |

### 特殊処理

- **アーティスト名**: `Artists` 配列と `AlbumArtist` 両方がある場合は `Artists` を優先、無ければ `AlbumArtist`、さらに `Unknown Artist` にフォールバック。
- **長さ**: `RunTimeTicks` (100ns 単位) を `÷ 10_000` で ms に変換。
- **トラック番号**: `IndexNumber` と `ParentIndexNumber` の組合せからトラック/ディスク番号を構築。

## 3. モデル (`data/jellyfin/model/`)

| ファイル | 主要 `data class` | フィールド要約 |
| --- | --- | --- |
| `JellyfinSong.kt` | `JellyfinSong` | `id, title, artist, artistId?, album, albumId?, duration, trackNumber, discNumber, year, genre?, bitRate?, contentType?, path, size?, playCount` |
| `JellyfinAlbum.kt` | `JellyfinAlbum` | `id, name, artist, artistId?, songCount, duration, year, genre?` |
| `JellyfinArtist.kt` | `JellyfinArtist` | `id, name, albumCount` |
| `JellyfinPlaylist.kt` | `JellyfinPlaylist` | `id, name, songCount, duration, created, changed` |
| `JellyfinCredentials.kt` | `JellyfinCredentials` | `serverUrl, username, password, accessToken?, userId?` + 派生 `isValid`, `hasToken`, `normalizedServerUrl`, `connectionValidationError()` |

> 全モデルに `empty()` ファクトリと `@Parcelize` / `@Immutable` 注釈を付与 (UI 層で `LazyColumn` の安定キーとして使う想定)。

## 4. 内部実装メモ

### 認証フロー

```
1. authenticateByName(serverUrl, username, password)
   POST {server}/Users/AuthenticateByName
   Body: {"Username": ..., "Pw": ...}
   Header: MediaBrowser Client="PixelPlayer", ...
   → { "AccessToken": "...", "User": { "Id": "..." } }
2. JellyfinCredentials.copy(accessToken = ..., userId = ...)
3. prefs (EncryptedSharedPreferences "jellyfin_prefs") に永続化
```

### エラーハンドリング

- `Result<T>` でラップし、`Result.failure(Exception)` を返す。
- `IllegalStateException("No credentials configured")` は `request` 内で送出され、Repository 層で再ラップ。

### タイムアウト

- connect / read / write: 15s / 30s / 15s (基底 OkHttpClient の値を `newBuilder()` で上書き)。

## 5. 関連ファイル

- 上位層: [`streaming-jellyfin.md`](./streaming-jellyfin.md)
- 認証モデル: [`../../app/src/main/java/com/theveloper/pixelplay/data/jellyfin/model/JellyfinCredentials.kt`](../../app/src/main/java/com/theveloper/pixelplay/data/jellyfin/model/JellyfinCredentials.kt) (詳細)
- Coil Fetcher: [`image-fetchers.md`](./image-fetchers.md) §2
- 共通セキュリティ: [`streaming-cloud.md`](./streaming-cloud.md)
