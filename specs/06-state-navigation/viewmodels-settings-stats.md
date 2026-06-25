# viewmodels-settings-stats.md

> 設定 / セットアップ / 統計 / イコライザ / DailyMix 系 StateHolder / ViewModel の詳細仕様。`SettingsViewModel` (1490 行) / `SetupViewModel` (395 行) / `StatsViewModel` (196 行) / `EqualizerViewModel` (523 行) / `DailyMixStateHolder` / `ListeningStatsTracker` (335 行) を扱う。

---

## SettingsViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/SettingsViewModel.kt` (1490 行)
**アノテーション**: `@HiltViewModel`
**役割**: 設定画面の統合 ViewModel。テーマ / ナビバー / ライブラリ / 同期 / AI 12 プロバイダ / バックアップ / ファイルエクスプローラ / 統計を全カバー。

### 注入される依存 (14 個)

| 種類 | フィールド |
|------|----------|
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| Preferences | `aiPreferencesRepository: AiPreferencesRepository` |
| Preferences | `themePreferencesRepository: ThemePreferencesRepository` |
| Process | `colorSchemeProcessor: ColorSchemeProcessor` |
| Worker | `syncManager: SyncManager` |
| AI | `aiClientFactory: AiClientFactory` |
| AI | `geminiModelService: GeminiModelService` |
| DAO | `aiUsageDao: AiUsageDao` |
| Repository | `lyricsRepository: LyricsRepository` |
| Repository | `musicRepository: MusicRepository` |
| Backup | `backupManager: BackupManager` |
| Context | `context: Context` (ApplicationContext) |

### `SettingsUiState` (data class — 60+ フィールド)

主要フィールド:

| フィールド | 目的 |
|----------|------|
| `isLoadingDirectories: Boolean` | ディレクトリ読込中 |
| `appLanguageTag: String` | 言語コード |
| `appThemeMode: String` | ダーク / ライト / システム |
| `playerThemePreference: String` | プレイヤー色テーマ |
| `albumArtPaletteStyle: AlbumArtPaletteStyle` | パレットスタイル |
| `albumArtColorAccuracy: Int` | 精度 (1-100) |
| `mockGenresEnabled: Boolean` | モックジャンル |
| `navBarCornerRadius: Int` | ナビバー角丸 |
| `navBarStyle: String` | DEFAULT / FLOATING / ... |
| `navBarCompactMode: Boolean` | コンパクトモード |
| `carouselStyle: String` | カルーセルスタイル |
| `libraryNavigationMode: String` | TAB_ROW / DRAWER / LIST |
| `launchTab: String` | 起動タブ |
| `keepPlayingInBackground: Boolean` | バックグラウンド再生 |
| `disableCastAutoplay: Boolean` | キャスト自動再生無効 |
| `pauseOnVolumeZero: Boolean` | 音量ゼロで一時停止 |
| `resumeOnHeadsetReconnect: Boolean` | ヘッドセット再接続で再開 |
| `showQueueHistory: Boolean` | キュー履歴表示 |
| `isCrossfadeEnabled: Boolean` | クロスフェード |
| `hiFiModeEnabled: Boolean` | Hi-Fi モード |
| `crossfadeDuration: Int` | クロスフェード時間 (ms) |
| `persistentShuffleEnabled: Boolean` | シャッフル永続化 |
| `lyricsSourcePreference: LyricsSourcePreference` | 歌詞ソース優先度 |
| `autoScanLrcFiles: Boolean` | .lrc 自動スキャン |
| `blockedDirectories: Set<String>` | ブロックディレクトリ |
| `fullPlayerLoadingTweaks: FullPlayerLoadingTweaks` | フルプレイヤー遅延設定 |
| `albumArtQuality: AlbumArtQuality` | アート品質 |
| `albumArtCacheLimitMb: Int` | アートキャッシュ上限 |
| `collagePattern: CollagePattern` | コラージュパターン |
| `collageAutoRotate: Boolean` | コラージュ自動回転 |
| `minSongDuration: Int` | 最小曲長 (ms) |
| `minTracksPerAlbum: Int` | アルバムの最小曲数 |
| `replayGainEnabled: Boolean` | ReplayGain |
| `replayGainUseAlbumGain: Boolean` | アルバムゲイン使用 |
| `isSafeTokenLimitEnabled: Boolean` | 安全トークン制限 |
| `showScrollbar: Boolean` | スクロールバー表示 |
| `immersiveLyricsEnabled: Boolean` | 没入歌詞 |
| `immersiveLyricsTimeout: Long` | 没入歌詞タイムアウト |
| `useAnimatedLyrics: Boolean` | アニメーション歌詞 |
| `animatedLyricsBlurEnabled: Boolean` | ブラー |
| `animatedLyricsBlurStrength: Float` | ブラー強度 |
| `disableBlurAllOver: Boolean` | 全画面でブラー無効 |
| `backupInfoDismissed: Boolean` | バックアップ情報非表示 |
| `isDataTransferInProgress: Boolean` | バックアップ/リストア進行中 |
| `restorePlan: RestorePlan?` | リストア計画 |
| `backupHistory: List<BackupHistoryEntry>` | バックアップ履歴 |
| `backupValidationErrors: List<ValidationError>` | 検証エラー |
| `isInspectingBackup: Boolean` | バックアップ検査中 |
| `modelsFetchError: String?` | モデル取得エラー |
| `appRebrandDialogShown: Boolean` | リブランドダイアログ |
| `beta05CleanInstallDisclaimerDismissed: Boolean?` | 免責事項非表示 |

