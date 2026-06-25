# navigation.md

> Navigation Compose による画面遷移の仕様。`AppNavigation` (573 行) / `Screen` (sealed class) / `MainRootRoutes` / `NavControllerExtensions` / `Transitions` の 5 ファイル。

---

## AppNavigation

**パッケージ**: `com.theveloper.pixelplay.presentation.navigation`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/AppNavigation.kt` (573 行)
**アノテーション**: `@OptIn(UnstableApi::class)` `@SuppressLint("UnrememberedGetBackStackEntry")`
**役割**: `NavHost` を構築し、各 route → Composable Screen をバインド。`PlayerViewModel` を全画面で共有し、Bottom Nav / Push / Pop の遷移アニメーションを制御。

### 関数シグネチャ

```kotlin
@Composable
fun AppNavigation(
    playerViewModel: PlayerViewModel,
    navController: NavHostController,
    paddingValues: PaddingValues,
    userPreferencesRepository: UserPreferencesRepository,
    onSearchBarActiveChange: (Boolean) -> Unit,
    onOpenSidebar: () -> Unit
)
```

### パラメータ

| 名前 | 目的 |
|------|------|
| `playerViewModel` | 全画面共有の ViewModel |
| `navController` | NavHost への参照 |
| `paddingValues` | Scaffold / Bottom Bar の padding (各画面へ伝搬) |
| `userPreferencesRepository` | `launchTabFlow` を読込 → startDestination 決定 |
| `onSearchBarActiveChange` | 検索バー focus 状態通知 |
| `onOpenSidebar` | サイドバー (Drawer) を開く callback |

### 内部実装メモ

- **startDestination 動的決定**: `LaunchedEffect(Unit)` で `userPreferencesRepository.launchTabFlow.first()` を読込。`LaunchTab.HOME / SEARCH / LIBRARY` を `Screen` の route に変換 (private `String.toRoute()` 関数)。
- **Default transition**: `aospSharedAxisEnter() / Exit() / PopEnter() / PopExit()` (Transitions.kt)。
- **Bottom Nav 用 transition**: `Home / Search / Library / Settings` の 4 ルート間で `mainRootEnterTransition` / `mainRootExitTransition` を使用。Forward (右方向) と Backward (左方向) で `slideInHorizontally` の符号を反転。
- **`MainRootDirection`**: `mainRootRouteIndex` から方向判定 (HOME=0, SEARCH=1, LIBRARY=2, SETTINGS=3)。
- **デフォルト fallback**: route 判別不能時は `aospSharedAxisEnter()` 等のデフォルトアニメーション。
- **`ScreenWrapper`**: 全画面共通で `MiniPlayer` + `BottomBar` + プレイヤーシートを `AnimatedVisibilityScope` 内に包む。

### Composable destination 一覧

| Route | Screen | 補足 |
|-------|--------|------|
| `Screen.Home.route` | `HomeScreen` | 起動タブ可 (Forward/Backward 切替) |
| `Screen.Search.route` | `SearchScreen` | 〃 |
| `Screen.Library.route` | `LibraryScreen` | 〃 |
| `Screen.Settings.route` | `SettingsScreen` | デフォルト遷移 |
| `Screen.Accounts.route` | `AccountsScreen` | Provider Dashboard への jump |
| `Screen.SettingsCategory.route` | `SettingsCategoryScreen` | `categoryId: String` 引数 |
| `Screen.PaletteStyle.route` | `PaletteStyleSettingsScreen` | |
| `Screen.Experimental.route` | `ExperimentalSettingsScreen` | |
| `Screen.NavBarCrRad.route` | `NavBarCornerRadiusScreen` | `route = "nav_bar_corner_radius"` (Screen クラス外で定義) |
| `Screen.DailyMixScreen.route` | `DailyMixScreen` | |
| `Screen.RecentlyPlayed.route` | `RecentlyPlayedScreen` | |
| `Screen.Stats.route` | `StatsScreen` | |
| `Screen.PlaylistDetail.route` | `PlaylistDetailScreen` | `playlistId: String` 引数 + 別 `PlaylistViewModel` (`hiltViewModel()`) |
| `Screen.DJSpace.route` | `MashupScreen` | |
| `Screen.GenreDetail.route` | `GenreDetailScreen` | `genreId: String` 引数 (URL-decode 必要) |
| `Screen.AlbumDetail.route` | `AlbumDetailScreen` | `albumId: String` (Long) |
| `Screen.ArtistDetail.route` | `ArtistDetailScreen` | `artistId: String` (Long) |
| `Screen.EditTransition.route` | `EditTransitionScreen` | `playlistId: String?` (nullable) |
| `Screen.About.route` | `AboutScreen` | |
| `Screen.EasterEgg.route` | `EasterEggScreen` | |
| `Screen.ArtistSettings.route` | `ArtistSettingsScreen` | |
| `Screen.DelimiterConfig.route` | `DelimiterConfigScreen` | |
| `Screen.WordDelimiterConfig.route` | `WordDelimiterConfigScreen` | |
| `Screen.Equalizer.route` | `EqualizerScreen` | |
| `Screen.DeviceCapabilities.route` | `DeviceCapabilitiesScreen` | |
| `Screen.NeteaseDashboard.route` | `NeteaseDashboardScreen` | |
| `Screen.QqMusicDashboard.route` | `QqMusicDashboardScreen` | |
| `Screen.NavidromeDashboard.route` | `NavidromeDashboardScreen` | |
| `Screen.JellyfinDashboard.route` | `JellyfinDashboardScreen` | |

### 内部定数 (private)

| 名前 | 値 | 目的 |
|------|---|------|
| `BOTTOM_NAV_TRANSITION_DURATION` | 380 | Bottom Nav 遷移時間 (0.5x scale で ~190ms) |
| `BottomNavEasing` | `CubicBezierEasing(0.2f, 0f, 0f, 1f)` | MD3 Expressive イージング |
| `MAIN_ROOT_TRANSITION_SPEC` | `tween<IntOffset>(380, BottomNavEasing)` | Offset アニメ |
| `MAIN_ROOT_FADE_SPEC` | `tween<Float>(190, BottomNavEasing)` | Fade アニメ |

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/Screen.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/MainRootRoutes.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/NavControllerExtensions.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/Transitions.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/ScreenWrapper.kt`

