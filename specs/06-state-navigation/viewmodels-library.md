# viewmodels-library.md

> ライブラリ画面系 StateHolder / ViewModel の詳細仕様。`LibraryStateHolder` (615 行) / `LibraryTabsStateHolder` / `SearchStateHolder` / `AlbumDetailViewModel` / `ArtistDetailViewModel` (305 行) / `GenreDetailViewModel` (317 行) を扱う。

---

## LibraryStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/LibraryStateHolder.kt` (615 行)
**アノテーション**: `@Singleton`
**役割**: 楽曲 / アルバム / アーティスト / フォルダ / ジャンルの全データを保持し、Paging フロー、並び替え、ストレージフィルタ、メモリトリム、ジャンル別カラーテーマ生成を統合管理。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `allSongs` | `StateFlow<ImmutableList<Song>>` | 全楽曲 (永続化) |
| `allSongsById` | `StateFlow<Map<String, Song>>` | ID 逆引き |
| `albums` | `StateFlow<ImmutableList<Album>>` | アルバム (並び替え後) |
| `artists` | `StateFlow<ImmutableList<Artist>>` | アーティスト |
| `musicFolders` | `StateFlow<ImmutableList<MusicFolder>>` | 音楽フォルダツリー |
| `isLoadingLibrary` | `StateFlow<Boolean>` | ライブラリ読込中 |
| `isLoadingCategories` | `StateFlow<Boolean>` | アルバム/アーティスト読込中 |
| `currentSongSortOption` | `StateFlow<SortOption>` | 楽曲並び順 |
| `currentAlbumSortOption` | `StateFlow<SortOption>` | アルバム並び順 |
| `currentArtistSortOption` | `StateFlow<SortOption>` | アーティスト並び順 |
| `currentFolderSortOption` | `StateFlow<SortOption>` | フォルダ並び順 |
| `currentFavoriteSortOption` | `StateFlow<SortOption>` | お気に入り並び順 |
| `currentStorageFilter` | `StateFlow<StorageFilter>` | ALL / OFFLINE / CLOUD |
| `genres` | `Flow<ImmutableList<Genre>>` | ジャンル一覧 (各 Genre に light/dark color 付き) |
| `songsPagingFlow` | `Flow<PagingData<Song>>` | 楽曲ページング (filter 連動) |
| `albumsPagingFlow` | `Flow<PagingData<Album>>` | アルバムページング |
| `artistsPagingFlow` | `Flow<PagingData<Artist>>` | アーティストページング |
| `favoritesPagingFlow` | `Flow<PagingData<Song>>` | お気に入りページング |
| `favoriteSongCountFlow` | `Flow<Int>` | お気に入り件数 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(scope: CoroutineScope)` | ソート設定を `UserPreferencesRepository` から復元 |
| `fun onCleared()` | Job キャンセル |
| `fun startObservingLibraryData()` | 4 並行 Job (songs / albums / artists / folders) で Repository を監視開始 |
| `fun loadSongsFromRepository()` / `loadAlbumsFromRepository()` / `loadArtistsFromRepository()` / `loadFoldersFromRepository()` | 強制再読込 |
| `fun loadSongsIfNeeded()` / `loadAlbumsIfNeeded()` / `loadArtistsIfNeeded()` | 初回のみ |
| `fun sortSongs(option: SortOption, persist: Boolean = true)` | 楽曲並び替え + 永続化 |
| `fun sortAlbums(option, persist)` | アルバム並び替え |
| `fun sortArtists(option, persist)` | アーティスト並び替え |
| `fun sortFolders(option, persist)` | フォルダ並び替え |
| `fun sortFavoriteSongs(option, persist)` | お気に入り並び替え |
| `fun sortAlbumsList(...)` / `sortArtistsList(...)` / `sortFoldersList(...)` (private) | 純粋関数 (テスト可能) |
| `fun updateSong(updatedSong: Song)` | 単曲更新 (metadata 編集後) |
| `fun removeSong(songId: String)` | 単曲削除 (ライブラリ / キュー) |
| `fun setStorageFilter(filter: StorageFilter)` | ストレージフィルタ |
| `fun trimMemory(level: Int)` | ComponentCallbacks2 経由のメモリ解放 |
| `fun restoreAfterTrimIfNeeded()` | メモリ復元 |

### 内部実装メモ

- **`effectiveStorageFilter`**: `currentStorageFilter` をベースに、`hideLocalMedia` フラグを反映 (i.e. `CLOUD` に強制)。Folders は `ENABLE_FOLDERS_STORAGE_FILTER = false` のため常に `ALL` 相当。
- **Paging flow 構成**:
  - `songsPagingFlow` = `Pager(PagingConfig(pageSize = 30)) { musicRepository.songsPagingSource(sort, filter) }.flow`
  - `albumsPagingFlow` / `artistsPagingFlow` / `favoritesPagingFlow` も同様。
- **Genres カラーテーマ**: `GenreThemeUtils.getGenreThemeColor(seed.id, isDark = false/true)` を各 Genre に適用し、`StateFlow<Pair<Int, Int>>` (light, dark) を保持。
- **メモリトリム**: `level >= TRIM_MEMORY_UI_HIDDEN` で `needsReloadAfterTrim = true` を立て、`onStart` で `restoreAfterTrimIfNeeded` を呼ぶ。
- **ソート永続化**: `persist = true` なら `userPreferencesRepository.setSortOption(...)` で `DataStore` に保存。次回起動時に復元。
- **起動時 Job 構築**: `startObservingLibraryData` で 4 つの `Job` を開始 (`songsJob`, `albumsJob`, `artistsJob`, `foldersJob`)。`onCleared` で全キャンセル。
- **`collectLatest`**: `_allSongs` 更新時に古いソート処理が走らないよう `mapLatest` / `collectLatest` でキャンセル可能。
- **並列ソート**: `withContext(Dispatchers.Default) { ... }` で `songs.toImmutableList()` / `songs.associateBy { it.id }` を実行 (CPU バウンド)。
- **Sort key 永続化キー**: `songsSortOptionFlow`, `albumsSortOptionFlow`, `artistsSortOptionFlow`, `foldersSortOptionFlow`, `likedSongsSortOptionFlow` の 5 種類。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/repository/MusicRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/model/SortOption.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/model/StorageFilter.kt`
- `app/src/main/java/com/theveloper/pixelplay/ui/theme/GenreThemeUtils.kt`
- 画面 ViewModel ラッパ: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/LibraryViewModel.kt`

---

## LibraryTabsStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/LibraryTabsStateHolder.kt` (72 行)
**アノテーション**: `@ViewModelScoped` (実際は `@Inject` のみ)
**役割**: ライブラリタブ (SONGS / ALBUMS / ARTISTS / FAVORITES / FOLDERS / GENRES) の表示制御。並び替えシートの表示状態管理と、タブ ID 解決を担当。