### `LyricsRefreshProgress` (data class)

| フィールド | 目的 |
|----------|------|
| `totalSongs: Int` | 全体数 |
| `currentCount: Int` | 処理済み |
| `savedCount: Int` | 保存成功 |
| `notFoundCount: Int` | 歌詞見つからず |
| `skippedCount: Int` | スキップ |
| `isComplete: Boolean` | 完了 |
| `failedSongs: List<FailedSongInfo>` | 失敗曲 |

### StateFlow / SharedFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<SettingsUiState>` | 画面状態 |
| `aiProvider` | `StateFlow<String>` | 現在の AI プロバイダ |
| `currentAiApiKey` / `currentAiModel` / `currentAiSystemPrompt` / `currentAiBaseUrl` | 現在の AI 設定 |
| `geminiApiKey` / `geminiModel` / `geminiSystemPrompt` | Gemini 専用 |
| `deepseekApiKey` / `deepseekModel` / `deepseekSystemPrompt` | DeepSeek 専用 |
| `groqApiKey` / `groqModel` / `groqSystemPrompt` | Groq 専用 |
| `mistralApiKey` / `mistralModel` / `mistralSystemPrompt` | Mistral 専用 |
| `nvidiaApiKey` / `nvidiaModel` / `nvidiaSystemPrompt` | NVIDIA 専用 |
| `kimiApiKey` / `kimiModel` / `kimiSystemPrompt` | Kimi 専用 |
| `glmApiKey` / `glmModel` / `glmSystemPrompt` | GLM 専用 |
| `openaiApiKey` / `openaiModel` / `openaiSystemPrompt` | OpenAI 専用 |
| `openrouterApiKey` / `openrouterModel` / `openrouterSystemPrompt` | OpenRouter 専用 |
| `ollamaApiKey` / `ollamaModel` / `ollamaSystemPrompt` | Ollama 専用 |
| `customApiKey` / `customModel` / `customSystemPrompt` / `customBaseUrl` | Custom 専用 |
| `aiTemperature` / `aiTopP` / `aiTopK` / `aiMaxTokens` | サンプリングパラメータ |
| `aiPresencePenalty` / `aiFrequencyPenalty` | ペナルティ |
| `aiSampleSize` / `aiDigestMode` / `aiIncludeExtendedFields` | プレイリスト生成設定 |
| `isSafeTokenLimitEnabled: StateFlow<Boolean>` | 安全トークン制限 |
| `recentAiUsage: StateFlow<List<AiUsageEntity>>` | 直近 20 件 |
| `totalPromptTokens: StateFlow<Int>` | プロンプト累計 |
| `totalOutputTokens: StateFlow<Int>` | 出力累計 |
| `totalThoughtTokens: StateFlow<Int>` | 思考累計 |
| `currentPath: StateFlow<File>` | ファイルエクスプローラ現在パス (FileExplorerStateHolder 経由) |
| `currentDirectoryChildren: StateFlow<List<DirectoryEntry>>` | 現在ディレクトリ |
| `blockedDirectories` / `availableStorages` / `selectedStorageIndex` | ファイルエクスプローラ |
| `isLoadingDirectories` / `isExplorerPriming` / `isExplorerReady` / `isCurrentDirectoryResolved` | ローディング状態 |
| `isSyncing: StateFlow<Boolean>` | 同期中 |
| `syncProgress: StateFlow<SyncProgress>` | 同期進捗 |
| `dataTransferEvents: SharedFlow<String>` | バックアップ/リストアのイベント |
| `dataTransferProgress: StateFlow<BackupTransferProgressUpdate?>` | データ転送進捗 |

