# viewmodels-playback.md

> 再生系 StateHolder / ViewModel の詳細仕様。`PlaybackStateHolder` / `PlaybackDispatchStateHolder` / `QueueStateHolder` / `SleepTimerStateHolder` / `LyricsStateHolder` / `TransitionViewModel` を扱う。

---

## PlaybackStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlaybackStateHolder.kt` (1133 行)
**アノテーション**: `@Singleton`
**役割**: MediaController / `DualPlayerEngine` からの再生状態を `StablePlayerState` として集約。再生位置 (Position) の tick 駆動、Repeat / Shuffle / Seek / Next / Prev のコア操作。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Engine | `dualPlayerEngine: DualPlayerEngine` |
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| StateHolder | `castStateHolder: CastStateHolder` |
| StateHolder | `queueStateHolder: QueueStateHolder` |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `stablePlayerState` | `StateFlow<StablePlayerState>` | 再生スナップショット (currentSong, currentMediaItemIndex, isPlaying, playWhenReady, totalDuration, isShuffleEnabled, isShuffleTransitionInProgress, repeatMode, isLoadingLyrics, lyrics, isBuffering) |
| `currentPosition` | `StateFlow<Long>` | 現在再生位置 (ms) |
| `mediaController` | `var MediaController?` (getter/setter) | 現在の MediaController (複数スタック可) |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(snapshot: StablePlayerState, callbacks: PlaybackStateHolderCallbacks)` | 初期化 + snapshot 復元 (UI 復元用) |
| `fun setMediaController(controller: MediaController?)` | MediaController 登録 (スタック push) |
| `fun clearMediaController(controller: MediaController?)` | スタックから除去 |
| `fun updateStablePlayerState(update: (StablePlayerState) -> StablePlayerState)` | StateFlow を immutable に更新 |
| `fun setCurrentPosition(positionMs: Long)` | 位置強制設定 |
| `fun syncCurrentPositionFromPlayer(mediaId: String?, reportedPositionMs: Long)` | Player からの同期 |
| `fun ensureCurrentPlaybackOccurrence(mediaId: String?)` | 再生 occurrence を activate |
| `fun onPlaybackOccurrenceTransition(mediaId: String?)` | メディア切替時のコール |
| `fun rememberPausedPositionOverride(mediaId: String?, positionMs: Long)` | 一時停止位置の freeze |
| `fun clearCurrentPositionHints(mediaId: String? = null)` | override / cold start 状態を消去 |
| `fun playPause()` | 再生/一時停止 (Cast 中はリモート制御) |
| `fun seekTo(position: Long)` | シーク (ローカル / Cast 別) |
| `fun previousSong()` / `fun nextSong()` | 前/次の曲 |
| `fun cycleRepeatMode()` | OFF → ALL → ONE → OFF (ローカル & Cast 同期) |
| `fun setRepeatMode(mode: Int)` | 直接設定 |
| `fun toggleShuffle(forceShuffle: Boolean? = null, onCastSeekBlocked: () -> Unit = {})` | シャッフル ON/OFF + Cast 切替 |
| `fun resolveDurationForPlaybackState(...)` | 曲の duration 解決 (reported vs hint vs min) |
| `fun startProgressUpdates()` / `fun stopProgressUpdates()` | tick ループの制御 |
| `fun onCleared()` | クリア |

### 内部実装メモ

- **Position 解決 (`resolveUiPosition`)**:
  1. paused override (4 秒以内) が最も優先
  2. cold start snapshot (アプリ起動直後)
  3. Player reported position
  4. 上記を `abs(reported - preferred) < 1500ms` でドリフト許容
- **Progress Tick 間隔**:
  - `SLIDER_TICK_MS = 250L` (スライダー操作中)
  - `MINIPLAYER_TICK_MS = 1000L` (ミニプレイヤー表示中)
  - `BACKGROUND_TICK_MS = 1000L` (バックグラウンド)
  - リモート (Cast) は別計算
- **Shuffle 二段階**: `isShuffleEnabled` と `isShuffleTransitionInProgress` を分離。`isShuffleTransitionInProgress` 中は UI のチラつき防止。
- **Cast Repeat Mode マッピング**: Media3 の `REPEAT_MODE_OFF / ONE / ALL` と Cast の `REPEAT_MODE_REPEAT_OFF / REPEAT_MODE_REPEAT_SINGLE / REPEAT_MODE_REPEAT_ALL / REPEAT_MODE_REPEAT_ALL_AND_SHUFFLE` 間を相互変換。
- **シャッフルトグル冷却時間**: `SHUFFLE_TOGGLE_COOLDOWN_MS = 400L` で連続タップを 1 回に。
- **キュー再構築 (`buildQueueReplacement` / `buildQueueSegments`)**: `MediaItem` 列を構築し、current を中央に固定して before/after に分割。バルク置換 (`BULK_REPLACE_THRESHOLD = 80`) を境に in-place reorder か完全置換を選択。
- **Shuffle 適用順**:
  1. `reorderQueueInPlace` (現状キューと一致するなら in-place move)
  2. `buildQueueSegments` で currentIndex 周辺を温存しつつ before/after 置換
  3. 失敗時は完全置換 (`buildQueueReplacement`)

### 関連ファイル

- 再生エンジン: `../04-engine/music-service.md` (`DualPlayerEngine`, `MusicService`)
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/CastStateHolder.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/QueueStateHolder.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/StablePlayerState.kt` (データクラス定義)

