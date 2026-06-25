# theme — カラー / タイポグラフィ / シェイプ / 動的カラースキーム

> **パッケージ**: `com.theveloper.pixelplay.ui.theme`
> **役割**: Material3 ベースの `ColorScheme` / `Typography` / `Shapes` を提供し、アルバムアートやジャンルに基づく動的カラー生成を担当する。

## ファイル一覧

| ファイル | 行数 | 役割 |
|----------|------|------|
| `Color.kt` | 23 | 静的パレット (Purple / Pink / Orange / LightBackground ほか) |
| `ColorRoles.kt` | 729 | アルバムアート → 動的 `ColorSchemePair` 生成 (Hct + QuantizerCelebi) |
| `GenreColors.kt` | 246 | ジャンル ID → 固定 20 色パレット + 動的生成ブリッジ |
| `Shape.kt` | 11 | Material3 `Shapes` (small=8 / medium=16 / large=24) |
| `ShapeCache.kt` | 48 | `AbsoluteSmoothCornerShape` シングルトンキャッシュ (8-32dp + Pill) |
| `Theme.kt` | 148 | `PixelPlayTheme` / `PixelPlayStatusBarStyle` / CompositionLocal |
| `Type.kt` | 220 | `Typography` (Google Sans Flex Rounded) + `MontserratFamily` + `ExpTitleTypography` |

---

## Color.kt

**パッケージ**: `com.theveloper.pixelplay.ui.theme`
**役割**: ダーク / ライト両用の静的パレットを Color 定数として公開する。

### 公開定数 (val)

| 定数 | 値 | 用途 |
|------|-----|------|
| `PixelPlayPurpleDark` | `0xFF1E1234` | ダーク background |
| `PixelPlayPurplePrimary` | `0xFFAB47BC` | ダーク primary |
| `PixelPlayPink` | `0xFFF06292` | secondary (ダーク / ライト共通) |
| `PixelPlayOrange` | `0xFFFF8A65` | tertiary (ダーク / ライト共通) |
| `PixelPlayLightPurple` | `0xFFE1BEE7` | ダーク onSurface |
| `PixelPlayWhite` | `0xFFFFFFFF` | onPrimary / onError |
| `PixelPlayBlack` | `0xFF000000` | ライト onTertiary |
| `PixelPlaySurface` | `0xFF2A1F40` | ダーク surface |
| `LightBackground` | `0xFFF7F2FF` | ライト background |
| `LightSurface` | `0xFFFBF8FF` | ライト surface |
| `LightSurfaceVariant` | `0xFFE8DEF9` | ライト surfaceVariant |
| `LightOnSurface` | `0xFF1E1237` | ライト onSurface / onBackground |
| `LightOnSurfaceVariant` | `0xFF4D4165` | ライト onSurfaceVariant |
| `LightPrimary` | `0xFF6C4FF5` | ライト primary / surfaceTint |
| `LightPrimaryContainer` | `0xFFE3DBFF` | ライト primaryContainer |
| `LightOnPrimaryContainer` | `0xFF23005C` | ライト onPrimaryContainer |
| `LightOutline` | `0xFF78659A` | ライト outline / outlineVariant (alpha 0.6) |

### 呼び出し元

- `app/src/main/java/com/theveloper/pixelplay/ui/theme/Theme.kt:71-106` — `DarkColorScheme` / `LightColorScheme` の組み立て
- `app/src/main/java/com/theveloper/pixelplay/ui/theme/ColorRoles.kt` — フォールバックカラー

---

## ColorRoles.kt

**パッケージ**: `com.theveloper.pixelplay.ui.theme`
**役割**: アルバムアート画像 (`Bitmap`) から代表色を抽出し、`light` / `dark` 双方の Material3 `ColorScheme` を生成する。Material Color Utilities (Hct / QuantizerCelebi / SchemeTonalSpot ほか) を使用。

### 依存関係

- **上流**: `presentation/viewmodel/ColorSchemePair.kt` (結果型), `data/preferences/AlbumArtColorAccuracy.kt`, `data/preferences/AlbumArtPaletteStyle.kt`
- **下流**: `com.google.android.material.color.utilities.*` — DynamicScheme / Hct / MathUtils / QuantizerCelebi / SchemeExpressive / SchemeFruitSalad / SchemeMonochrome / SchemeTonalSpot / SchemeVibrant
- **下流**: `androidx.core.graphics.scale` — Bitmap 縮小

### public API

