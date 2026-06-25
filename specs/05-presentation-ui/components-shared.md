# 共有 / 共通コンポーネント仕様

> すべての画面から横断的に利用される「TopBar / Drawer / Dialog / Sidebar / テーマ連動 UI / Scoped コンポーネント」の仕様。

---

## ScreenWrapper

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/ScreenWrapper.kt`
- **用途**: 各 Screen をラップする共通コンテナ。フルプレイヤー BottomSheet (`UnifiedPlayerSheetV2.kt`) を全画面でホストし、`AnimatedVisibilityScope` を消費する。
- **シグネチャ**: `fun ScreenWrapper(navController, playerViewModel, animatedVisibilityScope, content)`
- **呼び出し元**: `AppNavigation.kt` のすべての `composable(...)` ブロック内。

### 内部実装メモ

- `UnifiedPlayerSheetV2` を画面コンテンツの上に `Box` で配置。
- `SharedTransitionLayout` / `AnimatedContent` を必要に応じて有効化。
- Bottom Navigation (`AppSidebarDrawer.kt` 経由) との統合は呼び出し側。

---

## AppSidebarDrawer

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/AppSidebarDrawer.kt`
- **用途**: Home 画面の `ModalNavigationDrawer` の本体。設定・ナビ・About 等へのリンクを格納。

### 状態ホルダー連携

| Holder | 役割 |
|---|---|
| `PlayerViewModel` | ナビゲート |
| `SettingsViewModel` | 表示用設定 |

---

## CollapsibleCommonTopBar

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/CollapsibleCommonTopBar.kt`
- **用途**: スクロールで Collapse / Expand する共通 TopBar。アルバム / アーティスト / プレイリスト / ジャンル / 設定カテゴリなどで利用。

### 内部実装メモ

- `nestedScroll` で `topBarHeight` を縮小。
- `useSmoothCorners` 設定で角丸スタイルを切替。
- `LocalPixelPlayDarkTheme` で暗色対応。
- 動的 ColorScheme (`ColorSchemePair`) を渡せる。

---

## GradientTopBar

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/GradientTopBar.kt`
- **用途**: グラデーション背景 TopBar。HomeScreen で `HomeGradientTopBar` として使われる。

### 内部実装メモ

- `Brush.verticalGradient` で `surfaceContainer` から `primary` へグラデーション。
- スクロールで α フェード。

---

## ExpressiveTopBarContent

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/ExpressiveTopBarContent.kt`
- **用途**: スクロール連動 Expressive な TopBar コンテンツ。タイトル / サブタイトル / スクロール位置で α フェード。

---

## SmartImage / OptimizedAlbumArt

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/SmartImage.kt` / `OptimizedAlbumArt.kt`
- **用途**: Coil ベースの画像表示。アルバムアート用 `SmartImageListTargetSize` / `SmartImageCompactListTargetSize` 定数を持つ。

---

## ShimmerBox

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/ShimmerBox.kt`
- **用途**: ロード中 Shimmer プレースホルダ矩形。

---

## ExpressiveScrollBar / ExpressiveScrollBarMetrics / ExpressiveScrollBarLabelResolvers

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/ExpressiveScrollBar.kt` / `ExpressiveScrollBarMetrics.kt` / `ExpressiveScrollBarLabelResolvers.kt`
- **用途**: ファストスクロール可能な Expressive スクロールバー。

### 内部実装メモ

- スクロール位置 / ドラッグ / ラベル解決を `LocalShowScrollbar` で全体制御。
- `LazyListState.layoutInfo` からメトリクス取得。

---

## MarqueeText / AutoScrollingText / AutoScrollingTextOnDemand

(前述: `components-controls.md`)

---

