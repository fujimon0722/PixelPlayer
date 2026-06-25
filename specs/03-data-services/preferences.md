# Preferences

DataStore Preferences をバックエンドとする設定値の読み書きレイヤー。各種 `PreferencesRepository` と、UI / バリュークラス / enum 定義を含む。

## パッケージ

`com.theveloper.pixelplay.data.preferences`

---

## 依存関係

### 上流
- `presentation/viewmodel/SettingsViewModel.kt`
- `presentation/viewmodel/PlayerViewModel.kt`
- `presentation/viewmodel/LibraryViewModel.kt`
- `data/repository/MusicRepositoryImpl.kt` — 設定値の Flow を購読
- `data/worker/SyncWorker.kt` — `getLastSyncTimestamp()` / `allowedDirectoriesFlow` 等を pull
- `data/diagnostics/AdvancedPerformanceDiagnosticsController.kt` — 設定値を購読してセッションを制御
- `data/diagnostics/DebugPerformanceReportCollector.kt` — 設定値を pull
- `data/equalizer/EqualizerManager.kt` — `EqualizerManager.restoreState(...)` の呼び出し元

### 下流
- `androidx.datastore.preferences.core.Preferences`
- `data/database/LocalPlaylistDao.kt` (Playlist)
- `data/database/EngagementDao.kt`, `AiUsageDao.kt` (バックアップ経由)

---

## ファイル一覧

| ファイル | 行 | 役割 |
|---|---|---|
| `UserPreferencesRepository.kt` | 1389 | メイン設定ストア（ほぼ全設定値の SSOT） |
| `ThemePreferencesRepository.kt` | 74 | テーマ / アルバムアート配色専用 |
| `EqualizerPreferencesRepository.kt` | 276 | EQ 専用（DataStore 直接） |
| `PlaylistPreferencesRepository.kt` | 239 | プレイリスト操作（Room DAO + DataStore 設定） |
| `AiPreferencesRepository.kt` | 215 | AI プロバイダ別 API キー等 |
| `PreferenceBackupEntry.kt` | 14 | バックアップ用 DTO |
| `AlbumArtColorAccuracy.kt` | 10 | 数値定数 |
| `AlbumArtPaletteStyle.kt` | 19 | enum |
| `AppLanguage.kt` | 45 | enum (locale tag) |
| `CarouselStyle.kt` | 7 | const String |
| `CollagePattern.kt` | 20 | enum |
| `EqualizerViewMode.kt` | 5 | enum |
| `FullPlayerLoadingTweaks.kt` | 15 | data class |
| `LaunchTab.kt` | 7 | const String |
| `LibraryNavigationMode.kt` | 6 | const String |
| `NavBarStyle.kt` | 6 | const String |
| `TelegramTopicDisplayMode.kt` | 17 | enum |

---

## 拡張 / 定数

### `Context.dataStore` (`UserPreferencesRepository.kt:35`)

```kotlin
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")
```

全 PreferencesRepository がこの DataStore を共有する。

### `AlbumArtQuality` enum (`UserPreferencesRepository.kt:64`)

| 定数 | maxSize | label |
|---|---|---|
| LOW | 256 | "Low (256px) - Better performance" |
| MEDIUM | 512 | "Medium (512px) - Balanced" |
| HIGH | 800 | "High (800px) - Best quality" |
| ORIGINAL | 0 | "Original - Maximum quality" |

### `AdvancedPerformanceDiagnosticsSettings` data class (`UserPreferencesRepository.kt:71`)

`enabled: Boolean`, `sessionStartedEpochMs: Long?`, `expiresAtEpochMs: Long?`。
派生 `isActive(nowEpochMs)` で `enabled && expiresAt > now` を判定。

### `MIN_NAV_BAR_CORNER_RADIUS = 0` / `MAX = 60` (`UserPreferencesRepository.kt:50-51`)
`sanitizeNavBarCornerRadius(radius)` で `[0, 60]` にクランプ。

### `ThemePreference`, `AppThemeMode` const Strings (`UserPreferencesRepository.kt:37-48`)
UI から文字列で扱うテーマ識別子。

### `DEFAULT_ARTIST_DELIMITERS = listOf(";")` (`UserPreferencesRepository.kt:1368`)

旧 `LEGACY_DEFAULT_ARTIST_DELIMITERS = listOf("/", ";", ",", "+", "&")` からマイグレート。

### `DEFAULT_ARTIST_WORD_DELIMITERS` (`UserPreferencesRepository.kt:1373`)

`["featuring", "feat.", "feat", "ft.", "ft", "vs.", "vs", "versus", "with", "prod.", "prod"]`