| シグネチャ | 戻り値 | 目的 | 呼び出し元 |
|------------|--------|------|-----------|
| `fun clearExtractedColorCache()` | `Unit` | LRU キャッシュ (32 件) の `evictAll` | 設定画面 / メモリ圧迫時 |
| `fun extractSeedColor(bitmap: Bitmap, config: ColorExtractionConfig = ColorExtractionConfig()): Color` | `Color` | アルバムアートから代表シード色を抽出 (キャッシュ済みなら再利用) | アルバム詳細画面 / プレイヤー画面 |
| `fun generateColorSchemeFromSeed(seedColor: Color, paletteStyle: AlbumArtPaletteStyle = default): ColorSchemePair` | `ColorSchemePair` | シード色から dynamic ColorScheme (light/dark) ペアを生成。グレースケール判定時はグレースケール版に強制変換 | プレイヤー画面 / アルバム詳細 |
| `fun generateMonochromeColorSchemeFromSeed(seedColor: Color): ColorSchemePair` | `ColorSchemePair` | SchemeMonochrome ベースの強制モノクロペアを生成 | `GenreColors.getColorSchemeFromSeed` (forceMonochrome 時) |
| `internal fun selectSeedColorArgbFromPixels(pixels: IntArray, config: ColorExtractionConfig): Int` | `Int` (ARGB) | ピクセル配列 → シード色 ARGB (内部: extractSeedColor から呼び出し) | — |

### 公開データクラス

| 名前 | フィールド | 説明 |
|------|-----------|------|
| `ColorScoringConfig` | `targetChroma=48.0`, `weightProportion=0.7`, `weightChromaAbove=0.3`, `weightChromaBelow=0.1`, `cutoffChroma=5.0`, `cutoffExcitedProportion=0.01`, `maxColorCount=4`, `maxHueDifference=90`, `minHueDifference=15` | 候補色スコアリング重み |
| `ColorExtractionConfig` | `downscaleMaxDimension=128`, `quantizerMaxColors=128`, `scoring=ColorScoringConfig()`, `accuracyLevel=AlbumArtColorAccuracy.DEFAULT` | 抽出パイプライン設定。`normalizedAccuracy` プロパティで 0.0-1.0 に正規化 |

### 内部 private 関数 (主要)

| 関数 | 役割 |
|------|------|
| `resizeForExtraction(bitmap, maxDimension): Bitmap` | maxDimension に縮小 (inSampleSize 計算なし、直接 scale) |
| `scoreQuantizedColors(...): Int` | 候補色 Hct をスコアリング (彩度 / 出現率 / 代表色への忠実度 / 過剰彩度ペナルティ) |
| `calculateRepresentativeArtworkColor(pixels, accuracy): RepresentativeArtworkColor?` | ピクセル重み付き平均で代表色を計算 (透過 / 彩度低 / グレースケール除外) |
| `calculateRepresentativeFidelityScore(candidate, representative, accuracy): Double` | 候補色が代表色にどれだけ近いか (hue / chroma / tone 距離の重み付きスコア) |
| `calculateExcessChromaPenalty(candidate, representative, accuracy): Double` | 候補色の彩度が代表色より高すぎる場合のペナルティ |
| `refineSeedColorArgb(...)` | 局所平均 (hue ウィンドウ) と代表色との混合でシード色を精製 |
| `blendArgb(firstArgb, secondArgb, ratio): Int` | ARGB を線形補間 |
| `lerpDouble(start, stop, fraction): Double` | 精度レベルによるパラメータ補間 |
| `lerpFloat(start, stop, fraction): Float` | Float 版 |
| `createDynamicScheme(sourceHct, paletteStyle, isDark): DynamicScheme` | paletteStyle に応じて SchemeTonalSpot / Vibrant / Expressive / FruitSalad を選択 |
| `averageColorArgb(pixels): Int` | 単純 RGB 平均 |
| `isMostlyNeutralArtwork(colorsToPopulation): Boolean` | 中性色 (低彩度) 主体の画像かを判定 (REQUIRED_NEUTRAL_POPULATION=0.92) |
| `shouldUseNeutralArtworkScheme(argb, sourceHct): Boolean` | グレースケールスキーム強制使用の判定 |
| `isArgbNearGrayscale(argb): Boolean` | RGB チャンネル間の差が MAX_GRAYSCALE_CHANNEL_DELTA (10) 以内 |
| `ColorScheme.toGrayscaleColorScheme(): ColorScheme` | 全ロールを HSL で脱色 (saturation=0) |
| `DynamicScheme.toComposeColorScheme(): ColorScheme` | Material Color Utilities → Compose Material3 ColorScheme 変換 |