---

## PlaybackDispatchStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlaybackDispatchStateHolder.kt` (1166 行)
**アノテーション**: `@ViewModelScoped` (実際は `@Inject` + コンストラクタ)
**役割**: 再生開始リクエストを集約し、トークン化 (`fullQueuePlaybackToken` / `directPlaybackToken`) で古いリクエストを無効化しながら Cast / ローカル切替を安全に実行する。

### 注入される依存 (14 個)

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| Engine | `dualPlayerEngine: DualPlayerEngine` |
| Util | `appShortcutManager: AppShortcutManager` |
| Worker | `syncManager: SyncManager` |
| StateHolder | `externalMediaStateHolder: ExternalMediaStateHolder` |
| StateHolder | `playbackStateHolder: PlaybackStateHolder` |
| StateHolder | `queueStateHolder: QueueStateHolder` |
| StateHolder | `libraryStateHolder: LibraryStateHolder` |
| StateHolder | `castStateHolder: CastStateHolder` |
| StateHolder | `castTransferStateHolder: CastTransferStateHolder` |
| StateHolder | `connectivityStateHolder: ConnectivityStateHolder` |
| StateHolder | `themeStateHolder: ThemeStateHolder` |

### `PlaybackDispatchCallbacks` (data class)

`PlayerViewModel` から注入される callback 群:

