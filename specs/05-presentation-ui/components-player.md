# Player 系コンポーネント仕様 (FullPlayer / Queue / Lyrics / Cast / SongPicker / PlayerInternalNavigationBar / PlaylistContainer)

> Player 系は `UnifiedPlayerSheetV2.kt` をルートとして、`FullPlayerContent`, `QueueBottomSheet`, `LyricsSheet`, `CastBottomSheet`, `SongPickerBottomSheet`, `PlayerInternalNavigationBar` などの下位シートを切り替える。各シートは `ScreenWrapper` (Screen.kt 周辺) 経由でフルプレイヤーシート内に組み込まれる。

---

## FullPlayerContent

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/player/FullPlayerContent.kt` (2644 行)
- **用途**: フルプレイヤーシート本体。アルバムカバー (Carousel) / 曲メタデータ / シークバー / トランスポート (Prev/PlayPause/Next) / ボトム行 (Favorite/Timer/Lyrics/Queue/Cast) / イマーシブモード (Lyrics) を含む。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | `currentSong`, `currentPlaybackQueue`, `currentMediaItemIndex`, `playbackPosition`, `repeatMode`, `lyrics`, `lyricsSearchUiState`, `fullPlayerSlice`, `stablePlayerState`, `playbackAudioMetadata`, `isLoadingLyrics`, `albumArtQuality`, `carouselStyle`, `fullPlayerLoadingTweaks`, `selectedRouteName`, `isBluetoothEnabled`, `bluetoothName`, `isRemotePlaybackActive`, `isCastConnecting`, `currentSongArtists`, `immersiveLyricsEnabled`, `immersiveLyricsTimeout`, `isImmersiveTemporarilyDisabled` |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `FullPlayerContent(currentSong, lyricsProvider, onPrevious, onNext, isPlayingProvider, currentPlaybackQueue, currentMediaItemIndex, playWhenReadyProvider, totalDurationProvider, currentPositionProvider, playerViewModel, ...)` | `FullPlayerContent.kt:192` | フルプレイヤー本体。 | `UnifiedPlayerSheetV2.kt` 等 |
| `FullPlayerAlbumCoverSection(...)` (private) | `FullPlayerContent.kt:1004` | アルバムカバー + アルバムカークル | FullPlayerContent |
| `FullPlayerControlsSection(...)` (private) | `FullPlayerContent.kt:1110` | Prev / Play / Next / 10s スキップ | FullPlayerContent |
| `FullPlayerProgressSection(...)` (private) | `FullPlayerContent.kt:1199` | シークバー | FullPlayerContent |
| `FullPlayerSongMetadataSection(...)` (private) | `FullPlayerContent.kt:1305` | 曲メタ + 操作 | FullPlayerContent |
| `FullPlayerPortraitContent(...)` (private) | `FullPlayerContent.kt:1374` | 縦向き合成 | FullPlayerContent |
| `FullPlayerLandscapeContent(...)` (private) | `FullPlayerContent.kt:1409` | 横向き合成 | FullPlayerContent |
| `SongMetadataDisplaySection(...)` (private) | `FullPlayerContent.kt:1453` | タイトル/アーティスト/ファイル情報 | FullPlayerSongMetadataSection |
| `PlayerProgressBarSection(...)` (private) | `FullPlayerContent.kt:1632` | 進捗バー本体 (WavySliderExpressive 利用) | FullPlayerProgressSection |
| `EfficientSlider(value, onValueChange, ...)` (private) | `FullPlayerContent.kt:1853` | ハプティクス連動スライダー | PlayerProgressBarSection |
| `EfficientTimeLabels(...)` (private) | `FullPlayerContent.kt:1899` | 時間ラベル (00:00 / 03:21) | PlayerProgressBarSection |
| `DelayedContent(...)` (private) | `FullPlayerContent.kt:1971` | 出現/退去の遅延 (Loading Tweaks) | FullPlayerContent |
| `PlayerSongInfo(...)` (private) | `FullPlayerContent.kt:2133` | タイトル / アーティスト | FullPlayerContent |
| `PlaceholderBox(...)` (private) | `FullPlayerContent.kt:2232` | プレースホルダ矩形 | DelayedContent |
| `AlbumPlaceholder(...)` (private) | `FullPlayerContent.kt:2246` | アルバムプレースホルダ | DelayedContent |
| `MetadataPlaceholder(...)` (private) | `FullPlayerContent.kt:2272` | メタデータプレースホルダ | DelayedContent |
| `ProgressPlaceholder(...)` (private) | `FullPlayerContent.kt:2351` | 進捗プレースホルダ | DelayedContent |
| `ControlsPlaceholder(color, onColor)` (private) | `FullPlayerContent.kt:2438` | 操作プレースホルダ | DelayedContent |
| `TransportButtonColors` (private data) | `FullPlayerContent.kt:2525` | container/content | FullPlayerControlsSection |
| `expressivePlayPauseButtonColors(colorScheme)` (private) | `FullPlayerContent.kt:2530` | 再生/停止ボタンのカラー | FullPlayerContent |
| `expressiveSkipButtonColors(colorScheme)` (private) | `FullPlayerContent.kt:2537` | スキップボタンのカラー | FullPlayerContent |
| `BottomToggleRow(...)` (private) | `FullPlayerContent.kt:2545` | 下部行 (Favorite/Queue/Lyrics/Cast) | FullPlayerContent |
| `predictSkipCarouselIndex(direction)` (private fn) | `FullPlayerContent.kt:460` | スキップ先のキューインデックス予測 | FullPlayerContent |
| `requestSkip(direction)` (private fn) | `FullPlayerContent.kt:483` | 楽観的 UI でスキップ | FullPlayerContent |
| `resolveQueueIndex(queue, targetSongId)` (private fn) | `FullPlayerContent.kt:1258` | ID からキュー内位置 | FullPlayerContent |
| `predictSkipNextCarouselIndex(...)` (private fn) | `FullPlayerContent.kt:1269` | 次の曲予測 | FullPlayerContent |
| `predictSkipPreviousCarouselIndex(...)` (private fn) | `FullPlayerContent.kt:1285` | 前の曲予測 | FullPlayerContent |
| `formatAudioMetaLabel(mimeType, bitrate, sampleRate)` (private fn) | `FullPlayerContent.kt:1612` | メタ情報文字列 | SongMetadataDisplaySection |
| `validateLyricsImport(context, uri)` (private suspend fn) | `FullPlayerContent.kt:157` | LRC インポート検証 | FullPlayerContent |

enum / data:

| Name | 場所 | 内容 |
|---|---|---|
| `SkipDirection` (private) | `FullPlayerContent.kt:155` | PREVIOUS / NEXT |
| `DelayedContentFrame` (private) | `FullPlayerContent.kt:2125` | rawExpansionFraction, effectiveExpansionFraction, isExpandedOverride |

### 内部実装メモ

- アルバムカバー: `AlbumCarouselSection` (`components/AlbumCarouselSection`/`AlbumCarouselSelection.kt`) で「アルバム単位の Carousel」化。
- 縦/横向き切替: `BoxWithConstraints` + `isLandscape` で `FullPlayerPortraitContent` / `FullPlayerLandscapeContent` を切替。
- 10 秒スキップ: `LyricsSheet` のジェスチャから `onLyricsClick` で `requestSkip(SkipDirection.NEXT)`。
- イマーシブモード: `immersiveMode` フラグ + `lastInteractionTime` でタイマー。再インタラクションで復帰。
- 歌詞シートは下層別 Composable。`LyricsSheet` を `showLyricsSheet = true` で表示。
- ファイルインポート: `filePickerLauncher = rememberLauncherForActivityResult(OpenDocument)` で LRC ファイルを取り込み、`validateLyricsImport` でバリデーション後 `lyricsRepository.applyImport()`。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/components/AlbumCarouselSelection.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/UnifiedPlayerSheetV2.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/UnifiedPlayerSheetShared.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/UnifiedPlayerOverlaysLayer.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/UnifiedPlayerSheetLayers.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/player/AnimatedPlaybackControls.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/player/BottomToggleRow.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/player/PlayerArtistPickerBottomSheet.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/WavySliderExpressive.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/ToggleSegmentButton.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/subcomps/FetchLyricsDialog.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/scoped/*` (Player Sheet 制御群)

