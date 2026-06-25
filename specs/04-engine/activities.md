# Activities — MainActivity / ExternalPlayerActivity

UI エントリーポイント。`MainActivity` は Compose ホスト + 多数の Intent / Setup ロジック、`ExternalPlayerActivity` は外部 Intent (ACTION_SEND 等) からの音声再生を受け取る軽量オーバーレイ。

---

## MainActivity.kt

**パッケージ**: `com.theveloper.pixelplay`
**役割**: アプリ唯一の Activity。Compose ホスト、Splash 制御、Permission ゲート、Setup ゲート、MediaController バインド、Custom Intent ハンドリング、Glance Widget / Mini Player レイアウトを担当する 1268 行の大物。

**依存 (上流)**: Android Launcher (LAUNCHER Intent), Quick Settings Tile (custom action), Wear OS (`WearIntents.ACTION_OPEN_PLAYER`), 外部アプリ (`ACTION_VIEW` / `ACTION_SEND` audio)
**依存 (下流)**: `MusicService` (MediaController 経由), `PlayerViewModel`, `MainViewModel`, `SyncManager`, `UserPreferencesRepository`, `ThemePreferencesRepository`, `AppLocaleManager`, `CrashHandler`, `CrashReportDialog`, `AppNavigation`, `SetupScreen`, `UnifiedPlayerSheetV2`, `AppSidebarDrawer`, `PlayerInternalNavigationBar`, `GitHubAnnouncementPropertiesService`, `PlayStoreAnnouncementDialog`, `SyncProgress`, `BuildConfig`

### ファイル先頭の型

| 名前 | 種類 | 説明 |
|------|------|------|
| `BottomNavItem` | `data class` | bottom nav の表示エントリ。`label: String` (literal), `labelResId: Int`, `iconResId: Int`, `selectedIconResId: Int?`, `screen: Screen` を持つ (literal フィールドは Compose Preview 用) |
| `DismissUndoBarSlice` | `private data class` | Mini Player を dismiss した時の Undo 表示状態 (`isVisible: Boolean`, `durationMillis: Long`) |
| `MainActivity` | `class : ComponentActivity()` (`@UnstableApi @AndroidEntryPoint`) | アプリ唯一の Activity。Hilt エントリポイント |
| `BlurEffectCache` | `private class` | 展開時の背景 blur をキャッシュする (RenderEffect オブジェクトを再利用するため) |
| `NavBarShapeCache` | `private class` | bottom bar の corner shape (top/bottom radius, smooth flag) をキャッシュする |
| `DynamicSmoothCornerShape` | `private class : Shape` | `AbsoluteSmoothCornerShape` (smooth 有) or `RoundedCornerShape` (smooth 無) を委譲。`createOutline` の結果を `(size, layoutDirection)` でメモ化 |

### public/protected API (主要メソッド)

| シグネチャ | 戻り値 | 目的 | 呼び出し元 |
|------------|--------|------|-----------|
| `attachBaseContext(newBase: Context)` | `Unit` (`@CallSuper`) | `AppLocaleManager.wrapContext(newBase)` でロケールを適用してからスーパークラスへ | フレームワーク |
| `onCreate(savedInstanceState: Bundle?)` | `Unit` | SplashScreen 装着 → edge-to-edge → ベンチマークモード検出 → Compose セット (PixelPlayTheme + SetupScreen / MainAppContent 切替 + CrashReportDialog) | フレームワーク |
| `onNewIntent(intent: Intent)` | `Unit` | `handleIntent(intent)` へ委譲 | フレームワーク |
| `onStart()` | `Unit` (`@OptIn(UnstableApi)`) | `playerViewModel.onMainActivityStart()` → `SessionToken` 経由で `MediaController.buildAsync` | フレームワーク |
| `onStop()` | `Unit` | `MediaController.releaseFuture` でリソース解放 | フレームワーク |
| `onResume()` | `Unit` | スーパークラスのみ | フレームワーク |

