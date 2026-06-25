# viewmodels-core.md

> コアとなる `PlayerViewModel` (3118 行) と軽量 ViewModel (`MainViewModel`, `LibraryViewModel`) の詳細仕様。

---

## PlayerViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlayerViewModel.kt` (3118 行)
**アノテーション**: `@HiltViewModel`
**役割**: アプリ全体の再生 / ライブラリ / 設定 / 通知を統括する単一のオーケストレータ ViewModel。29 個の StateHolder / Repository を内包し、Compose 側へ `StateFlow` / `SharedFlow` として公開する。

### 注入される依存 (29 個)

| 種類 | フィールド | 役割 |
|------|----------|------|
| Repository | `musicRepository: MusicRepository` | 楽曲 / プレイリスト / 同期クエリ |
| Preferences | `userPreferencesRepository: UserPreferencesRepository` | ユーザー設定 (全般) |
| Preferences | `aiPreferencesRepository: AiPreferencesRepository` | AI プロバイダ API キー |
| Preferences | `themePreferencesRepository: ThemePreferencesRepository` | カラーパレット / プレイヤー色テーマ |
| Worker | `syncManager: SyncManager` | MediaStore 同期制御 |
| Engine | `dualPlayerEngine: DualPlayerEngine` | Master/Preview ExoPlayer |
| DI (Lazy) | `telegramCacheManagerProvider: Lazy<TelegramCacheManager>` | Telegram キャッシュ (遅延取得) |
| Tracker | `listeningStatsTracker: ListeningStatsTracker` | 再生履歴 / 統計 |
| StateHolder | `dailyMixStateHolder: DailyMixStateHolder` | DailyMix の永続化 |
| StateHolder | `lyricsStateHolder: LyricsStateHolder` | 歌詞取得 / 同期オフセット |
| StateHolder | `castStateHolder: CastStateHolder` | MediaRoute / CastSession |
| StateHolder | `castRouteStateHolder: CastRouteStateHolder` | ルート切替の安全制御 |
| StateHolder | `queueStateHolder: QueueStateHolder` | シャッフル元キューの保持 |
| StateHolder | `queueUndoStateHolder: QueueUndoStateHolder` | キュー削除取り消し |
| StateHolder | `playlistDismissUndoStateHolder: PlaylistDismissUndoStateHolder` | プレイリスト閉じ取り消し |
| StateHolder | `playbackStateHolder: PlaybackStateHolder` | StablePlayerState / Progress |
| StateHolder | `connectivityStateHolder: ConnectivityStateHolder` | Wi-Fi / Bluetooth 状態 |
| StateHolder | `sleepTimerStateHolder: SleepTimerStateHolder` | スリープタイマー / EOT |
| StateHolder | `searchStateHolder: SearchStateHolder` | 検索デバウンス / 履歴 |
| StateHolder | `aiStateHolder: AiStateHolder` | AI プレイリスト生成 |
| StateHolder | `libraryStateHolder: LibraryStateHolder` | 楽曲 / アルバム / アーティスト一覧 |
| StateHolder | `folderNavigationStateHolder: FolderNavigationStateHolder` | フォルダ階層ナビゲーション |
| StateHolder | `libraryTabsStateHolder: LibraryTabsStateHolder` | ライブラリタブ切替 |
| StateHolder | `castTransferStateHolder: CastTransferStateHolder` | キャストへのキュー転送 |
| StateHolder | `metadataEditStateHolder: MetadataEditStateHolder` | 楽曲メタデータ書込 |
| StateHolder | `songRemovalStateHolder: SongRemovalStateHolder` | 楽曲削除 (物理/論理) |
| StateHolder | `themeStateHolder: ThemeStateHolder` | アルバムアート由来 ColorScheme |
| StateHolder | `multiSelectionStateHolder: MultiSelectionStateHolder` | 複数選択モード |
| StateHolder | `playlistSelectionStateHolder: PlaylistSelectionStateHolder` | プレイリスト選択 |
| StateHolder | `playbackDispatchStateHolder: PlaybackDispatchStateHolder` | 再生ディスパッチ |
| StateHolder | `mediaControllerSyncStateHolder: MediaControllerSyncStateHolder` | MediaController 同期 |
| Session | `sessionToken: SessionToken` | MediaSession 接続用 |
| DI (Lazy) | `mediaControllerFactory: MediaControllerFactory` | MediaController 取得 (Lazy) |