### 注入される依存

なし (純粋関数 + Repository 経由のルックアップ)

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun showSortingSheet(isSortingSheetVisible: MutableStateFlow<Boolean>)` | 並び替えシート表示 |
| `fun hideSortingSheet(isSortingSheetVisible: MutableStateFlow<Boolean>)` | 並び替えシート非表示 |
| `fun onLibraryTabSelected(tabIndex: Int, libraryTabs: List<String>, setCurrentTab: (LibraryTabId) -> Unit, setLastTabIndex: (Int) -> Unit, lastTabIndexFlow: StateFlow<Int>)` | タブ選択 + インデックス記憶 |

### 内部実装メモ

- **タブ ID 解決**: `libraryTabs[tabIndex]` を `LibraryTabId.toLibraryTabIdOrNull()` でパース。失敗時は `LibraryTabId.SONGS` にフォールバック。
- **同一タブ再タップ**: `tabIndex == lastTabIndexFlow.value` の場合は何もしない (= スクロールトップ動作は Composable 側で実装)。
- **`setLastTabIndex`**: 選択されたタブ位置を `PlayerViewModel` の `lastLibraryTabIndexFlow` へ保存。
- **`MutableStateFlow` を受け取る設計**: `PlayerViewModel` 内の `_isSortingSheetVisible` への参照を引数で渡し、外部から操作可能にする疎結合パターン。

### 関連ファイル

- データモデル: `app/src/main/java/com/theveloper/pixelplay/data/model/LibraryTabId.kt` (詳細は `../05-presentation-ui/library-model-stats-utils.md`)
- 呼び出し元: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlayerViewModel.kt` の `onLibraryTabSelected` 内部
- Repository 設定: `app/src/main/java/com/theveloper/pixelplay/data/preferences/UserPreferencesRepository.kt` (`libraryTabsOrderFlow`)