### `DEFAULT_ALBUM_ART_CACHE_LIMIT_MB = 200`

### `backupExcludedKeyNames = {"app_rebrand_dialog_shown"}` (`UserPreferencesRepository.kt:86`)

バックアップ時に除外するキー（オンボーディング用フラグ）。

---

## `UserPreferencesRepository` (`UserPreferencesRepository.kt:80`)

全 DataStore を背負う SSOT レポジトリ。`@Singleton` / `@Inject` で `DataStore<Preferences>` と `kotlinx.serialization.json.Json` を受ける。

### 主要 Flow プロパティ（主要部のみ。全体は 1389 行）

| Flow プロパティ | 行 | デフォルト / 型 | 用途 |
|---|---|---|---|
| `appRebrandDialogShownFlow` | 280 | `false: Boolean` | リブランドダイアログ |
| `beta05CleanInstallDisclaimerDismissedFlow` | 287 | `false: Boolean` | β 警告 |
| `backupInfoDismissedFlow` | 294 | `false: Boolean` | バックアップ説明 |
| `initialSetupDoneFlow` | 301 | `false: Boolean` | 初期セットアップ完了 |
| `repeatModeFlow` | 310 | `Player.REPEAT_MODE_OFF: Int` | 再生モード |
| `isShuffleOnFlow` | 317 | `false: Boolean` | シャッフル |
| `persistentShuffleEnabledFlow` | 324 | `false: Boolean` | 永続シャッフル |
| `isCrossfadeEnabledFlow` | 331 | `false: Boolean` | クロスフェード |
| `crossfadeDurationFlow` | 338 | `2000 ms (clamp 1000..12000)` | クロスフェード時間 |
| `hiFiModeEnabledFlow` | 345 | `false: Boolean` | Hi-Fi モード |
| `keepPlayingInBackgroundFlow` | 352 | `true: Boolean` | バックグラウンド再生 |
| `disableCastAutoplayFlow` | 359 | `false: Boolean` | Cast 自動再生 |
| `resumeOnHeadsetReconnectFlow` | 366 | `false: Boolean` | ヘッドセット再接続時再開 |
| `showQueueHistoryFlow` | 373 | `false: Boolean` | キュー履歴 |
| `playbackQueueSnapshotFlow` | 380 | `PlaybackQueueSnapshot?` (JSON) | 永続キュー |
| `showPlayerFileInfoFlow` | 402 | `true: Boolean` | ファイル情報表示 |
| `fullPlayerLoadingTweaksFlow` | 409 | `FullPlayerLoadingTweaks` | フルプレイヤー表示遅延 |
| `globalTransitionSettingsFlow` | 499 | `TransitionSettings` (durationMs=crossfadeDuration) | グローバルトランジション |
| `favoriteSongIdsFlow` | 512 | `Set<String>` | お気に入り ID |
| `playlistSongOrderModesFlow` | 542 | `Map<String, String>` (JSON) | プレイリスト別再生順 |
| `legacyUserPlaylistsFlow` | 566 | `List<Playlist>` (JSON) | 旧プレイリスト（マイグレ用） |
| `allowedDirectoriesFlow` | 579 | `Set<String>` (distinctUntilChanged) | 許可ディレクトリ |
| `blockedDirectoriesFlow` | 582 | `Set<String>` (distinctUntilChanged) | ブロックディレクトリ |
| `lastSyncTimestampFlow` | 606 | `Long` (0L) | 最終同期時刻 |
| `directoryRulesVersionFlow` | 609 | `Int` (0) | ディレクトリルールバージョン |
| `lastAppliedDirectoryRulesVersionFlow` | 612 | `Int` (0) | 適用済バージョン |
| `dailyMixSongIdsFlow` | 628 | `List<String>` (JSON) | デイリーミックス ID |
| `yourMixSongIdsFlow` | 648 | `List<String>` (JSON) | ユアミックス ID |
| `isGenreGridViewFlow` | 668 | `true: Boolean` | ジャンルグリッド表示 |
| `isAlbumsListViewFlow` | 679 | `false: Boolean` | アルバムリスト表示 |
| `lastDailyMixUpdateFlow` | 690 | `Long` (0L) | デイリーミックス最終更新 |
| `minSongDurationFlow` | 701 | `10000 ms (clamp 0..120000)` | 最小再生時間 |
| `minTracksPerAlbumFlow` | 716 | `1: Int` | アルバム内最低トラック数 |
| `replayGainEnabledFlow` | 727 | `false: Boolean` | ReplayGain |
| `replayGainUseAlbumGainFlow` | 732 | `false: Boolean` | アルバムゲイン使用 |
| `pauseOnVolumeZeroFlow` | 751 | `false: Boolean` | 音量 0 で一時停止 |
| `showScrollbarFlow` | 762 | `true: Boolean` | スクロールバー |
| `songsSortOptionFlow` | 775 | `SongTitleAZ` | 楽曲ソート |
| `albumsSortOptionFlow` | 778 | `AlbumTitleAZ` | アルバムソート |
| `artistsSortOptionFlow` | 781 | `ArtistNameAZ` | アーティストソート |
| `playlistsSortOptionFlow` | 784 | `PlaylistNameAZ` | プレイリストソート |
| `foldersSortOptionFlow` | 787 | `FolderNameAZ` | フォルダソート |
| `likedSongsSortOptionFlow` | 790 | `LikedSongDateLiked` | お気に入りソート |
| `lastLibraryTabIndexFlow` | 856 | `0: Int` | 最後に開いたタブ |
| `lastStorageFilterFlow` | 863 | `StorageFilter.ALL` | 最終ストレージフィルタ |
| `mockGenresEnabledFlow` | 876 | `false: Boolean` | モックジャンル |
| `libraryTabsOrderFlow` | 883 | `String?` | タブ順 (JSON) |
| `isFolderFilterActiveFlow` | 909 | `false: Boolean` | フォルダフィルタ |
| `isFoldersPlaylistViewFlow` | 916 | `false: Boolean` | フォルダ = プレイリスト表示 |
| `showTelegramCloudPlaylistsFlow` | 923 | `true: Boolean` | Telegram クラウド表示 |
| `hideLocalMediaFlow` | 930 | `false: Boolean` | ローカル非表示 |
| `telegramTopicDisplayModeFlow` | 937 | `TelegramTopicDisplayMode` | トピック表示モード |
| `foldersSourceFlow` | 944 | `FolderSource` | フォルダソース |
| `folderBackGestureNavigationFlow` | 951 | `true: Boolean` | バックジェスチャ |
| `navBarCornerRadiusFlow` | 960 | `32 (clamp 0..60)` | ナビバー角丸 |
| `navBarStyleFlow` | 967 | `NavBarStyle.DEFAULT` | ナビバースタイル |
| `navBarCompactModeFlow` | 974 | `false: Boolean` | ナビバーコンパクト |
| `libraryNavigationModeFlow` | 981 | `LibraryNavigationMode.TAB_ROW` | ライブラリナビ |
| `carouselStyleFlow` | 988 | `CarouselStyle.NO_PEEK` | カルーセルスタイル |
| `launchTabFlow` | 995 | `LaunchTab.HOME` | 起動タブ |
| `useSmoothCornersFlow` | 1002 | `false: Boolean` | スムーズコーナー |
| `artistDelimitersFlow` | 1011 | `[";"]` | アーティスト区切り（レガシーマイグレ付き） |
| `artistWordDelimitersFlow` | 1028 | `["featuring", "feat.", ...]` | アーティストワード区切り |
| `extractArtistsFromTitleFlow` | 1040 | `true: Boolean` | タイトルからアーティスト抽出 |
| `groupByAlbumArtistFlow` | 1050 | `false: Boolean` | アルバムアーティストでグループ化 |
| `artistSettingsRescanRequiredFlow` | 1060 | `false: Boolean` | アーティスト設定変更後の再スキャン要求 |
| `lyricsSyncOffsetsFlow` (private) | 1073 | `Map<String, Int>` | 歌詞同期オフセット |
| `lyricsSourcePreferenceFlow` | 1088 | `LyricsSourcePreference` | 歌詞ソース優先度 |
| `autoScanLrcFilesFlow` | 1095 | `false: Boolean` | .lrc 自動スキャン |
| `immersiveLyricsEnabledFlow` | 1102 | `false: Boolean` | 没入歌詞 |
| `immersiveLyricsTimeoutFlow` | 1109 | `4000 ms: Long` | 没入歌詞タイムアウト |
| `useAnimatedLyricsFlow` | 1116 | `false: Boolean` | アニメーション歌詞 |
| `animatedLyricsBlurEnabledFlow` | 1123 | `true: Boolean` | アニメーション歌詞ぼかし |
| `animatedLyricsBlurStrengthFlow` | 1130 | `2.5f: Float` | ぼかし強度 |
| `customGenresFlow` | 1139 | `Set<String>` | カスタムジャンル |
| `customGenreIconsFlow` | 1142 | `Map<String, Int>` | カスタムジャンルアイコン |
| `disableBlurAllOverFlow` | 1159 | `false: Boolean` | 全体ぼかし無効 |
| `collagePatternFlow` | 1172 | `CollagePattern.COSMIC_SWIRL` | コラージュパターン |
| `collageAutoRotateFlow` | 1179 | `false: Boolean` | コラージュ自動回転 |
| `lastPlaylistIdFlow` | 1188 | `String?` | 最後のプレイリスト ID |
| `lastPlaylistNameFlow` | 1191 | `String?` | 最後のプレイリスト名 |
| `albumArtQualityFlow` | 1214 | `AlbumArtQuality.MEDIUM` | アート品質 |
| `albumArtCacheLimitMbFlow` | 1225 | `200 MB (clamp 50..1500)` | アートキャッシュ上限 |
| `tapBackgroundClosesPlayerFlow` | 1233 | `false: Boolean` | 背景タップで閉じる |
| `hapticsEnabledFlow` | 1240 | `true: Boolean` | ハプティクス |
| `advancedPerformanceDiagnosticsSettingsFlow` | 1249 | `AdvancedPerformanceDiagnosticsSettings` | 詳細診断セッション |
| `playerThemePreferenceFlow` (ThemePreferences) | 28 | `ThemePreference.ALBUM_ART` | プレイヤーテーマ |
| `appThemeModeFlow` (ThemePreferences) | 24 | `AppThemeMode.FOLLOW_SYSTEM` | アプリテーマ |