| 名前 | 目的 |
|------|------|
| `scope: CoroutineScope` | コルーチン起動元 |
| `getController: () -> MediaController?` | 現在の MediaController |
| `getUiState: () -> PlayerUiState` | UI 状態取得 |
| `updateUiState: ((PlayerUiState) -> PlayerUiState) -> Unit` | UI 状態書込 |
| `showSheet: () -> Unit` | プレイヤーシート展開 |
| `collapseSheetState: () -> Unit` | 閉じる |
| `showPlayer: () -> Unit` | フルプレイヤー表示 |
| `sendToast: (String) -> Unit` | 即時トースト |
| `emitToast: suspend (String) -> Unit` | suspend トースト |
| `showNoInternetDialog: () -> Unit` | オフライン警告 |
| `ensureTelegramObservers: () -> Unit` | Telegram 監視開始 |
| `cancelTransitionScheduler: () -> Unit` | クロスフェードキャンセル |
| `incrementSongScore: (Song) -> Unit` | スコア加算 |
| `resetPredictiveBackState: () -> Unit` | 予測バック状態リセット |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(callbacks: PlaybackDispatchCallbacks)` | コールバック注入 |
| `fun onCleared()` | 進行中 Job キャンセル |
| `fun showAndPlaySongFromLibrary(songs, startIndex, queueName)` | ライブラリ楽曲から再生 (ライブラリ順) |
| `fun showAndPlaySongFromFavorites(songs, startIndex, queueName)` | お気に入りから再生 |
| `suspend fun getSongsForCurrentLibrarySelection(): List<Song>` | ライブラリ全曲 (sort+filter 適用) |
| `suspend fun getSongsForCurrentFavoriteSelection(): List<Song>` | お気に入り全曲 |
| `fun launchLatestFullQueuePlayback(sortedIdsProvider: suspend () -> List<Long>)` | 最新トークンで全曲再生 (重複防止) |
| `fun cancelPendingFullQueuePlayback()` | 古いリクエストをキャンセル |
| `fun showAndPlaySong(song: Song, ...)` | 1 曲再生 (Cast 時はジャンプ再利用判定) |
| `fun showAndPlaySong(song: Song)` (overload) | PlayerUiState から context を取得 |
| `suspend fun hydrateSongsIfNeeded(songs: List<Song>): List<Song>` | 不足プロパティを DB から補完 |
| `fun playSongs(songsToPlay, startSong, queueName, playlistId?)` | 任意キューで再生 |
| `fun playSongsShuffled(songsToPlay, queueName, startAtZero, isShuffleUserForced)` | シャッフル再生 |
| `fun playExternalUri(uri: Uri)` | External URI 再生 (ExternalMediaStateHolder 経由) |
| `fun triggerShuffleAllFromTile()` | ホームタイル用シャッフル全曲 |
| `suspend fun internalPlaySongs(songs, startSong, queueName, playlistId?)` | 内部実装 (Local + Cast 振り分け) |
| `suspend fun buildResolvedPlaybackMediaItem(song: Song): MediaItem` | Cloud URI 解決 (Telegram 等) |
| `fun loadAndPlaySong(song: Song)` | 1 曲を MediaController 経由で読込 |
| `fun addSongToQueue(song: Song)` | キュー末尾追加 |
| `fun addSongNextToQueue(song: Song)` | 次の曲として追加 |
| `fun playPause()` | 再生/一時停止 (Cast 時は remoteMediaClient に委譲) |
| `fun preparePlaybackQueueSegments(...)` / `attachPreparedQueueSegmentsIfCurrent(...)` | キュー分割読込 |
| `fun setPreparingSong(songId: String?)` / `fun beginPreparingSong(song: Song)` | 準備中フラグ |
| `fun clearPreparingSongIfMatching(mediaId: String?)` | 準備中解除 |
| `fun songRequiresHydration(song: Song): Boolean` | 補完要否判定 |
| `fun flushPendingPlaybackAction()` | pending action を即時実行 |

### 内部実装メモ

- **Token によるリクエスト無効化**:
  - `fullQueuePlaybackToken: Long` — `launchLatestFullQueuePlayback` で increment。古い Job は `throwIfFullQueuePlaybackRequestIsStale(token)` で中断。
  - `directPlaybackToken: Long` — `playSongs` / `playExternalUri` などで使用。
- **Cast Jump 最適化**: `contextMatchesRemoteSnapshot` で Cast 側の `lastRemoteQueue` と同じ曲順なら `itemId` を使ってジャンプのみ実行 (キュー再ロード不要)。
- **Hydration**: Telegram 曲など Song の必須プロパティ (`title`, `album`, `duration` 等) が欠落しているものは `getSongsByIdsChunked` で `SONG_ID_QUERY_CHUNK_SIZE = 900` 毎にチャンクして DB から取得。
- **Persistent Shuffle**: `userPreferencesRepository.persistentShuffleEnabledFlow` が true なら、シャッフル再生を強制。
- **Cast 接続中**: `playPause()` は `localQueue` を snapshot し、Cast の `mediaStatus` と比較。一致すれば `mediaStatus.play()` のみ実行。
- **`Song.requiresHydration()`**: タイトル空 + アルバム空 + duration 0 のいずれかで `true`。
- **AppShortcutManager**: ホーム画面ショートカットから呼ばれた際のキューリストも `playSongs` 経由で処理。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlaybackStateHolder.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/QueueStateHolder.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/CastTransferStateHolder.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ExternalMediaStateHolder.kt`

---

## QueueStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/QueueStateHolder.kt` (289 行)
**アノテーション**: `@Singleton`
**役割**: シャッフル前の「オリジナルキュー」を保持 / シャッフル順序生成 / Album/Artist 単位のシャッフル / 再生。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |

### 主要 state (var — UI へ `@Singleton` スコープで公開)