#### 内部 Composable / 関数

| シグネチャ | 戻り値 | 目的 |
|------------|--------|------|
| `handleIntent(intent: Intent?)` | `Unit` | ACTION_SHUFFLE_ALL / ACTION_OPEN_PLAYLIST / ACTION_SHOW_PLAYER / ACTION_VIEW / ACTION_SEND (audio/) / ACTION_PLAY_SONG をディスパッチ |
| `resolveStreamUri(intent: Intent)` | `Uri?` | API 33+ は型指定 `getParcelableExtra`、それ未満は deprecated API、`clipData` も fallback |
| `persistUriPermissionIfNeeded(intent: Intent, uri: Uri)` | `Unit` | `FLAG_GRANT_PERSISTABLE_URI_PERMISSION` があれば `takePersistableUriPermission` を呼ぶ (SecurityException / IllegalArgumentException を握りつぶす) |
| `clearExternalIntentPayload(intent: Intent)` | `Unit` | `intent.data = null`, `clipData = null`, `removeExtra(EXTRA_STREAM)` で再起動時の重複発火を防ぐ |
| `openExternalUrl(url: String)` | `Unit` | Play Store URL (`play.google.com` or `market.android.com` の https) のみ許可するホワイトリスト方式。Defense-in-depth (GitHub リモート設定が汚染されても `intent://` を起動しない) |
| `PlayStoreAnnouncementRemoteConfig.toUiModel(context: Context)` | `PlayStoreAnnouncementUiModel` | GitHub 由来のリモート設定に `PlayStoreAnnouncementDefaults.localizedTemplate` のフォールバックをマージ |
| `SetupGateLoadingScreen()` | `@Composable Unit` | セットアップ状態の判定待ち中に表示する CircularWavyProgressIndicator + "Preparing setup..." テキスト |
| `MainAppContent(playerViewModel, mainViewModel)` | `@Composable Unit` | ライブラリ状態 (isSyncing / isLibraryEmpty) を観察し、初回同期中は LoadingOverlay を表示。NavController を保持し pending playlist navigation を処理。`Trace.beginSection("MainActivity.MainAppContent")` |
| `MainUI(playerViewModel, navController)` | `@Composable Unit` | Scaffold + bottomBar (PlayerInternalNavigationBar in Surface with custom shape + blur graphicsLayer) + コンテンツ (AppNavigation) + UnifiedPlayerSheetV2 + DismissUndoBar + PlayStoreAnnouncementDialog |
| `LoadingOverlay(syncProgress: SyncProgress)` | `@Composable Unit` | `CircularWavyProgressIndicator` + `LinearWavyProgressIndicator` + 進行カウント表示 (バックグラウンドクリック無効化) |

### 内部実装メモ

- **Splash 制御**: MD3 最適化として Splash 解除条件は `setKeepOnScreenCondition { false }` (UI スケルトンを即表示) (`MainActivity.kt:222`)。
- **ベンチマークモード**: `is_benchmark` extra で起動すると `shouldBenchmarkRebuildDatabase` で `SyncManager.rebuildDatabase()` を enqueue してから `playerViewModel.prepareBenchmarkPlayerFromLibrary()` を実行 (`MainActivity.kt:225-240`)。
- **Permission Gate**: API 33+ は `READ_MEDIA_AUDIO + POST_NOTIFICATIONS`、それ未満は `READ_EXTERNAL_STORAGE`。`rememberMultiplePermissionsState` で管理。
- **Setup Gate**: `isSetupComplete` (DataStore) × `permissionsValid` の AND で `showSetupScreen` を派生。null の間は `SetupGateLoadingScreen()`。
- **SetContent 内 3 つの LaunchedEffect**:
  1. Setup 完了で `mainViewModel.startSync()` 起動 (`MainActivity.kt:276-281`)
  2. クラッシュログ残存時 (`CrashHandler.hasCrashLog()`) は CrashReportDialog を表示 (`MainActivity.kt:284-289`)
  3. 100ms delay で `contentVisible = true` (`MainActivity.kt:302-306`)