### 主要 public 関数 (抜粋)

#### 設定 setter (各 1 行)

`setAppRebrandDialogShown`, `setBeta05CleanInstallDisclaimerDismissed`, `setPlayerThemePreference`, `setAlbumArtPaletteStyle`, `setCollagePattern`, `setCollageAutoRotate`, `setAppThemeMode`, `setAppLanguage`, `setNavBarStyle`, `setNavBarCompactMode`, `setLibraryNavigationMode`, `setCarouselStyle`, `setShowPlayerFileInfo`, `setShowScrollbar`, `setLaunchTab`, `setKeepPlayingInBackground`, `setDisableCastAutoplay`, `setPauseOnVolumeZero`, `setResumeOnHeadsetReconnect`, `setHiFiModeEnabled`, `setShowQueueHistory`, `setCrossfadeEnabled`, `setCrossfadeDuration`, `setPersistentShuffleEnabled`, `setFolderBackGestureNavigation`, `setLyricsSourcePreference`, `setAutoScanLrcFiles`, `setUseAnimatedLyrics`, `setAnimatedLyricsBlurEnabled`, `setAnimatedLyricsBlurStrength`, `setDisableBlurAllOver`, `setSafeTokenLimitEnabled`, `setMinSongDuration`, `setMinTracksPerAlbum`, `setReplayGainEnabled`, `setReplayGainUseAlbumGain`, `setImmersiveLyricsEnabled`, `setImmersiveLyricsTimeout`, `setAlbumArtQuality`, `setAlbumArtCacheLimitMb`, `setUseSmoothCorners`, `setTapBackgroundClosesPlayer`, `setHapticsEnabled`, `setBackupInfoDismissed`, `setNavBarCornerRadius`

#### フルプレイヤー遅延設定 (Full Player Tweaks)

`setDelayAllFullPlayerContent`, `setDelayAlbumCarousel`, `setDelaySongMetadata`, `setDelayProgressBar`, `setDelayControls`, `setFullPlayerPlaceholders`, `setTransparentPlaceholders`, `setFullPlayerPlaceholdersOnClose`, `setFullPlayerSwitchOnDragRelease`, `setFullPlayerAppearThreshold`, `setFullPlayerCloseThreshold`

#### ディレクトリ管理

| シグネチャ | 目的 |
|-----------|------|
| `fun toggleDirectoryAllowed(file: File)` | 許可/ブロック トグル |
| `fun applyPendingDirectoryRuleChanges()` | 保留中の変更を永続化 |
| `fun loadDirectory(file: File)` | ディレクトリ読込 |
| `fun primeExplorer()` | 初期化 (プリロード) |
| `fun openExplorer()` | 開く |
| `fun navigateUp()` | 親へ |
| `fun refreshExplorer()` | 再読込 |
| `fun selectStorage(index: Int)` | ストレージ選択 |
| `fun refreshAvailableStorages()` | ストレージ一覧更新 |
| `fun isAtRoot(): Boolean` | ルート判定 |
| `fun explorerRoot(): File` | ルート取得 |

#### AI

| シグネチャ | 目的 |
|-----------|------|
| `fun onAiApiKeyChange(apiKey: String)` | 現在のプロバイダの API キー書込 |
| `fun onGeminiApiKeyChange` ... `onCustomApiKeyChange` | 各プロバイダ個別 |
| `fun onAiTemperatureChange` ... `onAiFrequencyPenaltyChange` | パラメータ |
| `fun onAiSampleSizeChange` / `onAiDigestModeChange` / `onAiIncludeExtendedFieldsChange` | 生成設定 |
| `fun onAiModelChange(model)` | 現在のプロバイダのモデル |
| `fun onGeminiModelChange` ... `onCustomModelChange` | 個別 |
| `fun onAiSystemPromptChange(prompt)` | システムプロンプト |
| `fun onGeminiSystemPromptChange` ... `onCustomSystemPromptChange` | 個別 |
| `fun resetAiSystemPrompt` ... `resetCustomSystemPrompt` | デフォルト復元 |
| `fun clearAiUsageData()` | 履歴クリア |
| `fun onAiProviderChange(provider: String)` | プロバイダ切替 |
| `fun loadModelsForCurrentProvider()` | モデル一覧取得 |
| `private fun fetchAvailableModels(apiKey, providerName)` | 内部 |
| `private fun formatModelDisplayName(modelName)` | 表示名整形 |

