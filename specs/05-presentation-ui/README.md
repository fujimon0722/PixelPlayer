# PixelPlayer Presentation UI 仕様書

`app/src/main/java/com/theveloper/pixelplay/presentation/` 配下の **Screen / Component / BottomSheet** 層の Composable 単位仕様書。

| Layer | Directory | 役割 |
|---|---|---|
| Screen | `presentation/screens/` | ナビゲーショングラフの目的地。トップレベル UI 画面。 |
| Component | `presentation/components/` | 画面や他コンポーネントから再利用される Composable・BottomSheet・Dialog・カスタム UI 要素。 |

## 1. ナビゲーション定義

`presentation/navigation/Screen.kt` に `sealed class Screen(val route: String)` があり、すべてのルート文字列を集約。`AppNavigation.kt` (`presentation/navigation/AppNavigation.kt:67`) の `NavHost` で各 route → Screen のマッピングを行う。

| Screen object | Route | 仕様書 |
|---|---|---|
| `Home` | `home` | [screens-main.md](./screens-main.md#homescreen) |
| `Search` | `search` | [screens-main.md](./screens-main.md#searchscreen) |
| `Library` | `library` | [screens-main.md](./screens-main.md#libraryscreen) |
| `Settings` | `settings` | [screens-settings.md](./screens-settings.md#settingsscreen) |
| `Accounts` | `settings_accounts` | [screens-settings.md](./screens-settings.md#accountsscreen) |
| `SettingsCategory` | `settings_category/{categoryId}` | [screens-settings.md](./screens-settings.md#settingscategoryscreen) |
| `PaletteStyle` | `palette_style_settings` | [screens-settings.md](./screens-settings.md#palettestylesettingsscreen) |
| `Experimental` | `experimental_settings` | [screens-settings.md](./screens-settings.md#experimentalsettingsscreen) |
| `NavBarCrRad` | `nav_bar_corner_radius` | [screens-settings.md](./screens-settings.md#navbarcornerradiusscreen) |
| `PlaylistDetail` | `playlist_detail/{playlistId}` | [screens-detail.md](./screens-detail.md#playlistdetailscreen) |
| `DailyMixScreen` | `daily_mix` | [screens-specialized.md](./screens-specialized.md#dailymixscreen) |
| `RecentlyPlayed` | `recently_played` | [screens-specialized.md](./screens-specialized.md#recentlyplayedscreen) |
| `Stats` | `stats` | [screens-specialized.md](./screens-specialized.md#statsscreen) |
| `GenreDetail` | `genre_detail/{genreId}` | [screens-detail.md](./screens-detail.md#genredetailscreen) |
| `DJSpace` | `dj_space` | [screens-specialized.md](./screens-specialized.md#mashupscreen) |
| `AlbumDetail` | `album_detail/{albumId}` | [screens-detail.md](./screens-detail.md#albumdetailscreen) |
| `ArtistDetail` | `artist_detail/{artistId}` | [screens-detail.md](./screens-detail.md#artistdetailscreen) |
| `EditTransition` | `edit_transition?playlistId={playlistId}` | [screens-specialized.md](./screens-specialized.md#edittransitionscreen) |
| `About` | `about` | [screens-specialized.md](./screens-specialized.md#aboutscreen) |
| `EasterEgg` | `easter_egg` | [screens-specialized.md](./screens-specialized.md#eastereggscreen) |
| `ArtistSettings` | `artist_settings` | [screens-settings.md](./screens-settings.md#artistsettingsscreen) |
| `DelimiterConfig` | `delimiter_config` | [screens-settings.md](./screens-settings.md#delimiterconfigscreen) |
| `WordDelimiterConfig` | `word_delimiter_config` | [screens-settings.md](./screens-settings.md#worddelimiterconfigscreen) |
| `Equalizer` | `equalizer` | [screens-settings.md](./screens-settings.md#equalizerscreen) |
| `DeviceCapabilities` | `device_capabilities` | [screens-settings.md](./screens-settings.md#devicecapabilitiesscreen) |
| `NeteaseDashboard` / `QqMusicDashboard` / `NavidromeDashboard` / `JellyfinDashboard` | 外部ダッシュボード | screens-settings.md で参照 |

## 2. 主要な状態ホルダー (ViewModel / StateHolder)

`presentation/viewmodel/` に集約。詳細仕様は別冊 `specs/06-state-navigation/viewmodels.md` を参照。

| ViewModel | 役割 | 主な consumer |
|---|---|---|
| `PlayerViewModel` | プレイヤー再生状態・キュー・お気に入り・AI 連携 | すべての画面 |
| `SettingsViewModel` | アプリ設定永続化・テーマ・ライブラリ | Settings 系, Home |
| `LibraryViewModel` | ライブラリ Paging フロー | LibraryScreen, SongPicker |
| `PlaylistViewModel` | プレイリスト作成/編集/同期 (Telegram, AI) | Library, PlaylistDetail |
| `AlbumDetailViewModel` | アルバム詳細 | AlbumDetailScreen |
| `ArtistDetailViewModel` | アーティスト詳細 | ArtistDetailScreen |
| `GenreDetailViewModel` | ジャンル詳細 | GenreDetailScreen |
| `StatsViewModel` | 再生統計 | StatsScreen, Home |
| `EqualizerViewModel` | EQ プリセット・帯域 | EqualizerScreen |
| `TransitionViewModel` | クロスフェード設定 | EditTransitionScreen |
| `MashupViewModel` | DJ ミキシング | MashupScreen |
| `DeviceCapabilitiesViewModel` | デバイス性能 | DeviceCapabilitiesScreen |
| `ArtistSettingsViewModel` | アーティスト分割ルール | ArtistSettingsScreen |
| `SetupViewModel` | 初回セットアップ | SetupScreen |
| `AccountsViewModel` | 外部サービス連携 | AccountsScreen |
| `MainViewModel` | ホーム Daily Mix | HomeScreen, DailyMixScreen |
| `SongInfoBottomSheetViewModel` | 曲情報シート操作 | SongInfoBottomSheet, 詳細画面 |

StateHolder（軽量・Composable 内で利用）:

| StateHolder | 用途 |
|---|---|
| `MultiSelectionStateHolder` | ライブラリ/検索の複数選択 |
| `PlaylistSelectionStateHolder` | プレイリスト複数選択 |
| `QueueStateHolder` / `QueueSheetState` | キュー操作 |
| `LyricsStateHolder` | 歌詞表示 |
| `PlayerSheetState` | フルプレイヤー BottomSheet |
| `PlaybackStateHolder` / `StablePlayerState` | 再生状態の安定 Flow |
| `FolderNavigationStateHolder` | フォルダツリー操作 |
| `SleepTimerStateHolder` | スリープタイマー |
| `CastRouteStateHolder` / `CastStateHolder` / `CastTransferStateHolder` | Cast 連携 |
| `ThemeStateHolder` | 動的テーマ状態 |
| `MetadataEditStateHolder` | 楽曲メタデータ編集 |
| `SyncProgress` / `LibraryStateHolder` | 同期 |
| `ConnectivityStateHolder` / `ExternalMediaStateHolder` | 接続性 |

## 3. 共有ユーティリティ / Custom Composable

| ファイル | 用途 |
|---|---|
| `ScreenWrapper.kt` | 各 Screen に共通でラップする Scaffold / フルプレイヤーシート |
| `AppSidebarDrawer.kt` | サイドバードロワー (HomeScreen) |
| `CollapsibleCommonTopBar.kt` / `GradientTopBar.kt` / `ExpressiveTopBarContent.kt` | スクロール連動 TopBar |
| `ExpressiveScrollBar.kt` (+ `ExpressiveScrollBarMetrics.kt` / `ExpressiveScrollBarLabelResolvers.kt`) | カスタムスクロールバー |
| `SmartImage.kt` / `OptimizedAlbumArt.kt` | Coil ベースの画像表示 |
| `AlbumArtCollage.kt` / `PlaylistArtCollage.kt` / `CollagePatterns.kt` / `AlbumCarouselSelection.kt` | コラージュ生成 |
| `InfiniteListHandler.kt` | スクロール末尾プリフェッチ |
| `StreamingProviderSheet.kt` | ストリーミング プロバイダ選択 |
| `HomeOptionsBottomSheet.kt` | ホーム右上のオプションメニュー |
| `SmartImage.kt` / `ShimmerBox.kt` | プレースホルダ/画像 |
| `NoInternetComponents.kt` / `ExpressiveOfflineState.kt` | オフライン UI |
| `AppRebrandDialog.kt` / `Beta05CleanInstallDisclaimerDialog.kt` / `BetaInfoBottomSheet.kt` / `ChangelogBottomSheet.kt` / `PlayStoreAnnouncementDialog.kt` / `CrashReportDialog.kt` / `PermissionIconCollage.kt` | ダイアログ/告知系 |
| `AppSidebarDrawer.kt` | Home ドロワー |
| `StatsOverviewCard.kt` | Stats ヒーロー |
| `PermissionIconCollage.kt` | 権限アイコンコラージュ |

詳細は [components-shared.md](./components-shared.md) を参照。

## 4. コンポーネント分類

| 区分 | 仕様書 |
|---|---|
| Player UI (Full Player, Queue, Lyrics, Cast, SongPicker) | [components-player.md](./components-player.md) |
| Library / 複数選択 (PlaylistBottomSheet, MultiSelection*, AlbumCarousel) | [components-library.md](./components-library.md) |
| Custom Controls (WavySlider, MarqueeText, EnhancedSongListItem, ToggleSegmentButton) | [components-controls.md](./components-controls.md) |
| BottomSheet 系 (EditSongSheet, SongInfoBottomSheet, FetchLyricsDialog, AiPlaylistSheet 他) | [components-bottomsheets.md](./components-bottomsheets.md) |
| Shared / Dialog / TopBar / Drawer | [components-shared.md](./components-shared.md) |

## 5. 命名規約・相対パス方針

- 仕様書内のソースコード参照はすべて `app/src/main/java/com/theveloper/pixelplay/...` からの相対パス (`file:line` 形式)。
- ViewModel 仕様は `specs/06-state-navigation/viewmodels.md` に分離。
- データ層は `specs/01-data-foundation/`, `specs/02-data-network/`, `specs/03-data-services/`。
- エンジン層 (プレイヤー本体) は `specs/04-engine/`。
- テーマ/UI システム層は `specs/07-ui-system/`。