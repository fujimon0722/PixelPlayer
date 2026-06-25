# Repositories

データレイヤーの中核。Room DAO / MediaStore / Network API を統合して、楽曲・アルバム・アーティスト・プレイリスト・歌詞・トランジション・ルールの読み書き API を提供します。

## パッケージ

`com.theveloper.pixelplay.data.repository`

---

## 依存関係

### 上流（呼び出し元）
- `presentation/viewmodel/...` — `LibraryViewModel`, `PlayerViewModel`, `HomeViewModel`, `SearchViewModel`, `PlaylistViewModel` 等
- `service/...` — `MusicService`（お気に入りや再生カウント）
- `worker/SyncWorker.kt` — Telegram / Netease / Navidrome 同期時に Repository 呼び出し
- `di/AppModule.kt` — Hilt @Provides で `MusicRepository` を `MusicRepositoryImpl` に bind

### 下流（呼ばれる側）
- `data/database/MusicDao.kt`, `FavoritesDao.kt`, `LyricsDao.kt`, `SearchHistoryDao.kt`, `TransitionDao.kt`, `TelegramDao.kt`, `LocalPlaylistDao.kt`
- `data/preferences/UserPreferencesRepository.kt`, `PlaylistPreferencesRepository.kt`
- `data/observer/MediaStoreObserver.kt`
- `data/network/.../DeezerApiService.kt`, `LrcLibApiService.kt`
- `utils/AlbumArtUtils.kt`, `utils/DirectoryFilterUtils.kt`, `utils/DirectoryRuleResolver.kt`
- `utils/LocalArtworkUri.kt`, `utils/StorageUtils.kt`

---

## ファイル一覧

| ファイル | 役割 |
|---|---|
| `MusicRepository.kt` | 統合 API の interface。`Flow`/`suspend`/`PagingData` のミックス |
| `MusicRepositoryImpl.kt` | 実装本体（1192 行）。Room 中心、`SearchPrefs` 構造体でキャッシュ |
| `MediaStoreSongRepository.kt` | MediaStore 直読み版（外部楽曲専用、`SongRepository` を実装） |
| `SongRepository.kt` | 単純なフロー中心の楽曲リポジトリ interface |
| `LyricsRepository.kt` | 歌詞取得 API interface |
| `LyricsRepositoryImpl.kt` | LRCLIB + AMLLDB + ローカル .lrc + 埋め込み歌詞の多段取得実装（1722 行） |
| `TransitionRepository.kt` | 楽曲間トランジションルール interface |
| `TransitionRepositoryImpl.kt` | ルール解決 + DataStore 設定フォールバック |
| `ArtistImageRepository.kt` | Deezer 経由のアーティスト画像 URL 取得 + LRU キャッシュ + ユーザー設定画像 |
| `FolderTreeBuilder.kt` | `FolderSongRow` 群から階層的な `MusicFolder` ツリーを構築 |

---

## `MusicRepository` (interface)

`MusicRepository.kt:17`

ライブラリ全体の公開 API。実装は `MusicRepositoryImpl`、内部で `SongRepository` / `PlaylistPreferencesRepository` / `LyricsRepository` を委譲呼び出し。

