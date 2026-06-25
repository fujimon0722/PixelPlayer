# 特化画面仕様 (Stats / CreatePlaylist / EditTransition / DailyMix / Mashup / RecentlyPlayed / QuickFill / EasterEgg / FolderExplorer)

---

## StatsScreen

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/StatsScreen.kt` (2632 行)
- **ルート**: `Screen.Stats.route` (`"stats"`) (`AppNavigation.kt:298`)
- **概要**: ユーザーの再生統計。期間 (`DAY` / `WEEK` / `MONTH` / `YEAR` / `ALL`) タブ切替、ヒーロー数値タイル、タイムライン (ListeningTime / PlayCount / AvgDuration)、カテゴリ (Song / Album / Artist) 別トップチャート、トラック集中度 (ドーナツチャート)、24 時間リズム可視化 (DAY/WEEK のみ)、Pull-to-refresh。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `StatsViewModel` | `uiState` (PlaybackStatsRepository.Summary), `homeOverview`, `isRefreshing` |
| `PlaybackStatsRepository` | 統計データソース |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `StatsScreen(navController)` | `StatsScreen.kt:141` | Stats 画面のエントリ | `AppNavigation.kt:302` |
| `StatsHeroSection(summary, ...)` (private) | `StatsScreen.kt:397` | ヒーローセクション | StatsScreen |
| `HeroCard(summary, ...)` (private) | `StatsScreen.kt:434` | 総再生時間 / 回数 | StatsHeroSection |
| `StatsEmptyState()` (private) | `StatsScreen.kt:465` | データなしの空状態 | StatsScreen |
| `SummaryPill(...)` (private) | `StatsScreen.kt:521` | サマリーチップ | StatsHeroSection |
| `SummaryHeroTile(...)` (private) | `StatsScreen.kt:546` | ヒーロー数値タイル | StatsHeroSection |
| `SummaryProgressRow(label, progress, accent, supporting)` (private) | `StatsScreen.kt:588` | プログレス行 | StatsHeroSection |
| `RangeTabsHeader(ranges, selected, onSelect)` (private) | `StatsScreen.kt:639` | 期間タブ | StatsScreen |
| `ListeningHabitsCard(summary, ...)` (private) | `StatsScreen.kt:697` | 24h ピーク時間表示 | StatsScreen |
| `HabitMetric(label, value, supporting)` (private) | `StatsScreen.kt:779` | 習慣指標 | ListeningHabitsCard |
| `formatMinutesWindowLabel(startMinute, endMinute)` (private) | `StatsScreen.kt:814` | 時刻範囲ラベル | ListeningHabitsCard |
| `formatHourLabel(minute)` (private) | `StatsScreen.kt:820` | 時刻ラベル | ListeningHabitsCard |
| `HighlightRow(...)` (private) | `StatsScreen.kt:828` | ハイライト行 | ListeningHabitsCard |
| `rememberStatsSectionTitleStyle()` (private) | `StatsScreen.kt:965` | セクションタイトルスタイル | StatsScreen |
| `rememberStatsAxisLabelStyle(range)` (private) | `StatsScreen.kt:986` | 軸ラベルスタイル | StatsScreen |
| `rememberStatsMetricValueStyle(compact)` (private) | `StatsScreen.kt:1012` | 指標値スタイル | StatsScreen |
| `ListeningTimelineSection(summary, selectedMetric, onMetricChange)` (private) | `StatsScreen.kt:1034` | タイムラインセクション | StatsScreen |
| `CategoryMetricsSection(summary, selectedDimension, onDimensionChange)` (private) | `StatsScreen.kt:1190` | カテゴリ別指標 | StatsScreen |
| `CategoryHorizontalBarChart(entries, palette)` (private) | `StatsScreen.kt:1332` | 水平棒グラフ | CategoryMetricsSection |
| `CategoryRankBadge(rank, highlighted, ...)` (private) | `StatsScreen.kt:1417` | ランキングバッジ | CategoryHorizontalBarChart |
| `TimelineMetricBadge(metric, ...)` (private) | `StatsScreen.kt:1455` | メトリックバッジ | ListeningTimelineSection |
| `TimelineBarChart(range, entries, metric)` (private) | `StatsScreen.kt:1480` | 期間別チャート | ListeningTimelineSection |
| `VerticalTimelineBarChart(range, entries, metric, ...)` (private) | `StatsScreen.kt:1515` | 縦棒チャート | TimelineBarChart |
| `HorizontalTimelineBarChart(range, entries, metric, ...)` (private) | `StatsScreen.kt:1615` | 横棒チャート | TimelineBarChart |
| `timelineSupportingCopy(...)` (private) | `StatsScreen.kt:1697` | 補足コピー | ListeningTimelineSection |
| `formatTimelineLabelForRange(label, range, blankLabel, use24Hour)` (private) | `StatsScreen.kt:1713` | ラベルフォーマット | StatsScreen |
| `convertHourLabel(label, use24Hour)` (private) | `StatsScreen.kt:1740` | 12h/24h 変換 | formatTimelineLabelForRange |
| `monthThreeLetters(label, blankLabel)` (private) | `StatsScreen.kt:1788` | 月名短縮 | formatTimelineLabelForRange |
| `timelineChartSpecFor(range, entryCount)` (private) | `StatsScreen.kt:1794` | 期間/件数別のチャート仕様 | TimelineBarChart |
| `categoryPaletteFor(dimension)` (private) | `StatsScreen.kt:1855` | カテゴリ別パレット | CategoryMetricsSection |
| `TopArtistsCard(summary, ...)` (private) | `StatsScreen.kt:1888` | Top アーティスト | StatsScreen |
| `ArtistAvatar(name, avatarUrl, ...)` (private) | `StatsScreen.kt:1975` | アバター | TopArtistsCard |
| `TopAlbumsCard(summary, ...)` (private) | `StatsScreen.kt:2005` | Top アルバム | StatsScreen |
| `SongStatsCard(summary, ...)` (private) | `StatsScreen.kt:2095` | Top Songs | StatsScreen |
| `TrackConcentrationCard(summary)` (private) | `StatsScreen.kt:2263` | トラック集中度 | StatsScreen |
| `TrackDistributionOverview(slices, totalDurationMs, ...)` (private) | `StatsScreen.kt:2358` | 集中度概要 | TrackConcentrationCard |
| `TrackDistributionStats(slices, totalDurationMs)` (private) | `StatsScreen.kt:2468` | 統計行 | TrackConcentrationCard |
| `TrackDistributionDonut(slices, totalDurationMs)` (private) | `StatsScreen.kt:2540` | ドーナツ描画 | TrackConcentrationCard |

enum / data:

| Name | 場所 | 内容 |
|---|---|---|
| `TimelineMetric` (private) | `StatsScreen.kt:873` | `ListeningTime` / `PlayCount` / `AvgDurationPerPlay` |
| `CategoryDimension` (private) | `StatsScreen.kt:907` | `Song` / `Album` / `Artist` |
| `TimelineChartLayout` (private) | `StatsScreen.kt:935` | `Sparse` / `Dense` |
| `CategoryMetricEntry` (private data) | `StatsScreen.kt:929` | label, durationMs, supporting |
| `TimelineChartSpec` (private data) | `StatsScreen.kt:940` | layout, min/maxItemWidth, maxVisibleItems, chartHeight, labelMaxLines, horizontalContentPadding |
| `CategoryChartPalette` (private data) | `StatsScreen.kt:950` | containerColor, contentColor, accentColor, accentOnColor |
| `TrackShareSlice` (private data) | `StatsScreen.kt:957` | label, durationMs, color |

### 内部実装メモ

- `selectedTimelineMetric` / `selectedCategoryDimension` を `rememberSaveable` (StatsScreen.kt:210-211)。
- Pull-to-refresh: `rememberPullToRefreshState()` + `onPullRefresh` 内で `scope.launch` して `statsViewModel.refresh()`。`PULL_TO_REFRESH_MIN_DURATION_MS = 3500L` で最低 3.5s スピナーを保持。
- カスタムフォント: `ExpTitleTypography` の `rememberYourMixTitleStyle` 系を流用 (`rememberStatsSectionTitleStyle` 等)。
- `TrackDistributionDonut` は `Canvas` で円弧描画 (stroke + ギャップ)。
- DAY/WEEK のとき `showDailyRhythm = true` で `ListeningHabitsCard` を表示。

---

## CreatePlaylistScreen

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/CreatePlaylistScreen.kt` (1708 行)
- **ルート**: 直接ルーティングなし。Library の FAB → `CreatePlaylistDialog` → `CreatePlaylistContent` / `EditPlaylistDialog` → `EditPlaylistContent` の構造。
- **概要**: プレイリスト作成 / 編集ダイアログ。マニャアル作成 / AI 作成 / スマートプレイリスト / 画像クロップ / 色 / アイコン / 形状 (`PlaylistShapeType.Circle/Square/RoundedRect/RotatedPill/Star/SmoothRect`) の選択 UI。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | `playlistPickerStorageFilter` |
| `PlaylistViewModel` | プレイリスト作成・更新の最終処理 |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `smartPlaylistRuleTitle(rule)` (private) | `CreatePlaylistScreen.kt:181` | スマートプレイリストタイトル | CreatePlaylistContent |
| `smartPlaylistRuleSubtitle(rule)` (private) | `CreatePlaylistScreen.kt:192` | スマートプレイリストサブタイトル | CreatePlaylistContent |
| `PlaylistCreationMode` (private enum) | `CreatePlaylistScreen.kt:202` | MANUAL/SMART/AI | CreatePlaylistContent |
| `CreatePlaylistDialog(onDismiss, onCreatePlaylist)` | `CreatePlaylistScreen.kt:208` | 名前入力 + 作成 | LibraryScreen 等 |
| `EditPlaylistDialog(playlistId, ...)` | `CreatePlaylistScreen.kt:244` | 既存編集ダイアログ | PlaylistDetailScreen |
| `CreatePlaylistContent(onDismiss, onCreatePlaylist, ...)` (private) | `CreatePlaylistScreen.kt:295` | マニャアル / AI / スマート分岐 UI | CreatePlaylistDialog |
| `EditPlaylistContent(playlistId, ...)` (public) | `CreatePlaylistScreen.kt:725` | 編集 UI | EditPlaylistDialog |
| `PlaylistFormContent(...)` (private) | `CreatePlaylistScreen.kt:951` | 共通フォーム (画像/色/アイコン/形状) | CreatePlaylistContent, EditPlaylistContent |
| `getIconByName(name)` (public) | `CreatePlaylistScreen.kt:1546` | アイコン取得ヘルパ | PlaylistFormContent |
| `ExpressiveButtonGroup(items, selectedIndex, onSelect)` (public) | `CreatePlaylistScreen.kt:1561` | セグメントボタン | PlaylistFormContent |
| `ShapeParameterCard(...)` (public) | `CreatePlaylistScreen.kt:1618` | 形状パラメータ (star curve / corner radius) | PlaylistFormContent |
| `ThickSlider(...)` (public) | `CreatePlaylistScreen.kt:1667` | 太めスライダー | ShapeParameterCard |
| `getThemeContentColor(colorArgb, scheme)` (public) | `CreatePlaylistScreen.kt:1703` | カラーに対するテキスト色決定 | PlaylistFormContent |