---

## SearchStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/SearchStateHolder.kt` (206 行)
**アノテーション**: `@Singleton`
**役割**: 検索デバウンス + 履歴管理 + フィルタ切替。`SEARCH_DEBOUNCE_MS = 300L` で連続入力時に古いリクエストを無効化。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `searchResults` | `StateFlow<ImmutableList<SearchResultItem>>` | 検索結果 (楽曲/アルバム/アーティスト/プレイリスト混在) |
| `selectedSearchFilter` | `StateFlow<SearchFilterType>` | ALL / SONGS / ALBUMS / ARTISTS / PLAYLISTS |
| `searchHistory` | `StateFlow<ImmutableList<SearchHistoryItem>>` | 検索履歴 (最大 15 件) |
| `searchRequests` | `MutableSharedFlow<SearchRequest>` (private) | リクエストストリーム |
| `latestSearchRequestId` | `AtomicLong` (private) | 最新リクエスト ID |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(scope: CoroutineScope)` | リクエスト監視開始 + 履歴復元 |
| `fun updateSearchFilter(filterType: SearchFilterType)` | フィルタ切替 (即時再フィルタ) |
| `fun loadSearchHistory(limit: Int = 15)` | DB から履歴復元 |
| `fun onSearchQuerySubmitted(query: String)` | 検索確定 (履歴追加) |
| `fun performSearch(query: String)` | 即時検索 (デバウンス無し) |
| `fun deleteSearchHistoryItem(query: String)` | 履歴から 1 件削除 |
| `fun clearSearchHistory()` | 全消去 |
| `fun onCleared()` | Job キャンセル |

### 内部実装メモ

- **検索パイプライン**:
  1. `PlayerViewModel` から `searchQuery` 変化時、`SearchStateHolder` 内の `MutableSharedFlow<SearchRequest>` へ emit
  2. `observeSearchRequests` 内で `latestSearchRequestId.incrementAndGet()` を発行し、`requestId == latest` なら結果採用
  3. `debounce(300L)` (実装内コメント参照) でタイピング中の不要な検索を回避
  4. `musicRepository.search(query, filter)` を実行
  5. 結果を `SearchResultItem` に変換し、`SearchHistoryItem` 優先度順にソート
- **`FlowPreview`** 注釈付き: `debounce` 使用のため。
- **履歴ソート**: 履歴の最新マッチを優先するため、`sortedWith(compareByDescending { historyItem.indexOf(query) })`。
- **`@FlowPreview`**: `debounce` の使用に必須 (実験的 API)。
- **検索ソース**: `MusicRepository` 内に SQL `LIKE` クエリを構築し、`@Query` で DB から直接フェッチ (Room + Paging ではなく単純な Flow 返却)。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/repository/MusicRepository.kt` (`search` メソッド)
- `app/src/main/java/com/theveloper/pixelplay/data/model/SearchResultItem.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/model/SearchFilterType.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/SearchScreen.kt`

---

## AlbumDetailViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/AlbumDetailViewModel.kt` (110 行)
**アノテーション**: `@HiltViewModel`
**役割**: アルバム詳細画面。`SavedStateHandle["albumId"]` からアルバム ID を受け取り、アルバム情報 + 収録曲を読み込み。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |
| State | `savedStateHandle: SavedStateHandle` |

### `AlbumDetailUiState` (data class)