### 主要メソッド（Flow）

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `getAudioFiles()` | 22 | `Flow<List<Song>>` | 許可ディレクトリとブロックディレクトリを考慮した楽曲リストを流す |
| `getAlbums(storageFilter, minTracks)` | 140 | `Flow<List<Album>>` | アルバム一覧（`Offline`/`Online`/`All`） |
| `getArtists(storageFilter)` | 149 | `Flow<List<Artist>>` | アーティスト一覧（トラック数付き） |
| `getSongsForAlbum(albumId)` | 203 | `Flow<List<Song>>` | アルバム内の楽曲をトラック番号順 |
| `getSongsForArtist(artistId)` | 210 | `Flow<List<Song>>` | アーティストの全楽曲 |
| `getSongsByIds(songIds)` | 217 | `Flow<List<Song>>` | ID リスト順を保持した楽曲マップ |
| `getAlbumById(id)` | 190 | `Flow<Album?>` | 単一アルバム監視 |
| `getArtistById(artistId)` | 287 | `Flow<Artist?>` | 単一アーティスト監視 |
| `getArtistsForSong(songId)` | 289 | `Flow<List<Artist>>` | 楽曲に紐づくアーティスト群 |
| `getSong(songId)` | 286 | `Flow<Song?>` | 単一楽曲（`Long.parseLong` 失敗時は Telegram テーブルから検索） |
| `getFavoriteSongIdsFlow()` | 279 | `Flow<Set<String>>` | お気に入り ID の reactive セット |
| `getFavoriteSongCountFlow(storageFilter)` | 76 | `Flow<Int>` | お気に入り件数 |
| `getSongCountFlow()` | 84 | `Flow<Int>` | DB 内総楽曲数 |
| `getCloudSongCountFlow()` | 89 | `Flow<Int>` | クラウドソース楽曲数（Telegram+Netease+Navidrome+GDrive+QQ） |
| `getAllUniqueAlbumArtUris()` | 234 | `Flow<List<Uri>>` | テーマプレキャッシュ用のアート URI 群 |
| `getDistinctAlbumArtSongs()` | 163 | `Flow<List<Song>>` | アルバムアート代表楽曲 |
| `getHomeMixPreviewSongs(limit)` | 168 | `Flow<List<Song>>` | Home 用プレビュー楽曲 |
| `searchSongs(query, titleOnly)` | 238 | `Flow<List<Song>>` | タイトル / アーティスト検索 |
| `searchAlbums(query, minTracks)` | 239 | `Flow<List<Album>>` | アルバム検索 |
| `searchArtists(query)` | 240 | `Flow<List<Artist>>` | アーティスト検索 |
| `searchAll(query, filterType)` | 242 | `Flow<List<SearchResultItem>>` | 4 カテゴリ横断検索 |
| `getMusicByGenre(genreId)` | 255 | `Flow<List<Song>>` | ジャンル別楽曲（`unknown` 特殊分岐あり） |
| `getGenres()` | 295 | `Flow<List<Genre>>` | ジャンル一覧（Mock / DB 由来、`unknown` 含む） |
| `getMusicFolders(storageFilter)` | 327 | `Flow<List<MusicFolder>>` | フォルダ階層ツリー |
| `getAllTelegramChannels()` | 339 | `Flow<List<TelegramChannelEntity>>` | Telegram 登録チャンネル一覧 |
| `getAllTelegramTopics()` | 345 | `Flow<List<TelegramTopicEntity>>` | Telegram トピック一覧 |

### 主要メソッド（PagingData）

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `getPaginatedSongs(sortOption, storageFilter)` | 28 | `Flow<PagingData<Song>>` | 楽曲の `LazyPagingItems` 用 |
| `getPaginatedAlbums(sortOption, storageFilter, minTracks)` | 33 | `Flow<PagingData<Album>>` | アルバムページング |
| `getPaginatedArtists(sortOption, storageFilter)` | 42 | `Flow<PagingData<Artist>>` | アーティストページング |
| `getPaginatedFavoriteSongs(sortOption, storageFilter)` | 51 | `Flow<PagingData<Song>>` | お気に入りページング |