---

## QueueBottomSheet

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/QueueBottomSheet.kt` (2150 行)
- **用途**: 再生キュー (現曲 + 待機列 + 履歴) の表示・並び替え・保存・クリア・削除 (Undo バー)。`FullPlayer` の Bottom Toggle Row から起動。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | `queue`, `currentSong`, `currentMediaItemIndex`, `stablePlayerState`, `isRemotePlaybackActive`, `selectedRouteName`, `navBarCompactMode`, `repeatMode`, `isShuffleEnabled`, `isPlaying`, `playbackQueueTitle` |
| `SettingsViewModel` | `uiState.showQueueHistory` (履歴表示フラグ) |
| `QueueSheetState` | キューシートの状態管理 |
| `PlaylistViewModel` | キュー → プレイリスト保存 (`SaveQueueAsPlaylistSheet`) |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `QueueBottomSheet(...)` | `QueueBottomSheet.kt:213` | キューシート本体 | `UnifiedPlayerSheetV2.kt` |
| `QueueToolbarMenuButton(...)` (private) | `QueueBottomSheet.kt:1218` | ツールバー項目 | QueueBottomSheet |
| `QueueHeaderSection(...)` (private) | `QueueBottomSheet.kt:1266` | 上部セクション (タイトル / 曲数) | QueueBottomSheet |
| `QueueHeader(...)` (private) | `QueueBottomSheet.kt:1312` | ヘッダー (再生曲情報含む) | QueueHeaderSection |
| `QueueSourceBadge(...)` (private) | `QueueBottomSheet.kt:1370` | ソース表示 | QueueHeader |
| `QueueControlsToolbar(...)` (private) | `QueueBottomSheet.kt:1405` | 操作ツールバー (Repeat/Shuffle/Clear/Save/Timer) | QueueBottomSheet |
| `SaveQueueAsPlaylistSheet(...)` | `QueueBottomSheet.kt:1485` | キュー→プレイリスト保存シート | QueueBottomSheet |
| `QueuePlaylistSongItem(...)` | `QueueBottomSheet.kt:1880` | 曲 1 行 (スワイプ削除 / ドラッグ並び替え対応) | QueueBottomSheet |

データクラス:

| Name | 場所 | 内容 |
|---|---|---|
| `QueueUndoBarProjection` (private) | `QueueBottomSheet.kt:197` | isVisible, removedSongTitle |

### 内部実装メモ

- 並び替え: `rememberReorderableLazyListState` (sh.calvin.reorderable) で `ReorderableItem`。`pendingReorderExpectedIds` で ViewModel からの非同期結果を待つ間ローカル表示を保持。
- スワイプ削除: `QueueItemDismissGestureHandler` (`components/scoped/QueueItemDismissGestureHandler.kt`) でジェスチャ判定、`dismissOffsetAnimatable` を `Animatable` で滑らかに。
- 履歴表示: `settingsState.showQueueHistory` で `displaySongs` のオフセット (`queueIndexOffset`) を切替。
- 削除 Undo: `QueueUndoBarProjection` 経由で `QueueUndoStateHolder` のタイマー + 復元ボタンを表示 (`DismissUndoBar` 共通)。
- シート高さ: `CastSheetContainer` 同様、`Animatable` で隠す / 表示を制御。
- 予測バック: `predictiveBackProgress.value` でスケール・角丸補間。

---

## LyricsSheet

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/LyricsSheet.kt` (2085 行)
- **用途**: 歌詞表示。同期歌詞 (LRC) / 静的歌詞 / ローマ字 / 翻訳 / 検索 / フェッチ (`FetchLyricsDialog`) / ファイルインポート / イマーシブモード / 自動スクロールスナップ。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | `stablePlayerState`, `lyrics`, `isLoadingLyrics`, `currentSong`, `lyricsSearchUiState`, `fullPlayerSlice.lyricsSyncOffset` |
| `LyricsStateHolder` | 歌詞保持 / 同期オフセット |
| `DataStore` (直接) | `lyricsAlignment`, `showLyricsTranslation`, `showLyricsRomanization`, `useAnimatedLyrics`, `animatedLyricsBlurEnabled`, `disableBlurAllOver`, `animatedLyricsBlurStrength`, `keepScreenOn` |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `LyricsSheet(...)` | `LyricsSheet.kt:224` | 歌詞シート本体 | `UnifiedPlayerSheetV2.kt` |
| `LyricsPlaybackSeekBar(...)` (private) | `LyricsSheet.kt:1164` | プレビュー用シークバー | LyricsSheet |
| `SyncedLyricsList(...)` | `LyricsSheet.kt:1192` | 同期歌詞リスト (スナップ / パララックス) | LyricsSheet |
| `LyricLineRow(...)` | `LyricsSheet.kt:1396` | 1 行表示 (現在行強調 / スケール / ブラー) | SyncedLyricsList |
| `LyricWordSpan(...)` | `LyricsSheet.kt:1660` | 単語単位ハイライト | LyricLineRow |
| `PlainLyricsLine(...)` | `LyricsSheet.kt:1726` | 静的歌詞 1 行 | PlainLyricsList |
| `LyricsTrackInfo(...)` (private) | `LyricsSheet.kt:1986` | 上部トラック情報 | LyricsSheet |