| フィールド | 型 | 目的 |
|----------|---|------|
| `album` | `Album?` | アルバム情報 |
| `songs` | `List<Song>` | 収録曲 (Track# 順) |
| `isLoading` | `Boolean` | 読込中 |
| `error` | `String?` | エラー |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<AlbumDetailUiState>` | 画面用統合状態 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun update(songs: List<Song>)` | 楽曲リスト更新 (drag-to-reorder 用) |

### 内部実装メモ

- **データ取得**: `init { loadAlbumData(albumId) }` で `combine(albumDetailsFlow, albumSongsFlow)` を `catch` で囲み、エラー時に `uiState.error` を設定。
- **DB クエリ**: `musicRepository.getAlbumById(id)` + `getSongsForAlbum(id)`。
- **`@HiltViewModel` + `SavedStateHandle`**: Navigation の `albumId` 引数 (`Screen.AlbumDetail.createRoute(albumId)`) を自動取得。
- **`albumId.toLongOrNull()`**: 文字列 ID を Long に変換。変換失敗時は `uiState` が `album = null` のまま。

### 関連ファイル

- データレイヤー: `app/src/main/java/com/theveloper/pixelplay/data/database/MusicDao.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/AlbumDetailScreen.kt`
- Navigation: `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/Screen.kt` (`AlbumDetail.createRoute`)

---

## ArtistDetailViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ArtistDetailViewModel.kt` (305 行)
**アノテーション**: `@HiltViewModel`
**役割**: アーティスト詳細画面。アルバム別グルーピング + カスタム画像設定 + カラースキーム生成。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |
| Repository | `artistImageRepository: ArtistImageRepository` |
| StateHolder | `themeStateHolder: ThemeStateHolder` (公開プロパティ) |
| State | `savedStateHandle: SavedStateHandle` |

### `ArtistDetailUiState` (data class)

| フィールド | 型 | 目的 |
|----------|---|------|
| `artist` | `Artist?` | アーティスト情報 |
| `songs` | `List<Song>` | 全楽曲 |
| `albumSections` | `List<ArtistAlbumSection>` | アルバム別セクション |
| `effectiveImageUrl` | `String?` | カスタム / Deezer / デフォルト画像 |
| `isLoading` | `Boolean` | 読込中 |
| `error` | `String?` | エラー |

### `ArtistAlbumSection` (data class)

| フィールド | 型 | 目的 |
|----------|---|------|
| `albumId` | `Long` | アルバム ID |
| `title` | `String` | アルバムタイトル |
| `year` | `Int?` | 発売年 |
| `albumArtUriString` | `String?` | アルバムアート |
| `songs` | `List<Song>` | 収録曲 |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<ArtistDetailUiState>` | 画面状態 |
| `artistColorScheme` | `StateFlow<ColorSchemePair?>` | 画像から生成されたカラースキーム |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun setCustomImage(sourceUri: Uri)` | カスタム画像設定 (内部保存 + バスター付き URL で再読込) |
| `fun clearCustomImage()` | Deezer 画像に戻す |
| `fun removeSongFromAlbumSection(songId: String)` | 1 曲削除 (UI から即時除去) |
| `private fun buildAlbumSections(songs: List<Song>): List<ArtistAlbumSection>` | アルバム/年でソートされたセクション生成 |

### 内部実装メモ

- **画像ソース解決の優先順位**:
  1. カスタム (`ArtistImageRepository.setCustomArtistImage` の結果)
  2. Deezer (`getArtistImageUrl(artist.name, artist.id)`)
  3. デフォルト (なし)
- **`effectiveImageUrl`**: 上記解決 + キャッシュバスター (`?t=${currentTimeMillis()}`) を付与。
- **アルバムグルーピング**: `discNumber → trackNumber → title` の複合キー。`albumYear` は `albumSongs.mapNotNull { it.year.takeIf { it > 0 } }.maxOrNull()`。
- **カラースキーム**: `themeStateHolder.getAlbumColorSchemeFlow(uriString)` を使い、画像から ColorScheme を自動生成 → ヘッダー背景に適用。
- **画像再生成**: `setCustomImage` 後 `ThemeStateHolder.forceRegenerateColorScheme(uri, regenerateAllStyles = false)` で再生成。
- **Deezer フォールバック**: `clearCustomImage` 時に Deezer URL が null なら `effectiveImageUrl = null`、色スキームは `null` のまま。
- **`removeSongFromAlbumSection`**: UI からの楽観的削除。`currentState.albumSections.map { ... }` で `songs.filterNot { it.id == songId }` を実施。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/repository/ArtistImageRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ThemeStateHolder.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/ArtistDetailScreen.kt`

---

## GenreDetailViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/GenreDetailViewModel.kt` (317 行)
**アノテーション**: `@HiltViewModel`
**役割**: ジャンル詳細画面。アーティスト/アルバム別のセカンダリリスト生成と、3 つのソートモード。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |
| State | `savedStateHandle: SavedStateHandle` |

### `SortOption` (ローカル enum)

| 値 | 用途 |
|----|------|
| `ARTIST` | アーティスト → アルバム → Track |
| `ALBUM` | アルバム → Track |
| `TITLE` | タイトル |

### `SectionData` (sealed class)

| 派生 | 用途 |
|------|------|
| `ArtistSection(id, artistName, albums)` | アーティスト別アルバム集 |
| `AlbumSection(id, album: AlbumData)` | アルバム単体 |
| `FlatList(songs)` | 曲のみ (TITLE モード時) |

### `GenreDetailListItem` (sealed class — LazyColumn 用)

| 派生 | 用途 |
|------|------|
| `ArtistHeader(key, artistName, imageUrl?)` | アーティスト見出し |
| `AlbumHeader(key, album, useArtistStyle)` | アルバム見出し |
| `SongItem(key, song, isFirstInAlbum, isLastInAlbum, isLastAlbumInSection, useArtistStyle)` | 曲行 |
| `Spacer(key, heightDp, useSurfaceBackground)` | スペーサー |
| `Divider(key)` | 区切り |

### `GenreDetailUiState` (data class)

| フィールド | 型 | 目的 |
|----------|---|------|
| `genre` | `Genre?` | ジャンル情報 |
| `songs` | `List<Song>` | ジャンル内全曲 |
| `sortedSongs` | `List<Song>` | TITLE モード時のソート結果 |
| `displaySections` | `List<SectionData>` | セクション |
| `flattenedItems` | `List<GenreDetailListItem>` | LazyColumn 用 |
| `sortOption` | `SortOption` | 現在の並び順 |
| `isLoadingGenreName` | `Boolean` | ジャンル名解決中 |
| `isLoadingSongs` | `Boolean` | 楽曲読込中 |
| `error` | `String?` | エラー |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<GenreDetailUiState>` | 画面状態 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun updateSortOption(newSort: SortOption)` | ソート切替 + sections / flattened 再計算 |
| `private fun flattenSections(sections, artistMap): List<GenreDetailListItem>` | LazyColumn 用 flat list |
| `private fun buildDisplaySections(songs, sort): List<SectionData>` | グループ化 |

### 内部実装メモ

- **ジャンル ID 復元**: `java.net.URLDecoder.decode(genreId, "UTF-8")` でデコード (URL-safe 名前のデコード)。
- **3 段並列取得**:
  1. `musicRepository.getGenres().first()` で全 Genre
  2. 該当 Genre を ID マッチで発見
  3. `musicRepository.getMusicByGenre(genre.name).first()` で楽曲
  4. `musicRepository.getArtists().first()` でアーティスト名マップ
- **アーティスト名マップ**: アーティスト ID → 表示名 (プロフィール画像 URL も)。
- **グルーピング**: ARTIST モード時 `songs.groupBy { artist ?: "Unknown Artist" }` → アルバムで再グルーピング → Track# ソート。
- **LazyColumn 最適化**: `flattenedItems` を `LazyColumn` に渡すことで Compose の `key` ベースの差分更新を活用。
- **`useArtistStyle`**: アーティストヘッダー配下の曲にアーティスト色を適用するフラグ。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/model/Genre.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/repository/MusicRepository.kt` (`getMusicByGenre`, `getArtists`, `getGenres`)
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/GenreDetailScreen.kt`
- Navigation: `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/Screen.kt` (`GenreDetail.createRoute`)