### 主要メソッド（suspend / 単発）

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `getAllSongsOnce()` | 157 | `List<Song>` | DB スナップショット |
| `getAllAlbumsOnce(storageFilter, minTracks)` | 174 | `List<Album>` | DB スナップショット |
| `getAllArtistsOnce()` | 183 | `List<Artist>` | DB スナップショット |
| `getFavoriteSongsOnce(storageFilter)` | 59 | `List<Song>` | お気に入り全件（シャッフル用） |
| `getFavoriteSongsPage(limit, offset, sort, filter)` | 66 | `List<Song>` | 範囲指定のお気に入り（Queue 構築用） |
| `getSongsPage(limit, offset, sort, filter)` | 102 | `List<Song>` | 範囲指定楽曲ページ |
| `getAlbumsPage(...)` | 112 | `List<Album>` | 範囲指定アルバム |
| `getArtistsPage(...)` | 123 | `List<Artist>` | 範囲指定アーティスト |
| `getRandomSongs(limit)` | 97 | `List<Song>` | DB レベルの `ORDER BY RANDOM()` |
| `getFirstPlayableSong()` | 134 | `Song?` | 起動時のフォールバック再生用 |
| `getSongByPath(path)` | 224 | `Song?` | パス指定の単発検索 |
| `getAllUniqueAudioDirectories()` | 232 | `Set<String>` | MediaStore を一度走査してディレクトリ列挙 |
| `searchPlaylists(query)` | 241 | `List<Playlist>` | プレイリスト検索 |
| `getFavoriteSongIdsOnce()` | 274 | `Set<String>` | お気に入り ID スナップショット |
| `setFavoriteStatus(songId, isFavorite)` | 269 | `Unit` | お気に入りセット / 解除 |
| `toggleFavoriteStatus(songId)` | 262 | `Boolean` | 反転、新状態を返す |
| `addSearchHistoryItem(query)` | 245 | `Unit` | 検索履歴追加（重複削除→新エントリ） |
| `getRecentSearchHistory(limit)` | 246 | `List<SearchHistoryItem>` | 履歴取得 |
| `deleteSearchHistoryItemByQuery(query)` | 247 | `Unit` | 履歴個別削除 |
| `clearSearchHistory()` | 248 | `Unit` | 履歴全消去 |
| `getArtistIdByName(name)` | 288 | `Long?` | 名前→アーティスト ID |
| `getLyrics(song, sourcePreference, forceRefresh)` | 297 | `Lyrics?` | 歌詞取得（埋め込み/API/ローカル優先度指定） |
| `getStoredLyrics(song)` | 303 | `Pair<Lyrics, String>?` | DB 保存済み歌詞のみ |
| `getLyricsFromRemote(song)` | 305 | `Result<Pair<Lyrics, String>>` | LRCLIB から取得 |
| `searchRemoteLyrics(song)` | 312 | `Result<Pair<String, List<LyricsSearchResult>>>` | 複数候補を返す |
| `searchRemoteLyricsByQuery(title, artist)` | 319 | `Result<Pair<String, List<LyricsSearchResult>>>` | クエリ直接指定 |
| `updateLyrics(songId, lyrics)` | 321 | `Unit` | 歌詞を保存 |
| `resetLyrics(songId)` | 323 | `Unit` | 単曲削除 |
| `resetAllLyrics()` | 325 | `Unit` | 全削除 |
| `invalidateCachesDependentOnAllowedDirectories()` | 236 | `Unit` | キャッシュ無効化（no-op 実装、Flow で対応済） |
| `deleteById(id)` | 331 | `Unit` | 楽曲 1 件削除 |
| `saveTelegramSongs(songs)` | 332 | `Unit` | Telegram 楽曲一括保存 + SyncWorker enqueue |
| `replaceTelegramSongsForChannel(chatId, songs)` | 334 | `Unit` | チャンネル単位の差分置換 |
| `clearTelegramData()` | 336 | `Unit` | Telegram データ全消去 |
| `saveTelegramChannel(channel)` | 338 | `Unit` | チャンネル登録 + Sync enqueue |
| `deleteTelegramChannel(chatId)` | 340 | `Unit` | チャンネル削除 + 関連楽曲/プレイリスト全消去 |
| `saveTelegramTopics(chatId, topics)` | 341 | `Unit` | トピック保存 |
| `replaceTopicsForChannel(chatId, freshTopics)` | 343 | `Unit` | トピック差分更新（消えたものは削除） |
| `getTopicsForChannel(chatId)` | 344 | `List<TelegramTopicEntity>` | トピック単発取得 |
| `replaceTelegramSongsForTopic(chatId, threadId, topicName, songs)` | 346 | `Unit` | トピック別楽曲置換 + playlist 同期 |
| `getSongIdsSorted(sort, filter)` | 350 | `List<Long>` | ソート済み ID 列（Queue 構築） |
| `getFavoriteSongIdsSorted(sort, filter)` | 355 | `List<Long>` | お気に入りソート済み ID 列 |
| `getSongIdByContentUri(contentUri)` | 365 | `Long?` | Content URI → 統一 ID |
| `requestTelegramUnifiedSync()` | 374 | `Unit` | `KEEP` ポリシーで SyncWorker enqueue |

### プロパティ

| 名前 | 型 | 目的 |
|---|---|---|
| `telegramRepository` | `TelegramRepository` | `Lazy<>` 経由の遅延取得 |

---

## `MusicRepositoryImpl`

`MusicRepositoryImpl.kt:88`

`@Singleton` / `@Inject` で全 DAO + Preferences + Lazy<TelegramRepository/CacheManager> + ArtistImageRepository + FolderTreeBuilder + SongRepository + LyricsRepository を受ける。