- **BottomBar hide animation**: 2 つの graphicsLayer で重なる方式。1 つ目で `translationY` による slide-down hide (resize を起こさない)、2 つ目で corner shape のキャッシュ化 (`MainActivity.kt:855-894`)。
- **Blur quantization**: `quantizedBlurPx = (fraction * 120f / 2f).roundToInt() * 2f` で 2px 単位の量子化。RenderEffect のオブジェクト生成を ~25 回に抑える (`MainActivity.kt:961`)。
- **Open External URL セキュリティ**: GitHub 由来 URL が汚染されても `intent://` や `javascript:` を起動しないよう、`scheme=https && host ∈ {play.google.com, market.android.com}` のみ許可 (`MainActivity.kt:467-478`)。
- **Pending playlist navigation**: `_pendingPlaylistNavigation` の StateFlow を 50 回 / 100ms までリトライする `LaunchedEffect` (`MainActivity.kt:534-560`)。Nav graph がマウントされるまでのレース対策。
- **Predictive back collapse**: `predictiveBackCollapseFraction` で fractional collapse を反映 (`MainActivity.kt:957`)。
- **Routes with hidden nav bar**: 多数の Screen を列挙した Set で一元管理し、プレフィックス一致 (`{` 前まで) で派生 route を包含 (`MainActivity.kt:622-664`)。
- **Hot field**: `setupScreen = null` の間はスピナー表示 → 状態確定後 AnimatedContent で Setup ↔ Main を 400ms / 450ms で切り替え (`MainActivity.kt:315-336`)。
- **NavBarShapeCache**: 0.5px 以内の変化では既存 Outline を再利用、`(smooth flag)` が同じであれば micro アニメーション中の Outline 再計算を回避 (`MainActivity.kt:1181-1212`)。
- **DynamicSmoothCornerShape**: `AbsoluteSmoothCornerShape` (外部依存) と標準 `RoundedCornerShape` の切替。`smoothnessAsPercent=60` で軽めのベジエ。
- **internal class (file-private)**: `BlurEffectCache` / `NavBarShapeCache` / `DynamicSmoothCornerShape` の 3 つはファイル末尾 (`MainActivity.kt:1155-1267`)。

### 関連ファイル
- 上流: `AndroidManifest.xml` (LAUNCHER activity)
- 下流:
  - ViewModel: `../06-state-navigation/player-viewmodel.md`, `../06-state-navigation/main-viewmodel.md`
  - Navigation: `../06-state-navigation/navigation.md`
  - Player sheet: `../05-presentation-ui/unified-player-sheet.md`
  - Sync: `../03-data-services/sync-manager.md`
  - Service: `music-service.md`
- 関連: `ExternalPlayerActivity.kt`, `MainActivityIntentContract.kt`

---

## ExternalPlayerActivity.kt

**パッケージ**: `com.theveloper.pixelplay`
**役割**: 外部 Intent (`ACTION_VIEW` / `ACTION_SEND` with `audio/*`) からの音声再生を軽量オーバーレイ UI で受け取る Activity。

**依存 (上流)**: 外部アプリ (Share intent), ランチャー
**依存 (下流)**: `MainActivity`, `PlayerViewModel.playExternalUri`, `ExternalPlayerOverlay`, `ThemePreferencesRepository`

### クラス / オブジェクト
| 名前 | 種類 | 説明 |
|------|------|------|
| `ExternalPlayerActivity` | `class : ComponentActivity()` (`@UnstableApi @AndroidEntryPoint`) | シェア受領 Activity。フル Activity ではなくオーバーレイ的 |

### public API (主要メソッド)