### 内部実装メモ

- LRU キャッシュ容量: 32 件 (key = `bitmap.hashCode() * 31 + config.hashCode()`)。
- グレースケール判定閾値は複数段階 (GRAYSCALE_CHROMA_THRESHOLD=12.0, NEUTRAL_PIXEL_CHROMA_THRESHOLD=8.0, HIGH_CHROMA_THRESHOLD=18.0) で多層防御。
- 精度レベル (`accuracyLevel`) は `ACCURATE_*` 定数との `lerpDouble` で段階的に重みを切り替え (例: FIDELITY_HUE_WINDOW 90→52, CHROMA_WINDOW 32→18, etc.)。
- 候補色は Hct 色空間でスコアリングされ、最終的に `maxColorCount=4` まで hue 重複 (`minHueDifference`/`maxHueDifference`) を避けつつ選別。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/ui/theme/Theme.kt` — フォールバック ColorScheme
- `app/src/main/java/com/theveloper/pixelplay/ui/theme/GenreColors.kt` — ジャンル別スキーム
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ColorSchemePair.kt` — 戻り値型

---

## GenreColors.kt

**パッケージ**: `com.theveloper.pixelplay.ui.theme`
**役割**: ジャンル ID / `Genre` データモデルに対応する `GenreThemeColor` または完全 `ColorScheme` を提供する。`Genre.darkColorHex` / `lightColorHex` / `onDarkColorHex` / `onLightColorHex` を尊重し、なければ固定 20 色パレットにフォールバック。

### 依存関係

- **上流**: `data/model/Genre` (id, darkColorHex, lightColorHex, onDarkColorHex, onLightColorHex), `data/preferences/AlbumArtPaletteStyle`
- **下流**: `android.util.LruCache` (ColorScheme キャッシュ 96 件)
- **下流**: `ColorRoles.kt` (動的 ColorScheme 生成)

### 公開型

| 名前 | 種類 | フィールド | 説明 |
|------|------|-----------|------|
| `GenreThemeColor` | data class | `container: Color`, `onContainer: Color` | ジャンル用 2 色ペア |
| `GenreThemeUtils` | object | — | ファサード |

### GenreThemeUtils public API

| シグネチャ | 戻り値 | 目的 | 呼び出し元 |
|------------|--------|------|-----------|
| `fun getGenreThemeColor(genreId: String, isDark: Boolean): GenreThemeColor` | `GenreThemeColor` | 文字列 ID → 20 色パレット (hash ベース) | ジャンルカード UI |
| `fun getGenreThemeColor(genre: Genre?, isDark: Boolean, fallbackGenreId: String = "unknown"): GenreThemeColor` | `GenreThemeColor` | `Genre` モデル → カスタムカラー or ハッシュ色。`onContainer` は `onDarkColorHex`/`onLightColorHex`、なければ luminance から自動算出 (90% 白/黒と lerp) | ジャンルヘッダー / カード |
| `fun getGenreColorScheme(genre: Genre?, isDark: Boolean, genreIdFallback: String = "unknown", paletteStyle: AlbumArtPaletteStyle = EXPRESSIVE): ColorScheme` | `ColorScheme` | 完全な Material3 `ColorScheme` を取得 (キャッシュ済みなら再利用) | ジャンル詳細画面 |
| `fun getGenreColorScheme(genreId: String, isDark: Boolean, paletteStyle: AlbumArtPaletteStyle = EXPRESSIVE): ColorScheme` | `ColorScheme` | ID 直接版 | — |
| `fun getGenreDetailColorScheme(genre: Genre?, isDark: Boolean, fallbackGenreId: String = "unknown", paletteStyle: AlbumArtPaletteStyle = default): ColorScheme` | `ColorScheme` | 詳細画面用 (常に Genre 由来のコンテナ色を seed として使用) | ジャンル詳細 |
| `fun getColorSchemeFromSeed(seedColor: Color, isDark: Boolean, paletteStyle: AlbumArtPaletteStyle = EXPRESSIVE, forceMonochrome: Boolean = false): ColorScheme` | `ColorScheme` | seed → 完全 `ColorScheme` への変換 (キャッシュ対応) | 内部 / 直接利用 |

### 内部実装メモ