### 内部実装メモ

#### `CachedDirFilter`
`MusicRepositoryImpl.kt:133`
```
data class CachedDirFilter(val allowedParentDirs: List<String>, val applyFilter: Boolean)
```
- `cachedDirFilter: StateFlow<CachedDirFilter>` (`MusicRepositoryImpl.kt:136`) を `combine(allowedDirectoriesFlow, blockedDirectoriesFlow).flatMapLatest { … }` で計算。
- `blockedDirs.isEmpty()` のときはテーブル走査を省略して `applyFilter=false` を即座に返す（コメント `MusicRepositoryImpl.kt:142` 参照）。
- `blockedDirs` が空でない場合は `musicDao.getDistinctParentDirectoriesFlow()` を購読し、変更があるたびに `DirectoryFilterUtils.computeAllowedParentDirs(...)` を再評価する。
- 旧版は起動時に空スナップショットで固定されていた → プレイバックキューが 1 曲しか再生できない不具合があった（コメント `MusicRepositoryImpl.kt:147-152` 参照）。

#### ディレクトリフィルタ計算
`computeAllowedDirs` (`MusicRepositoryImpl.kt:422`) は `allowed` から `blocked` を引いた集合を返し、各ページング取得で `allowedParentDirs + applyDirectoryFilter` のフラグを DAO に渡す。

#### Telegram 同期エンハンスメント
`ensureTelegramDownloadSyncObserverStarted` (`MusicRepositoryImpl.kt:167`) は `telegramRepository.songFileUpdated` を購読し、`SyncWorker.incrementalSyncWork()` を発火。これにより Telegram ファイルがローカルへ DL 完了した時点で統一 songs テーブルへ反映される。

#### Favorite / Song ID 文字列⇔Long
- `getFavoriteSongIdsOnce()` は `Set<Long>` を `Set<String>` に変換して返す（`MusicRepositoryImpl.kt:827`）。
- `setFavoriteStatus(songId, isFavorite)` は `songId.toLongOrNull()` を使い、文字列 ID は無視する（`MusicRepositoryImpl.kt:814`）。

#### `getSong(songId)` の二段解決
`MusicRepositoryImpl.kt:847`
- `songId.toLongOrNull()` 成功時は `musicDao.getSongById(id)` を監視。
- 失敗時（Telegram 由来 ID）は `telegramDao.getSongsByIds(listOf(songId)) + getAllChannels()` で `channelTitle` 付きで `Song` を生成。

#### Genre 構築
`getGenres()` (`MusicRepositoryImpl.kt:864`) は `combine(getUniqueGenres, hasUnknownGenre)` を `flow { … }.flatMapLatest { it }` で結合。`buildGenre(name)` (`MusicRepositoryImpl.kt:906`) は `GenreThemeUtils.getGenreThemeColor(id, isDark)` で Material 3 ライクなコンテナ色を生成。

#### `getArtists` のプリフェッチ
`MusicRepositoryImpl.kt:460` で返却直前に `prefetchJob?.cancel()` を行い、新しい emission で前回のプリフェッチをキャンセル。これにより重複起動を防いでいる（連続 sync 時の重複防止）。

#### `getArtistsForSong` のプリフェッチ
`MusicRepositoryImpl.kt:506` の `onEach` 内で `currentSongArtistPrefetchSongId` を切り替え、同一楽曲では Room の再 emit に対してプリフェッチをキャンセルしない。`return@onEach` で抜ける（`MusicRepositoryImpl.kt:521`）。

### 状態保持

| フィールド | 用途 |
|---|---|
| `directoryScanMutex` (`MusicRepositoryImpl.kt:112`) | `getAllUniqueAudioDirectories` の同時呼び出し抑止 |
| `repositoryScope` (`MusicRepositoryImpl.kt:113`) | `SupervisorJob() + Dispatchers.IO` バックグラウンド用 |
| `prefetchJob` (`MusicRepositoryImpl.kt:120`) | 現在の artist image プリフェッチ |
| `currentSongArtistPrefetchJob` / `currentSongArtistPrefetchSongId` (`MusicRepositoryImpl.kt:121-122`) | 現在再生中楽曲に対する artist image プリフェッチ |
| `telegramDownloadSyncObserverStarted` (`MusicRepositoryImpl.kt:123`) | 二重購読抑止フラグ |
| `cachedDirFilter` (`MusicRepositoryImpl.kt:136`) | `StateFlow<CachedDirFilter>` キャッシュ |