### 主要な StateFlow / SharedFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `playerUiState` | `StateFlow<PlayerUiState>` | UI 用の複合状態 (queue, 検索, ライブラリ, AI シート, 並び替え, フォルダ, undo, etc.) |
| `queueFlow` | `StateFlow<ImmutableList<Song>>` | 現在の再生キュー (alias of `_playerUiState.map { it.currentPlaybackQueue }`) |
| `stablePlayerState` | `StateFlow<StablePlayerState>` | 再生状態の安定スナップショット (current song, isPlaying, repeat, shuffle, lyrics) |
| `currentPlaybackPosition` | `StateFlow<Long>` | 現在の再生位置 ms (PlayBackStateHolder 経由) |
| `playbackHistory` | `StateFlow<List<PlaybackHistoryEntry>>` | 再生履歴 (ListeningStatsTracker 経由) |
| `albumArtPaletteStyle` | `StateFlow<AlbumArtPaletteStyle>` | アルバムアートパレットスタイル |
| `castRoutes` / `selectedRoute` | `StateFlow<...>` | CastStateHolder から |
| `isRemotePlaybackActive` / `isCastConnecting` / `remotePosition` | `StateFlow<...>` | Cast 状態 |
| `isWifiEnabled` / `isWifiRadioOn` / `wifiName` | `StateFlow<...>` | ネットワーク |
| `isBluetoothEnabled` / `bluetoothName` / `bluetoothAudioDeviceStates` | `StateFlow<...>` | Bluetooth |
| `trackVolume` | `StateFlow<Float>` | トラックボリューム (UI 調整) |
| `favoriteSongIds` | `StateFlow<Set<String>>` | お気に入り ID セット |
| `isCurrentSongFavorite` | `StateFlow<Boolean>` | current song がお気に入りか |
| `allSongsFlow` / `albumsFlow` / `artistsFlow` | `StateFlow<ImmutableList<...>>` | LibraryStateHolder 経由 |
| `dailyMixSongs` / `yourMixSongs` | `StateFlow<ImmutableList<Song>>` | DailyMix |
| `paginatedSongs` / `favoritesPagingFlow` | `Flow<PagingData<Song>>` | Paging |
| `sheetState` | `StateFlow<PlayerSheetState>` | EXPANDED / COLLAPSED / HIDDEN |
| `isQueueSheetVisible` / `isCastSheetVisible` | `StateFlow<Boolean>` | BottomSheet 可視性 |
| `selectedSongForInfo` | `StateFlow<Song?>` | 楽曲情報シート選択中 |
| `searchQuery` | `mutableStateOf<String>` (非 StateFlow) | 検索バー入力 |
| `toastEvents` | `SharedFlow<String>` | トースト発行 |
| `writePermissionRequest` | `SharedFlow<IntentSender>` | メタデータ書込時の権限要求 |
| `deletePermissionRequest` | `SharedFlow<IntentSender>` | 削除時の権限要求 |
| `albumNavigationRequests` / `artistNavigationRequests` | `SharedFlow<Long>` | ナビジャンプ要求 |
| `searchNavDoubleTapEvents` | `SharedFlow<Unit>` | 検索アイコン 2 連打 |
| `scrollToIndexEvent` | `SharedFlow<Int>` | スクロール先要求 |
| `showNoInternetDialog` | `SharedFlow<Unit>` | オフライン再生拒否 |
| `lyricsSearchUiState` | `StateFlow<LyricsSearchUiState>` | 歌詞検索 UI 状態 |
| `currentSongLyricsSyncOffset` | `StateFlow<Int>` | 現在の曲歌詞オフセット |
| `homeMixPreviewSongs` | `StateFlow<ImmutableList<Song>>` | ホーム画面用プレビュー |
| `paletteRegenerationTargets` | `StateFlow<List<Song>>` | パレット再生成対象 |
| `currentSongArtists` | `StateFlow<List<Artist>>` | 現在の曲のアーティスト |
| `navBarCornerRadius` / `navBarStyle` / `navBarCompactMode` | `StateFlow<...>` | UI 設定 |
| `playerConfigSlice` | `StateFlow<PlayerConfigSlice>` | プレイヤー設定 (合成) |
| `fullPlayerSlice` | `StateFlow<FullPlayerSlice>` | フルプレイヤー UI 用 (合成) |