- 20 色の固定パレット (dark / light 各 20) は `darkColors` / `lightColors` としてハードコード (Blue, Rose, Pink, Cyan, Green, Gold, Slate, Purple, Red, Lime, Teal, Indigo, Maroon, Yellow, Navy, Steel Blue, Brick Red, Grey, Violet, Amber)。
- "unknown" / "unknown genre" / "unknown_genre" は特別扱いされ、`unknownSeedColor = 0xFF7C7D84` ベースのモノクロスキームに強制。
- `isUnknownGenreId` は case-insensitive + trim で判定。
- LRU キャッシュ: 96 件。`buildColorSchemeCacheKey` で `seed.toArgb * 31 + isDark + paletteStyle.ordinal * 31 + forceMonochrome` をキーに。
- `contrastContentColor`: luminance ≤ 0.5 で白 90% と lerp、> 0.5 で黒 90% と lerp。`genre` の onColorHex がない場合のフォールバック。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/model/Genre.kt` — 入力モデル
- `app/src/main/java/com/theveloper/pixelplay/data/preferences/AlbumArtPaletteStyle.kt` — スキーム生成スタイル

---

## Shape.kt

**パッケージ**: `com.theveloper.pixelplay.ui.theme`
**役割**: Material3 `Shapes` の最小限定義。アプリ全体では `ShapeCache` のスムーズ角丸が併用される。

### 公開定数

| 定数 | 値 | 用途 |
|------|-----|------|
| `Shapes` | Material3 `Shapes(small=8dp, medium=16dp, large=24dp)` | ボタン / カード / ボトムシートのデフォルト |

### 内部実装メモ

- `RoundedCornerShape` ベース。スムーズ (AbsoluteSmoothCornerShape) 版は `ShapeCache` 経由で利用。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/ui/theme/ShapeCache.kt` — スムーズ角丸キャッシュ
- `app/src/main/java/com/theveloper/pixelplay/ui/theme/Theme.kt:144` — MaterialTheme.shapes に注入

---

## ShapeCache.kt

**パッケージ**: `com.theveloper.pixelplay.ui.theme`
**役割**: `AbsoluteSmoothCornerShape` (Bézier 計算で生成される高コストなスムーズ角丸) のシングルトンキャッシュ。LazyColumn 等での Path 再構築を防ぐ OPT #6 パフォーマンス改善。

### 公開プロパティ (object ShapeCache)

| プロパティ | 角丸 | 用途 |
|-----------|------|------|
| `smooth8` | 8dp | コンパクトチップ・小サーフェス |
| `smooth10` | 10dp | — |
| `smooth12` | 12dp | 曲リストアイテム・小カード |
| `smooth14` | 14dp | — |
| `smooth16` | 16dp | アルバムカード・プレイリストアイテム |
| `smooth20` | 20dp | 大型カード |
| `smooth24` | 24dp | ダイアログ |
| `smooth28` | 28dp | — |
| `smooth32` | 32dp | ボトムシート・フローティングパネル |
| `smoothPill` | 50dp | ボタン・チップ (pill 形) |

全て `smoothnessAsPercent = 60`。

### 内部実装メモ

- 外部ライブラリ `racra.compose.smooth_corner_rect_library.AbsoluteSmoothCornerShape` を使用。
- 各プロパティは `val` (eager 初期化) で、`Shape` 実装を毎回生成するコストを排除。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/ui/theme/Theme.kt:144` — デフォルト Shapes は併用
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/` — 各 Composable で `Modifier.clip(ShapeCache.smoothN)` として使用

---

## Theme.kt

**パッケージ**: `com.theveloper.pixelplay.ui.theme`
**役割**: アプリ全体の `MaterialTheme` 適用、StatusBar / NavigationBar 制御、CompositionLocal 提供。

### 公開 CompositionLocal

| 名前 | デフォルト値 | 用途 |
|------|-------------|------|
| `LocalPixelPlayDarkTheme` | `false` | 現在ダークテーマか |
| `LocalShowScrollbar` | `true` | スクロールバー表示フラグ |

### 公開 Composable / 定数