---

## `SongRepository` (interface)

`SongRepository.kt:8`

`MusicRepository` の下位に位置する、シンプルな楽曲中心の API。実装は `MediaStoreSongRepository`。

| メソッド | 戻り値 | 目的 |
|---|---|---|
| `getSongs()` | `Flow<List<Song>>` | 全曲（MediaStore 直読み） |
| `getSongsByAlbum(albumId)` | `Flow<List<Song>>` | アルバム楽曲 |
| `getSongsByArtist(artistId)` | `Flow<List<Song>>` | アーティスト楽曲 |
| `searchSongs(query)` | `suspend List<Song>` | 検索単発 |
| `getSongById(songId)` | `Flow<Song?>` | 単曲監視 |
| `getPaginatedSongs()` | `Flow<PagingData<Song>>` | ページング（既定） |
| `getPaginatedSongs(sortOption, storageFilter)` | `Flow<PagingData<Song>>` | ソート指定ページング |
| `getPaginatedFavoriteSongs(sort, filter)` | `Flow<PagingData<Song>>` | お気に入りページング |
| `getFavoriteSongsOnce(filter)` | `suspend List<Song>` | お気に入り単発 |
| `getFavoriteSongCountFlow(filter)` | `Flow<Int>` | お気に入り件数 |

---

## `MediaStoreSongRepository`

`MediaStoreSongRepository.kt:55`

`@Singleton` / `@Inject` で `Context`, `MediaStoreObserver`, `FavoritesDao`, `MusicDao`, `UserPreferencesRepository` を受ける。`SongRepository` を実装する。

### 内部実装メモ

#### `SearchPrefs`
`MediaStoreSongRepository.kt:45` で検索に必要なユーザー設定（許可ディレクトリ、ブロックディレクトリ、アーティスト区切り、最小再生時間など）をまとめて保持。`searchSongs()` 内で `coroutineScope { async { ... } }` で並行取得。

#### `observeSongs` の Flow 構造
`MediaStoreSongRepository.kt:73` は以下 8 つの Flow を `combine` する：
1. `mediaStoreObserver.mediaStoreChanges` (MediaStore 更新通知)
2. `favoritesDao.getFavoriteSongIds()` (お気に入り ID)
3. `userPreferencesRepository.allowedDirectoriesFlow`
4. `userPreferencesRepository.blockedDirectoriesFlow`
5. `userPreferencesRepository.artistDelimitersFlow`
6. `userPreferencesRepository.minSongDurationFlow`
7. `userPreferencesRepository.artistWordDelimitersFlow`
8. `userPreferencesRepository.extractArtistsFromTitleFlow`

#### `fetchSongsFromMediaStore`
`MediaStoreSongRepository.kt:107` で `ContentResolver.query()` を実行。`buildLocalAudioSelection(minDurationMs)` (`utils/`) で `IS_MUSIC`/`DURATION` の基本フィルタを構築し、追加条件があれば AND で連結。`directoryResolver.isBlocked(parent)` で除外。

#### `getSongIdToGenreMap`
`MediaStoreSongRepository.kt:278` で `Genres.EXTERNAL_CONTENT_URI` → `Genres.Members.getContentUri(...)` の 2 段階クエリ。多重ジャンルは後勝ち（コメント `MediaStoreSongRepository.kt:306`）。

#### `getPaginatedSongs`
- 引数なし版 (`MediaStoreSongRepository.kt:373`) は `PagingSource` ベースの `MediaStorePagingSource` を使う。
- sortOption 版 (`MediaStoreSongRepository.kt:466`) は Room の `musicDao.getSongsPaginated(...)` を使い、`PagingConfig(pageSize=50, maxSize=250)` で構築。

#### `computeAllowedDirs`
`MediaStoreSongRepository.kt:447` で `DirectoryFilterUtils.computeAllowedParentDirs(...)` を呼び、`musicDao.getDistinctParentDirectories()` から親ディレクトリ列挙→ブロック差分。

---

## `LyricsRepository` (interface)

`LyricsRepository.kt:7`

