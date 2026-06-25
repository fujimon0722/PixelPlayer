# streaming-navidrome.md

> Navidrome の楽曲同期 (Repository) とストリーミングプロキシ (StreamProxy) 仕様。
> **最大規模** のサブシステム (Repository は 960 行)。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/navidrome/
├─ NavidromeRepository.kt          // 960 lines (最大)
├─ NavidromeStreamProxy.kt         // 77 lines
└─ model/
    ├─ NavidromeAlbum.kt           // 51 lines
    ├─ NavidromeArtist.kt          // 36 lines
    ├─ NavidromeCredentials.kt     // 86 lines
    ├─ NavidromeMusicFolder.kt     // 27 lines
    ├─ NavidromePlaylist.kt        // 51 lines
    └─ NavidromeSong.kt            // 93 lines
```

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `presentation/navidrome/*`, `data/repository/MusicRepositoryImpl.kt`, `data/image/NavidromeCoilFetcher.kt`, `data/worker/NavidromeSyncWorker.kt`, `data/service/MusicService.kt` (scrobble) |
| 下流 (依存先) | `data/network/navidrome/{NavidromeApiService, NavidromeResponseParser}`, `data/database/{NavidromeDao, MusicDao}`, `data/preferences/PlaylistPreferencesRepository`, `data/stream/CloudMusicUtils`, `androidx.security.crypto.EncryptedSharedPreferences` |

---

## 1. `NavidromeRepository` (`@Singleton`)

### 依存 (`@Inject constructor`)

| 依存 | 用途 |
| --- | --- |
| `NavidromeApiService` | Subsonic API 呼び出し |
| `NavidromeDao` (Room) | Navidrome 専用エンティティ DAO |
| `MusicDao` | 統合 Song/Album/Artist DAO |
| `PlaylistPreferencesRepository` | アプリ内プレイリスト統合 |
| `@ApplicationContext Context` | EncryptedSharedPreferences |

### 定数

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `SYNC_THRESHOLD_MS` | `24 * 60 * 60 * 1000L` | 24 時間間隔のフルシンク閾値 |
| `TAG` | `"NavidromeRepo"` | |
| `PREFS_NAME` | `"navidrome_prefs"` | |
| `KEY_SERVER_URL` | `"server_url"` | |
| `KEY_USERNAME` | `"username"` | |
| `KEY_PASSWORD` | `"password"` | Navidrome はパスワード永続化 (トークン認証) |
| `KEY_LAST_FULL_SYNC` | `"last_full_sync"` | 最終フルシンク時刻 (ms) |
| `NAVIDROME_SONG_ID_OFFSET` | `9_000_000_000_000L` | 統合 Song ID |
| `NAVIDROME_ALBUM_ID_OFFSET` | `10_000_000_000_000L` | 統合 Album ID |
| `NAVIDROME_ARTIST_ID_OFFSET` | `11_000_000_000_000L` | 統合 Artist ID |
| `NAVIDROME_PARENT_DIRECTORY` | `"/Cloud/Navidrome"` | 統合 Song.parentDirectory |
| `NAVIDROME_GENRE` | `"Navidrome"` | 統合 Song.genre |
| `NAVIDROME_PLAYLIST_PREFIX` | `"navidrome_playlist:"` | アプリ内プレイリスト ID |
| `LIBRARY_PLAYLIST_ID` | `"__library__"` | 全楽曲仮想プレイリスト ID |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `prefs` | `SharedPreferences` | EncryptedSharedPreferences (失敗時通常 prefs) |
| `_isLoggedInFlow` | `MutableStateFlow<Boolean>` | |
| `isLoggedInFlow` | `StateFlow<Boolean>` | |
| `lastFullSyncTime` | `var lastFullSyncTime: Long` | メモリ内 (起動時に prefs から復元) |

### 派生プロパティ

| プロパティ | 型 | 説明 |
| --- | --- | --- |
| `isLoggedIn` | `Boolean` | `credentials?.isValid == true` |
| `serverUrl` | `String?` | `api.getServerUrl()` |
| `username` | `String?` | 永続値 |

### 認証情報管理

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `initFromSavedCredentials` | `private fun initFromSavedCredentials()` | prefs から復元し `api.setCredentials` を呼ぶ。 |
| `login` | `suspend fun login(serverUrl, username, password): Result<String>` | `credentials.connectionValidationError()` 検証 → `api.setCredentials` → `api.ping` → prefs 永続化 → `dao.clearAll` → `_isLoggedInFlow.value = true`。 |
| `logout` | `suspend fun logout()` | `dao.clearAll` + アプリ内プレイリスト削除 + prefs 消去 + `api.clearCredentials` + flow false。 |

### 同期

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `syncPlaylists` | `suspend fun syncPlaylists(): Result<List<NavidromePlaylistEntity>>` | `api.getPlaylists` → パース → DB upsert → stale 削除。 |
| `syncPlaylistSongs` | `suspend fun syncPlaylistSongs(playlistId): Result<Int>` | `api.getPlaylist(id)` → songs パース → DB upsert → アプリ内プレイリスト同期。 |
| `syncLibrarySongs` | `suspend fun syncLibrarySongs(onProgress: ((Float, String) -> Unit)?): Result<Int>` | アルバム単位並列取得 (`Semaphore(5)`)。進捗 0.1 → 0.9。 |
| `syncAllPlaylistsAndSongs` | `suspend fun syncAllPlaylistsAndSongs(onProgress: ((Float, String) -> Unit)?): Result<BulkSyncResult>` | ライブラリ → プレイリスト → 各プレイリスト楽曲。 |

### 並列アルバム取得 (`syncLibrarySongs`)

```kotlin
val totalAlbums = fetchedAlbums.size
val semaphore = Semaphore(5)  // concurrencyLimit = 5
val processedCount = AtomicInteger(0)

coroutineScope {
    val albumSongLists = fetchedAlbums.map { albumJson ->
        async {
            semaphore.withPermit {
                val songsResult = api.getAlbum(albumJson.optString("id", ""))
                val currentProcessed = processedCount.incrementAndGet()
                val progress = 0.1f + (currentProcessed.toFloat() / totalAlbums.coerceAtLeast(1) * 0.8f)
                onProgress?.invoke(progress, "Processing album ...")
                NavidromeResponseParser.parseSongsFromAlbumResponse(songsResult.getOrNull() ?: JSONObject())
            }
        }
    }.awaitAll()
}
```

### `fetchAllAlbums(pageSize)`

```kotlin
private suspend fun fetchAllAlbums(pageSize: Int): List<JSONObject> {
    var offset = 0
    val allAlbums = mutableListOf<JSONObject>()
    while (true) {
        val albumsResult = api.getAlbumList(type = "alphabeticalByName", size = pageSize, offset = offset)
        val albumJsons = albumsResult.getOrNull()?.optJSONObject("albumList2")?.optJSONArray("album") ?: break
        if (albumJsons.length() == 0) break
        albumJsons.toList().forEach { allAlbums.add(it as JSONObject) }
        offset += pageSize
    }
    return allAlbums
}
```

### クエリ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getPlaylists` | `fun getPlaylists(): Flow<List<NavidromePlaylistEntity>>` | DAO 監視。 |
| `getPlaylistSongs` | `fun getPlaylistSongs(playlistId): Flow<List<Song>>` | |
| `getAllSongs` | `fun getAllSongs(): Flow<List<Song>>` | |
| `searchSongs` | `suspend fun searchSongs(query, limit = 30): Result<List<Song>>` | `api.searchSongs` → `parseSongs` → `toSong()`。 |
| `searchLocalSongs` | `fun searchLocalSongs(query): Flow<List<Song>>` | DB ローカル LIKE 検索。 |
| `getStreamUrl` | `fun getStreamUrl(songId, maxBitRate = 0): String` | `api.getStreamUrl` |
| `getCoverArtUrl` | `fun getCoverArtUrl(coverArtId?, size = 500): String?` | `api.getCoverArtUrl` |
| `getLyrics` | `suspend fun getLyrics(songId): Result<String>` | `api.getLyricsBySongId` → 空なら DB から artist/title 取得してフォールバック。 |

### Scrobble / スター

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `reportPlayback` | `suspend fun reportPlayback(navidromeId, isPlaying, positionMs?): Result<Unit>` | `api.reportPlayback` |
| `scrobble` | `suspend fun scrobble(navidromeId, submission = true): Result<Unit>` | `api.scrobble` |
| `deletePlaylist` | `suspend fun deletePlaylist(playlistId)` | DB とアプリ内プレイリスト削除。 |

### 統合レイヤー

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `syncUnifiedLibrarySongsFromNavidrome` | `suspend fun syncUnifiedLibrarySongsFromNavidrome()` | `NavidromeSongEntity` → 統合 Song/Album/Artist upsert。 |
| `parseArtistNames` | `private fun parseArtistNames(rawArtist): List<String>` | 区切り文字で分割 (`utils.splitArtistsByDelimiters`)。 |
| `toUnifiedSongId` | `private fun toUnifiedSongId(navidromeId): Long` | `NAVIDROME_SONG_ID_OFFSET + navidromeId.hashCode().absoluteValue` |
| `toUnifiedAlbumId` | `private fun toUnifiedAlbumId(albumId?, albumName): Long` | |
| `toUnifiedArtistId` | `private fun toUnifiedArtistId(artistName): Long` | |
| `getAppPlaylistIdForNavidrome` | `private fun getAppPlaylistIdForNavidrome(navidromePlaylistId): String` | `"navidrome_playlist:$id"` |
| `updateAppPlaylistForNavidromePlaylist` | `private suspend fun updateAppPlaylistForNavidromePlaylist(...)` | アプリ内プレイリスト upsert |
| `deleteAppPlaylistForNavidromePlaylist` | `private suspend fun deleteAppPlaylistForNavidromePlaylist(...)` | 削除 |
| `NavidromeSong.toSong()` | `fun NavidromeSong.toSong(): Song` | ドメイン Song 変換 |

---

## 2. `NavidromeStreamProxy` (`@Singleton`)

`CloudStreamProxy<String>` の実装。

### 抽象メンバー実装

| メンバー | 値 | 説明 |
| --- | --- | --- |
| `allowedHostSuffixes` | (実装) | Navidrome サーバホストの派生。 |
| `cacheExpirationMs` | `30L * 60 * 1000` (30 分) | |
| `proxyTag` | `"NavidromeStreamProxy"` | |
| `routePath` | `"/navidrome/{songId}"` | |
| `routeParamName` | `"songId"` | |
| `uriScheme` | `"navidrome"` | |
| `routePrefix` | `"/navidrome"` | |

### 抽象メソッド実装

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `parseRouteParam` | `override fun parseRouteParam(value: String): String?` | Subsonic の song ID をそのまま返す。 |
| `validateId` | `override fun validateId(id: String): Boolean` | `CloudStreamSecurity.validateNavidromeSongId` (regex)。 |
| `formatIdForUrl` | `override fun formatIdForUrl(id: String): String` | |
| `resolveStreamUrl` | `override suspend fun resolveStreamUrl(id: String): String?` | `repository.getStreamUrl(id)` |

### 独自メソッド

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `extractIdFromUri` | `override fun extractIdFromUri(uri: Uri): String?` | `uri.host` を songId として抽出。 |
| `resolveNavidromeUri` | `fun resolveNavidromeUri(uriString: String): String?` | `resolveUri(uriString)` ラッパ。 |
| `warmUpStreamUrl` | `suspend fun warmUpStreamUrl(uriString: String)` | URL を解決してキャッシュ温め。 |

---

## 3. `data/navidrome/model/*`

### `NavidromeCredentials`

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `serverUrl` | `String` | |
| `username` | `String` | |
| `password` | `String` | 永続化される (Subsonic トークン認証は毎回パスワードハッシュ) |
| `clientId` | `String` | 既定 `"PixelPlayer"` |

派生:
- `isValid: Boolean`
- `normalizedHttpUrlOrNull: HttpUrl?`
- `normalizedServerUrl: String`
- `connectionValidationError(requireHttps: Boolean = true): String?` — **HTTPS デフォルト強制**

### `NavidromeSong.resolvedMimeType`

`contentType` もしくは `suffix` から MIME 推定。Media3 の ContentType に渡す。

### 他モデル

`@Parcelize @Immutable data class`。詳細は [`network-navidrome.md`](./network-navidrome.md) §3。

---

## 4. 内部実装メモ

### パスワード永続化の判断

Subsonic API は `t = md5(password + salt)` で毎回認証するため、**パスワードを保持しないと認証できない** (token 方式でも salt ごとに password が必要)。EncryptedSharedPreferences で AES-256-GCM 暗号化保存。

### 24 時間フルシンク閾値

```kotlin
if (System.currentTimeMillis() - lastFullSyncTime < SYNC_THRESHOLD_MS) return
```

> `NavidromeSyncWorker` から呼ばれることを想定。

### アルバム並列度 5 の根拠

IO バウンドな GET アルバム楽曲 API のため。Navidrome サーバの Subsonic 実装は単一プロセスなので過負荷を避ける。

### playCount / 再生統計

`NavidromeSong.playCount` を `SongEntity.playCount` にコピーし、`PlaybackStatsRepository` 経由で UI 表示。

---

## 5. 関連ファイル

- 低レベル API: [`network-navidrome.md`](./network-navidrome.md)
- 同期ワーカー: `data/worker/NavidromeSyncWorker.kt`
- Scrobble 連携: `data/service/MusicService.kt`
- Stream Proxy 抽象基底: [`streaming-cloud.md`](./streaming-cloud.md) §1
- Coil: [`image-fetchers.md`](./image-fetchers.md) §3
- DB: [`../01-data-foundation/database.md`](../01-data-foundation/database.md)