### 主要 suspend set メソッド（代表）

| メソッド | 行 | 目的 |
|---|---|---|
| `setCrossfadeEnabled` | 334 | クロフェ ON/OFF |
| `setCrossfadeDuration(duration)` | 341 | clamp 1000..12000 |
| `setKeepPlayingInBackground` | 355 | バックグラウンド再生 |
| `setFavoriteSong(songId, isFavorite)` | 519 | 旧 DataStore のお気に入り追加 / 削除 |
| `toggleFavoriteSong(songId)` | 528 | 旧 DataStore の反転 |
| `clearFavoriteSongIds()` | 536 | 全消去 |
| `setPlaylistSongOrderMode` / `setPlaylistSongOrderModes` / `clearPlaylistSongOrderMode` | 547-563 | プレイリスト別再生順 |
| `updateAllowedDirectories` / `updateDirectorySelections` | 585-602 | ディレクトリ変更 + バージョン bump |
| `markDirectoryRulesVersionApplied` | 624 | 適用バージョン記録 |
| `saveDailyMixSongIds` / `saveYourMixSongIds` | 642-665 | ミックス保存 |
| `setGenreGridView` / `setAlbumsListView` | 673-687 | 表示モード |
| `saveLastDailyMixUpdateTimestamp` | 695 | 更新タイムスタンプ |
| `setMinSongDuration` / `setMinTracksPerAlbum` | 706-724 | フィルタ |
| `setReplayGainEnabled` / `setReplayGainUseAlbumGain` | 737-746 | ReplayGain |
| `setPauseOnVolumeZero` / `setShowScrollbar` | 756-769 | UI |
| `setSongsSortOption` 等 6 種 | 793-817 | ソート |
| `ensureLibrarySortDefaults()` | 820 | 起動時のマイグレ |
| `migrateSortPreference(...)` (private) | 844 | 不正値を許容値に置換 |
| `saveLastStorageFilter` | 872 | StorageFilter |
| `setLibraryTabsOrder` / `resetLibraryTabsOrder` / `migrateTabOrder` | 886-906 | タブ順 |
| `setFoldersSource` | 947 | FolderSource |
| `setNavBarCornerRadius` | 963 | clamp 0..60 |
| `setLibraryNavigationMode` / `setCarouselStyle` / `setLaunchTab` / `setUseSmoothCorners` | 984-1006 | UI |
| `setArtistDelimiters` / `setArtistWordDelimiters` | 1018-1037 | `ARTIST_SETTINGS_RESCAN_REQUIRED=true` も同時セット |
| `setExtractArtistsFromTitle` / `setGroupByAlbumArtist` | 1043-1057 | 同上 |
| `clearArtistSettingsRescanRequired` | 1063 | 再スキャン要求クリア |
| `setLyricsSyncOffset(songId, offsetMs)` | 1082 | `editJsonMap`、0 なら remove |
| `setLyricsSourcePreference` | 1091 | 優先度 |
| `setAutoScanLrcFiles` | 1098 | 自動 LRC |
| `setImmersiveLyricsEnabled` / `setImmersiveLyricsTimeout` | 1105-1113 | 没入歌詞 |
| `setUseAnimatedLyrics` / `setAnimatedLyricsBlurEnabled` / `setAnimatedLyricsBlurStrength` | 1119-1133 | アニメーション歌詞 |
| `addCustomGenre(genre, iconResId?)` | 1145 | ジャンル + アイコン同時追加 |
| `setCollagePattern` / `setCollageAutoRotate` | 1175-1183 | コラージュ |
| `setLastPlaylist` / `clearLastPlaylist` | 1194-1206 | 最終プレイリスト |
| `setAlbumArtQuality` / `setAlbumArtCacheLimitMb` | 1221-1229 | アート品質 |
| `setTapBackgroundClosesPlayer` / `setHapticsEnabled` | 1236-1244 | UI |
| `setAdvancedPerformanceDiagnosticsEnabled` | 1260 | 24h セッション開始 |
| `disableExpiredAdvancedPerformanceDiagnostics` | 1276 | 期限切れ自動 OFF |
| `clearPreferencesByKeys(keyNames)` | 1288 | 特定キーのみ削除 |
| `clearPreferencesExceptKeys(excludedKeyNames)` | 1300 | 保護キー以外削除 |
| `exportPreferencesForBackup()` | 1312 | 全 DataStore を `PreferenceBackupEntry` の List に変換 |
| `importPreferencesFromBackup(entries, clearExisting)` | 1332 | バックアップから復元 |
| `setFullPlayerPlaceholders` / `setTransparentPlaceholders` / `setFullPlayerPlaceholdersOnClose` | 461-473 | フルプレイヤー |
| `setFullPlayerSwitchOnDragRelease` / `setFullPlayerAppearThreshold` / `setFullPlayerCloseThreshold` | 476-489 | フルプレイヤー |
| `setDelayAllFullPlayerContent` | 436 | 4 フラグ同時 |
| `clearDeprecatedPlayerSheetPreference` | 493 | 旧 player sheet v2 削除 |
| `getLastSyncTimestamp` / `getDirectoryRulesVersion` / `getLastAppliedDirectoryRulesVersion` | 615-618 | `.first()` ヘルパ |
| `getLyricsSyncOffset(songId)` | 1079 | 単曲オフセット取得 |
| `getMinSongDuration()` | 712 | `.first()` ヘルパ |
| `getLegacyUserPlaylistsOnce()` | 571 | 旧プレイリスト |