## SyncProgressBar

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/SyncProgressBar.kt`
- **用途**: ライブラリ同期中、上部に表示するインライン プログレス。

---

## StatsOverviewCard

- **パッケージ**: `app/src/main/java/com/theveloper/pixelplay/presentation/components/StatsOverviewCard.kt`
- **用途**: Home 画面の統計概要カード。

---

## StreamingProviderSheet / HomeOptionsBottomSheet

(前述)

---

## Dialog 群

| ファイル | 用途 |
|---|---|
| `presentation/components/AppRebrandDialog.kt` | リブランディング告知 |
| `presentation/components/PlayStoreAnnouncementDialog.kt` | Play Store 移行告知 |
| `presentation/components/CrashReportDialog.kt` | クラッシュレポート送信 |
| `presentation/components/Beta05CleanInstallDisclaimerDialog.kt` | クリーンインストール免責 |
| `presentation/components/BetaInfoBottomSheet.kt` | ベータ版告知 |
| `presentation/components/ChangelogBottomSheet.kt` | 変更履歴 |
| `presentation/components/PermissionIconCollage.kt` | 権限アイコンコラージュ |
| `presentation/components/AllFilesAccessDialog.kt` | MANAGE_EXTERNAL_STORAGE 要求 |
| `presentation/components/BackupModuleSelectionDialog.kt` | バックアップ対象選択 |

---

## Scoped Components (Component-local state holders)

`presentation/components/scoped/` 配下に「Composable のスコープ内で完結する状態・副作用」をモジュール化。Player Sheet / Queue Sheet / Lyrics Sheet 等の複雑な BottomSheet で活躍。

| ファイル | 役割 |
|---|---|
| `SheetVisualState.kt` | シートの視覚状態 (fraction / peek / height) |
| `SheetMotionController.kt` | シートの動き制御 (drag / fling / settle) |
| `SheetInteractionState.kt` | インタラクション状態 |
| `SheetOverlayState.kt` | オーバーレイ制御 |
| `SheetModalOverlayController.kt` | Modal 風オーバーレイ制御 |
| `SheetBackAndDragState.kt` | バック/ドラッグの調停 |
| `SheetThemeState.kt` | シート内テーマ |
| `SheetVerticalDragMath.kt` | 縦ドラッグの計算 (fraction / velocity) |
| `SheetVerticalDragGestureHandler.kt` | 縦ドラッグ ジェスチャ |
| `Expansion.kt` | シートの拡張ロジック |
| `ComposeLoader.kt` | Composable ローダ |
| `PrefetchAlbumNeighbors.kt` | アルバム Carousel のプリフェッチ |
| `PrewarmFullPlayerState.kt` | フルプレイヤー状態プリウォーム |
| `FullPlayerCompositionPolicy.kt` | フルプレイヤー合成ポリシー |
| `FullPlayerRuntimePolicy.kt` | フルプレイヤー実行時ポリシー |
| `FullPlayerVisualState.kt` | フルプレイヤー視覚状態 |
| `LyricsPredictiveBackHandler.kt` | 歌詞シート予測バック |
| `PlayerSheetPredictiveBackHandler.kt` | プレイヤーシート予測バック |
| `PlayerArtistNavigationEffect.kt` | アーティスト詳細への遷移副作用 |
| `PlayerAlbumNavigationEffect.kt` | アルバム詳細への遷移副作用 |
| `MiniPlayerDismissGestureHandler.kt` | ミニプレイヤー Dismiss ジェスチャ |
| `QueueItemDismissGestureHandler.kt` | キューアイテム スワイプ削除 |
| `QueueSheetController.kt` | キューシート制御 |
| `QueueSheetState.kt` | キューシート状態 |
| `QueueSheetRuntimeEffects.kt` | キューシート実行時副作用 |
| `CastSheetState.kt` | Cast シート状態 |
| `CustomNavigationBarItem.kt` | カスタム NavBar アイテム |
| `KeylineListScope.kt` | キーライン リスト スコープ |

### 内部実装メモ

- いずれも Composable の `remember` で状態を持ち、`LaunchedEffect` で副作用を扱う。PlayerViewModel への依存は最小限。
- `Expansion.kt` は Player Sheet / Queue Sheet 共通の Expansion (fraction / expanded / peeked) をモデル化。
- `SheetMotionController.kt` / `SheetVerticalDragGestureHandler.kt` / `SheetVerticalDragMath.kt` の 3 点セットで「ドラッグ → fraction 計算 → settle」のパイプラインを実装。
- `PlayerArtistNavigationEffect.kt` / `PlayerAlbumNavigationEffect.kt` は曲クリック時の遷移ロジックを集約。

---

## Cross-cutting な Theme Hooks

- `LocalPixelPlayDarkTheme` (`ui/theme/`) — ダーク/ライト判定を CompositionLocal で公開。
- `LocalShowScrollbar` (`ui/theme/`) — スクロールバー表示可否を CompositionLocal で公開。
- `LocalMaterialTheme` (`components/`) — フルプレイヤー内で ColorScheme を差し替えるための CompositionLocal。
- `LocalAppHapticsConfig` (`presentation/utils/`) — ハプティクス設定の CompositionLocal。
- `ShapeCache` (`ui/theme/`) — 動的 Shape キャッシュ。

---

## 全体アーキテクチャ

```mermaid
flowchart TB
  subgraph Per-Screen
    HomeScreen
    LibraryScreen
    SearchScreen
    AlbumDetail
    ArtistDetail
    PlaylistDetail
    GenreDetail
    SettingsScreen
    StatsScreen
    DailyMixScreen
    RecentlyPlayedScreen
    MashupScreen
    AboutScreen
    SetupScreen
    EqualizerScreen
    DeviceCapabilitiesScreen
  end

  subgraph Shared Composables
    ScreenWrapper
    AppSidebarDrawer
    CollapsibleCommonTopBar
    GradientTopBar
    ExpressiveTopBarContent
    SmartImage
    OptimizedAlbumArt
    ShimmerBox
    ExpressiveScrollBar
    MarqueeText
    AutoScrollingTextOnDemand
    SyncProgressBar
    StatsOverviewCard
  end

  subgraph Scoped Components
    SheetVisualState
    SheetMotionController
    Expansion
    QueueSheetController
    CastSheetState
    PlayerArtistNavigationEffect
    PlayerAlbumNavigationEffect
  end

  Per-Screen --> ScreenWrapper
  Per-Screen --> Shared Composables
  Per-Screen --> Scoped Components
  ScreenWrapper --> UnifiedPlayerSheetV2
  Shared Composables --> ui/theme
```