### 主要 public 関数

#### 再生ディスパッチ (PlaybackDispatchStateHolder へ委譲)

| シグネチャ | 目的 | 委譲先 |
|-----------|------|--------|
| `fun showAndPlaySongFromLibrary(songs: List<Song>, startIndex: Int, queueName: String)` | ライブラリ全体から再生 | `playbackDispatchStateHolder.showAndPlaySongFromLibrary` |
| `fun showAndPlaySongFromFavorites(...)` | お気に入りから再生 | `playbackDispatchStateHolder.showAndPlaySongFromFavorites` |
| `fun showAndPlaySong(song: Song, ...)` | 1 曲再生 (リモート時はキャスト転送) | `playbackDispatchStateHolder.showAndPlaySong` |
| `fun playAlbum(album: Album)` | アルバム全曲再生 | `playbackDispatchStateHolder.playAlbum` |
| `fun playArtist(artist: Artist)` | アーティスト全曲再生 | `playbackDispatchStateHolder.playArtist` |
| `fun shuffleAllSongs(queueName: String)` | 全曲シャッフル | `playbackDispatchStateHolder.shuffleAll` |
| `fun shuffleFavoriteSongs()` | お気に入りシャッフル | `playbackDispatchStateHolder.shuffleFavorites` |
| `fun shuffleRandomAlbum()` / `shuffleRandomArtist()` | ランダムアルバム / アーティスト | 委譲 |
| `fun playRandomSong()` | 1 曲ランダム | `playbackDispatchStateHolder.playRandom` |
| `fun loadAndPlaySong(song: Song)` | External URI ではない 1 曲読込 | `playbackDispatchStateHolder.loadAndPlaySong` |
| `fun addSongToQueue(song: Song)` | キュー末尾追加 | `playbackDispatchStateHolder.addSongToQueue` |
| `fun addSongNextToQueue(song: Song)` | 次の曲として追加 | `playbackDispatchStateHolder.addSongNextToQueue` |
| `fun playPause()` / `fun nextSong()` / `fun previousSong()` / `fun seekTo(pos: Long)` | 基本操作 | `playbackStateHolder.*` |
| `fun toggleShuffle()` | シャッフル ON/OFF | `playbackStateHolder.toggleShuffle` |
| `fun cycleRepeatMode()` | リピートモード切替 | `playbackStateHolder.cycleRepeatMode` |

#### キュー編集

| シグネチャ | 目的 |
|-----------|------|
| `fun removeSongFromQueue(songId: String)` | キューから 1 曲削除 (取り消し可能) |
| `fun undoRemoveSongFromQueue()` | 直前の削除を取り消す |
| `fun hideQueueItemUndoBar()` | Undo バーを閉じる |
| `fun reorderQueueItem(fromIndex: Int, toIndex: Int)` | 並び替え |
| `fun moveQueueItemToPosition(songId: String, targetIndex: Int)` | 指定位置へ移動 |
| `fun clearQueue()` | キュー全消去 |