| メソッド | 戻り値 | 目的 |
|---|---|---|
| `getStoredLyrics(song)` | `suspend Pair<Lyrics, String>?` | DB / JSON / LRC の永続化された歌詞のみ |
| `getLyrics(song, sourcePreference, forceRefresh)` | `suspend Lyrics?` | 優先度指定で歌詞取得（埋め込み / API / ローカル） |
| `fetchFromRemote(song)` | `suspend Result<Pair<Lyrics, String>>` | LRCLIB から強制取得 |
| `searchRemote(song)` | `suspend Result<Pair<String, List<LyricsSearchResult>>>` | 楽曲メタから検索 |
| `searchRemoteByQuery(title, artist?)` | `suspend Result<Pair<String, List<LyricsSearchResult>>>` | 任意クエリ検索 |
| `updateLyrics(songId, lyricsContent)` | `suspend Unit` | DB / JSON キャッシュ更新 |
| `resetLyrics(songId)` | `suspend Unit` | 単曲削除 |
| `resetAllLyrics()` | `suspend Unit` | 全削除 |
| `clearCache()` | `Unit` | メモリキャッシュのみ消去 |
| `scanAndAssignLocalLrcFiles(songs, onProgress)` | `suspend Int` | `.lrc` ファイルを走査して DB にマージ |

---

## `LyricsRepositoryImpl`

`LyricsRepositoryImpl.kt:115`

### 主要メソッド（実装）

| メソッド | 行 | 目的 |
|---|---|---|
| `getLyrics(song, sourcePreference, forceRefresh)` | 360 | キャッシュ → DB → `fetchFromXxx` の順で解決。`EMBEDDED_FIRST` / `API_FIRST` / `LOCAL_FIRST` を切替 |
| `getStoredLyrics(song)` | 456 | DB / ローカル `.lrc` / ローカル JSON のみ（ネットワークなし） |
| `fetchFromRemote(song)` | 1272 | LRCLIB API 経由の取得 + DB 保存 |
| `searchRemote(song)` | 1373 | 候補一覧 |
| `searchRemoteByQuery(title, artist?)` | 1453 | クエリ直接指定 |
| `updateLyrics(songId, lyricsContent)` | 1502 | DB + JSON disk cache 更新 |
| `resetLyrics(songId)` | 1525 | DB + JSON disk cache 削除 |
| `resetAllLyrics()` | 1544 | 全削除 |
| `scanAndAssignLocalLrcFiles(songs, onProgress)` | 1560 | `.lrc` ファイル走査 + DB 反映 |
| `clearCache()` | 1664 | `LruCache` クリア |

### 内部実装メモ

- **キャッシュ**:
  - `lyricsCache: LruCache<String, Lyrics>(MAX_LYRICS_CACHE_SIZE = 150)` (`LyricsRepositoryImpl.kt:221`)
  - JSON disk cache は `cacheDir/<songId>.json` 形式
- **Rate Limit**:
  - `lastApiCalls: ConcurrentHashMap<String, Long>`, `apiCallCounts: ConcurrentHashMap<String, RateLimitWindow>` (`LyricsRepositoryImpl.kt:224-225`)
  - `MAX_CALLS_PER_MINUTE = 30`, `LRCLIB_MIN_DELAY = 100L` (`LyricsRepositoryImpl.kt:130-131`)
- **検索戦略 (`runSearchStrategiesFast`)**: `LyricsRepositoryImpl.kt:234` で複数戦略（`track+artist`, `track`, `plain` など）を `Channel` 経由で並行実行し、最初にヒットした時点で他をキャンセル。
- **マッチランキング (`rankRemoteLyricsMatches`)**: `titleMatchScore` / `artistMatchScore` / `remoteDurationToleranceSeconds` (`LyricsRepositoryImpl.kt:614-679`) で候補をスコアリング。
- **Netease 特殊パス**: `isNeteaseSong(song)` で分岐し、`AMLLDB_NCM_LYRICS_BASE_URL = "https://amlldb.bikonoo.com/lyrics/ncm-lyrics/"` から word-by-word 歌詞を取得 (`LyricsRepositoryImpl.kt:132`)。
- **リトライ**: `withNetworkRetry(...)` を全 API 呼び出しに適用 (`NETWORK_RETRY_ATTEMPTS = 3`, `NETWORK_RETRY_INITIAL_DELAY_MS = 500L`) (`LyricsRepositoryImpl.kt:133-134`)。
- **マッチング補助データ**: `BRACKETED_QUALIFIER_REGEX`, `FEATURE_QUALIFIER_REGEX`, `TITLE_SEPARATOR_REGEX`, `TIMING_VARIANT_KEYWORDS`, `TITLE_DROP_QUALIFIERS`, `UNKNOWN_ARTISTS`, `ARTIST_CONNECTOR_TOKENS` (`LyricsRepositoryImpl.kt:136-214`) を全て定数化。