---

## Screen

**パッケージ**: `com.theveloper.pixelplay.presentation.navigation`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/Screen.kt` (59 行)
**役割**: 全 route の sealed class。`createRoute(...)` で引数付き route を生成。

### `Screen` (sealed class)

```kotlin
@Immutable
sealed class Screen(val route: String)
```

### 派生クラス一覧

| 派生 | route パターン | 引数 | 用途 |
|------|--------------|------|------|
| `Home` | `"home"` | なし | ホーム画面 |
| `Search` | `"search"` | なし | 検索画面 |
| `Library` | `"library"` | なし | ライブラリ画面 |
| `Settings` | `"settings"` | なし | 設定画面 |
| `Accounts` | `"settings_accounts"` | なし | アカウント一覧 |
| `SettingsCategory` | `"settings_category/{categoryId}"` | `categoryId: String` | 設定カテゴリ詳細 |
| `PaletteStyle` | `"palette_style_settings"` | なし | パレットスタイル |
| `Experimental` | `"experimental_settings"` | なし | 実験的設定 |
| `NavBarCrRad` | `"nav_bar_corner_radius"` | なし | ナビバー角丸設定 |
| `PlaylistDetail` | `"playlist_detail/{playlistId}"` | `playlistId: String` | プレイリスト詳細 |
| `DailyMixScreen` | `"daily_mix"` | なし | Daily Mix |
| `RecentlyPlayed` | `"recently_played"` | なし | 最近再生 |
| `Stats` | `"stats"` | なし | 統計 |
| `GenreDetail` | `"genre_detail/{genreId}"` | `genreId: String` | ジャンル詳細 (URL-decode) |
| `DJSpace` | `"dj_space"` | なし | DJ マッシュアップ |
| `AlbumDetail` | `"album_detail/{albumId}"` | `albumId: Long` | アルバム詳細 |
| `ArtistDetail` | `"artist_detail/{artistId}"` | `artistId: Long` | アーティスト詳細 |
| `EditTransition` | `"edit_transition?playlistId={playlistId}"` | `playlistId: String?` (optional) | クロスフェード設定 |
| `About` | `"about"` | なし | このアプリについて |
| `EasterEgg` | `"easter_egg"` | なし | イースターエッグ |
| `ArtistSettings` | `"artist_settings"` | なし | アーティスト設定 |
| `DelimiterConfig` | `"delimiter_config"` | なし | 区切り文字設定 |
| `WordDelimiterConfig` | `"word_delimiter_config"` | なし | 単語区切り |
| `Equalizer` | `"equalizer"` | なし | イコライザ |
| `DeviceCapabilities` | `"device_capabilities"` | なし | デバイス性能 |
| `NeteaseDashboard` | `"netease_dashboard"` | なし | Netease Dashboard |
| `QqMusicDashboard` | `"qqmusic_dashboard"` | なし | QQ Music Dashboard |
| `NavidromeDashboard` | `"navidrome_dashboard"` | なし | Navidrome Dashboard |
| `JellyfinDashboard` | `"jellyfin_dashboard"` | なし | Jellyfin Dashboard |

### `createRoute` ヘルパ

| クラス | 関数 |
|--------|------|
| `SettingsCategory` | `fun createRoute(categoryId: String) = "settings_category/$categoryId"` |
| `PlaylistDetail` | `fun createRoute(playlistId: String) = "playlist_detail/$playlistId"` |
| `GenreDetail` | `fun createRoute(genreId: String) = "genre_detail/$genreId"` |
| `AlbumDetail` | `fun createRoute(albumId: Long) = "album_detail/$albumId"` |
| `ArtistDetail` | `fun createRoute(artistId: Long) = "artist_detail/$artistId"` |
| `EditTransition` | `fun createRoute(playlistId: String?) = "edit_transition?playlistId=$playlistId"` (nullable) |

### 内部実装メモ

- **`@Immutable`**: Compose のリコンポジション最適化用。
- **object 宣言**: 全派生が `object` (Singleton) — ルート文字列の single source of truth。
- **`Long` への変換**: `AppNavigation` 内で `backStackEntry.arguments?.getString("albumId")?.toLongOrNull()` でパース。
- **URL エンコード**: `genreId` は URL safe な文字 (e.g. `Rock%20%26%20Roll`) で渡る想定で、ViewModel 内で `URLDecoder.decode(...)` する。
- **nullable 引数**: `EditTransition` の `playlistId` は `null` 可能 (`?playlistId={playlistId}` クエリ形式)。

### 関連ファイル

- すべて `AppNavigation.kt` から参照される

---

## MainRootRoutes

**パッケージ**: `com.theveloper.pixelplay.presentation.navigation`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/MainRootRoutes.kt` (16 行)
**役割**: Main Root 4 ルート (Home / Search / Library / Settings) 判定ヘルパ。`AppNavigation` の遷移方向計算用。