data / helper:

| Name | 場所 | 目的 |
|---|---|---|
| `LyricsSheetColors` (internal) | `LyricsSheet.kt:142` | シート用カラーセット |
| `lyricsSheetColors(scheme)` (internal) | `LyricsSheet.kt:156` | カラーセット生成 |
| `preferredContrastColor(...)` (private) | `LyricsSheet.kt:181` | 背景に対するコントラスト色選択 |
| `contrastRatio(fg, bg)` (private) | `LyricsSheet.kt:195` | コントラスト比計算 |
| `Color.relativeLuminance()` (private ext) | `LyricsSheet.kt:203` | 相対輝度 |
| `Color.encodedSrgbArgb()` (private ext) | `LyricsSheet.kt:211` | sRGB ARGB 整数化 |
| `linearizedChannel(channel)` (private) | `LyricsSheet.kt:213` | sRGB → linear |
| `sanitizeLyricLineText(raw)` (internal) | `LyricsSheet.kt:1811` | 行頭の `v1:` 等を除去 |
| `sanitizeSyncedWords(words)` (internal) | `LyricsSheet.kt:1814` | 単語サニタイズ |
| `SyncedWordCluster` (internal data) | `LyricsSheet.kt:1830` | クラスタ情報 |
| `clusterSyncedWords(words)` (internal) | `LyricsSheet.kt:1835` | 連続する単語を 1 クラスタに |
| `normalizeWordEndTime(...)` (internal) | `LyricsSheet.kt:1861` | 単語の終了時刻正規化 |
| `resolveLineEndTimeMs(line, nextStartMs)` (internal) | `LyricsSheet.kt:1871` | 行の終了時刻 |
| `resolveHighlightedWordIndex(...)` (internal) | `LyricsSheet.kt:1877` | 単語ハイライトインデックス |
| `resolveSeekPositionMs(...)` (internal) | `LyricsSheet.kt:1887` | シーク位置 |
| `HighlightZoneMetrics` (internal data) | `LyricsSheet.kt:1892` | ハイライトゾーン寸法 |
| `calculateHighlightMetrics(...)` (internal) | `LyricsSheet.kt:1899` | メトリクス計算 |
| `highlightSnapOffsetPx(...)` (internal) | `LyricsSheet.kt:1922` | スナップオフセット |
| `animateToSnapIndex(...)` (internal suspend) | `LyricsSheet.kt:1936` | スナップスクロール |
| `snapToSnapIndex(...)` (internal suspend) | `LyricsSheet.kt:1959` | 即時スナップ |
| `resolveCurrentLineIndex(lines, position)` (internal) | `LyricsSheet.kt:1972` | 現在行インデックス |
| `LeadingTagRegex` (private) | `LyricsSheet.kt:1809` | `v\d+:` 正規表現 |