### 内部実装メモ

- **Flow → DataStore**: 共通ヘルパ `pref { ... }` (`UserPreferencesRepository.kt:254`) で `dataStore.data.map(transform)` を一行化。
- **JSON decode**: `decodeJsonPref<T>(preferences, key, default)` (`UserPreferencesRepository.kt:258`) は失敗時に `default` を返す fail-safe。
- **JSON Map 編集**: `editJsonMap<V>(key) { ... }` (`UserPreferencesRepository.kt:267`) で読み出し→編集→保存を一発。
- **ソートマイグレーション**: `ensureLibrarySortDefaults()` (`UserPreferencesRepository.kt:820`) で `SONGS_SORT_OPTION_MIGRATED` フラグを使って Z→A や displayName を AZ に書き換え。
- **バージョン管理**: ディレクトリ変更時に `directoryRulesVersion` を incrementWrapped で bump し、`lastAppliedDirectoryRulesVersion` との比較で sync を促す。
- **バックアップ互換性**: `exportPreferencesForBackup` (`UserPreferencesRepository.kt:1312`) は `backupExcludedKeyNames` を除外、`Set<*>` を `string_set` 型として扱う。

---

## `ThemePreferencesRepository` (`ThemePreferencesRepository.kt:14`)

`@Singleton` / `@Inject` で `DataStore<Preferences>` を受ける。