| 名前 | 型 | 目的 |
|------|---|------|
| `originalQueueOrder` | `List<Song>` (getter) | シャッフル前のキュー (トグルで戻す用) |
| `originalQueueName` | `String` (getter) | キュー名 ("None" = 未設定) |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun saveOriginalQueueState(queue: List<Song>, queueName: String)` | オリジナルキュー記憶 |
| `fun setOriginalQueueOrder(queue: List<Song>)` | 強制設定 |
| `fun hasOriginalQueue(): Boolean` | 記憶済み判定 |
| `fun getOriginalQueueForRestore(): List<Song>` | 復元用にコピー取得 |
| `fun clearOriginalQueue()` | クリア |
| `fun createShuffledQueue(currentQueue, currentSongId, forceStartFromCurrent): Pair<List<Song>, Int>?` | 現在位置を維持してシャッフル |
| `fun prepareShuffledQueue(songs, queueName): Pair<List<Song>, Song>?` | ランダム開始点 |
| `fun prepareShuffledQueueWithStart(songs, startSong, queueName): List<Song>` | 指定曲から開始 |
| `suspend fun prepareShuffledQueueSuspending(songs, queueName, startAtZero): Pair<List<Song>, Song>?` | `Dispatchers.Default` で実行 |
| `fun shuffleAll(currentStorageFilter, onQueueReady)` | 全曲シャッフル (500 件サンプリング) |
| `fun playRandom(callbacks: ShufflePlaybackCallbacks)` | 完全ランダム 1 曲 |
| `fun shuffleFavorites(callbacks)` | お気に入りからシャッフル再生 |
| `fun shuffleRandomAlbum(callbacks)` | ランダムアルバム |
| `fun shuffleRandomArtist(callbacks)` | ランダムアーティスト |
| `fun playAlbum(album, callbacks: PlaybackSourceCallbacks)` | アルバム全曲再生 (Track# → Title 順) |
| `fun playArtist(artist, callbacks)` | アーティスト全曲再生 |

### `ShufflePlaybackCallbacks` / `PlaybackSourceCallbacks` (data class)

| Callback | 目的 |
|----------|------|
| `scope: CoroutineScope` | 実行スコープ |
| `currentStorageFilter: () -> StorageFilter` | フィルタ取得 |
| `albums: () -> List<Album>` | アルバム一覧 |
| `artists: () -> List<Artist>` | アーティスト一覧 |
| `playShuffled: (songs, queueName) -> Unit` | シャッフル再生 (PlayerViewModel 実装) |
| `playSongs: (songs, startSong, queueName, playlistId?) -> Unit` | 通常再生 |

### 内部実装メモ

- **シャッフル定数**:
  - `SHUFFLE_SAMPLE_LIMIT = 500` — 全曲シャッフル時の取得上限
  - `ALL_SONGS_SHUFFLED_QUEUE = "All Songs (Shuffled)"`
  - `FAVORITES_SHUFFLED_QUEUE = "Liked Songs (Shuffled)"`
- **シャッフルアルゴリズム**: `QueueUtils.buildAnchoredShuffleQueue` を使用 (現在曲を先頭に固定して残りを Fisher-Yates)。
- **プレイリスト再生順序**: `playAlbum` は `discNumber` → `trackNumber` → `title` で安定ソート。
- **`Dispatchers.Default`**: `prepareShuffledQueueSuspending` は CPU バウンド処理のため Default ディスパッチャで実行。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/utils/QueueUtils.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlaybackStateHolder.kt`

---

## SleepTimerStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/SleepTimerStateHolder.kt` (309 行)
**アノテーション**: `@Singleton`
**役割**: スリープタイマー (指定分数後に停止) と EOT (End Of Track) タイマーを統合管理。AlarmManager ブロードキャストでカウントダウン。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Context | `context: Context` (ApplicationContext) |
| System | `alarmManager: AlarmManager` |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `sleepTimerEndTimeMillis` | `StateFlow<Long?>` | タイマー終了予定時刻 (epoch ms) |
| `isEndOfTrackTimerActive` | `StateFlow<Boolean>` | EOT モード有効 |
| `activeTimerValueDisplay` | `StateFlow<String?>` | 表示用残り時間 (e.g. "00:14:32") |
| `activeTimerDurationMinutes` | `StateFlow<Int?>` | 設定分数 |
| `playCount` | `StateFlow<Float>` | 残り曲数 (0.5 単位で部分曲サポート) |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(scope, mediaControllerProvider, currentSongIdProvider, songTitleResolver, toastEmitter)` | 依存注入 |
| `fun setSleepTimer(durationMinutes: Int)` | タイマー設定 (AlarmManager 登録) |
| `fun playCounted(count: Int)` | N 曲再生後に停止 (1 曲 = 1.0) |
| `fun cancelCountedPlay()` | 曲数指定キャンセル |
| `fun setEndOfTrackTimer(enable: Boolean, currentSongId: String?)` | EOT モード切替 |
| `fun cancelSleepTimer(overrideToastMessage, suppressDefaultToast)` | 全タイマー停止 |
| `fun onCleared()` | Job キャンセル |

### `EotStateHolder` (singleton object — 別ファイル)

`com.theveloper.pixelplay.data.EotStateHolder` で EOT 対象の曲 ID / タイトルを保持。
複数 StateHolder 間で参照される。

### 内部実装メモ

- **AlarmManager 連携**: `Intent(context, SleepTimerReceiver::class.java)` を `PendingIntent` 化し、`setExactAndAllowWhileIdle` で登録。
- **`SLEEP_TIMER_ACTION = "com.theveloper.pixelplay.action.SLEEP_TIMER_EXPIRED"`**: Receiver 起動時の識別子。
- **EOT 監視**: `EotStateHolder.eotTargetSongId` の Flow を `filterNotNull` → `distinctUntilChanged` で監視し、曲遷移を検出したらタイマー解除。
- **`activeTimerValueDisplay` 更新**: タイマー Job 内で `while(isActive) { delay(1000); _activeTimerValueDisplay.value = formatRemaining(endTime - now) }`。
- **曲数指定**: `playCount` を 0.5 / 1.0 刻みで増減 (曲遷移時に `playCount -= 1.0`)。`Float` なのは 0.5 単位で「現在の曲の途中で停止」サポートのため。
- **データ層 Receiver**: `data.service.SleepTimerReceiver` がブロードキャストを受けて `MusicService` に pause を指示。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/EotStateHolder.kt` (object)
- `app/src/main/java/com/theveloper/pixelplay/data/service/SleepTimerReceiver.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/service/MusicService.kt`