#### アルバムアート

| シグネチャ | 目的 |
|-----------|------|
| `fun setAlbumArtPaletteSettings(style, accuracy)` | スタイル + 精度 |
| `suspend fun getAlbumArtPalettePreview(uri, style, accuracy): ColorSchemePair?` | プレビュー生成 |

#### 同期 / 統計

| シグネチャ | 目的 |
|-----------|------|
| `fun refreshLibrary()` | 増分同期 |
| `fun fullSyncLibrary()` | 全同期 |
| `fun rebuildDatabase()` | DB 再構築 |

#### バックアップ / リストア

| シグネチャ | 目的 |
|-----------|------|
| `fun exportAppData(uri: Uri, sections: Set<BackupSection>)` | エクスポート |
| `fun inspectBackupFile(uri: Uri)` | 検査 |
| `fun updateRestorePlanSelection(selectedModules: Set<BackupSection>)` | リストア計画更新 |
| `fun restoreFromPlan(uri: Uri)` | リストア実行 |
| `fun clearRestorePlan()` | 計画クリア |
| `fun removeBackupHistoryEntry(entry)` | 履歴削除 |

#### 開発

| シグネチャ | 目的 |
|-----------|------|
| `fun triggerTestCrash()` | クラッシュテスト |
| `fun resetSetupFlow()` | セットアップリセット |

### 内部実装メモ

- **初期化**: `init { /* 100+ 個の UserPreferencesRepository Flow を listen */ }` で巨大な Combine / MapLatest を構築。`combine` を使って 60 フィールドの `SettingsUiState` を合成。
- **3 つの内部 Group**: `SettingsUiUpdate.Group1` (UI テーマ系) / `Group2` (再生系) への分割。
- **AI プロバイダ切替**: `onAiProviderChange` で `AiProvider.fromString(provider)` 変換し、`getApiKey` / `getModel` / `getSystemPrompt` を各プロバイダごとに取得。`apiKey` 空なら `loadModelsForCurrentProvider` を呼ばない。
- **モデル取得**:
  - Gemini: `geminiModelService.listModels(apiKey)`
  - それ以外: `aiClientFactory.create(provider).listModels()` (対応プロバイダのみ)
- **ファイルエクスプローラ**: 内部で `FileExplorerStateHolder(userPreferencesRepository, viewModelScope, context)` を `private val` として構築 (他画面とは独立したインスタンス)。
- **バックグラウンド色設定の最適化**: `_uiState.update { it.copy(...) }` で部分更新し、全フィールド再評価を避ける。
- **Hi-Fi モード**: `HiFiCapabilityChecker.isDeviceSupported(context)` の結果と `userPreferencesRepository.hiFiModeEnabledFlow` を combine。
- **テーマプレビュー**: `getAlbumArtPalettePreview` は一時的にスタイル / 精度を変えて ColorScheme を生成 (永続化しない)。
- **`buildAlbumArtPaletteSettings`**: 1 関数でスタイル + 精度を同時設定 (2 つの setter 呼び出しを 1 トランザクションに)。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/preferences/UserPreferencesRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/preferences/AiPreferencesRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/preferences/ThemePreferencesRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/backup/BackupManager.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/ai/GeminiModelService.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/ai/provider/AiClientFactory.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/repository/LyricsRepository.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/SettingsScreen.kt`

---

## SetupViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/SetupViewModel.kt` (395 行)
**アノテーション**: `@HiltViewModel`
**役割**: 初回セットアップウィザード画面。権限要求 / ディレクトリ選択 / ナビ設定 / バックアップ復元 / 同期 / 完了を統合。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| Preferences | `themePreferencesRepository: ThemePreferencesRepository` |
| Worker | `syncManager: SyncManager` |
| Backup | `backupManager: BackupManager` |
| Repository | `musicRepository: MusicRepository` |
| Context | `context: Context` (ApplicationContext) |

### `SetupUiState` (data class)