### Flow プロパティ

| プロパティ | 行 | デフォルト | 用途 |
|---|---|---|---|
| `appThemeModeFlow` | 24 | `AppThemeMode.FOLLOW_SYSTEM` | アプリ全体テーマ |
| `playerThemePreferenceFlow` | 28 | `ThemePreference.ALBUM_ART` | プレイヤーテーマ |
| `albumArtPaletteStyleFlow` | 32 | `AlbumArtPaletteStyle.fromStorageKey` | アルバムアート配色 |
| `albumArtColorAccuracyFlow` | 36 | `AlbumArtColorAccuracy.clamp(...)` (DEFAULT=0) | 色精度 |

### suspend set メソッド

| メソッド | 行 |
|---|---|
| `setPlayerThemePreference(themeMode)` | 40 |
| `setAppThemeMode(themeMode)` | 45 |
| `initializeAppThemeMode(themeMode)` | 50 (null の場合のみ設定) |
| `setAlbumArtPaletteStyle(style)` | 57 |
| `setAlbumArtColorAccuracy(level)` | 62 (clamp 0..10) |
| `setAlbumArtPaletteSettings(style, accuracyLevel)` | 67 (同時セット) |

---

## `EqualizerPreferencesRepository` (`EqualizerPreferencesRepository.kt:20`)