---

## `TransitionRepository` (interface)

`TransitionRepository.kt:12`

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `resolveTransitionSettings(playlistId, fromTrackId, toTrackId)` | 21 | `Flow<TransitionResolution>` | 優先度 specific → playlist default → global で解決 |
| `getAllRulesForPlaylist(playlistId)` | 30 | `Flow<List<TransitionRule>>` | プレイリスト全ルール |
| `getPlaylistDefaultRule(playlistId)` | 36 | `Flow<TransitionRule?>` | デフォルトルールのみ |
| `saveRule(rule)` | 43 | `suspend Unit` | upsert（`id=0` で新規） |
| `deleteRule(ruleId)` | 48 | `suspend Unit` | 単一削除 |
| `deletePlaylistDefaultRule(playlistId)` | 53 | `suspend Unit` | デフォルト削除 |
| `getGlobalSettings()` | 58 | `Flow<TransitionSettings>` | DataStore 由来 |
| `saveGlobalSettings(settings)` | 63 | `suspend Unit` | DataStore 保存 |

---

## `TransitionRepositoryImpl`

`TransitionRepositoryImpl.kt:18`

`resolveTransitionSettings` (`TransitionRepositoryImpl.kt:25`) は `transitionDao.getSpecificRule` をまず取得し、`null` なら `getPlaylistDefaultRule`、それも `null` なら `userPreferences.globalTransitionSettingsFlow` にフォールバックする 3 段 `flatMapLatest` チェーン。

---

## `ArtistImageRepository`

`ArtistImageRepository.kt:37`

Deezer API によるアーティスト画像 URL の取得 + LRU キャッシュ + ユーザー設定画像（`artist_art_<id>.jpg` を `filesDir` に保存）。

### 公開メソッド

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `getArtistImageUrl(artistName, artistId)` | 86 | `suspend String?` | メモリキャッシュ → DB → Deezer の順で取得 |
| `prefetchArtistImages(artists)` | 127 | `suspend Unit` | バッチプリフェッチ（並列度 3、`chunked(12)` で OOM 防止） |
| `clearCache()` | 258 | `Unit` | LRU + failedFetches 消去 |
| `getEffectiveArtistImageUrl(artistId, artistName)` | 268 | `suspend String?` | ユーザー設定画像優先、なければ Deezer |
| `setCustomArtistImage(context, artistId, sourceUri)` | 286 | `suspend String?` | 画像取り込み + JPEG 保存 + DB 更新 |
| `clearCustomArtistImage(context, artistId)` | 399 | `suspend Unit` | ファイル削除 + DB クリア |

### 内部実装メモ

- **二重取得抑止**: `fetchMutex` + `pendingFetches: MutableSet<String>` (`ArtistImageRepository.kt:71-72`) で同一アーティストへの並行 API を防ぐ。
- **失敗キャッシュ**: `failedFetches: MutableSet<String>` (`ArtistImageRepository.kt:78`) で再試行を抑制（`SocketTimeoutException` 以外）。
- **高解像度アップグレード**: `upgradeToHighResDeezerUrl` (`ArtistImageRepository.kt:417`) で `/<dim>x<dim>-` を `/1000x1000-` に置換（`dzcdn.net/images/artist` のみ）。
- **画像デコード**: `decodeCustomArtistBitmap` (`ArtistImageRepository.kt:321`) で MIME / サイズ / 寸法 / ピクセル数を多段検証してから `BitmapFactory.decodeStream`（OOM も catch）。
- **圧縮**: `scaleBitmapIfNeeded` (`ArtistImageRepository.kt:381`) で長辺 ≤ 2048 px まで縮小、JPEG quality 90 で保存。
- **診断**: `prefetchArtistImages` 開始 / 終了を `AdvancedPerformanceDiagnostics.recordEventIfEnabled(WORKER, "artist_image_prefetch_*")` で記録。