#### 検索

| シグネチャ | 目的 |
|-----------|------|
| `fun updateSearchQuery(query: String)` | 検索バー入力 (SearchStateHolder へデバウンス転送) |
| `fun requestLocateCurrentSong()` | 現在の曲にスクロール (曲/アルバム/アーティストへの nav ジャンプ要求発行) |
| `fun onSearchNavIconDoubleTapped()` | 検索アイコン 2 連打イベント発行 |

#### ライブラリ

| シグネチャ | 目的 |
|-----------|------|
| `fun setStorageFilter(filter: StorageFilter)` | ストレージフィルタ変更 (ALL / OFFLINE / CLOUD) |
| `fun setPlaylistPickerStorageFilter(filter: StorageFilter)` | プレイリストピッカー用フィルタ |
| `fun setHideLocalMedia(hide: Boolean)` | ローカル非表示 |
| `fun toggleStorageFilter()` | 3 値を循環 |
| `fun setCurrentLibraryTab(tab: LibraryTabId)` | ライブラリタブ切替 |
| `fun setSortOption(tab: String, option: SortOption)` | 並び替え |
| `fun showSortingSheet()` / `fun hideSortingSheet()` | 並び替えシート表示 |
| `fun loadSongsIfNeeded()` / `loadAlbumsIfNeeded()` / `loadArtistsIfNeeded()` | 遅延ロード (LibraryStateHolder へ) |
| `fun loadFoldersFromRepository()` | フォルダ読込 |

#### Sleep / EOT

| シグネチャ | 目的 |
|-----------|------|
| `fun setSleepTimer(minutes: Int)` | スリープタイマー設定 |
| `fun cancelSleepTimer()` | キャンセル |
| `fun setEndOfTrackTimer(enable: Boolean)` | EOT モード切替 |
| `fun setPlayCount(count: Int)` | 残り曲数指定再生 |
| `fun cancelPlayCount()` | 曲数指定キャンセル |

#### お気に入り

| シグネチャ | 目的 |
|-----------|------|
| `fun toggleFavorite(song: Song)` | お気に入り追加/削除 |
| `fun addAllFavorite(...)` / `removeAllFavorite(...)` | 一括操作 |
| `fun isFavorite(song: Song): Boolean` | 判定 |

#### DailyMix

| シグネチャ | 目的 |
|-----------|------|
| `fun forceUpdateDailyMix()` | 即時再生成 |
| `fun removeFromDailyMix(songId: String)` | 1 曲除外 |
| `fun observeSong(songId: String?): Flow<Song?>` | 単一曲監視 |

#### キャスト

| シグネチャ | 目的 |
|-----------|------|
| `fun selectCastRoute(route: MediaRouter.RouteInfo)` | ルート選択 |
| `fun disconnectCast()` | 切断 |
| `fun setRouteVolume(volume: Int)` | Cast ボリューム |
| `fun refreshCastRoutes()` | 再スキャン |
| `fun setTrackVolume(volume: Float)` | トラックボリューム |

#### AI

| シグネチャ | 目的 |
|-----------|------|
| `fun showAiPlaylistSheet()` / `dismissAiPlaylistSheet()` | シート表示 |
| `fun generateAiPlaylist(prompt: String, ...)` | プレイリスト生成 |
| `fun retryLastAiPlaylistGeneration()` | 再試行 |
| `fun clearAiError()` | エラークリア |

#### BottomSheet 状態