### 内部実装メモ

- 画像: `imagePickerLauncher = rememberLauncherForActivityResult(PickVisualMedia)` で選択。`ImageCropView` でクロップ UI (`presentation/components/ImageCropView.kt`)。
- 色: `selectedColor: Int?` (ARGB)。`defaultColor = MaterialTheme.colorScheme.primaryContainer.toArgb()`。
- 形状: `PlaylistShapeType` enum (Circle/Square/RoundedRect/RotatedPill/Star/SmoothRect)。
- `PlaylistFormContent` は 3 タブ (画像 / 色 / アイコン) を持ち、選択したものを `imageUriString`/`color`/`icon` で出力。

---

## EditTransitionScreen

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/EditTransitionScreen.kt` (756 行)
- **ルート**: `Screen.EditTransition.route` (`"edit_transition?playlistId={playlistId}"`) (`AppNavigation.kt:388`)
- **概要**: 曲のクロスフェード設定。グローバル / プレイリスト単位のスコープ、モード (Overlap / None)、時間 (0–12s)、フェードカーブ (`Curve` enum) を編集。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `TransitionViewModel` | `uiState`, `playlistId`, `useGlobalDefaults`, `rule`, `displayedSettings`, `mode`, `durationMs`, `curve` |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `EditTransitionScreen(navController)` | `EditTransitionScreen.kt:91` | 編集画面エントリ | `AppNavigation.kt:396` |
| `TransitionSummaryCard(...)` (private) | `EditTransitionScreen.kt:255` | サマリーカード | EditTransitionScreen |
| `TransitionModeSection(selectedOption, onSelect)` (private) | `EditTransitionScreen.kt:348` | モード切替 | EditTransitionScreen |
| `ExpressiveMorphingToggle(options, selectedOption, onSelect)` (private) | `EditTransitionScreen.kt:375` | モーフィングトグル | TransitionModeSection |
| `TransitionDurationSection(settings, onChange)` (private) | `EditTransitionScreen.kt:445` | 時間スライダー | EditTransitionScreen |
| `CrossfadeVisualizer(durationMs)` (private) | `EditTransitionScreen.kt:509` | クロスフェード可視化 | EditTransitionScreen |
| `TransitionCurvesSection(selected, onSelect)` (private) | `EditTransitionScreen.kt:655` | カーブ選択 | EditTransitionScreen |
| `CurveSelectionColumn(...)` (private) | `EditTransitionScreen.kt:697` | カーブ列 | TransitionCurvesSection |

### 内部実装メモ

- スクロール連動で `LargeTopAppBar` 状態管理 (`scrollBehavior = TopAppBarDefaults.exitUntilCollapsedScrollBehavior()`)。
- メッセージ: `messageRes` を `useGlobalDefaults` 等の状態で出し分け。
- `CrossfadeVisualizer` は `overlapFactor = durationMs.coerceIn(0, 12000) / 12000f` を `animateFloatAsState` で滑らかに描画。

---

## DailyMixScreen

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/DailyMixScreen.kt` (619 行)
- **ルート**: `Screen.DailyMixScreen.route` (`"daily_mix"`) (`AppNavigation.kt:278`)
- **概要**: 日替わり自動生成プレイリスト。3 つのアルバムアートを 3D モーフィング表示する Hero、シャッフル / 連続再生、AI プレイリスト生成、メニュー (`DailyMixMenu`) 連携。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | `dailyMixSongs`, `stablePlayerState`, `favoriteSongIds`, `selectedSongForInfo`, `showAiPlaylistSheet`, `isGeneratingAiPlaylist`, `aiStatus`, `aiError`, `aiSuccess`, `navBarCompactMode` |
| `PlaylistViewModel` | プレイリスト追加 (`PlaylistBottomSheet`) |
| `MainViewModel` | Daily Mix 関連 |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `DailyMixScreen(playerViewModel, navController)` | `DailyMixScreen.kt:101` | Daily Mix エントリ | `AppNavigation.kt:282` |
| `ExpressiveDailyMixHeader(songs, scrollState, ...)` (private) | `DailyMixScreen.kt:429` | 3D モーフ Hero ヘッダー | DailyMixScreen |
| `rememberDailyMixTitleStyle()` (private) | `DailyMixScreen.kt:595` | タイトルスタイル | DailyMixScreen |