### 内部実装メモ

- `LyricsPredictiveBackHandler` (`components/scoped/LyricsPredictiveBackHandler.kt`) でバック操作。
- `rememberLazyListSnapperLayoutInfo` + `rememberSnapperFlingBehavior` でスナップ。
- アニメーション: `useAnimatedLyrics` が true なら現在行を強調 (scale / padding / alpha / blur)、過去・未来行はフェード。
- イマーシブモード: `immersiveMode = true` で全画面表示 + 一定時間後にコントロール自動非表示 (`immersiveLyricsTimeout`)。
- Keep screen on: 歌詞表示中 `FLAG_KEEP_SCREEN_ON` を toggle。

---

## CastBottomSheet

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/CastBottomSheet.kt` (2045 行)
- **用途**: Cast / Bluetooth / Wi-Fi デバイス選択、アクティブデバイスの音量調整、クイック設定 (Wi-Fi/Bluetooth トグル)。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | `castRoutes`, `selectedRoute`, `routeVolume`, `isRefreshingRoutes`, `isWifiEnabled`, `isWifiRadioOn`, `wifiName`, `isBluetoothEnabled`, `bluetoothName`, `bluetoothAudioDeviceStates`, `isRemotePlaybackActive`, `isCastConnecting`, `trackVolume`, `stablePlayerState.isPlaying`, `navBarCompactMode` |
| `CastStateHolder` / `CastRouteStateHolder` | ルート・スキャン状態 |
| `MediaRouter` (AndroidX) | 実際のメディアルート検出 |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `CastBottomSheet(...)` | `CastBottomSheet.kt:164` | Cast シート本体 | UnifiedPlayerSheetV2 |
| `CastPermissionStep(...)` (private) | `CastBottomSheet.kt:458` | 権限要求ステップ | CastBottomSheet |
| `PermissionHighlight(label, granted)` (private) | `CastBottomSheet.kt:523` | 権限状態表示 | CastPermissionStep |
| `missingCastPermissions(context, permissions)` (private) | `CastBottomSheet.kt:561` | 不足権限リスト | CastBottomSheet |
| `CastSheetContent(state, ...)` (private) | `CastBottomSheet.kt:568` | シート本体 (Pager 切替) | CastBottomSheet |
| `CastControlsTabContent(state, ...)` (private) | `CastBottomSheet.kt:753` | アクティブデバイスの音量 / 状態 | CastSheetContent |
| `CastDevicesTabContent(state, onSelect, onRefresh, ...)` (private) | `CastBottomSheet.kt:818` | デバイスリスト | CastSheetContent |
| `CastSheetContainer(state, ...)` (private) | `CastBottomSheet.kt:886` | シート可動コンテナ | CastBottomSheet |
| `CollapsibleCastTopBar(state, pagerState, ...)` (private) | `CastBottomSheet.kt:1123` | シート TopBar | CastSheetContainer |
| `DeviceSectionHeader(text, action)` (private) | `CastBottomSheet.kt:1234` | セクション見出し | CastDevicesTabContent |
| `ActiveDeviceHero(device, onVolumeChange)` (private) | `CastBottomSheet.kt:1274` | 接続中デバイスのヒーロー | CastControlsTabContent |
| `buildVolumeLabel(value, max)` (private) | `CastBottomSheet.kt:1475` | 音量ラベル | ActiveDeviceHero |
| `EmptyDeviceState()` (private) | `CastBottomSheet.kt:1484` | 空状態 | CastDevicesTabContent |
| `CastDeviceRow(device, onClick, onVolumeChange)` (private) | `CastBottomSheet.kt:1520` | デバイス 1 行 | CastDevicesTabContent |
| `BluetoothMetricIndicator(device)` (private) | `CastBottomSheet.kt:1671` | Bluetooth 電池残量 / 音量 | CastDeviceRow |
| `buildBluetoothVolumePercent(device)` (private) | `CastBottomSheet.kt:1708` | Bluetooth 音量%算出 | BluetoothMetricIndicator |
| `BadgeChip(text, ...)` (private) | `CastBottomSheet.kt:1716` | バッジチップ | CastDeviceRow |
| `QuickSettingsRow(...)` (private) | `CastBottomSheet.kt:1745` | Wi-Fi / Bluetooth トグル行 | CastControlsTabContent |
| `QuickSettingTile(...)` (private) | `CastBottomSheet.kt:1784` | 個別タイル | QuickSettingsRow |
| `ScanningPlaceholderList()` (private) | `CastBottomSheet.kt:1873` | スキャン中プレースホルダ | CastDevicesTabContent |
| `ScanningIndicator(isActive)` (private) | `CastBottomSheet.kt:1894` | スキャンインジケータ | ScanningPlaceholderList |
| `CastSheetScanningPreview()` (private) | `CastBottomSheet.kt:1921` | プレビュー | preview |
| `CastSheetDevicesPreview()` (private) | `CastBottomSheet.kt:1956` | プレビュー | preview |
| `CastSheetWifiOffPreview()` (private) | `CastBottomSheet.kt:2015` | プレビュー | preview |

data:

| Name | 場所 | 内容 |
|---|---|---|
| `CastDeviceUi` (private) | `CastBottomSheet.kt:415` | id, name, deviceType, playbackType, connectionState, volumeHandling, volume, volumeMax, isSelected, batteryPercent, isBluetooth |
| `ActiveDeviceUi` (private) | `CastBottomSheet.kt:429` | id, title, subtitle, isRemote, icon, isConnecting, volume, volumeRange, connectionLabel |
| `CastSheetUiState` (private) | `CastBottomSheet.kt:441` | wifiRadioOn, wifiEnabled, wifiSsid, isScanning, isRefreshing, devices, activeDevice, isBluetoothEnabled, bluetoothName |

### 内部実装メモ

- `HorizontalPager` で Controls / Devices の 2 タブ。
- スキャン: `rememberInfiniteTransition` でスキャン中パルスを `Canvas` 描画 (`ScanningIndicator`)。
- Bluetooth 電池残量 / 音量を統合した `BluetoothMetricIndicator`。
- 権限要求ステップ: `rememberLauncherForActivityResult(ActivityResultContracts.RequestMultiplePermissions)` で段階的に要求。

---

## SongPickerBottomSheet

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/SongPickerBottomSheet.kt` (883 行)
- **用途**: プレイリストに追加する曲をライブラリから選択する BottomSheet。Paging ベース + 検索 + お気に入りフィルタ + ストレージフィルタ。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | `playlistPickerStorageFilter`, `hasCloudSongsFlow`, `playlistPickerSongs`, `playlistPickerFavoriteSongs`, `favoriteSongIds` |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `SongPickerBottomSheet(...)` | `SongPickerBottomSheet.kt:101` | 曲選択シート | PlaylistDetailScreen 等 |
| `SongPickerContent(...)` | `SongPickerBottomSheet.kt:130` | タブ + リスト | SongPickerBottomSheet |
| `SongPickerSelectionPane(...)` | `SongPickerBottomSheet.kt:286` | 検索 + リストペイン | SongPickerContent |
| `SongPickerSearchField(value, onChange, ...)` (private) | `SongPickerBottomSheet.kt:434` | 検索フィールド | SongPickerSelectionPane |
| `SongPickerPagingList(...)` | `SongPickerBottomSheet.kt:473` | Paging ベースリスト | SongPickerSelectionPane |
| `SongPickerRow(...)` (private) | `SongPickerBottomSheet.kt:602` | 1 行 | SongPickerPagingList |
| `SongPickerPlaceholderRow()` (private) | `SongPickerBottomSheet.kt:658` | ロード中行 | SongPickerPagingList |
| `SongPickerEmptyState(tabId)` | `SongPickerBottomSheet.kt:712` | 空状態 | SongPickerSelectionPane |
| `SongPickerList(...)` | `SongPickerBottomSheet.kt:781` | 静的リスト | SongPickerSelectionPane |