| フィールド | 目的 |
|----------|------|
| `mediaPermissionGranted: Boolean` | メディア読み取り権限 |
| `notificationsPermissionGranted: Boolean` | 通知権限 (Tiramisu+) |
| `alarmsPermissionGranted: Boolean` | 正確なアラーム権限 (S+) |
| `isLoadingDirectories: Boolean` | ディレクトリ読込中 |
| `blockedDirectories: Set<String>` | ブロックディレクトリ |
| `libraryNavigationMode: String` | ライブラリモード |
| `navBarStyle: String` | ナビバースタイル |
| `navBarCornerRadius: Int` | ナビバー角丸 |
| `appThemeMode: String` | アプリテーマ |
| `isInspectingBackup: Boolean` | バックアップ検査中 |
| `isRestoringBackup: Boolean` | リストア中 |
| `restorePlan: RestorePlan?` | リストア計画 |
| `backupTransferProgress: BackupTransferProgressUpdate?` | データ転送進捗 |

### `SetupEvent` (sealed interface)

| 派生 | 用途 |
|------|------|
| `Message(value)` | トースト |
| `RestoreCompleted(message)` | リストア成功 |

### StateFlow / SharedFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<SetupUiState>` | 画面状態 |
| `events` | `SharedFlow<SetupEvent>` | イベント |
| `isSyncing` | `StateFlow<Boolean>` | 同期中 |
| `fileExplorerStateHolder.currentPath` | 現在パス | (FileExplorer 委譲) |
| `currentDirectoryChildren`, `blockedDirectories`, `availableStorages`, `selectedStorageIndex`, `isLoadingDirectories`, `isExplorerPriming`, `isExplorerReady`, `isCurrentDirectoryResolved` | FileExplorer 公開 | (同上) |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun checkPermissions(context: Context)` | 3 つの権限を確認 (`ContextCompat.checkSelfPermission` / `AlarmManager.canScheduleExactAlarms`) |
| `fun loadMusicDirectories()` | ディレクトリセットアップ |
| `fun toggleDirectoryAllowed(file: File)` | ディレクトリトグル |
| `fun applyPendingDirectoryRuleChanges()` | 永続化 |
| `fun loadDirectory(file: File)` | 移動 |
| `fun selectStorage(index: Int)` | ストレージ選択 |
| `fun refreshAvailableStorages()` | 更新 |
| `fun refreshCurrentDirectory()` | 再読込 |
| `fun primeExplorer()` / `openExplorer()` / `navigateUp()` | ナビゲーション |
| `fun isAtRoot(): Boolean` / `fun explorerRoot(): File` | ナビゲーション情報 |
| `fun setLibraryNavigationMode(mode: String)` | 設定 |
| `fun setNavBarStyle(style: String)` | 設定 |
| `fun setNavBarCornerRadius(radius: Int)` | 設定 |
| `fun setAppThemeMode(mode: String)` | 設定 |
| `fun setSetupComplete()` | セットアップ完了フラグ |
| `fun retrySync()` | 同期再試行 |
| `fun inspectBackupFile(uri: Uri)` | バックアップファイル解析 |
| `fun updateRestorePlanSelection(modules: Set<BackupSection>)` | リストア計画更新 |
| `fun clearRestorePlan()` | 計画クリア |
| `fun restoreFromPlan(uri: Uri)` | リストア実行 |
| `fun completeSetup(syncAfter: Boolean)` | 完了処理 |
| `fun onCleared()` | クリーンアップ |

### 内部実装メモ

- **権限判定 (Android 13+)**: `READ_MEDIA_AUDIO` が必要 (Tiramisu+)。それ未満は `READ_EXTERNAL_STORAGE`。
- **AlarmManager.canScheduleExactAlarms()**: スリープタイマーの `setExactAndAllowWhileIdle` で必要。S+ で必須。
- **通知権限**: Tiramisu+ で `POST_NOTIFICATIONS`。
- **`isRestoringBackup` → `canFinishSetup`**: `result.succeeded.isNotEmpty() || !result.rolledBack` のいずれかでリストア成功判定。
- **同期トリガ**: `completeSetup(syncAfter = true)` で `syncManager.startSync()` を呼んでフル同期。
- **ファイルエクスプローラ**: `SettingsViewModel` と同様に内部で `FileExplorerStateHolder` を構築。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/preferences/UserPreferencesRepository.kt` (`initialSetupDoneFlow`)
- `app/src/main/java/com/theveloper/pixelplay/data/worker/SyncManager.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/SetupScreen.kt`

---

## StatsViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/StatsViewModel.kt` (196 行)
**アノテーション**: `@HiltViewModel`
**役割**: 統計画面。時間範囲別 (DAY / WEEK / MONTH / YEAR / ALL) の再生統計。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `playbackStatsRepository: PlaybackStatsRepository` |
| Repository | `musicRepository: MusicRepository` |

### `StatsUiState` (data class)

