# network-navidrome.md

> Navidrome / Subsonic API クライアント。
> Subsonic プロトコル互換 (Gonic / Airsonic-Advanced / Funkwhale 等も対応)。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/network/navidrome/
├─ NavidromeApiService.kt        // Subsonic API クライアント (568 lines)
└─ NavidromeResponseParser.kt    // JSON -> NavidromeSong/Album/Artist/Playlist (256 lines)
```

## ファイル概要

| ファイル | 行数 | 概要 |
| --- | ---: | --- |
| `NavidromeApiService.kt` | 568 | Subsonic API v1.16.1 を実装。`md5(password + salt)` トークン認証。 |
| `NavidromeResponseParser.kt` | 256 | Subsonic レスポンス (`subsonic-response` キー) をパース。 |

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `data/navidrome/NavidromeRepository.kt`, `data/navidrome/NavidromeStreamProxy.kt`, `data/image/NavidromeCoilFetcher.kt`, `data/worker/NavidromeSyncWorker.kt` |
| 下流 (依存先) | `data/navidrome/model/NavidromeCredentials.kt`, `org.json.JSONObject`, `okhttp3`, `java.security.MessageDigest` (MD5) |

## 1. `NavidromeApiService`

### 定数 (`companion object`)

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `TAG` | `"NavidromeApi"` | ログタグ |
| `API_VERSION` | `"1.16.1"` | Subsonic プロトコルバージョン |
| `DEFAULT_CLIENT_ID` | `"PixelPlayer"` | `c=` クエリ |
| `DEFAULT_FORMAT` | `"json"` | `f=json` クエリ |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `credentials` | `@Volatile NavidromeCredentials?` | 認証情報 |
| `okHttpClient` | `OkHttpClient` | 15s/30s/15s タイムアウト (Subsonic は大量楽曲取得で時間がかかることがある) |

### 認証メカニズム

`generateAuthParams(password)` (L86) で `(token, salt)` を生成:
- `salt = UUID.randomUUID().toString().take(6)` — 先頭 6 文字。
- `token = md5(password + salt)` — hex 文字列。
- クエリ: `u=<user>&t=<token>&s=<salt>&v=1.16.1&c=PixelPlayer&f=json`。

> `p` (plaintext) 認証も Subsonic プロトコル上は可能だが、本実装は `t/s` トークン認証のみ。

### 公開 API メソッド

#### 認証情報管理

| メソッド | シグネチャ | 目的 / 戻り値 | 呼び出し元 |
| --- | --- | --- | --- |
| `setCredentials` | `fun setCredentials(credentials: NavidromeCredentials)` | 認証情報設定。 | `NavidromeRepository.login()` |
| `clearCredentials` | `fun clearCredentials()` | 認証情報破棄。 | `NavidromeRepository.logout()` |
| `hasCredentials` | `fun hasCredentials(): Boolean` | `credentials?.isValid == true`。 | Repository |
| `getServerUrl` | `fun getServerUrl(): String?` | 正規化済み URL。 | UI 表示 |

#### システム

| メソッド | シグネチャ | 目的 / 戻り値 |
| --- | --- | --- |
| `ping` | `suspend fun ping(): Result<Boolean>` | `/rest/ping.view` で疎通確認。 |
| `getLicense` | `suspend fun getLicense(): Result<JSONObject>` | `/rest/getLicense.view` でサーバライセンス情報。 |
| `getMusicFolders` | `suspend fun getMusicFolders(): Result<List<JSONObject>>` | `/rest/getMusicFolders.view`。 |

#### ブラウズ

| メソッド | シグネチャ | 目的 / 戻り値 |
| --- | --- | --- |
| `getArtists` | `suspend fun getArtists(musicFolderId?): Result<List<JSONObject>>` | `/rest/getArtists.view`。インデックス → アーティスト配列を平坦化。 |
| `getArtist` | `suspend fun getArtist(id): Result<JSONObject>` | `/rest/getArtist.view?id=...`。 |
| `getAlbumList` | `suspend fun getAlbumList(type, size=10, offset=0, musicFolderId?): Result<List<JSONObject>>` | `/rest/getAlbumList2.view`。`type` は `newest`, `random`, `frequent`, `recent`, `alphabeticalByName`, `alphabeticalByArtist`, `starred`, `byYear`, `byGenre`。 |
| `getAlbum` | `suspend fun getAlbum(id): Result<List<JSONObject>>` | `/rest/getAlbum.view?id=...` の songs 配列を返す。 |
| `getSong` | `suspend fun getSong(id): Result<JSONObject>` | `/rest/getSong.view?id=...`。 |

#### プレイリスト

| メソッド | シグネチャ | 目的 / 戻り値 |
| --- | --- | --- |
| `getPlaylists` | `suspend fun getPlaylists(): Result<List<JSONObject>>` | `/rest/getPlaylists.view`。`playlists.playlist` または root 直下の `playlist` を吸収。 |
| `getPlaylist` | `suspend fun getPlaylist(id): Result<Pair<JSONObject, List<JSONObject>>>` | `/rest/getPlaylist.view` の `playlist` JSON + songs 配列。`entry`/`song` 両対応。 |

#### 検索

| メソッド | シグネチャ | 目的 / 戻り値 |
| --- | --- | --- |
| `search3` | `suspend fun search3(query, artistCount=20, albumCount=20, songCount=20): Result<JSONObject>` | `/rest/search3.view`。生 JSON を返す。 |
| `searchSongs` | `suspend fun searchSongs(query, count=30): Result<List<JSONObject>>` | `searchResult3.song` を抽出。 |
| `searchAlbums` | `suspend fun searchAlbums(query, count=20): Result<List<JSONObject>>` | `searchResult3.album` を抽出。 |
| `searchArtists` | `suspend fun searchArtists(query, count=10): Result<List<JSONObject>>` | `searchResult3.artist` を抽出。 |

#### ストリーム / カバーアート

| メソッド | シグネチャ | 目的 / 戻り値 |
| --- | --- | --- |
| `getStreamUrl` | `fun getStreamUrl(songId, maxBitRate=0, format?): String` | `/rest/stream.view?id=...&maxBitRate=...&format=...`。 |
| `getCoverArtUrl` | `fun getCoverArtUrl(coverArtId, size=500): String` | `/rest/getCoverArt.view?id=...&size=500`。 |

#### 歌詞

| メソッド | シグネチャ | 目的 / 戻り値 |
| --- | --- | --- |
| `getLyrics` | `suspend fun getLyrics(artist?, title?): Result<String>` | `/rest/getLyrics.view`。`artist`/`title` 指定検索。 |
| `getLyricsBySongId` | `suspend fun getLyricsBySongId(songId): Result<String>` | `/rest/getLyrics.view?songId=...`。`structuredLyrics[].line[].value` を `\n` 連結。 |

#### Last.fm 風 scrobble / スター

| メソッド | シグネチャ | 目的 / 戻り値 |
| --- | --- | --- |
| `reportPlayback` | `suspend fun reportPlayback(songId, isPlaying, positionMs?): Result<Unit>` | `/rest/scrobble.view` の薄いラッパ。 |
| `scrobble` | `suspend fun scrobble(id, submission=true): Result<Unit>` | submission=true で実際に scrobble、false で "now playing"。 |
| `star` | `suspend fun star(id?, albumId?, artistId?): Result<Boolean>` | `/rest/star.view`。 |
| `unstar` | `suspend fun unstar(id?, albumId?, artistId?): Result<Boolean>` | `/rest/unstar.view`。 |

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `md5` | `private fun md5(input: String): String` | `MessageDigest.getInstance("MD5")` → hex。 |
| `buildApiUrl` | `private fun buildApiUrl(endpoint, extraParams): String` | 認証クエリ + 任意追加クエリで URL を組み立てる。 |
| `request` | `private suspend fun request(endpoint, params): Result<String>` | OkHttp GET。HTTP エラーや IOException を `Result.failure` 化。 |
| `parseResponse` | `private fun parseResponse(raw: String): Result<JSONObject>` | `subsonic-response` の `status` を確認し、`failed` ならエラーコード/メッセージを `Result.failure`。 |
| `requestAndParse` | `private suspend fun requestAndParse(endpoint, params): Result<JSONObject>` | `request` + `parseResponse` を連結。 |

### Subsonic レスポンス解釈

`parseResponse` の動作:
```
subsonic-response.status == "ok"  → Result.success(JSON)
subsonic-response.status == "failed" → Result.failure(Exception("Subsonic error code=..., message=..."))
```

## 2. `NavidromeResponseParser` (`object`)

### 定数

| 定数 | 値 |
| --- | --- |
| `TAG` | `"NavidromeParser"` |

### 公開関数

| 関数 | シグネチャ | 目的 |
| --- | --- | --- |
| `parseMusicFolder` | `fun parseMusicFolder(json: JSONObject): NavidromeMusicFolder` | `id`/`name`。 |
| `parseArtist` | `fun parseArtist(json: JSONObject): NavidromeArtist` | `id`, `name`, `coverArt`, `albumCount` (派生), `artistImageUrl`。 |
| `parseArtists` | `fun parseArtists(jsonArray: List<JSONObject>): List<NavidromeArtist>` | リスト変換。 |
| `parseAlbum` | `fun parseAlbum(json: JSONObject): NavidromeAlbum` | `id`/`name`/`artist`/`coverArt`/`songCount`/`duration`/`year`/`genre`/`playCount`。 |
| `parseAlbums` | `fun parseAlbums(jsonArray: List<JSONObject>): List<NavidromeAlbum>` | リスト変換。 |
| `parseSong` | `fun parseSong(json: JSONObject): NavidromeSong` | `id`/`title`/`artist`/`album`/`duration`/`track`/`discNumber`/`year`/`genre`/`bitRate`/`contentType`/`suffix`/`coverArt`/`size`/`playCount`。 |
| `parseSongs` | `fun parseSongs(jsonArray: List<JSONObject>): List<NavidromeSong>` | リスト変換。 |
| `parseSongsFromAlbumResponse` | `fun parseSongsFromAlbumResponse(response: JSONObject): List<NavidromeSong>` | `getAlbum.view` の `album.song` を抽出。 |
| `parsePlaylist` | `fun parsePlaylist(json: JSONObject): NavidromePlaylist` | `id`/`name`/`comment`/`owner`/`songCount`/`duration`/`coverArt`/`public`/`created`/`changed`。 |
| `parsePlaylists` | `fun parsePlaylists(jsonArray: List<JSONObject>): List<NavidromePlaylist>` | リスト変換。 |
| `parseSongsFromPlaylistResponse` | `fun parseSongsFromPlaylistResponse(response: JSONObject): Pair<NavidromePlaylist?, List<NavidromeSong>>` | `playlist` JSON を playlist + songs に分解。 |
| `parseSearchResults` | `fun parseSearchResults(response: JSONObject): SearchResults` | `searchResult3` から artists/albums/songs を一括抽出。 |

### 内部データクラス

```kotlin
data class SearchResults(
    val artists: List<NavidromeArtist> = emptyList(),
    val albums: List<NavidromeAlbum> = emptyList(),
    val songs: List<NavidromeSong> = emptyList()
)
```

### 非公開ヘルパ

| 関数 | シグネチャ | 目的 |
| --- | --- | --- |
| `parseTimestamp` | `private fun parseTimestamp(timestamp: String?): Long` | ISO-8601 (`2006-01-02T15:04:05.000Z` 形式) を epoch ms に変換。 |

## 3. モデル (`data/navidrome/model/`)

| ファイル | 主要 `data class` | フィールド要約 |
| --- | --- | --- |
| `NavidromeSong.kt` | `NavidromeSong` | `id, title, artist, artistId?, album, albumId?, coverArt?, duration, trackNumber, discNumber, year, genre?, bitRate?, contentType?, suffix?, path, size?, playCount` |
| `NavidromeAlbum.kt` | `NavidromeAlbum` | `id, name, artist, artistId?, coverArt?, songCount, duration, playCount, year, genre?` |
| `NavidromeArtist.kt` | `NavidromeArtist` | `id, name, coverArt?, albumCount, artistImageUrl?` |
| `NavidromePlaylist.kt` | `NavidromePlaylist` | `id, name, comment?, owner?, songCount, duration, coverArt?, public, created, changed` |
| `NavidromeMusicFolder.kt` | `NavidromeMusicFolder` | `id, name` |
| `NavidromeCredentials.kt` | `NavidromeCredentials` | `serverUrl, username, password, clientId` + 派生 `isValid`, `normalizedServerUrl`, `connectionValidationError()` |

> `NavidromeCredentials.connectionValidationError(requireHttps: Boolean = true)` は **デフォルトで HTTPS を要求** (LAN 内 HTTP サーバも明示許可が必要)。

## 4. 内部実装メモ

### API バージョン固定

`API_VERSION = "1.16.1"` を Subsonic 仕様で要求。一部サーバは 1.16.1 未満のフィールド欠落があるため `opt*` 系で安全アクセス。

### 大量楽曲同期

Navidrome は数千〜数万曲規模になることが多く、`syncLibrarySongs` 側でアルバム単位並列取得 (`Semaphore(5)`) を行う (`streaming-navidrome.md` 参照)。本サービス単体では逐次 GET。

### Stream URL の format 引数

`format=raw|mp3|flac|opus` をサーバに通知。Navidrome の Transcoding 設定に従う。

## 5. 関連ファイル

- 上位層: [`streaming-navidrome.md`](./streaming-navidrome.md)
- 認証モデル: `data/navidrome/model/NavidromeCredentials.kt`
- 同期ワーカー: `data/worker/NavidromeSyncWorker.kt`
- Coil Fetcher: [`image-fetchers.md`](./image-fetchers.md) §3
- 共通セキュリティ: [`streaming-cloud.md`](./streaming-cloud.md)