### 内部実装メモ

- 4 タブ構成: `LibraryTabId.SONGS` / `FAVORITES` / `DOWNLOADED` / `CLOUD` (Cloud は `hasCloudSongs` が true のときのみ表示)。
- チェックボックス: `selectedSongIds` は呼び出し側が `remember { mutableStateMapOf<String, Boolean>() }` で所有。
- `SongPickerPagingList` は `paging.compose.collectAsLazyPagingItems` で Paging 3 を読込、`pagedSongs.loadState.refresh` が `LoadState.Error` ならそのメッセージを表示。

---

## PlayerInternalNavigationBar

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/PlayerInternalNavigationBar.kt`
- **用途**: フルプレイヤーシート内でアルバム / アーティスト / キュー / 歌詞 / Cast のシート切替を行う下部ナビゲーションバー。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | `currentSong` 関連 |
| `PlayerSheetState` | 現在のシート |

---

## PlaylistContainer

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/PlaylistContainer.kt` (636 行)
- **用途**: Library の「Playlists」タブのメイン表示。プレイリスト一覧 + Pull-to-refresh + 並べ替え + 選択 + 作成 (CreatePlaylistDialogRedesigned)。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlaylistViewModel` | `uiState.playlists` |
| `PlayerViewModel` | `multiSelectionStateHolder` は使わず、`playlistSelectionStateHolder` を使用 |
| `PlaylistSelectionStateHolder` | プレイリスト複数選択 |

### 主要 Composable

| Composable | 場所 | 目的 | 呼び出し元 |
|---|---|---|---|
| `PlaylistContainer(...)` | `PlaylistContainer.kt:95` | プレイリスト一覧コンテナ | LibraryScreen |
| `PlaylistItems(...)` | `PlaylistContainer.kt:236` | リスト本体 | PlaylistContainer |
| `PlaylistItem(playlist, onClick, isSelected, selectionIndex, onToggle)` | `PlaylistContainer.kt:350` | プレイリスト 1 行 | PlaylistItems |
| `CreatePlaylistDialogRedesigned(...)` | `PlaylistContainer.kt:562` | 新規作成ダイアログ | PlaylistContainer FAB |

### 内部実装メモ

- 並べ替え: `currentSortOption` の `storageKey` 変化時にスクロールリセット。
- カバー: `PlaylistArtCollage` で複数曲のコラージュ。
- 選択モード: `isSelected`, `selectionIndex` を Animated で表示 (`selectionScale`, `selectionBorderWidth`).

---

## UnifiedPlayerSheet (周辺)

- `UnifiedPlayerSheetV2.kt` — フルプレイヤー / キュー / 歌詞 / Cast のシート統合コンテナ。
- `UnifiedPlayerSheetLayers.kt` — 各レイヤー分割。
- `UnifiedPlayerSheetShared.kt` — 共通 Composable (`QueuePlaylistSongItem` 等)。
- `UnifiedPlayerOverlaysLayer.kt` — オーバーレイレイヤー (Brick Breaker, Diagnostic 等)。

詳細は各ファイル参照。

---

## SubComponents (Player 系)

- `presentation/components/player/AnimatedPlaybackControls.kt` — Prev/PlayPause/Next アニメーション統合。
- `presentation/components/player/BottomToggleRow.kt` — フルプレイヤー下部のアイコントグル。
- `presentation/components/player/PlayerArtistPickerBottomSheet.kt` — 曲に紐づくアーティスト選択シート (複数アーティスト対応)。