| フィールド | 目的 |
|----------|------|
| `selectedRange: StatsTimeRange` | 現在の時間範囲 |
| `isLoading: Boolean` | 読込中 |
| `isRefreshing: Boolean` | 再読込中 |
| `summary: PlaybackStatsSummary?` | サマリ |
| `availableRanges: List<StatsTimeRange>` | 利用可能範囲 |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<StatsUiState>` | 画面状態 |
| `weeklyOverview` | `StateFlow<PlaybackStatsSummary?>` | 週概要 (ホーム画面用キャッシュ) |
| `homeOverview` | `StateFlow<PlaybackStatsSummary?>` | ホーム概要 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun onRangeSelected(range: StatsTimeRange)` | 範囲切替 + 自動再計算 |
| `fun refreshWeeklyOverview()` | 週概要再計算 |
| `fun refreshHomeOverview()` | ホーム概要再計算 |
| `fun requestStatsRefresh()` | 再生イベント発火時に手動更新 |
| `fun forceRegenerateStats()` | 完全再生成 |
| `private fun observeStatsRefreshFlow()` | 自動更新 Flow 監視 |
| `private suspend fun loadSongs(): List<Song>` | DB から全曲取得 (キャッシュ) |

### 内部実装メモ

- **3 つのスコープで Summary を保持**: `uiState.summary` (選択中), `weeklyOverview` (週固定), `homeOverview` (ホーム画面用)。
- **`HomeOverviewRanges`**: `private val` で `listOf(StatsTimeRange.DAY, WEEK, MONTH, YEAR, ALL)` を保持。
- **キャッシュ**: `cachedSongs: List<Song>?` で DB 全曲を 1 回取得して再利用 (suspend 関数)。
- **再生イベント駆動**: `playbackStatsRepository.refreshEvents` を collect し、新しいイベントで関連 Summary を invalidate。
- **`hasListeningActivity()`**: Summary の listenedCount > 0 または listenedMs > 0 で `true`。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/stats/PlaybackStatsRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/stats/StatsTimeRange.kt` (詳細は `../05-presentation-ui/library-model-stats-utils.md`)
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/StatsScreen.kt`

---

## EqualizerViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/EqualizerViewModel.kt` (523 行)
**アノテーション**: `@HiltViewModel`
**役割**: イコライザ画面。10 バンド EQ + Bass Boost / Virtualizer / Loudness Enhancer + カスタムプリセット。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Engine | `equalizerManager: EqualizerManager` |
| Preferences | `equalizerPreferencesRepository: EqualizerPreferencesRepository` |
| Engine | `dualPlayerEngine: DualPlayerEngine` |
| Context | `context: Context` (ApplicationContext) |

### `EqualizerUiState` (data class)