---

## LyricsStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/LyricsStateHolder.kt` (509 行)
**アノテーション**: `@Singleton`
**役割**: 歌詞読込 (埋め込み / ローカル .lrc / リモート検索) / 同期オフセット管理 / 歌詞インポート / AI 翻訳 / エラー管理。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| Media | `songMetadataEditor: SongMetadataEditor` |

### 主要 StateFlow / SharedFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `currentSongSyncOffset` | `StateFlow<Int>` | 現在の曲の歌詞オフセット (ms) |
| `searchUiState` | `StateFlow<LyricsSearchUiState>` | 検索 UI 状態 (Idle / Loading / PickResult / Success / NotFound / Error) |
| `songUpdates` | `SharedFlow<Pair<Song, Lyrics?>>` | 歌詞保存後の Song 更新通知 |
| `messageEvents` | `SharedFlow<String>` | ユーザー向けメッセージ |

### `LyricsSearchUiState` (sealed interface)

| 派生 | 用途 |
|------|------|
| `Idle` | 初期状態 |
| `Loading` | 検索中 |
| `PickResult(query, results)` | 候補表示 |
| `Success(lyrics)` | 取得成功 |
| `NotFound(message, allowManualSearch)` | 見つからず |
| `Error(message, query?)` | エラー |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(scope, loadCallback: LyricsLoadCallback)` | 依存注入 + 歌詞読込コールバック設定 |
| `fun loadLyricsForSong(song: Song, sourcePreference: LyricsSourcePreference)` | 歌詞読込 (埋め込み → ローカル → リモート) |
| `fun cancelLoading()` | 進行中読込をキャンセル |
| `fun setSyncOffset(songId: String, offsetMs: Int)` | オフセット保存 |
| `suspend fun updateSyncOffsetForSong(songId: String)` | 現在の offset を再読込 |
| `fun setSearchState(state: LyricsSearchUiState)` | 検索状態強制設定 |
| `fun resetSearchState()` | Idle に戻す |
| `fun fetchLyricsForSong(song, sourcePreference): Lyrics?` | suspend 取得 |
| `fun searchLyricsManually(title, artist)` | 手動検索 (LyricsSearchResult 取得) |
| `fun acceptLyricsSearchResult(result, currentSong)` | 検索結果を採用 |
| `fun importLyricsFromFile(songId, validatedImport, currentSong)` | .lrc ファイルからインポート |
| `fun translateLyricsViaAi(currentSong, lyricsObj, cb: LyricsTranslationCallbacks)` | AI 翻訳 |
| `fun resetLyrics(songId)` | 歌詞クリア |
| `fun resetAllLyrics()` | 全曲リセット |
| `fun onCleared()` | クリーンアップ |

### 内部実装メモ

- **歌詞ソース優先順位 (`LyricsSourcePreference`)**:
  1. `EMBEDDED_FIRST` (デフォルト) — 埋め込み → ローカル .lrc → リモート
  2. `LOCAL_FILE_FIRST` — ローカル → 埋め込み → リモート
  3. `REMOTE_FIRST` — リモート → 埋め込み → ローカル
  4. `EMBEDDED_ONLY` / `LOCAL_ONLY` / `REMOTE_ONLY` — 単一ソース
- **セキュリティ検証**: `LyricsImportSecurity.validateImportedLrcContent` / `validateLocalLyricsFile` でインポート時のサニタイズ (XSS / LRC Injection 防止)。
- **AI 翻訳**: `LyricsTranslationCallbacks` で `translate: suspend (String) -> Result<String>` を PlayerViewModel 側でラップ。`aiHandler` 経由。
- **歌詞保存**: `persistLyricsToFileMetadataIfPossible` で埋め込みタグに書き戻し、`albumArtUriString` も再生成 (アートワーク変更検知)。
- **同期オフセット永続化**: `userPreferencesRepository.setLyricsSyncOffset(songId, offsetMs)` で曲別保存。
- **曲遷移時フック**: `loadCallback` (`LyricsLoadCallback`) で `onLoadingStarted` / `onLyricsLoaded` を呼び、`StablePlayerState` の `isLoadingLyrics` / `lyrics` を更新。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/utils/LyricsImportSecurity.kt`
- `app/src/main/java/com/theveloper/pixelplay/utils/LyricsUtils.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/media/SongMetadataEditor.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/repository/MusicRepository.kt` (`searchLyricsManually` 等)