| シグネチャ | 目的 |
|-----------|------|
| `fun setSheetState(state: PlayerSheetState)` | プレイヤーシート状態 |
| `fun updateQueueSheetVisibility(visible: Boolean)` | キューシート |
| `fun updateCastSheetVisibility(visible: Boolean)` | キャストシート |
| `fun setSelectedSongForInfo(song: Song?)` | 楽曲情報シート |
| `fun updatePredictiveBackCollapseFraction(fraction: Float)` | 予測バック用 |
| `fun updatePredictiveBackSwipeEdge(edge: Int?)` | 予測バック用 |
| `fun resetPredictiveBackState()` | リセット |
| `fun setMiniPlayerDismissing(dismissing: Boolean)` | ミニプレイヤー dismiss アニメ |
| `fun setImmersiveTemporarilyDisabled(disabled: Boolean)` | 没入歌詞モード一時無効化 |

#### 歌詞

| シグネチャ | 目的 |
|-----------|------|
| `fun setLyricsSyncOffset(songId: String, offsetMs: Int)` | 歌詞オフセット |
| `fun loadLyricsForCurrentSong()` | 現在の曲歌詞読込 |
| `fun reloadLyricsForCurrentSong()` | 再読込 |

#### その他

| シグネチャ | 目的 |
|-----------|------|
| `fun sendToast(message: String)` | トースト発行 |
| `fun onMainActivityStart()` | Activity onStart フック (DailyMix 更新等) |
| `fun resetAndLoadInitialData(caller: String)` | 全データ再読込 |
| `fun preloadThemesAndInitialData()` | 初期テーマ先読み |
| `fun triggerShuffleAllFromTile()` | ホームタイルから全曲シャッフル |
| `fun refreshLocalConnectionInfo(refreshBluetoothDevices: Boolean)` | Wi-Fi/BT 再取得 |
| `fun incrementSongScore(song: Song)` | 選好スコア加算 |
| `fun onCustomCommand(...)` | MediaController カスタムコマンド受信 |

### 内部実装メモ

- **ファサードパターン**: 29 個の StateHolder を「薄い API セット」で公開し、UI からは `PlayerViewModel` のみが見える構造。委譲ロジックは `controllerSyncCallbacks` / `selectionActionCallbacks` / `playbackSourceCallbacks` / `shufflePlaybackCallbacks` などの callback オブジェクトに分離。
- **初期化順序**: `init` ブロックで各 StateHolder の `initialize()` を呼び、スコープと callback を注入。`viewModelScope` を各 StateHolder へ伝搬する。
- **永続化統合**: `MediaControllerSyncStateHolder` の `ControllerSyncCallbacks` で、シャッフル / リピート / 曲変更 / メディア項目遷移をフックし、`UserPreferencesRepository` へ即時書込み。
- **トリムメモリ**: `_playerUiState.update { it.copy(isLoadingLibrary = false) }` などでメモリ解放可能。
- **Paging 共有**: `paginatedSongs` / `favoritesPagingFlow` は `LibraryStateHolder` の `cachedIn(viewModelScope)` 結果。
- **searchQuery は mutableStateOf**: `var searchQuery by mutableStateOf("")` — 検索バーは Compose ツリー内で直接参照するため StateFlow ではなく snapshot 可能な `MutableState` を使用。
- **DailyMix 更新判定**: `checkAndUpdateDailyMixIfNeeded()` は `lastDailyMixUpdateFlow` の日付ベース判定 (前日と異なる場合のみ)。
- **AI API キー集約**: `hasActiveAiProviderApiKey` は 12 プロバイダ全ての API キーを `combine` して算出。
- **Telegram キャッシュ**: `telegramPlaybackObserversStarted` フラグで `ensureTelegramPlaybackObserversStarted()` を冪等に。

### 合成 StateFlow (中間型)

- `FullPlayerSlice` — フルプレイヤー画面用 (currentSongArtists, lyricsSyncOffset, albumArtQuality, audioMetadata, showPlayerFileInfo, immersiveLyricsEnabled, immersiveLyricsTimeout, isImmersiveTemporarilyDisabled, isRemotePlaybackActive, selectedRouteName, isBluetoothEnabled, bluetoothName)
- `PlayerConfigSlice` — プレイヤー設定 (navBarCornerRadius, navBarStyle, carouselStyle, fullPlayerLoadingTweaks, tapBackgroundClosesPlayer, useSmoothCorners, playerThemePreference)
- `BluetoothSlice` (private) — BT 結合中間