| フィールド | 目的 |
|----------|------|
| `isEnabled: Boolean` | EQ 全体 ON/OFF |
| `currentPreset: EqualizerPreset` | 現在のプリセット |
| `bandLevels: List<Int>` | 10 バンドのゲイン (dB) |
| `editingPresetName: String?` | 編集中のカスタムプリセット名 |
| `bassBoostEnabled: Boolean` | Bass Boost |
| `bassBoostStrength: Float` | Bass Boost 強度 (0-1000) |
| `virtualizerEnabled: Boolean` | Virtualizer |
| `virtualizerStrength: Float` | Virtualizer 強度 (0-1000) |
| `loudnessEnhancerEnabled: Boolean` | Loudness Enhancer |
| `loudnessEnhancerStrength: Float` | Loudness Enhancer 強度 (0-1000) |
| `isBassBoostSupported: Boolean` | デバイスサポート |
| `isVirtualizerSupported: Boolean` | 〃 |
| `isLoudnessEnhancerSupported: Boolean` | 〃 |
| `viewMode: EqualizerViewMode` | SLIDERS / KNOBS |
| `isBassBoostDismissed: Boolean` | BB 初回非表示フラグ |
| `isVirtualizerDismissed: Boolean` | 〃 |
| `isLoudnessDismissed: Boolean` | 〃 |
| `customPresets: List<EqualizerPreset>` | カスタムプリセット |
| `pinnedPresetsNames: List<String>` | ピン留めプリセット (順序保持) |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<EqualizerUiState>` | 画面状態 |
| `systemVolume` | `StateFlow<Float>` | システム音量 (0-1) |

### 主要 public 関数

#### システム音量

| シグネチャ | 目的 |
|-----------|------|
| `fun setSystemVolume(percent: Float)` | 0-1 範囲で設定 |

#### プリセット / バンド

| シグネチャ | 目的 |
|-----------|------|
| `fun cycleViewMode()` | SLIDERS → KNOBS → ... |
| `fun setEnabled(enabled: Boolean)` | EQ 全体 ON/OFF |
| `fun toggleEqualizer()` | ON/OFF トグル |
| `fun selectPreset(preset: EqualizerPreset)` | プリセット選択 |
| `fun setBandLevel(bandIndex: Int, level: Int)` | -15 〜 +15 |
| `fun saveCurrentAsCustomPreset(name: String)` | カスタム保存 |
| `fun deleteCustomPreset(preset: EqualizerPreset)` | 削除 |
| `fun renameCustomPreset(oldName: String, newName: String)` | リネーム |
| `fun updateCustomPresetBands(presetName: String)` | 編集中バンドの保存 |
| `fun togglePinPreset(presetName: String)` | ピン留めトグル |
| `fun updatePinnedPresetsOrder(newOrder: List<String>)` | 順序変更 |
| `fun resetPinnedPresetsToDefault()` | デフォルト復元 |

#### Bass Boost / Virtualizer / Loudness

| シグネチャ | 目的 |
|-----------|------|
| `fun setBassBoostEnabled(enabled)` / `setBassBoostStrength(strength: Int)` | 0-1000 |
| `fun setVirtualizerEnabled(enabled)` / `setVirtualizerStrength(strength: Int)` | 0-1000 |
| `fun setLoudnessEnhancerEnabled(enabled)` / `setLoudnessEnhancerStrength(strength: Int)` | 0-1000 |
| `fun setBassBoostDismissed(dismissed)` | 初回非表示 |
| `fun setVirtualizerDismissed(dismissed)` | 〃 |
| `fun setLoudnessDismissed(dismissed)` | 〃 |

#### Audio Session 再アタッチ

| シグネチャ | 目的 |
|-----------|------|
| `fun reattachToPlayer()` | Audio Session ID が変わった場合に再バインド |

### 内部実装メモ

- **Audio Session ID**: `dualPlayerEngine.getAudioSessionId()` で取得し、Equalizer / BassBoost / Virtualizer / LoudnessEnhancer にアタッチ。
- **永続化デバウンス**: `persistBandLevelsJob` / `persistBassBoostJob` / `persistVirtualizerJob` / `persistLoudnessJob` の 4 つの Job で `SLIDER_PERSIST_DEBOUNCE_MS = 150L` のデバウンス。
- **全設定の `combine`**: `observeEqualizerState` で `enabled` / `preset` / `customBands` / `bassBoost*` / `virtualizer*` / `loudness*` / `bbDismissed` / `vDismissed` / `lDismissed` / `viewMode` / `customPresets` / `pinnedPresets` の 15+ Flow を `combine` し、`EqualizerUiState` を合成。
- **システム音量**: `AudioManager.getStreamVolume(STREAM_MUSIC)` を `0..max` で割って 0-1 化。
- **`editingPresetName`**: バンドを変更したカスタムプリセット名。一時的な編集マーカー。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/equalizer/EqualizerManager.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/equalizer/EqualizerPreset.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/preferences/EqualizerPreferencesRepository.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/EqualizerScreen.kt`

---

## DailyMixStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/DailyMixStateHolder.kt` (183 行)
**アノテーション**: `@Singleton`
**役割**: ホーム画面用の Daily Mix (日替わり) と Your Mix (永続) の保持 / 生成 / 永続化。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Engine | `dailyMixManager: DailyMixManager` |
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| Repository | `musicRepository: MusicRepository` |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `dailyMixSongs` | `StateFlow<ImmutableList<Song>>` | 今日の Daily Mix |
| `yourMixSongs` | `StateFlow<ImmutableList<Song>>` | あなたの Mix (永続) |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(coroutineScope: CoroutineScope)` | スコープ注入 + persisted mix 復元 |
| `fun removeFromDailyMix(songId: String)` | 1 曲除外 |
| `fun updateDailyMix(favoriteSongIdsFlow: Flow<Set<String>>)` | 再生成 |
| `fun loadPersistedDailyMix()` | DB から復元 |
| `fun forceUpdate(favoriteSongIdsFlow)` | 即時強制更新 |
| `fun checkAndUpdateIfNeeded(favoriteSongIdsFlow)` | 日付ベース判定で条件付き更新 |
| `fun setDailyMixSongs(songs: List<Song>)` | 強制設定 |
| `suspend fun getCandidatePool(songs, favoriteIds, maxSize, prompt?): List<Song>` | AI 再生成用候補プール |
| `fun onCleared()` | クリーンアップ |

### 内部実装メモ

- **日替わり判定**: `Calendar.getInstance().get(Calendar.DAY_OF_YEAR)` を `lastDailyMixUpdateFlow` のタイムスタンプと比較。異なる場合のみ更新。
- **Daily Mix 生成**: `dailyMixManager.generateDailyMix(allSongs, favoriteIds)` で 48 曲のプレイリスト。
- **Your Mix**: `generateYourMix(allSongs, favoriteIds)` でより大きな永続プレイリスト。
- **永続化**: `userPreferencesRepository.dailyMixSongIdsFlow` / `yourMixSongIdsFlow` に ID 列を保存。次回起動時に `getSongsByIds` で復元。
- **AI 再生成 (候補プール)**: `getCandidatePool` は `DailyMixManager.generateDailyMix` を再度呼んで AI プロンプト用の候補を抽出。
- **強制更新**: `forceUpdate` は日付判定をスキップして必ず再生成。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/DailyMixManager.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/HomeScreen.kt`