---

## TransitionViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/TransitionViewModel.kt` (155 行)
**アノテーション**: `@HiltViewModel`
**役割**: プレイリスト単位の Transition (クロスフェード) 設定画面の状態管理。`SavedStateHandle["playlistId"]` からプレイリスト ID を取得し、Global / Playlist Override を切替。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `transitionRepository: TransitionRepository` |
| State | `savedStateHandle: SavedStateHandle` (NavArg 注入) |

### `TransitionUiState` (data class)

| フィールド | 目的 |
|----------|------|
| `rule: TransitionRule?` | プレイリスト固有ルール |
| `globalSettings: TransitionSettings` | グローバル設定 |
| `isLoading: Boolean` | 初期読込中 |
| `isSaved: Boolean` | 保存成功フラグ |
| `useGlobalDefaults: Boolean` | プレイリストオーバーライド無効化 |
| `playlistId: String?` | 編集中プレイリスト |

### `TransitionSettings` (data class — データレイヤー)

| フィールド | 型 | 意味 |
|----------|---|------|
| `durationMs: Int` | フェード時間 (0–12000) |
| `mode: TransitionMode` | GAPLESS / CROSSFADE / FADE_OUT_FADE_IN |
| `curveIn: Curve` | LINEAR / EASE_IN / EASE_OUT / EASE_IN_OUT / EASE_IN_CUBIC / EASE_OUT_CUBIC |
| `curveOut: Curve` | 同上 |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<TransitionUiState>` | 画面用統合状態 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun updateDuration(durationMs: Int)` | duration スライダー |
| `fun updateMode(mode: TransitionMode)` | モード切替 |
| `fun updateCurveIn(curve: Curve)` | 入力カーブ |
| `fun updateCurveOut(curve: Curve)` | 出力カーブ |
| `fun useGlobalDefaults()` | プレイリストオーバーライド解除 |
| `fun enablePlaylistOverride()` | プレイリストオーバーライド有効化 (Global の値をコピー) |
| `fun saveSettings()` | 永続化 |

### 内部実装メモ

- **初期化戦略**: `init { loadSettings() }` で `transitionRepository.getPlaylistDefaultRule(playlistId).first()` と `getGlobalSettings().first()` を並行取得。
- **`useGlobalDefaults == true`** なら `rule` を `null` として保存 → Repository 側で Global にフォールバック。
- **`enablePlaylistOverride()`**: 現在の Global 値を初期値として `TransitionRule` を生成。
- **保存戦略**: グローバル設定変更は直接 Repository へ (`setGlobalSettings`)。プレイリストルール変更は `setPlaylistRule(playlistId, rule)`。
- **ユースケース**: プレイリスト詳細画面 → 設定アイコン → `EditTransitionScreen` 遷移。

### 関連ファイル

- データモデル: `app/src/main/java/com/theveloper/pixelplay/data/model/TransitionRule.kt` / `TransitionSettings.kt`
- Repository: `app/src/main/java/com/theveloper/pixelplay/data/repository/TransitionRepository.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/EditTransitionScreen.kt`