| 名前 | シグネチャ | 目的 | 呼び出し元 |
|------|-----------|------|-----------|
| `PixelPlayStatusBarStyle` | `@Composable fun PixelPlayStatusBarStyle(color: Color, useDarkIcons: Boolean = luminance>0.55, navigationColor: Color? = null, useDarkNavigationIcons: Boolean = …)` | システムバーの色・アイコン明暗を `SideEffect` で適用。`Preview` モードでは no-op | アプリ全体の `Scaffold` / `MainActivity` |
| `DarkColorScheme` | `val ColorScheme` | ダーク fallback カラー (Purple ベース) | `Theme.kt` 内部フォールバック / `ColorRoles.kt` 例外時 |
| `LightColorScheme` | `val ColorScheme` | ライト fallback カラー (LightPrimary ベース) | `Theme.kt` 内部フォールバック / `ColorRoles.kt` 例外時 |
| `PixelPlayTheme` | `@Composable fun PixelPlayTheme(darkTheme: Boolean = isSystemInDarkTheme(), colorSchemePairOverride: ColorSchemePair? = null, content: @Composable () -> Unit)` | アプリ最外殻。優先順位: ① `colorSchemePairOverride` ② Android 12+ dynamic colors ③ 静的 `DarkColorScheme` / `LightColorScheme` | `MainActivity.onCreate` / `WearMainActivity` |

### 内部実装メモ

- `Context.findActivity()`: tailrec で `ContextWrapper` を剥がして `Activity` を取得。`PixelPlayStatusBarStyle` で使用。
- `PixelPlayStatusBarStyle` の `useDarkIcons` デフォルトは `ColorUtils.calculateLuminance(color.toArgb()) > 0.55` (W3C 基準)。
- `Build.VERSION.SDK_INT >= Q` (API 29) 以上で `isStatusBarContrastEnforced = false` / `isNavigationBarContrastEnforced = false` を設定 (半透明背景でもコントラスト強制を無効化)。
- `colorSchemePairOverride` はアルバムアート由来。`darkTheme` フラグで `light` / `dark` を選択。
- `LocalPixelPlayDarkTheme provides darkTheme` で配下に通知。
- `@Suppress("DEPRECATION")`: API < 30 の `window.statusBarColor` 等の非推奨 API を許容。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/MainActivity.kt` — ルート呼び出し
- `app/src/main/java/com/theveloper/pixelplay/ui/theme/Color.kt` — パレット
- `app/src/main/java/com/theveloper/pixelplay/ui/theme/Type.kt` — Typography
- `app/src/main/java/com/theveloper/pixelplay/ui/theme/Shape.kt` — Shapes
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ColorSchemePair.kt` — 動的スキーム受け取り型

---

## Type.kt

**パッケージ**: `com.theveloper.pixelplay.ui.theme`
**役割**: アプリ全体のタイポグラフィ定義。Google Sans Flex (Variable Font) を ROND 軸 100 で疑似的に丸ゴシック化し、`Montserrat` もサブセットとして用意。

### 公開プロパティ

| 名前 | 種類 | 説明 |
|------|------|------|
| `MontserratFamily` | `FontFamily` | 7 ウェイト (Black/ExtraBold/Bold/SemiBold/Medium/Normal/Light) を GoogleFonts Provider 経由で取得 |
| `ExpTitleTypography` | `Typography` | 実験的タイトル用 (`displayLarge` 60sp scaleX=1.5、`titleMedium` 32sp scaleX=1.3)。`TextGeometricTransform` で水平方向に伸長 |
| `GoogleSansRounded` | `FontFamily` | `R.font.gflex_variable` (Variable Font) を 5 ウェイト分、`ROND` 軸 = 100 で丸ゴシック化。`ExperimentalTextApi` 必須 |
| `Typography` | `Typography` | アプリ標準。Google Sans Rounded を全 13 ロール (display/headline/title/body/label × L/M/S) に適用。`displayLarge`=48sp Bold, `bodyMedium`=14sp Normal letterSpacing=0.25sp, `labelSmall`=11sp Medium letterSpacing=0.5sp |

### 内部実装メモ

- `montserrat` は `GoogleFont("Montserrat")` 宣言。
- `provider` は `com.google.android.gms.fonts` Authority + `R.array.com_google_android_gms_fonts_certs` 証明書配列。Play Services Fonts 互換。
- `GoogleSansRounded` の `ROND = 100f` で最大丸み。
- `PlatformTextStyle(includeFontPadding = false)` を含めることで Material3 既定の上部パディングを抑制 (ExpTitleTypography のみ)。`Typography` には含めず、各 Composable 側で制御する設計。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/ui/theme/Theme.kt:143` — `MaterialTheme(typography = Typography, ...)` として注入
- `app/src/main/res/font/gflex_variable.ttf` — Variable Font リソース
- `app/src/main/res/values/strings.xml` (com_google_android_gms_fonts_certs 配列)