| シグネチャ | 戻り値 | 目的 | 呼び出し元 |
|------------|--------|------|-----------|
| `onCreate(savedInstanceState: Bundle?)` | `Unit` | edge-to-edge 設定 + Compose (`PixelPlayTheme` + `ExternalPlayerOverlay`) + `handleIntent(intent)` | フレームワーク |
| `onNewIntent(intent: Intent)` | `Unit` | `handleIntent(intent)` へ委譲 | フレームワーク |
| `handleIntent(intent: Intent?)` | `Unit` | `ACTION_VIEW` → `intent.data` を `playExternalUri(uri)` / `ACTION_SEND` (audio/*) → `resolveStreamUri` で取り出した uri を `playExternalUri` に渡す。URI permission を保存し、`clearExternalIntentPayload` で再起動時の重複発火を防ぐ | 自身 |
| `openFullPlayer()` | `Unit` | `MainActivity` を `ACTION_SHOW_PLAYER=true` extra 付きで起動し、自分は `finish()` |
| `resolveStreamUri(intent: Intent)` | `Uri?` | API 33+ は型指定 `getParcelableExtra(EXTRA_STREAM, Uri::class.java)`、それ未満は deprecated、`clipData` も fallback、`intent.data` 最後 | 自身 |
| `persistUriPermissionIfNeeded(intent: Intent, uri: Uri)` | `Unit` | `FLAG_GRANT_PERSISTABLE_URI_PERMISSION` があれば `takePersistableUriPermission` (`runCatching` で握りつぶし) | 自身 |
| `clearExternalIntentPayload(intent: Intent)` | `Unit` | `intent.data = null`, `clipData = null`, `removeExtra(EXTRA_STREAM)` | 自身 |

### 内部実装メモ

- `MainActivity.handleIntent` と同じ URI 解決ロジック (`MainActivityIntentContract.kt` を使わず直接コピーされている)。
- `themePreferencesRepository.appThemeModeFlow` を購読して PixelPlayTheme の dark/light を切り替える。
- `MainActivity` との重複コード (`resolveStreamUri`, `persistUriPermissionIfNeeded`, `clearExternalIntentPayload`) は将来のリファクタ候補。

### 関連ファイル
- 上流: 外部アプリから `ACTION_SEND` で音声ファイル共有
- 下流: `PlayerViewModel.playExternalUri`, `MainActivity`
- 関連: `MainActivity.kt` (類似の intent ハンドリングコードあり)

---

## MainActivity 内部の補助 Composable (詳細)

### MainAppContent のライブラリ ゲート ロジック

`MainAppContent` は同期状態とライブラリ状態の両方を観察する:

```kotlin
val isSyncing by mainViewModel.isSyncing.collectAsStateWithLifecycle()
val isLibraryEmpty by mainViewModel.isLibraryEmpty.collectAsStateWithLifecycle()
val hasCompletedInitialSync by mainViewModel.hasCompletedInitialSync.collectAsStateWithLifecycle()
val syncProgress by mainViewModel.syncProgress.collectAsStateWithLifecycle()
```

ゲート条件 `shouldPotentiallyShowLoading = isSyncing && isLibraryEmpty && !hasCompletedInitialSync` が true になると、`LoadingOverlay(syncProgress)` を表示。デバウンス:
- 300ms delay で「短時間の同期フラッシング」を吸収
- 表示後 1.5 秒 (minimumDisplayDuration) の最低表示時間 → チラつき防止

### MainUI の Scaffold と Modifier 設計

`MainUI` のレイアウト階層:

```
Scaffold(bottomBar = { ... }) {
  BoxWithConstraints {
    Box(.graphicsLayer { renderEffect = blurEffectCache.get(quantizedBlurPx) }) {
      AppNavigation(...)
    }
    // フル展開時のみ背景 dim レイヤー (350ms fade)
    AnimatedVisibility(isExpandedOrExpanding) { Box(.background(alpha=0.35 or 0.6)) }
    UnifiedPlayerSheetV2(...)
    AnimatedVisibility(dismissUndoBarSlice.isVisible) { DismissUndoBar(...) }
    if (showPlayStoreAnnouncement) PlayStoreAnnouncementDialog(...)
  }
}
```

ポイント:
- bottomBar の Surface は 2 段階の graphicsLayer で
  1. translationY による slide-down hide (resize を起こさない)
  2. corner shape のキャッシュ化 (`NavBarShapeCache`)
- プレイヤー シート展開中は背景を blur + 暗転 (`colorScheme.surfaceContainerLowest.copy(alpha = 0.35 or 0.6)`)
- プレイヤー シート タップで `playerViewModel.collapsePlayerSheet()`

### haptics 制御

```kotlin
val hapticsEnabled by playerViewModel.hapticsEnabled.collectAsStateWithLifecycle()
LaunchedEffect(hapticsEnabled, rootView) {
    rootView.isHapticFeedbackEnabled = hapticsEnabled
    rootView.rootView?.isHapticFeedbackEnabled = hapticsEnabled
}
```

`AppHapticsConfig(enabled = hapticsEnabled)` → `LocalAppHapticsConfig provides` + `LocalHapticFeedback provides (if enabled platformHapticFeedback else NoOpHapticFeedback)` で無効時のタップをサイレント化。

### blur + shape の量子化キャッシュ戦略

| 対象 | キャッシュ | キー |
|------|----------|-----|
| `RenderEffect` (blur) | `BlurEffectCache` | 量子化された radius (`(fraction * 120 / 2).roundToInt() * 2f` px) |
| 角丸 Shape | `NavBarShapeCache` | `(topPx, bottomPx, smoothFlag)` の三重キー (0.5px 以内同一視) |

これらにより 60 FPS 描画中も Shape/RenderEffect オブジェクトの生成が ~25 回に抑えられる。

### Predictive Back との統合

API 34+ の predictive back gesture に対応:

```kotlin
val predictiveBackCollapseFraction by playerViewModel.predictiveBackCollapseFraction.collectAsStateWithLifecycle()
```

プレイヤー シートの `fraction = (expansion * (1f - predictiveBackCollapseFraction)).coerceIn(0f, 1f)` で、システムバック中の collapse 進行を反映。`blur radius` も同じ fraction から導出。

### Drawer 統合

`rememberDrawerState(initialValue = DrawerValue.Closed)` + `scope = rememberCoroutineScope()` + `AppSidebarDrawer(...)` を Scaffold の外側に配置 (`MainActivity.kt:741-797`)。`DrawerDestination` enum に応じて:
- `Home` → NavController で Home に navigate + popUpTo(Home, inclusive)
- `Equalizer` → `Screen.Equalizer.route`
- `Settings` → `Screen.Settings.route`
- `Telegram` → `TelegramLoginActivity` を startActivity

### 検索バーの状態管理

`var isSearchBarActive by remember { mutableStateOf(false) }` を `SearchScreen` のオン/オフに同期。`onSearchBarActiveChange = { isSearchBarActive = it }` を `AppNavigation` 経由で `SearchScreen` に伝搬。`shouldHideNavigationBar` が `Screen.Search.route && isSearchBarActive` の時に true になり、bottom bar を隠す。

### ロケール切替と AttachBaseContext

`@CallSuper override fun attachBaseContext(newBase: Context)` で `AppLocaleManager.wrapContext(newBase)` を先に呼び、ロケールが反映された `Configuration` を持つ Context でスーパークラスに渡す。これにより Activity 再生成を待たずにロケールが反映される。

### ベンチマークモード

`is_benchmark` extra で起動した時:
- `SyncManager.rebuildDatabase()` を enqueue
- 1.5 秒 delay → `playerViewModel.prepareBenchmarkPlayerFromLibrary()` でダミーデータ準備
- `isBenchmarkMode = true` なら `showSetupScreen = false` (強制的に Setup をスキップ)

`onStart` でも `is_benchmark` を確認し、ダミーデータロード分岐の hook を持つ (現状コメントのみ)。

### 通知 Channel の追加

`PixelPlayApplication.onCreate` で `pixelplay_music_channel` を作成する。MainActivity 自体では追加しない。`MusicService.setMediaNotificationProvider(LocalOnlyMediaNotificationProvider(this))` がこのチャンネルを使う。

### ThemeMode と Dark 判定

```kotlin
val systemDarkTheme = isSystemInDarkTheme()
val appThemeMode by themePreferencesRepository.appThemeModeFlow.collectAsStateWithLifecycle(initialValue = AppThemeMode.FOLLOW_SYSTEM)
val useDarkTheme = when (appThemeMode) {
    AppThemeMode.DARK -> true
    AppThemeMode.LIGHT -> false
    else -> systemDarkTheme
}
PixelPlayTheme(darkTheme = useDarkTheme) { ... }
```

3 つのモード (DARK / LIGHT / FOLLOW_SYSTEM) を持ち、`FOLLOW_SYSTEM` の場合だけ OS の `isSystemInDarkTheme` を使う。

### `LocalShowScrollbar` の CompositionLocal

`LocalShowScrollbar provides showScrollbar` で scroll bar 表示 / 非表示を CompositionLocal として配信。各 Composable (`LazyColumn` 等) は `LocalShowScrollbar.current` で受け取る。`userPreferencesRepository.showScrollbarFlow` で永続化。

### ベンチマーク + Compose Profiling

`Trace.beginSection("MainActivity.MainAppContent")` / `Trace.beginSection("MainActivity.MainUI")` を Systrace で計測可能にしている。Compose Layout Inspector の flame chart でも識別子が出る。

### まとめ: MainActivity の責務分割

| 責務 | 場所 |
|------|------|
| Compose host | `setContent` ブロック |
| Splash | `installSplashScreen()` + `setKeepOnScreenCondition { false }` |
| Edge-to-edge | `enableEdgeToEdge` + `window.isNavigationBarContrastEnforced = false` |
| Theme | `PixelPlayTheme(darkTheme = useDarkTheme)` |
| Navigation | `rememberNavController` + `AppNavigation` |
| Player sheet | `UnifiedPlayerSheetV2` + シート展開 fraction 監視 |
| Mini player bottom | `PlayerInternalNavigationBar` + dynamic shape |
| Drawer | `AppSidebarDrawer` |
| Announcement dialog | `PlayStoreAnnouncementDialog` |
| Crash report dialog | `CrashReportDialog` |
| Permission gate | `rememberMultiplePermissionsState` |
| Setup gate | `SetupScreen` vs `MainAppContent` AnimatedContent |
| Intent handling | `handleIntent` + `_pendingPlaylistNavigation` |
| MediaController bind | `onStart` で `SessionToken.buildAsync` |

---

## ExternalPlayerActivity 詳細

### Activity 起動のトリガ

`ExternalPlayerActivity` は `AndroidManifest.xml` で `<intent-filter>` が定義されており、外部アプリが `ACTION_SEND` で audio/* を共有、もしくは `ACTION_VIEW` で audio/* URI を開こうとすると起動する。Theme は `PixelPlayTheme(darkTheme = ...)` で適用。

### onCreate → setContent までの流れ

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    enableEdgeToEdge(statusBarStyle = ..., navigationBarStyle = ...)
    super.onCreate(savedInstanceState)
    setContent {
        val useDarkTheme = ... // themePreferencesRepository.appThemeModeFlow で判定
        PixelPlayTheme(darkTheme = useDarkTheme) {
            ExternalPlayerOverlay(
                playerViewModel = playerViewModel,
                onDismiss = { finish() },
                onOpenFullPlayer = { openFullPlayer() }
            )
        }
    }
    handleIntent(intent)
}
```

`ExternalPlayerOverlay` はミニマルな overlay UI。曲のタイトル / アートワーク / play/pause / open full player / dismiss のみ。

### Intent ハンドリングの重複コード

`MainActivity.kt` と `ExternalPlayerActivity.kt` は同じ URI 解決ロジックを持つ:

| 関数 | 用途 |
|------|------|
| `resolveStreamUri(intent: Intent)` | API 33+ は型指定、それ未満は deprecated、`clipData`、`intent.data` |
| `persistUriPermissionIfNeeded(intent: Intent, uri: Uri)` | `FLAG_GRANT_PERSISTABLE_URI_PERMISSION` で永続化 |
| `clearExternalIntentPayload(intent: Intent)` | `data = null`, `clipData = null`, `removeExtra(EXTRA_STREAM)` |

これらは将来のリファクタ候補 (Common 関数に抽出可能)。

### `openFullPlayer()` の動作

```kotlin
private fun openFullPlayer() {
    val fullPlayerIntent = Intent(this, MainActivity::class.java).apply {
        addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP or Intent.FLAG_ACTIVITY_SINGLE_TOP)
        putExtra("ACTION_SHOW_PLAYER", true)
    }
    startActivity(fullPlayerIntent)
    finish()
}
```

MainActivity を `ACTION_SHOW_PLAYER=true` で起動し、自分は `finish()` する。`MainActivity.handleIntent` でこの extra を見て `PlayerViewModel.showPlayer()` を呼ぶ。

### `playerViewModel.playExternalUri(uri)` との連携

`PlayerViewModel.playExternalUri(uri)` は URI を MediaItem に変換し、`MusicService` のキューに積む (内部で `MediaController.setMediaItems` を呼ぶ)。外部 URI の場合、ContentResolver で InputStream を取得 → 一時ファイルにコピー → その URI を MediaItem にする等の処理が必要。実装は `presentation/viewmodel/PlayerViewModel.kt` を参照。

### なぜ外部 Activity が独立しているか

MainActivity は Compose 全体 (Nav graph, Drawer, Mini Player 等) を持つ大物 Activity。外部 Intent から起動する度にこれ全体を初期化するのは重い。`ExternalPlayerActivity` は軽量 overlay のみで起動 → ユーザーが必要なら MainActivity に遷移、という二段構えで UX とパフォーマンスを両立。

### Theme の同期

`MainActivity` と `ExternalPlayerActivity` の両方が `themePreferencesRepository.appThemeModeFlow` を購読し、同じ theme を適用する。これにより「MainActivity から飛んできても ExternalPlayerActivity から開いても一貫した見た目」になる。

### ライフサイクル

- onCreate: setContent + handleIntent
- onNewIntent: handleIntent (再エントリ)
- ユーザーが close ボタンを押す → onDismiss = { finish() } で Activity 終了
- ユーザーが "Open Full Player" を押す → openFullPlayer() で MainActivity に遷移

### `MediaController` 経由 vs 直接 MediaItem

外部 URI を扱う場合、`PlayerViewModel.playExternalUri` は内部で `MediaController.setMediaItems` を使う。これにより URI 解決 (content:// / file:// / http://) は Media3 に任せられる。`MusicService` 側では特別扱いせず、通常の `setMediaItems` で処理される。

### Intent Action まとめ

| Intent Action | 受信側 | 処理 |
|--------------|------|------|
| `ACTION_VIEW` + `data: audio/* URI` | MainActivity / ExternalPlayerActivity | `playExternalUri(uri)` |
| `ACTION_SEND` + `type: audio/*` + `EXTRA_STREAM` | 同上 | `resolveStreamUri(intent)` → `playExternalUri(uri)` |

両 Activity とも同じハンドリングだが、MainActivity はフル UI を表示、ExternalPlayerActivity はオーバーレイのみ。