### 関数

| シグネチャ | 戻り値 | 目的 |
|-----------|--------|------|
| `internal fun isMainRootRoute(route: String?): Boolean` | Boolean | `route` が 4 root のいずれかか |
| `internal fun mainRootRouteIndex(route: String?): Int?` | Int? | 順序 (HOME=0, SEARCH=1, LIBRARY=2, SETTINGS=3) または `null` |

### 内部実装メモ

- **when 表現**: 4 ルートのみで判定 (拡張子や引数付きは `null` 返却)。
- **使用箇所**: `AppNavigation.mainRootDirection` / `mainRootEnterTransition` / `mainRootExitTransition`。
- **`internal` 可視性**: `app` モジュール内のみ。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/AppNavigation.kt`

---

## NavControllerExtensions

**パッケージ**: `com.theveloper.pixelplay.presentation.navigation`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/NavControllerExtensions.kt` (62 行)
**役割**: `NavController` への安全なナビゲーション拡張関数。

### 拡張関数

| シグネチャ | 目的 |
|-----------|------|
| `private fun NavController.isReadyForNavigation(): Boolean` | 内部: `lifecycle.currentState == Lifecycle.State.RESUMED` 確認 |
| `fun NavController.navigateSafely(route: String): Boolean` | 現在の lifecycle が RESUMED のときのみ遷移 |
| `fun NavController.navigateSafely(route: String, builder: NavOptionsBuilder.() -> Unit): Boolean` | 同上 + NavOptions ビルダ |
| `fun NavController.navigateSafelyReplacing(route: String, popUpToRoute: String?): Boolean` | popUpTo 指定で遷移 (スタック置換) |
| `fun NavController.navigateToTopLevelSafely(route: String): Boolean` | トップレベル (Bottom Nav) への切替。既存スタックをクリアして新規 destination を startDestination として push |