`@Singleton` / `@Inject` で `DataStore<Preferences>` と `Json` を受ける。**独自の DataStore は使わず、共有 `settings` DataStore を直接読み書き**。

### Flow プロパティ

| プロパティ | 行 | デフォルト | 用途 |
|---|---|---|---|
| `equalizerViewModeFlow` | 42 | `EqualizerViewMode.SLIDERS`（`is_graph_view` 旧キー互換） | 表示モード |
| `equalizerEnabledFlow` | 56 | `false: Boolean` | EQ 有効 |
| `equalizerPresetFlow` | 60 | `"flat"` | プリセット名 |
| `equalizerCustomBandsFlow` | 64 | `List(10) { 0 }` (JSON) | カスタムバンド |
| `bassBoostStrengthFlow` | 82 | `0: Int` | BassBoost 強度 |
| `virtualizerStrengthFlow` | 86 | `0: Int` | Virtualizer 強度 |
| `bassBoostEnabledFlow` | 90 | `false: Boolean` | BassBoost 有効 |
| `virtualizerEnabledFlow` | 94 | `false: Boolean` | Virtualizer 有効 |
| `loudnessEnhancerEnabledFlow` | 98 | `false: Boolean` | LoudnessEnhancer 有効 |
| `loudnessEnhancerStrengthFlow` | 102 | `0 (clamp 0..1000)` | LE 強度 |
| `bassBoostDismissedFlow` | 106 | `false: Boolean` | BB 警告 |
| `virtualizerDismissedFlow` | 110 | `false: Boolean` | V 警告 |
| `loudnessDismissedFlow` | 114 | `false: Boolean` | LE 警告 |
| `customPresetsFlow` | 118 | `List<EqualizerPreset>` (JSON) | カスタムプリセット |
| `pinnedPresetsFlow` | 131 | `ALL_PRESETS.map { it.name }` | ピン留め |

### suspend set メソッド

| メソッド | 行 | 目的 |
|---|---|---|
| `setEqualizerViewMode(mode)` | 144 | 表示モード |
| `setEqualizerEnabled(enabled)` | 149 | ON/OFF |
| `setEqualizerPreset(preset)` | 154 | プリセット名 |
| `setEqualizerCustomBands(bands)` | 159 | 10 要素に正規化 |
| `setBassBoostStrength` | 169 | clamp 0..1000 |
| `setVirtualizerStrength` | 174 | clamp 0..1000 |
| `setBassBoostEnabled` / `setVirtualizerEnabled` / `setLoudnessEnhancerEnabled` | 179-189 | ON/OFF |
| `setLoudnessEnhancerStrength` | 194 | clamp 0..1000 |
| `setBassBoostDismissed` / `setVirtualizerDismissed` / `setLoudnessDismissed` | 199-209 | 警告フラグ |
| `setPinnedPresets(presetNames)` | 214 | ピン留め更新 |
| `saveCustomPreset(preset)` | 219 | 既存同名削除 → 追加 |
| `deleteCustomPreset(presetName)` | 228 | 削除 + ピン留めからも除去 |
| `renameCustomPreset(oldName, newName)` | 241 | 名前変更 + activePreset とピン留め追従 |
| `updateCustomPresetBands(presetName, bandLevels)` | 266 | バンド値のみ更新 |

---

## `PlaylistPreferencesRepository` (`PlaylistPreferencesRepository.kt:19`)

`@Singleton` / `@Inject` で `LocalPlaylistDao` (Room) と `UserPreferencesRepository` (DataStore) を受ける。

### Flow プロパティ

| プロパティ | 行 | 型 | 用途 |
|---|---|---|---|
| `userPlaylistsFlow` | 32 | `Flow<List<Playlist>>` | プレイリスト一覧（Room 監視） |
| `playlistSongOrderModesFlow` | 42 | `Map<String, String>` | 再生順（DataStore） |
| `playlistsSortOptionFlow` | 44 | `String` | ソート |
| `showTelegramCloudPlaylistsFlow` | 45 | `Boolean` | Telegram 表示 |
| `telegramTopicDisplayModeFlow` | 47 | `TelegramTopicDisplayMode` | トピック表示モード |

### 主要 suspend メソッド