### 内部実装メモ

- 3D モーフ: `threeShapeSwitch(index, thirdShapeCornerRadius = 30.dp)` で複数の形状を切り替える `Layout` Composable (DailyMixScreen.kt:429-490)。
- パララックス: `parallaxOffset = scrollState.firstVisibleItemScrollOffset * 0.5f` (DailyMixScreen.kt:439)。
- Hero Alpha: スクロール距離で `headerAlpha` をアニメ。
- AI シート連携: `AiPlaylistSheet` を `showAiSheet` フラグで表示。

---

## MashupScreen (DJ Space)

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/MashupScreen.kt` (340 行)
- **ルート**: `Screen.DJSpace.route` (`"dj_space"`) (`AppNavigation.kt:327`)
- **概要**: DJ 風 2 デッキミキシング UI。Deck A / Deck B それぞれに曲を読み込み、Crossfader で切替。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `MashupViewModel` | `uiState`, `deck1`, `deck2`, `showSongPickerForDeck` |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `MashupScreen()` | `MashupScreen.kt:65` | DJ Space エントリ | `AppNavigation.kt:331` |
| `DeckUi(deck, onPickClick, onPlayPause, onVolume, onTempo)` (private) | `MashupScreen.kt:157` | 1 デッキ UI | MashupScreen |
| `SliderControl(label, value, onChange, ...)` (private) | `MashupScreen.kt:275` | スライダー | DeckUi |
| `Crossfader(value, onValueChange, modifier)` (private) | `MashupScreen.kt:289` | クロスフェーダー | MashupScreen |
| `SongPickerSheet(songs, onSongSelected)` (private) | `MashupScreen.kt:303` | 曲選択シート | MashupScreen |
| `SongPickerItem(song, onClick)` (private) | `MashupScreen.kt:320` | 曲 1 行 | SongPickerSheet |

---

## RecentlyPlayedScreen

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/RecentlyPlayedScreen.kt` (804 行)
- **ルート**: `Screen.RecentlyPlayed.route` (`"recently_played"`) (`AppNavigation.kt:288`)
- **概要**: 最近再生した曲を期間 (`DAY`/`WEEK`/`MONTH`) 別にグルーピング。タイムスタンプバケットで「2 hours ago」「Yesterday」「Tuesday」等の見出し。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | `playbackHistory`, `stablePlayerState`, `favoriteSongIds`, `selectedSongForInfo`, `navBarCompactMode` |
| `PlaylistViewModel` | プレイリスト追加 |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `RecentlyPlayedScreen(playerViewModel, navController)` | `RecentlyPlayedScreen.kt:103` | 履歴画面エントリ | `AppNavigation.kt:292` |
| `ExpressiveRecentlyPlayedHeader(songs, scrollState, ...)` (private) | `RecentlyPlayedScreen.kt:374` | ヘッダー | RecentlyPlayedScreen |
| `rememberRecentlyPlayedTitleStyle()` (private) | `RecentlyPlayedScreen.kt:455` | タイトルスタイル | RecentlyPlayedScreen |
| `RecentlyPlayedActions(firstSong, ...)` (private) | `RecentlyPlayedScreen.kt:481` | Play/Shuffle アクション | RecentlyPlayedScreen |
| `RecentlyPlayedTimestampDivider(group, ...)` (private) | `RecentlyPlayedScreen.kt:552` | タイムスタンプ区切り | RecentlyPlayedScreen |
| `RecentlyPlayedEmptyState()` (private) | `RecentlyPlayedScreen.kt:637` | 空状態 | RecentlyPlayedScreen |
| `TimestampGroup` (private data) | `RecentlyPlayedScreen.kt:678` | グルーピング単位 | RecentlyPlayedScreen |
| `groupRecentlyPlayedSongs(songs)` (private) | `RecentlyPlayedScreen.kt:685` | グルーピング処理 | RecentlyPlayedScreen |
| `TimestampBucket` (private data) | `RecentlyPlayedScreen.kt:738` | バケットキー / ラベル | groupRecentlyPlayedSongs |
| `resolveTimestampBucket(timestamp, zoneId)` (private) | `RecentlyPlayedScreen.kt:744` | バケット決定 | groupRecentlyPlayedSongs |