---

## `FolderTreeBuilder`

`FolderTreeBuilder.kt:18`

`List<FolderSongRow>` から階層的な `List<MusicFolder>` を構築するユーティリティ（状態を持たない `@Singleton`）。

### 公開メソッド

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `buildFolderTree(folderSongs, allowedDirs, blockedDirs, isFolderFilterActive, folderSource, context)` | 20 | `List<MusicFolder>` | フィルタ + ストレージ種別判定 + ツリー化の統合エントリ |

`buildFolderTreeForRoots` (`FolderTreeBuilder.kt:77`) は internal。

### 内部実装メモ

- **ディレクトリフィルタ**: `DirectoryRuleResolver(allowedDirs, blockedDirs)` で `isBlocked(parentPath)` を判定。
- **ストレージルート推定**: `StorageUtils.getAvailableStorages(context)` から `INTERNAL` と `removable` を取得。
- **`inferRemovableStorageRootFromPath`**: 既知のルートに含まれない親ディレクトリは `parts[0] == "storage" && parts[1] == "emulated"` 等のパターンから推測（`FolderTreeBuilder.kt:164`）。
- **`TempFolder`**: songs と subFolderPaths の二段 Map を組み立て、`buildImmutableFolder` で再帰的に `MusicFolder` に変換（`FolderTreeBuilder.kt:275-280`）。
- **songs は trackNumber → title でソート**: `buildImmutableFolder` 内 (`FolderTreeBuilder.kt:220-224`)。
- **`LocalArtworkUri`**: 共有アートワーク ContentProvider の URI 解決 (`LocalArtworkUri.isLocalArtworkUri / looksLikeVolatileArtworkUri / buildSongUri`)。

---

## 内部実装ノート (Repository 横断)

### Flow とディレクトリフィルタの合成パターン

```
combine(allowedDirectoriesFlow, blockedDirectoriesFlow) { a, b -> a to b }
    .flatMapLatest { (allowed, blocked) ->
        flow { /* sync 計算 */ emit(Pager { ... }.flow) }
    }
.map { pagingData -> pagingData.map { entity -> entity.toModel() } }
.flowOn(Dispatchers.IO)
```

すべてのページング Flow は `combine → flatMapLatest → emit Pager → map → flowOn(IO)` のパターンに従う。これにより設定変更時に Flow が自動で再起動する。

### `DataStore.first()` vs `Flow.first()`

- **設定値の単発取得**: 必ず `Flow.first()` を使い、`DataStore.data.first()` でなく `userPreferencesRepository.xxxFlow.first()` を呼ぶ。
- **Mutex** (`directoryScanMutex`, `editMutex`) は読み書き競合を防ぐための最終防衛線として実装。

### `Lazy<>` による循環参照回避

`MusicRepositoryImpl` が `Lazy<TelegramCacheManager>` / `Lazy<TelegramRepository>` を受ける (`MusicRepositoryImpl.kt:97-98`)。`@Inject` 直注入だと循環依存になるため、`Provider<>` 経由の遅延取得。

### Search 履歴の重複排除

`addSearchHistoryItem` (`MusicRepositoryImpl.kt:647`) は `deleteByQuery(query)` → `insert(new)` の順で実行し、最新クエリを先頭に保つ。

---

## 関連ファイル

- 上位: `presentation/.../viewmodel/`, `service/MusicService.kt`, `worker/SyncWorker.kt`
- 下位: `data/database/MusicDao.kt`, `FavoritesDao.kt`, `LyricsDao.kt`, `SearchHistoryDao.kt`, `TransitionDao.kt`, `TelegramDao.kt`, `LocalPlaylistDao.kt`
- 補助: `data/preferences/UserPreferencesRepository.kt`, `data/observer/MediaStoreObserver.kt`, `data/network/deezer/DeezerApiService.kt`, `data/network/lyrics/LrcLibApiService.kt`, `utils/AlbumArtUtils.kt`, `utils/DirectoryFilterUtils.kt`, `utils/LocalArtworkUri.kt`
- 詳細: [`../01-data-foundation/database.md`](../01-data-foundation/database.md), [`preferences.md`](./preferences.md), [`backup-system.md`](./backup-system.md), [`media-processing.md`](./media-processing.md)