### 内部実装メモ

- **Lifecycle ガード**: `lifecycle.currentState.isAtLeast(RESUMED)` を `isReadyForNavigation` で確認。RESUMED でないと遷移しない。
- **`navigateSafelyReplacing`**: 内部で `navigateSafely(route) { popUpTo(popUpToRoute) { inclusive = true } }` 形式を呼び出す。
- **`navigateToTopLevelSafely`**: `graph.startDestinationId` を確認し、その destination へ pop して新規 push。Bottom Nav 切替で重複スタックを防ぐ。
- **戻り値 `Boolean`**: 実際に遷移したか (`isReadyForNavigation == false` なら `false`)。

### 関連ファイル

- 呼び出し元: `AppNavigation.kt` / `AccountsScreen.kt` 等

---

## Transitions

**パッケージ**: `com.theveloper.pixelplay.presentation.navigation`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/Transitions.kt` (124 行)
**役割**: Compose Navigation の画面遷移アニメーション定義 (AOSP Shared Axis / Slide Horizontal)。

### 公開関数

| シグネチャ | 戻り値 | 目的 |
|-----------|--------|------|
| `fun aospSharedAxisEnter(): EnterTransition` | 横スライド + フェード (Forward 方向) |
| `fun aospSharedAxisExit(): ExitTransition` | 横スライド + フェード (Forward 方向) |
| `fun aospSharedAxisPopEnter(): EnterTransition` | 横スライド (Backward 方向 = 右→左) |
| `fun aospSharedAxisPopExit(): ExitTransition` | 横スライド (Backward 方向) |
| `fun enterTransition()` | デフォルト横スライド (Left→Right) |
| `fun exitTransition()` | デフォルト横スライド (Right→Left) |
| `fun popEnterTransition()` | Pop 用 (Right→Left 復帰) |
| `fun popExitTransition()` | Pop 用 (Left→Right) |

### 定数

| 名前 | 値 | 目的 |
|------|---|------|
| `M3EmphasizedEasing` | `CubicBezierEasing(0.2f, 0.0f, 0.0f, 1.0f)` | Material 3 Emphasized |
| `AOSP_TRANSITION_DURATION` | `350` (const) | AOSP 風遷移時間 (ms) |
| `TRANSITION_DURATION` | `450` (const) | スライド遷移時間 (ms) |
| `EmphasizedEasing` (private) | `CubicBezierEasing(0.2f, 0f, 0f, 1f)` | AOSP 用 |
| `EmphasizedDecelerateEasing` (private) | `CubicBezierEasing(0.2f, 0.85f, 0.7f, 1f)` | 減速 |
| `EmphasizedAccelerateEasing` (private) | `CubicBezierEasing(0.3f, 0f, 0.8f, 0.15f)` | 加速 |

### 内部実装メモ

- **AOSP Shared Axis**: Z 軸 (拡大縮小) + X 軸 (スライド) + フェードを組み合わせた Android 12 風アニメーション。`TransformOrigin(0f, 0f)` で左上からスケール。
- **デフォルト enterTransition**: 30% 幅スライド + 短いフェード (0–250ms)。
- **`aospSharedAxisEnter/Exit`**: Forward 方向のスライド + スケール (0.85→1.0)。
- **`popEnter/Exit`**: Backward 方向。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/navigation/AppNavigation.kt` (`NavHost` の enter/exit/pop 設定)
