# 07 — UI システム基盤

> **対象ディレクトリ**: `app/src/main/java/com/theveloper/pixelplay/ui/`
>
> Compose / Glance ベースの UI レイヤの基盤を定義する。
> アプリの「視覚的アイデンティティ」「色生成ロジック」「ホームウィジェット」がここに集約される。

## ディレクトリ構造

```
ui/
├── theme/                       # カラー / タイポグラフィ / シェイプ / 動的カラー生成
│   ├── Color.kt                 # 静的パレット定数
│   ├── ColorRoles.kt            # アルバムアートからの動的 ColorScheme 生成 (729 行)
│   ├── GenreColors.kt           # ジャンルごとの固定カラースキーム
│   ├── Shape.kt                 # Material3 Shapes
│   ├── ShapeCache.kt            # AbsoluteSmoothCornerShape のシングルトンキャッシュ
│   ├── Theme.kt                 # PixelPlayTheme Composable / StatusBar 制御
│   └── Type.kt                  # Typography (Montserrat + Google Sans Flex Rounded)
│
└── glancewidget/                # Glance ホームウィジェット
    ├── PixelPlayGlanceWidget.kt # 大型可変サイズ 1 系統 (1401 行)
    ├── BarWidget4x1.kt          # 4x1 バー型
    ├── BarWidget4x1Receiver.kt  # AppWidget 登録
    ├── ControlWidget4x2.kt      # 4x2 コントロール型
    ├── ControlWidget4x2Receiver.kt
    ├── GridWidget2x2.kt         # 2x2 グリッド型
    ├── GridWidget2x2Receiver.kt
    ├── PixelPlayGlanceWidgetReceiver.kt
    ├── PlayerInfoStateDefinition.kt  # Glance State (DataStore + JSON)
    ├── PlayerControlActionCallback.kt # onAction → MusicService Intent 発行
    ├── WidgetArtworkDecoder.kt  # ウィジェット用アルバムアートデコーダ
    ├── WidgetComponents.kt      # 共通 Compose 部品
    ├── WidgetUpdateReceiver.kt  # ブロードキャスト受信 → 全 widget 再描画
    ├── WidgetUtils.kt           # AlbumArtBitmapCache / WidgetColors
    ├── IntentProvider.kt        # MainActivity Intent 生成
    └── subcomponents/
        └── WavyLinearProgressIndicator.kt  # 波形プログレスバー
```

## サブスペック

| ファイル | 内容 |
|----------|------|
| [`theme.md`](./theme.md) | `theme/` 全ファイル。`PixelPlayTheme` / `Typography` / `Shapes` / `ShapeCache` / `ColorRoles` (アルバムアート → 動的 ColorScheme) / `GenreColors` (固定 20 色 × dark/light) |
| [`widgets.md`](./widgets.md) | `glancewidget/` 全ファイル。Glance ウィジェット 4 種 / State 永続化 / アクション配信 / アートワークデコード / 共通コンポーネント / サブコンポーネント |

## 上位レイヤとの関係

| 依存先 | 用途 |
|--------|------|
| `presentation/viewmodel/ColorSchemePair` | 動的生成された ColorScheme のペア (`light` / `dark`) |
| `data/preferences/AlbumArtColorAccuracy` | カラー抽出精度レベル |
| `data/preferences/AlbumArtPaletteStyle` | パレット生成スタイル (TonalSpot / Expressive / …) |
| `data/model/Genre` | ジャンル ID / onLightColorHex / onDarkColorHex |
| `data/model/PlayerInfo` | ウィジェットに表示する曲情報 |
| `data/service/MusicService` | ウィジェットアクションの実命令先 |
| `presentation/MainActivity` | ウィジェットクリック時の遷移先 |

## 下位レイヤ (Compose / Glance) との関係

| 依存 | 用途 |
|------|------|
| `androidx.compose.material3.*` | ColorScheme / MaterialTheme / Shapes / Typography |
| `androidx.glance.*` | GlanceAppWidget / GlanceStateDefinition / GlanceTheme |
| `com.google.android.material.color.utilities.*` | DynamicScheme / Hct / QuantizerCelebi / SchemeTonalSpot ほか |
| `racra.compose.smooth_corner_rect_library` | `AbsoluteSmoothCornerShape` (スムーズな角丸) |