### 関連ファイル

- 委譲先: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlaybackDispatchStateHolder.kt`
- 委譲先: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlaybackStateHolder.kt`
- 委譲先: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/MediaControllerSyncStateHolder.kt`
- データソース: `app/src/main/java/com/theveloper/pixelplay/data/repository/MusicRepository.kt`
- Repository: `../03-data-services/repositories.md`

---

## MainViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/MainViewModel.kt` (90 行)
**アノテーション**: `@HiltViewModel`
**役割**: `MainActivity` 直下に配置される軽量 ViewModel。アプリ全体の初期セットアップ判定 (Setup 画面 vs メイン画面) を提供。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| Repository | `musicRepository: MusicRepository` |
| Worker | `syncManager: SyncManager` |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `isSetupComplete` | `StateFlow<Boolean?>` | `initialSetupDoneFlow` (null = 初期読込中) |
| `hasCompletedInitialSync` | `StateFlow<Boolean>` | 初回同期完了 (lastSyncTimestamp > 0) |
| `isSyncing` | `StateFlow<Boolean>` | `SyncManager.isSyncing` |
| `syncProgress` | `StateFlow<SyncProgress>` | 同期進捗 |
| `isLibraryEmpty` | `StateFlow<Boolean>` | ライブラリ空判定 (songCount == 0) |

### public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun startSync()` | 同期をトリガ (viewModelScope 内 launch) |

### 内部実装メモ

- `isLibraryEmpty` は `MusicRepository.getSongCountFlow()` を `stateIn` で共有。
- `hasCompletedInitialSync` は `lastSyncTimestampFlow` の `map { it > 0L }` 結果。
- 役割は薄いが、`MainActivity` の Navigation 決定 (`SetupScreen` vs `HomeScreen`) で必須。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/MainViewModel.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/SetupViewModel.kt`

---

## LibraryViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/LibraryViewModel.kt` (26 行)
**アノテーション**: `@HiltViewModel`
**役割**: `LibraryScreen` / `AlbumsScreen` / `ArtistsScreen` / `FavoritesScreen` が `hiltViewModel()` で取得する軽量ラッパ。Paging フローを `cachedIn(viewModelScope)` で画面単位にキャッシュする。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| StateHolder | `libraryStateHolder: LibraryStateHolder` |

### public プロパティ

| 名前 | 型 | 目的 |
|------|---|------|
| `songsPagingFlow` | `Flow<PagingData<Song>>` | ライブラリ楽曲ページング |
| `albumsPagingFlow` | `Flow<PagingData<Album>>` | アルバムページング |
| `artistsPagingFlow` | `Flow<PagingData<Artist>>` | アーティストページング |
| `favoritesPagingFlow` | `Flow<PagingData<Song>>` | お気に入りページング |
| `favoriteSongCountFlow` | `Flow<Int>` | お気に入り件数 |
| `isLoadingLibrary` | `StateFlow<Boolean>` | ライブラリ読込中フラグ |

### 内部実装メモ

- **キャッシュ戦略**: 全 Paging フローを `cachedIn(viewModelScope)` で囲み、スクロール位置復帰時の再フェッチを回避。
- **Hilt 多重化**: 同じ `LibraryViewModel` クラスが同じ画面内の複数 Compose ノードで取得されても、`hiltViewModel()` のスコープキーで同一インスタンスが返る。
- `LibraryStateHolder` 自体は `@Singleton` で全画面共有。本 ViewModel は表示時の一時キャッシュ層。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/LibraryViewModel.kt`
- 委譲先: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/LibraryStateHolder.kt` (→ `viewmodels-library.md`)