| メソッド | 行 | 目的 |
|---|---|---|
| `setTelegramTopicDisplayMode(mode)` | 49 | プロキシ |
| `createPlaylist(...)` | 52 | プレイリスト生成（`UUID`、ソート順設定可、Source 既定 `"LOCAL"`） |
| `deletePlaylist(playlistId)` | 93 | 削除 + 再生順設定クリア |
| `renamePlaylist(playlistId, newName)` | 99 | `editMutex` 保護 |
| `updatePlaylist(playlist)` | 111 | `editMutex` 保護 |
| `addSongsToPlaylist(playlistId, songIds)` | 126 | 既存とマージ |
| `addOrRemoveSongFromPlaylists(songId, playlistIds)` | 135 | 一括追加 / 削除 |
| `removeSongFromPlaylist(playlistId, songId)` | 156 | 単曲削除 |
| `reorderSongsInPlaylist(playlistId, newOrderIds)` | 164 | 並び替え |
| `setPlaylistSongOrderMode / setPlaylistSongOrderModes` | 172-178 | DataStore |
| `clearPlaylistSongOrderMode` | 175 | DataStore |
| `setPlaylistsSortOption` | 181 | DataStore |
| `setShowTelegramCloudPlaylists` | 184 | DataStore |
| `getPlaylistsOnce()` | 187 | `.first()` ヘルパ |
| `replaceAllPlaylists(playlists)` | 192 | **トランザクション置換**（バックアップ復元用）+ `clearLegacyUserPlaylists` |
| `removeSongFromAllPlaylists(songId)` | 200 | 全プレイリストから削除 |
| `resetPlaylistPreferencesToDefaults` | 216 | 再生順クリア + デフォルトソート |

### 内部実装メモ

- **editMutex** (`PlaylistPreferencesRepository.kt:28`): 読み書き競合を防ぐため `addSongsToPlaylist` / `removeSongFromPlaylist` 等を直列化（issue #2391 修正）。
- **migrationMutex** (`PlaylistPreferencesRepository.kt:23`): 旧 DataStore プレイリストから Room への 1 回限りのマイグレーションを `ensureMigratedIfNeeded` (`PlaylistPreferencesRepository.kt:221`) で実施。
- **userPlaylistsFlow の `onStart`** (`PlaylistPreferencesRepository.kt:33`): Flow 購読開始時にマイグレーションを実行。

---

## `AiPreferencesRepository` (`AiPreferencesRepository.kt:17`)

`@Singleton` / `@Inject` で `DataStore<Preferences>` を受ける。AI プロバイダごとに API キー / モデル / システムプロンプト / BaseURL を保持。

### キー命名規則

```kotlin
Keys.getApiKey(provider)     = stringPreferencesKey("${provider.name.lowercase()}_api_key")
Keys.getModel(provider)      = stringPreferencesKey("${provider.name.lowercase()}_model")
Keys.getSystemPrompt(provider) = stringPreferencesKey("${provider.name.lowercase()}_system_prompt")
Keys.getBaseUrl(provider)    = stringPreferencesKey("${provider.name.lowercase()}_base_url")
```

### デフォルトシステムプロンプト

`DEFAULT_SYSTEM_PROMPT = """You are 'Vibe-Engine', ...` で全プロバイダ共有（`AiPreferencesRepository.kt:21-25`）。各プロバイダ用に `DEFAULT_<PROVIDER>_SYSTEM_PROMPT` が同じ値を指す。

### Flow プロパティ（一部）

| プロパティ | 行 | デフォルト |
|---|---|---|
| `aiProvider` | 139 | `"GEMINI"` |
| `isSafeTokenLimitEnabled` | 142 | `true: Boolean` |
| `aiTemperature` | 145 | `0.7f: Float` |
| `aiTopP` | 148 | `0.95f: Float` |
| `aiTopK` | 151 | `64: Int` |
| `aiMaxTokens` | 154 | `4096: Int` |
| `aiPresencePenalty` | 157 | `0.0f: Float` |
| `aiFrequencyPenalty` | 160 | `0.0f: Float` |
| `aiSampleSize` | 163 | `40: Int` |
| `aiDigestMode` | 166 | `"safe": String` |
| `aiIncludeExtendedFields` | 169 | `false: Boolean` |

### プロバイダ別 Flow プロパティ（例）

`geminiApiKey` / `geminiModel` / `geminiSystemPrompt` (`AiPreferencesRepository.kt:94-96`)、`deepseekApiKey` 等、計 11 プロバイダ分（GEMINI, DEEPSEEK, GROQ, MISTRAL, NVIDIA, KIMI, GLM, OPENAI, OPENROUTER, OLLAMA, CUSTOM）。

### suspend set メソッド

