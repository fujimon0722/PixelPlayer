# streaming-netease.md

> Netease Cloud Music の楽曲同期 (Repository) と ストリーミングプロキシ (StreamProxy) 仕様。
> 暗号化レイヤー (`NeteaseEncryption`) は [`network-netease.md`](./network-netease.md) 参照。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/netease/
├─ NeteaseRepository.kt          // 858 lines
└─ NeteaseStreamProxy.kt         // 45 lines
```

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `presentation/netease/*`, `data/repository/MusicRepositoryImpl.kt` |
| 下流 (依存先) | `data/network/netease/NeteaseApiService`, `data/database/{NeteaseDao, MusicDao}`, `data/preferences/{PlaylistPreferencesRepository, UserPreferencesRepository}`, `data/stream/CloudMusicUtils`, `androidx.security.crypto.EncryptedSharedPreferences` |

---

## 1. `NeteaseRepository` (`@Singleton`)

### 依存 (`@Inject constructor`)

| 依存 | 用途 |
| --- | --- |
| `NeteaseApiService` | API 呼び出し |
| `NeteaseDao` (Room) | Netease 専用 DAO |
| `MusicDao` | 統合 DAO |
| `PlaylistPreferencesRepository` | アプリ内プレイリスト |
| `@ApplicationContext Context` | EncryptedSharedPreferences |

### 定数

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `NETEASE_SONG_ID_OFFSET` | `3_000_000_000_000L` | 統合 Song ID |
| `NETEASE_ALBUM_ID_OFFSET` | `4_000_000_000_000L` | 統合 Album ID |
| `NETEASE_ARTIST_ID_OFFSET` | `5_000_000_000_000L` | 統合 Artist ID |
| `NETEASE_PARENT_DIRECTORY` | `"/Cloud/Netease"` | 統合 Song.parentDirectory |
| `NETEASE_GENRE` | `"Netease Cloud"` | 統合 Song.genre |
| `NETEASE_PLAYLIST_PREFIX` | `"netease_playlist:"` | アプリ内プレイリスト ID |
| `NETEASE_PLAYLIST_PAGE_SIZE` | `50` | 1 ページ最大件数 |
| `NETEASE_SONG_DETAIL_BATCH_SIZE` | `500` | `/song/detail` バッチサイズ |
| `NETEASE_MAX_PLAYLIST_PAGES` | `200` | プレイリスト取得ページ上限 |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `prefs` | `SharedPreferences` | EncryptedSharedPreferences (Cookie JSON 保存) |
| `_isLoggedInFlow` | `MutableStateFlow<Boolean>` | |
| `isLoggedInFlow` | `StateFlow<Boolean>` | |
| `inFlightSongUrlRequests` | `ConcurrentHashMap<Long, CompletableDeferred<Result<String>>>` | 同一曲の URL 取得を 1 リクエストに集約。 |
| `lastSongUrlAttemptAtMs` | `ConcurrentHashMap<Long, Long>` | 曲単位の最終試行時刻 |
| `songUrlRequestCooldownMs` | `1500L` | 同一曲 1.5 秒クールダウン |
| `neteaseSongUrlRequestMutex` | `Mutex` | 同一曲のミューテックス |
| `lastGlobalSongUrlRequestAtMs` | `var Long` | 全体最終試行時刻 |
| `globalSongUrlRequestIntervalMs` | `1100L` | 全体 1.1 秒間隔スロットル |

### 派生プロパティ

| プロパティ | 型 | 説明 |
| --- | --- | --- |
| `isLoggedIn` | `Boolean` | `api.hasLogin()` |
| `userId` | `Long` | `api.getCurrentUserId()` |
| `userNickname` | `String?` | `api` 初回呼び出し時のメモ (lazy) |
| `userAvatar` | `String?` | 同上 |

### 認証情報管理

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `initFromSavedCookies` | `fun initFromSavedCookies()` | prefs の `netease_cookies` JSON から復元し `api.setPersistedCookies`。 |
| `loginWithCookies` | `suspend fun loginWithCookies(cookieJson): Result<String>` | クッキー JSON を `api.setPersistedCookies` → `api.ensureWeapiSession` → `api.getCurrentUserAccount` → DB / prefs に保存。 |
| `logout` | `suspend fun logout()` | `dao.clearAll` + アプリ内プレイリスト削除 + prefs 消去 + `api.logout`。 |
| `saveUserInfo` | `private fun saveUserInfo(userId, nickname, avatarUrl)` | ユーザ情報を prefs に保存。 |
| `clearLoginState` | `private fun clearLoginState()` | 内部状態リセット。 |

### 同期

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `syncUserPlaylists` | `suspend fun syncUserPlaylists(): Result<List<NeteasePlaylistEntity>>` | `api.getUserPlaylists` をページング (`NETEASE_PLAYLIST_PAGE_SIZE`、`NETEASE_MAX_PLAYLIST_PAGES` 上限)。 |
| `syncPlaylistSongs` | `suspend fun syncPlaylistSongs(playlistId): Result<Int>` | `api.getPlaylistDetail` → embedded tracks を優先、不足分は `trackIds` 経由で `/song/detail` をバッチ取得。 |
| `syncAllPlaylistsAndSongs` | `suspend fun syncAllPlaylistsAndSongs(): Result<BulkSyncResult>` | `syncUserPlaylists` → 各プレイリスト楽曲同期。 |

### `syncPlaylistSongs` の特殊ロジック

```
1. playlist = api.getPlaylistDetail(playlistId)
2. embeddedTracks = playlist.tracks
3. trackIds = playlist.trackIds (id のみ)
4. entitiesBySongId に embedded tracks を順次挿入
5. orderedTrackIds に trackIds を順次追加 (順序保持)
6. 不足分 (missingTrackIds) を 500 件ずつ batch で /song/detail
7. 最終的に orderedTrackIds の順序で sort
```

### クエリ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getPlaylists` | `fun getPlaylists(): Flow<List<NeteasePlaylistEntity>>` | DAO 監視 |
| `getPlaylistSongs` | `fun getPlaylistSongs(playlistId): Flow<List<Song>>` | |
| `getAllSongs` | `fun getAllSongs(): Flow<List<Song>>` | |
| `searchLocalSongs` | `fun searchLocalSongs(query): Flow<List<Song>>` | |
| `searchOnline` | `suspend fun searchOnline(keywords, limit = 30): Result<List<Song>>` | `api.searchSongs` → 軽量 Song 変換 (ID のみ)。 |
| `getSongUrl` | `suspend fun getSongUrl(songId, quality = "lossless"): Result<String>` | レート制御 + 多重防止 + quality フォールバック。 |
| `getLyrics` | `suspend fun getLyrics(songId): Result<String>` | `api.getLyrics` → `lrc.lyric`。 |
| `deletePlaylist` | `suspend fun deletePlaylist(playlistId)` | DB + アプリ内プレイリスト削除。 |

### `getSongUrl` のレート制御

```
1. now = System.currentTimeMillis()
2. lastAttempt = lastSongUrlAttemptAtMs[songId]
3. if lastAttempt != null && now - lastAttempt < songUrlRequestCooldownMs(1500ms):
     delay + retry
4. requestDeferred = CompletableDeferred<Result<String>>()
5. existing = inFlightSongUrlRequests.putIfAbsent(songId, requestDeferred)
6. if existing != null:
     return existing.await()  // 既に進行中のリクエスト結果を再利用
7. withContext(Dispatchers.IO) {
     // 全曲 1.1 秒間隔スロットル
     // qualityFallbacks = linkedSetOf(quality, "exhigh", "higher", "standard")
     // 順に試して url 非空 + freeTrialInfo 無し を選ぶ
   }
```

### 内部データ変換

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `parseTrackToEntity` | `private fun parseTrackToEntity(track: JSONObject, playlistId: Long): NeteaseSongEntity` | `/song/detail` または playlist 内 embedded tracks を NeteaseSongEntity に変換。 |
| `parseTrackToSong` | `private fun parseTrackToSong(track: JSONObject): Song` | 検索結果の Song を軽量変換 (ID + タイトル + アーティスト + アルバム + 時間)。 |
| `requestSongUrl` | `private suspend fun requestSongUrl(songId, level): String` | 全体スロットル + `api.getSongDownloadUrl(songId, level)`。 |

### 統合レイヤー

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `syncUnifiedLibrarySongsFromNetease` | `private suspend fun syncUnifiedLibrarySongsFromNetease()` | `NeteaseSongEntity` → 統合 Song/Album/Artist upsert (neteaseArtistRefs も別途生成)。 |
| `parseArtistNames` | `private fun parseArtistNames(rawArtist): List<String>` | 区切り文字分割。 |
| `toUnifiedSongId` | `private fun toUnifiedSongId(neteaseId: Long): Long` | `NETEASE_SONG_ID_OFFSET + neteaseId.absoluteValue` |
| `toUnifiedAlbumId` | `private fun toUnifiedAlbumId(albumId: Long, albumName): Long` | albumId > 0 なら `albumId.absoluteValue`、それ以外は `albumName.lowercase().hashCode().absoluteValue`。 |
| `toUnifiedArtistId` | `private fun toUnifiedArtistId(artistName): Long` | |
| `getAppPlaylistIdForNetease` | `private suspend fun getAppPlaylistIdForNetease(neteasePlaylistId): String` | `"netease_playlist:$id"` |
| `updateAppPlaylistForNeteasePlaylist` | `private suspend fun updateAppPlaylistForNeteasePlaylist(...)` | アプリ内プレイリスト upsert |
| `deleteAppPlaylistForNeteasePlaylist` | `private suspend fun deleteAppPlaylistForNeteasePlaylist(...)` | 削除 |
| `jsonToMap` | `private fun jsonToMap(json: String): Map<String, String>` | JSON 文字列を `Map<String, String>` に展開。 |

---

## 2. `NeteaseStreamProxy` (`@Singleton`)

`CloudStreamProxy<Long>` の実装。**ID として `Long` を使用** (Netease は数値 ID)。

### 抽象メンバー実装

| メンバー | 値 | 説明 |
| --- | --- | --- |
| `allowedHostSuffixes` | `setOf("music.163.com", "music.126.net", "interface.music.163.com")` | 公式 Netease 配信ホスト |
| `cacheExpirationMs` | `15L * 60 * 1000` (15 分) | URL キャッシュ TTL |
| `proxyTag` | `"NeteaseStreamProxy"` | |
| `routePath` | `"/netease/{songId}"` | |
| `routeParamName` | `"songId"` | |
| `uriScheme` | `"netease"` | |
| `routePrefix` | `"/netease"` | |

### 抽象メソッド実装

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `parseRouteParam` | `override fun parseRouteParam(value: String): Long?` | `value.toLongOrNull()` |
| `validateId` | `override fun validateId(id: Long): Boolean` | `CloudStreamSecurity.validateNeteaseSongId(id)` (`id > 0`)。 |
| `formatIdForUrl` | `override fun formatIdForUrl(id: Long): String` | `id.toString()` |
| `resolveStreamUrl` | `override suspend fun resolveStreamUrl(id: Long): String?` | `repository.getSongUrl(id)` |

### 独自メソッド

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `resolveNeteaseUri` | `fun resolveNeteaseUri(uriString: String): String?` | `resolveUri(uriString)` ラッパ。 |

---

## 3. 内部実装メモ

### Quality フォールバック順

```
入力 quality (既定 "lossless") → "exhigh" → "higher" → "standard"
```

各レベルで `data[0].url` が空文字 or `freeTrialInfo != null` (试听) なら次へ。

### 全体スロットル 1.1 秒

```kotlin
private suspend fun requestSongUrl(songId: Long, level: String): String {
    val now = System.currentTimeMillis()
    val waitMs = globalSongUrlRequestIntervalMs - (now - lastGlobalSongUrlRequestAtMs)
    if (waitMs > 0) delay(waitMs)
    lastGlobalSongUrlRequestAtMs = System.currentTimeMillis()
    return api.getSongDownloadUrl(songId, level)
}
```

> Netease API は短い間隔で多数叩くと IP ベースでレート制限される。

### 同一曲の重複防止

`inFlightSongUrlRequests` で `CompletableDeferred<Result<String>>` を共有。同一曲の並行再生リクエストでも 1 つの API 呼び出しに集約。

### 歌詞の LRC

`api.getLyrics(songId)` → `{"lrc": {"lyric": "...LRC テキスト..."}, ...}` を返し、`Result.success(lyric)` でラップ。

---

## 4. 関連ファイル

- 低レベル API + 暗号化: [`network-netease.md`](./network-netease.md)
- Stream Proxy 抽象基底: [`streaming-cloud.md`](./streaming-cloud.md) §1
- セキュリティ: [`streaming-cloud.md`](./streaming-cloud.md) §2
- DB: [`../01-data-foundation/database.md`](../01-data-foundation/database.md)