---

## ListeningStatsTracker

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ListeningStatsTracker.kt` (335 行)
**アノテーション**: `@Singleton`
**役割**: 再生セッションを追跡し、`PlaybackStatsRepository` に永続化。`playbackHistory` Flow で UI へ履歴を公開。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Engine | `dailyMixManager: DailyMixManager` (選好スコア用) |
| Repository | `playbackStatsRepository: PlaybackStatsRepository` |

### `ActiveSession` (private data class)

| フィールド | 目的 |
|----------|------|
| `songId: String` | 曲 ID |
| `totalDurationMs: Long` | 曲の総時間 |
| `startedAtEpochMs: Long` | 開始時刻 |
| `lastKnownPositionMs: Long` | 最後の位置 |
| `accumulatedListeningMs: Long` | 累積再生時間 |
| `lastRealtimeMs: Long` | 内部時計 |
| `lastUpdateEpochMs: Long` | 最終更新 |
| `isPlaying: Boolean` | 再生中か |
| `isVoluntary: Boolean` | ユーザー選択曲か |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `playbackHistory` | `StateFlow<List<PlaybackHistoryEntry>>` | 再生履歴 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(coroutineScope: CoroutineScope)` | スコープ注入 |
| `fun onVoluntarySelection(songId: String)` | ユーザー選択フラグ |
| `fun onSongChanged(songId, totalDurationMs?, fallbackDurationMs?)` | 曲変更イベント |
| `fun onTrackChanged(songId, isVoluntary, totalDurationMs, fallbackDurationMs)` | 同上 (別オーバーロード) |
| `fun onPlayStateChanged(isPlaying: Boolean, positionMs: Long)` | 再生/一時停止 |
| `fun onProgress(positionMs: Long, isPlaying: Boolean)` | ポジションチック (1 秒毎) |
| `fun updateDuration(durationMs: Long)` | 曲の duration 補正 |
| `fun finalizeCurrentSession(forceSynchronousPersistence: Boolean)` | 現セッションを永続化 |
| `fun onPlaybackStopped()` | 停止時 |
| `fun onCleared()` | クリーンアップ |

### 内部実装メモ

- **`MIN_SESSION_LISTEN_MS = 5 秒`**: 5 秒未満のセッションは履歴に追加しない。
- **`MAX_INTERNAL_PLAYBACK_HISTORY_ITEMS = 500`**: 内部 StateFlow の履歴保持上限 (FIFO 削除)。
- **`isVoluntary`**: ユーザー操作 (曲タップ / アルバムタップ) で開始された再生のみ true。スキップ / 自動遷移は false。
- **Duration 解決**: `normalizeDuration(durationMs, fallbackDurationMs)` で 0 / 不正値を除外し、有効値を採用。
- **永続化タイミング**:
  - 曲変更時 (`onTrackChanged` / `onSongChanged`)
  - 停止時 (`onPlaybackStopped`)
  - 5 秒 tick の `onProgress` で `accumulatedListeningMs` 増分
- **PersistenceScope**: 専用の `SupervisorJob() + Dispatchers.IO` スコープで永続化を実行 (失敗が他セッションに伝播しない)。
- **履歴復元**: `playbackStatsRepository.getRecentHistory(MAX)` で DB から最新を読み出し。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/stats/PlaybackStatsRepository.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/StatsScreen.kt` / `RecentlyPlayedScreen.kt`