`setApiKey`, `setModel`, `setSystemPrompt`, `resetSystemPrompt`, `setBaseUrl` (`AiPreferencesRepository.kt:71-91`) が汎用。
`setAiProvider`, `setSafeTokenLimitEnabled`, `setAiTemperature`, `setAiTopP`, `setAiTopK`, `setAiMaxTokens`, `setAiPresencePenalty`, `setAiFrequencyPenalty`, `setAiSampleSize`, `setAiDigestMode`, `setAiIncludeExtendedFields` (`AiPreferencesRepository.kt:172-214`) がグローバル設定。

---

## enum / data class / const

| ファイル | 種類 | 主な定数 / 列挙 |
|---|---|---|
| `PreferenceBackupEntry.kt:3` | data class | `key`, `type` (`string/int/long/boolean/float/double/string_set`), `stringValue`, `intValue`, `longValue`, `booleanValue`, `floatValue`, `doubleValue`, `stringSetValue` |
| `AlbumArtColorAccuracy.kt:3` | object | `MIN=0`, `MAX=10`, `DEFAULT=MIN`, `STEPS=MAX-MIN-1`, `clamp(value)` |
| `AlbumArtPaletteStyle.kt:3` | enum | `TONAL_SPOT("tonal_spot", ...)`, `VIBRANT`, `EXPRESSIVE`, `FRUIT_SALAD`, `default = TONAL_SPOT` |
| `AppLanguage.kt:7` | enum | `SYSTEM("")`, `ENGLISH("en")`, `GERMAN("de")`, `SPANISH("es")`, `FRENCH("fr")`, `INDONESIAN("in")`, `ITALIAN("it")`, `KOREAN("ko")`, `NORWEGIAN_BOKMAL("nb")`, `RUSSIAN("ru")`, `SIMPLIFIED_CHINESE("zh-CN")`, `JAPANESE("ja")`, `TURKISH("tr")` |
| `CarouselStyle.kt:3` | object | `NO_PEEK`, `ONE_PEEK`, `TWO_PEEK` |
| `CollagePattern.kt:3` | enum | `COSMIC_SWIRL`, `HONEYCOMB_GROOVE`, `VINYL_STACK`, `PIXEL_MOSAIC`, `STARDUST_SCATTER` |
| `EqualizerViewMode.kt:3` | enum | `SLIDERS, GRAPH, HYBRID` |
| `FullPlayerLoadingTweaks.kt:3` | data class | `delayAll`, `delayAlbumCarousel`, `delaySongMetadata`, `delayProgressBar`, `delayControls`, `showPlaceholders`, `transparentPlaceholders`, `applyPlaceholdersOnClose`, `switchOnDragRelease`, `contentAppearThresholdPercent`, `contentCloseThresholdPercent` |
| `LaunchTab.kt:3` | object | `HOME`, `SEARCH`, `LIBRARY` |
| `LibraryNavigationMode.kt:3` | object | `TAB_ROW`, `COMPACT_PILL` |
| `NavBarStyle.kt:3` | object | `DEFAULT`, `FULL_WIDTH` |
| `TelegramTopicDisplayMode.kt:3` | enum | `CHANNELS_ONLY("channels_only")`, `TOPICS_ONLY("topics_only")`, `CHANNELS_AND_TOPICS("channels_and_topics")` |

---

## 内部実装メモ（横断）

### DataStore vs Room の使い分け

- **DataStore**: 単純な K-V、ユーザー設定、トグル / enum、軽量 JSON。
- **Room (LocalPlaylistDao)**: 構造化されたプレイリスト、楽曲 ID の順序は別テーブル (`playlist_song_xref`) で正規化。
- 旧プレイリストは DataStore に JSON で残されているが、`PlaylistPreferencesRepository` が初回購読時に Room へマイグレする。

### マイグレーションの流れ

1. `PlaylistPreferencesRepository.ensureMigratedIfNeeded()` が `migrationChecked` フラグで 1 回だけ実行。
2. Room に 0 件 かつ DataStore にデータあり → upsert。
3. `clearLegacyUserPlaylists()` で DataStore から旧データを削除。

### 並行性

- `editMutex` でプレイリスト書き込みを直列化。
- `migrationMutex` で二重マイグレを防止。

---

## 関連ファイル

- 上位: `presentation/viewmodel/SettingsViewModel.kt`, `presentation/viewmodel/PlayerViewModel.kt`, `presentation/viewmodel/LibraryViewModel.kt`
- 下位: `androidx.datastore:datastore-preferences`, `kotlinx.serialization`
- 関連: [`backup-system.md`](./backup-system.md), [`repositories.md`](./repositories.md), [`workers.md`](./workers.md), [`diagnostics.md`](./diagnostics.md), [`equalizer.md`](./equalizer.md)