### 内部実装メモ

- 期間タブ: `RecentlyPlayedRangeSelector` コンポーネント (`components/RecentlyPlayedRangeSelector.kt`)。
- タイムスタンプ書式: `AndroidDateFormat.is24HourFormat(LocalContext.current)` に応じて `DateTimeFormatter` 切替。
- パララックス Hero。

---

## QuickFillScreen (QuickFillDialog)

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/QuickFillScreen.kt` (468 行)
- **ルート**: 直接ルーティングなし。`GenreDetailScreen` から `QuickFillDialog` として表示。
- **概要**: ジャンルに基づいてプレイリストを自動生成するダイアログ。曲数 / 時間 / 並び替え / 上書き可否を選択。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | `getSongsForGenres`, `playlists` |
| `PlaylistViewModel` | 新規プレイリスト保存 |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `QuickFillDialog(...)` (public) | `QuickFillScreen.kt` 内 | クイックフィルダイアログ | GenreDetailScreen |

---

## FolderExplorerScreen

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/FolderExplorerScreen.kt`
- **概要**: 単体のフォルダツリーエクスプローラ。`FileExplorerBottomSheet` (`components/FileExplorerBottomSheet.kt`) と類似。

---

## EasterEggScreen

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/EasterEggScreen.kt`
- **概要**: `BrickBreakerOverlay` (`components/brickbreaker/BrickBreakerOverlay.kt`) を使った隠しゲーム。

---

## EditTransitionScreen 補足

`Screen.EditTransition.route` は optional 引数 (`playlistId`) を受け取り、`useGlobalDefaults` で分岐。

- `playlistId == null`: グローバル設定
- `playlistId != null`: 該当プレイリスト用のカスタムルール (TransitionRule)
- `useGlobalDefaults` トグルでカスタム ↔ グローバル切替。

---

## 関連共通ファイル

| ファイル | 用途 |
|---|---|
| `presentation/screens/TabAnimation.kt` | タブアニメーションプリセット |
| `presentation/screens/AiUsageComponents.kt` | AI 使用量表示 |
| `presentation/screens/search/components/GenreCategoriesGrid.kt` | ジャンルカテゴリグリッド |
| `presentation/screens/search/components/GenreTypography.kt` | ジャンル用タイポグラフィ |
| `presentation/screens/search/components/GenreiconProvider.kt` | ジャンルアイコン定義 |