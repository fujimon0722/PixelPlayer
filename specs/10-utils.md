# 10 — 共通ユーティリティ (`utils/`)

> **対象ディレクトリ**: `app/src/main/java/com/theveloper/pixelplay/utils/`
> **役割**: phone アプリ全体から横断的に使われるヘルパー。歌詞解析、アルバムアートキャッシュ / 抽出、ディレクトリ選択、ファイル削除、エンコード / デコード、ストレージ列挙、ショートカット管理、クラッシュログ、ネットワークリトライなど。

## 含まれるファイル一覧

| ファイル | 行数 | 主な役割 |
|----------|------|---------|
| `AlbumArtCacheManager.kt` | 298 | アルバムアートキャッシュ (200MB まで) のクリーンアップ / 孤立ファイル削除 |
| `AlbumArtUtils.kt` | 611 | アルバムアートの URI 解決 / キャッシュ / 埋め込み抽出 / 外部ファイル探索 |
| `AppLocaleManager.kt` | 61 | アプリ言語設定 (SharedPreferences 永続化、AppCompat 統合) |
| `AppShortcutManager.kt` | 79 | 「最後に再生したプレイリスト」ショートカット登録 / 削除 |
| `ArtworkTransportSanitizer.kt` | 132 | ウィジェット / Wear 向けアートワークのリサイズ / 再エンコード |
| `AudioDecoder.kt` | 127 | MediaCodec で音声を PCM Float 配列にデコード (ビジュアライザー用) |
| `AudioFileProvider.kt` | 141 | 音声ファイルをモノラル WAV にデコード (一時ファイル) |
| `ColorUtils.kt` | 142 | カラーコントラスト計算 / Hex→Color / 角丸 Bitmap 背景生成 |
| `CrashHandler.kt` | 136 | クラッシュログ保存 (SharedPreferences 経由、起動時表示) |
| `DirectoryFilterUtils.kt` | 26 | 許可 / 拒否ディレクトリ設定に基づくフィルタ |
| `DirectoryRuleResolver.kt` | 66 | ディレクトリ allowlist / blocklist 解決 |
| `Envelope.kt` | 28 | プログレス curve 計算 (`Curve` から値域 0-1 への envelope) |
| `Extensions.kt` | 201 | Color → Hex、文字列正規化、アーティスト分割 / 抽出 |
| `FileDeletionUtils.kt` | 174 | API 別 (11+ / 10 / legacy) ファイル削除 |
| `Formats.kt` | 100 | 再生時間 / 曲数 / 相対時間 / 総再生時間フォーマット |
| `LocalArtworkUri.kt` | 110 | `pixelplay_local_art://` カスタムスキーム URI ヘルパー |
| `LogUtils.kt` | 38 | Timber ラッパ (タグ長 23 文字カット、フォーマット) |
| `LyricsImportSecurity.kt` | 300 | 歌詞インポート時の検証 / サイズ制限 / デコード (UTF-8/16) |
| `LyricsUtils.kt` | 1444 | LRC / Kugou / TTML 歌詞解析 + 9 言語ローマ字化 (Kuromoji, pinyin4j, 各国 Map) + BubblesLine / ProviderText Composable |
| `MediaItemBuilder.kt` | 358 | `Song` → Media3 `MediaItem` 構築 (URI / MIME / 外部 extras / アート) |
| `MediaMetadataRetrieverPool.kt` | 85 | `MediaMetadataRetriever` の作成数追跡 (プールは no-op) |
| `MediaStorePermissionHelper.kt` | 325 | MediaStore URI 解決、削除 / 書き込み `IntentSender` 作成 |
| `MediaStoreSelectionUtils.kt` | 45 | MediaStore `WHERE` 句 (MIDI 除外) ビルダー |
| `NetworkRetryUtils.kt` | 44 | 指数バックオフ付きリトライ (`IOException` / `HttpException`) |
| `PlaylistCoverColors.kt` | 29 | プレイリストカバーコンテンツカラー (luminance 判定) |
| `QueueUtils.kt` | 196 | Fisher-Yates シャッフル (anchor 固定) + `yield` 対応版 |
| `StorageUtils.kt` | 146 | `StorageManager` で利用可能ストレージ列挙、SD カード検出 |
| `TtmlLyricsParser.kt` | 205 | TTML → 拡張 LRC 変換 (`DocumentBuilderFactory` ベース) |
| `WavHeader.kt` | 49 | モノラル WAV ヘッダー生成 / 修正 |
| `ZipShareHelper.kt` | 220 | 複数曲 → ZIP 圧縮 → `ACTION_SEND` Intent 起動 |
| `shapes/OtherShapes.kt` | 232 | ヘキサゴン / 三角 / セミサークル形状 (`Shape` 実装) |
| `shapes/PolygonShape.kt` | 59 | N 角形 `Shape` |
| `shapes/RoundedStarShape.kt` | 72 | 角丸星 `Shape` |

> `shapes/` 配下の形状は UI 側で `Modifier.clip(...)` に渡して使用。

---

## AlbumArtCacheManager.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: アルバムアートキャッシュ (`filesDir/album_art/`) の LRU 風クリーンアップ。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `DEFAULT_MAX_CACHE_SIZE_BYTES` | `const val = 200L * 1024 * 1024` (200 MiB) | デフォルト上限 |
| `configuredCacheLimitMb` | `var Long = 200L` | 外部から設定可能な上限 (MB) |
| `cleanCacheIfNeeded` | `suspend (context, maxCacheSizeMb = 200L): Int` | 上限超過時に `CLEANUP_PERCENTAGE` (25%) 分を LRU 削除。直近 5 分以内は skip (Mutex 保護) |
| `cleanOrphanedCacheFiles` | `suspend (context, validSongIds: Set<Long>): Int` | 存在しない songId 由来のキャッシュを削除 |
| `getCacheSizeBytes` | `suspend (context): Long` | 現キャッシュサイズ (bytes) |
| `getCacheSizeFormatted` | `suspend (context): String` | `"XX.X MB"` 形式 |
| `getCachedFileCount` | `(context): Int` | ファイル数 |
| `clearAllCache` | `suspend (context): Int` | 全削除 |

### 内部実装メモ

- ファイル命名: `song_art_{songId}_v4.jpg` (キャッシュ本体) / `song_art_{songId}_v4_no.jpg` (no-art マーカー)
- 25% 削除: `artFiles.sortedBy { it.lastModified() }` で古い順から削除
- `Mutex` + `lastCleanupTime` で最小 cleanup 間隔 5 分
- 5 分以上経過していなければ no-op

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/utils/AlbumArtUtils.kt` — キャッシュ書き込み側
- `data/diagnostics/PerformanceMetrics.kt` — 統計連携

---

## AlbumArtUtils.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: アルバムアートの URI / キャッシュ / 埋め込み / 外部ファイル探索を包括的に扱うユーティリティ。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `getAlbumArtUri` | `(context, songId: Long, songTitle: String, songPath: String, songAlbumId: Long): Uri?` | 優先順: キャッシュ → MediaStore → 埋め込み → 外部ファイル → null |
| `getAlbumArtUriForLibraryScan` | `(appContext, songId, songTitle, songPath, songAlbumId): Uri?` | スキャン専用。no-art マーカーがあれば早期 null |
| `getCachedAlbumArtUri` | `(appContext, songId: Long): Uri?` | `filesDir/album_art/song_art_{id}_v4.jpg` への FileProvider URI |
| `hasCachedAlbumArt` | `(appContext, songId: Long): Boolean` | キャッシュ有無 |
| `getEmbeddedAlbumArtUri` | `(context, filePath, songId): Uri?` | 埋め込みアート → 一時ファイル → FileProvider URI |
| `ensureAlbumArtCachedFile` | `(appContext, songId, filePath?, mimeType?, size?): File?` | キャッシュ生成 (存在しなければ生成) |
| `openArtworkInputStream` | `(appContext, uri: Uri): InputStream?` | LocalArtworkUri / `file://` / MediaStore / embedded を解決してストリーム取得 |
| `getExternalAlbumArtUri` | `(filePath): Uri?` | ディレクトリ内 `cover.jpg` 等 → FileProvider URI |
| `getMediaStoreAlbumArtUri` | `(appContext, albumId: Long): Uri?` | `content://media/external/audio/albumart/{id}` |
| `saveAlbumArtToCache` | `(appContext, bytes, songId): Uri` | キャッシュ書き込み + FileProvider URI 返却 |
| `clearCacheForSong` | `(appContext, songId: Long)` | 特定曲のキャッシュ削除 |
| `getAlbumArtDir` | `(appContext): File` | `filesDir/album_art` ディレクトリ |
| `getCachedAlbumArtFile` | `(appContext, songId): File` | キャッシュファイル参照 |
| `migrateLegacyCacheLocation` | `(appContext)` | 旧 `cacheDir` 配下から `filesDir/album_art` へマイグレ |

### 内部実装メモ

- キャッシュ命名: `song_art_{songId}_v4.jpg` + `_no.jpg` マーカー
- キャッシュサイズ上限: 1536px / JPEG 90% / 900KB (`OVERSIZED_CACHED_ART_BYTES`)
- `artworkShrinkInFlight: ConcurrentHashMap.newKeySet<String>()` で重複縮小防止
- `commonArtworkFileNames`: `cover.jpg`, `folder.jpg`, `album.jpg` 等の候補名
- `genericMixedDirectoryNames`: 信頼しないディレクトリ ("Various Artists", "Unknown Artist" 等)
- 埋め込みアート抽出は `MediaMetadataRetrieverPool` 経由
- `scheduleOversizedArtworkShrink` は IO スコープで非同期縮小

### 呼び出し元

- `presentation` のプレイヤー画面 / アルバム詳細 / ウィジェット等
- `data/service/http/MediaFileHttpServerService` — アルバムアート HTTP 配信用
- `ui/glancewidget/WidgetArtworkDecoder` — `openArtworkInputStream` 経由

---

## AppLocaleManager.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: アプリ全体の言語設定 (SharedPreferences 永続化、AppCompat 統合)。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `currentLanguageTag` | `(context): String` | 保存済言語タグ取得 (`"app_locale_preferences"`) |
| `applyLanguage` | `(context, languageTag: String)` | `AppCompatDelegate.setApplicationLocales(LocaleListCompat.forLanguageTags(...))` |
| `wrapContext` | `(base: Context): Context` | API 33+ は native、API < 33 は `Configuration.setLocale` で `createConfigurationContext` |

### 内部実装メモ

- `PREFERENCES_NAME = "app_locale_preferences"`
- `KEY_LANGUAGE_TAG = "app_language_tag"`
- `AppLanguage.normalize` で正規化

### 関連ファイル

- `data/preferences/AppLanguage.kt` — 正規化ロジック
- `MainActivity.attachBaseContext` で `wrapContext` 呼び出し

---

## AppShortcutManager.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: 「最後に再生したプレイリスト」のホームショートカット管理。

### 公開 API (class, `@Singleton @Inject`)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `updateLastPlaylistShortcut` | `(playlistId: String, playlistName: String)` | ショートカット作成 / 更新 (`SHORTCUT_ID_LAST_PLAYLIST = "last_playlist"`) |
| `removeLastPlaylistShortcut` | `()` | ショートカット削除 |

### 内部実装メモ

- `userPreferencesRepository` から現在のプレイリスト ID / 名前のリストを取得してショートカットに反映
- `MainActivity` + `MainActivityIntentContract` で開く

---

## ArtworkTransportSanitizer.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: ウィジェット / Wear 向けのアートワーク縮小 + 再エンコード。

### 公開データクラス

| 名前 | フィールド |
|------|-----------|
| `Config` | `maxDimensionPx: Int`, `maxBytes: Int`, `initialJpegQuality: Int`, `minJpegQuality: Int`, `jpegQualityStep: Int`, `sourceBytesLimit: Int` |

### 公開定数

| 名前 | 用途 |
|------|------|
| `WIDGET_CONFIG` | ウィジェット用設定 |
| `WEAR_CONFIG` | Wear 転送用設定 |

### 公開 API

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `sanitizeEncodedBytes` | `(data: ByteArray?, config: Config): ByteArray?` | inSampleSize で縮小 → 必要なら scale → JPEG エンコード (`initialJpegQuality` から段階的に下げて `maxBytes` 以下) |

### 内部実装メモ

- `decodeBoundedBitmap` で inSampleSize 自動算出
- `scaleBitmapIfNeeded` で長辺を `maxDimensionPx` に縮小
- `encodeBitmap` で `initialJpegQuality` → `jpegQualityStep` 刻みで `minJpegQuality` まで試行

---

## AudioDecoder.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: MediaCodec で音声ファイルを PCM Float 配列にデコード (ビジュアライザー / 波形用)。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `decodeToFloatArray` | `suspend (context, uri, requiredSamples: Int): Result<FloatArray>` | `MediaExtractor` → 音声トラック検索 → `MediaCodec.createDecoderByType` → PCM 16bit / Float を `FloatArray` に変換。`requiredSamples` に満たない場合は 0 で padding |

### 内部実装メモ

- `ENCODING_PCM_16BIT = 2` / `ENCODING_PCM_FLOAT = 4`
- `byteBufferToFloatArray` で `KEY_PCM_ENCODING` を見て分岐
- 失敗時は `Result.failure`

---

## AudioFileProvider.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: 任意の音声ファイルをモノラル WAV にデコード (キャッシュに保存)。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `getWavFile` | `suspend (context, uri): Result<File>` | MediaExtractor → MediaCodec で PCM にデコード → ステレオ to モノラル → WAV ヘッダー付与 → `cacheDir/input_mono_{rand}.wav` 作成 |

### 内部実装メモ

- `WavHeader(0, 0, 0, 0, 1)` で初期化 → 書き出し後に `updateHeader(file)` でサイズを埋める
- `stereoToMono` は左右平均 (16bit signed)
- `TIMEOUT_US = 1000L`

---

## ColorUtils.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: カラー操作ヘルパー。

### 公開 API

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `getContrastColor` | `(color: Color): Color` | luminance 0.5 しきい値で白 / 黒を選択 (Android ColorUtils ベース) |
| `hexToColor` | `(hex: String?, defaultColor: Color = Color.Gray): Color` | `#RRGGBB` / `RRGGBB` / `#AARRGGBB` を Compose Color に変換 |
| `createScalableBackgroundBitmap` | `(context, topLeft, topRight, bottomRight, bottomLeft, color, refWidth?, refHeight?): Bitmap` | 個別の角丸 (px) を持つ Bitmap を生成。サイズが角丸合計より小さい場合は自動 scale |

### 内部実装メモ

- 角丸 Bitmap は `addRoundRect(RectF, floatArrayOf(tl, tl, tr, tr, br, br, bl, bl), CW)` + `isAntiAlias = true`
- スケール計算: `leftUnstretchable = max(tl, bl)` 等から min 200x200 を保証

---

## CrashHandler.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: クラッシュログを SharedPreferences に保存し、次回起動時に表示。

### 公開型

| 名前 | 種類 | フィールド |
|------|------|-----------|
| `CrashLogData` | data class | `timestamp: Long`, `formattedDate: String`, `exceptionMessage: String`, `stackTrace: String` |
| | | `getFullLog(): String` — 全ログを文字列化 |

### 公開 API (object, `Thread.UncaughtExceptionHandler`)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `install` | `(context: Context)` | `Thread.setDefaultUncaughtExceptionHandler(this)` |
| `uncaughtException` | `(thread, throwable)` | `saveCrashLog` → default handler 委譲 |
| `hasCrashLog` | `(): Boolean` | `KEY_HAS_CRASH = true` |
| `getCrashLog` | `(): CrashLogData?` | ログ取得 |
| `clearCrashLog` | `()` | 削除 |

### 内部実装メモ

- `PREFS_NAME = "crash_handler_prefs"`
- `KEY_HAS_CRASH` / `KEY_TIMESTAMP` / `KEY_EXCEPTION_MESSAGE` / `KEY_STACK_TRACE`
- `getStackTraceString` は `StringWriter` + `PrintWriter`
- 日付フォーマット: `dd/MM/yyyy HH:mm:ss`

---

## DirectoryFilterUtils.kt / DirectoryRuleResolver.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: ファイルディレクトリベースの allowlist / blocklist による楽曲フィルタ。

### `DirectoryFilterUtils` (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `computeAllowedParentDirs` | `suspend (allowed: List<String>, blocked: List<String>, getAllParentDirs: suspend () -> List<File>): List<File>` | 許可ルート配下かつ非拒否ルートの親ディレクトリのみを返す |

### `DirectoryRuleResolver` (class)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `isBlocked` | `(path: String): Boolean` | `blocked` 配下なら true、`allowed` 配下なら false (最長一致) |
| `normalize` | `(path: String?): String?` (private) | 絶対パス正規化 |
| `isParentOrSame` | `(root, path): Boolean` (private) | root が path の親または同じ |

### 内部実装メモ

- 設定に基づき allowlist / blocklist を最深一致で評価
- 許可リストが空でも拒否リストだけあればブロック

---

## Envelope.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: `Curve` 型からプログレス値域 0-1 を計算。

### 公開 API

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `envelope` | `(progress: Float, curve: Curve): Float` | 0-1 入力を `Curve` に基づき 0-1 出力に写像 (アニメリニア/イーズ用) |

---

## Extensions.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: 文字列 / Color 拡張関数群。

### 公開拡張

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `Color.toHexString()` | `(): String` | ARGB を `#AARRGGBB` に |
| `String?.normalizeMetadataText()` | `(): String?` | Windows-1252 誤認 (Ã, â 等) を検出して UTF-8 再デコード |
| `String?.normalizeMetadataTextOrEmpty()` | `(): String` | 上記 + 空文字 fallback |
| `String.splitArtistsByDelimiters(delimiters, wordDelimiters = DEFAULT_WORD_DELIMITERS)` | `(): List<String>` | `featuring`, `feat.`, `vs.` 等で分割 |
| `String.extractArtistsFromTitle()` | `(): Pair<List<String>, String>` | タイトルから "feat. X" 等を抽出 (ブラケット含む) |
| `List<String>.joinArtistsForDisplay(separator = ", ")` | `(): String` | 結合 |

### 内部実装メモ

- `DEFAULT_WORD_DELIMITERS`: featuring, feat., feat, ft., ft, vs., vs, versus, with, prod., prod
- `ESCAPE_SEQUENCE = "\\\\"` / `ESCAPE_PLACEHOLDER = "\u0000ESCAPED\u0000"` でデリミタ内のバックスラッシュを保護
- `WINDOWS_1252` Charset で再デコード

---

## FileDeletionUtils.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: API 別 (11+ / 10 / legacy) ファイル削除。

### 公開データクラス

| 名前 | フィールド |
|------|-----------|
| `FileInfo` | `exists: Boolean`, `isFile: Boolean`, `size: Long`, `canRead: Boolean`, `canWrite: Boolean` |

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `deleteFile` | `suspend (context, filePath): Boolean` | API で分岐 (30+: MediaStore / 29: `Context.delete` / < 29: `File.delete`) |
| `getDeleteRequestIntentSender` | `(context, filePath): IntentSender?` | API 30+ 用の `createDeleteRequest` |
| `deleteFiles` | `suspend (context, filePaths): List<Boolean>` | 複数削除 |
| `canDeleteFile` | `suspend (filePath): Boolean` | 書き込み権限 / 所有者チェック |
| `getFileInfo` | `suspend (filePath): FileInfo` | ファイル stat |

### 内部実装メモ

- API 30+ (`Build.VERSION_CODES.R`): `MediaStore.createDeleteRequest` → `IntentSender`
- API 29 (`Q`): `ContextCompat.startForegroundService` 経由の `ACTION_DELETE` 不要
- legacy: `File.delete()` 直接

---

## Formats.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: フォーマット関数群。

### 公開 API

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `formatDuration` | `(milliseconds: Long): String` | `mm:ss` または `h:mm:ss` |
| `formatTotalDuration` | `(songs: List<Song>): String` | 全曲合計を "X 時間 Y 分" に |
| `formatListeningDurationLong` | `(milliseconds: Long): String` | 詳細 "X 時間 Y 分 Z 秒" |
| `formatListeningDurationCompact` | `(milliseconds: Long): String` | 簡略 "X 時間 Y 分" |
| `formatSongCount` | `(count: Int): String` | 単数 / 複数形で Resources から |
| `formatTimeAgo` | `(timestamp: Long): String` | "今 / X 分前 / X 時間前 / X 日前 / X か月前 / X 年前" |

### 内部実装メモ

- `PixelPlayApplication.instance.applicationContext` 経由で Resources 取得
- すべて Context 不要の純粋関数として利用可能

---

## LocalArtworkUri.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: `pixelplay_local_art://song/{id}` カスタムスキーム URI ヘルパー。

### 公開定数

| 名前 | 値 |
|------|-----|
| `SCHEME` | `"pixelplay_local_art"` |
| `HOST_SONG` | `"song"` (private) |
| `CACHE_BUST_QUERY` | `"t"` (private) |

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `buildSongUri` | `(songId: Long): String` | `"pixelplay_local_art://song/{id}"` |
| `buildSongUriWithTimestamp` | `(songId: Long): String` | `?t={System.currentTimeMillis()}` 付き (キャッシュバスティング) |
| `isLocalArtworkUri(uriString: String?)` | `(): Boolean` | プレフィックス判定 |
| `isLocalArtworkUri(uri: Uri?)` | `(): Boolean` | Uri 版 |
| `parseSongId` | `(uriString): Long?` | ID 抽出 (失敗時 null) |
| `looksLikeVolatileArtworkUri` | `(uriString: String?): Boolean` | キャッシュファイル名 / Shared 形式含む |
| `parseSongIdFromVolatileArtworkUri` | `(uriString: String?): Long?` | ファイル名から ID 抽出 |
| `extractCacheBustToken` | `(uriString: String?): String?` | `?t=...` 抽出 |
| `isLikelyLocalMedia` | `(contentUriString: String): Boolean` | content://media の判定 |
| `resolveSongArtworkUri` | `(storedUri, songId): String?` | stored URI があればそれ、なければ Local URI 構築 |

### 内部実装メモ

- `ContentProvider` 経由 (`SharedArtworkContentProvider`) でローカルキャッシュ画像を外部 (Wear / ウィジェット) に共有
- `song_art_*` ファイル名からも ID を抽出可能

---

## LogUtils.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: Timber ラッパ (タグ長 23 文字カット + フォーマット)。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `d` / `i` / `w` / `e` / `v` | `(tagProvider: Any, message: String, vararg args: Any?)` | ログレベル別。`getTag` で 23 文字カット。`e` は `throwable: Throwable? = null` 追加 |

### 内部実装メモ

- タグ: インスタンスなら `javaClass.simpleName`、文字列ならそのまま
- `buildLogMessage` は caller クラス名 (Thread stack) を先頭に付与

---

## LyricsImportSecurity.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: 歌詞インポート時の検証 (サイズ / 形式 / BOM 検出)。

### 公開型

| 名前 | 種類 | フィールド / 値 |
|------|------|----------------|
| `ValidatedLyricsImport` | data class | `sanitizedContent: String`, `parsedLyrics: Lyrics` |
| `LyricsImportFailureReason` | enum | `EMPTY_CONTENT`, `UNSUPPORTED_FORMAT`, `FILE_TOO_LARGE`, `INVALID_UTF`, `PARSE_FAILED`, `LINE_TOO_LONG`, etc. |
| `LyricsImportValidationResult` | sealed interface | `Valid(value: ValidatedLyricsImport)` / `Invalid(reason)` |

### 公開定数

| 名前 | 値 |
|------|-----|
| `MAX_LYRICS_FILE_BYTES` | `256 * 1024` |
| `MAX_TTML_FILE_BYTES` | `1024 * 1024` |
| `MAX_LYRICS_TEXT_CHARS` | `50_000` |

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `pickerMimeTypes` | `(): Array<String>` | MimePicker 用 (`text/*`, `application/xml`, etc.) |
| `supportedFileExtensions` | `(): List<String>` | `lrc`, `ttml`, `xml`, `txt` 等 |
| `validateImportedLyricsFile` | `(inputStream, fileName, mimeType?): LyricsImportValidationResult` | 全体検証 |
| `validateLocalLyricsFile` | `(file: File): LyricsImportValidationResult` | ローカルファイル版 |
| `validateImportedLrcContent` | `(rawText: String): LyricsImportValidationResult` | テキスト直接 |
| `messageFor` | `(reason): String` | 理由 → エラーメッセージ |

### 内部実装メモ

- `LyricsDocumentFormat` (private enum): `LRC`, `TTML`, `PLAIN_TEXT` (拡張子 + MIME 対応)
- `validatePayload` (private): BOM 検出 → UTF-8 / UTF-16LE / UTF-16BE デコード → 改行コード正規化
- `decodeText` (private): `CharsetDecoder` で `REPORT` / `REPLACE`
- `sanitizeImportedLyrics` (private): NUL 削除 + 改行統一
- BOM 検出: `startsWithUtf8Bom` / `Utf16LeBom` / `Utf16BeBom`

### 関連ファイル

- `LyricsUtils.kt` — `parseLyrics` 呼び出し
- `TtmlLyricsParser.kt` — TTML 変換

---

## LyricsUtils.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: 歌詞解析 (LRC / Kugou / TTML) + 9 言語ローマ字化 + 歌詞 UI Composable。

### 公開 object: `MultiLangRomanizer`

#### 公開 API (ローマ字化)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `isJapanese(text, entireLyricsHasKana = false)` | `(): Boolean` | ひらがな / カタカナの存在 |
| `isKorean(text)` | `(): Boolean` | Hangul Unicode 範囲 |
| `isHindi(text)` | `(): Boolean` | Devanagari |
| `isPunjabi(text)` | `(): Boolean` | Gurmukhi |
| `isCyrillic(text)` | `(): Boolean` | Cyrillic 範囲 |
| `isChinese(text)` | `(): Boolean` | CJK 漢字範囲 |
| `isScriptThatNeedsRomanization(text)` | `(): Boolean` | 上記いずれかに該当 |
| `romanizeJapanese` | `(text: String): String?` | Kuromoji (lazy) 経由。カタカナ化 → Latin 変換 |
| `romanizeChinese` | `(text: String): String?` | pinyin4j + `POLYPHONE_OVERRIDE` / `CONTEXT_PINYIN` 辞書 |
| `romanizeKorean` | `(text: String): String` | Hangul Jamo 分解 + `HANGUL_ROMAJA_MAP` |
| `romanizeHindi` | `(text: String): String` | `DEVANAGARI_ROMAJI_MAP` (2 文字 + 1 文字) |
| `romanizePunjabi` | `(text: String): String` | `GURMUKHI_ROMAJI_MAP` |
| `romanizeCyrillic` | `(text: String): String?` | 各国別 (`processUkrainian` / `processBelarusian` / `processCyrillicWordByWord`) |

#### 内部実装メモ

- `kuromojiTokenizer: Tokenizer?` (lazy; `Tokenizer.Builder().build()`)
- HANGUL_ROMAJA_MAP: cho / jung / jong の 3 階層 Map
- `HANGUL_ROMAJA_MAP["jong"]` の context-aware 変換 (例: ㄴㄱ → ng)
- POLYPHONE_OVERRIDE: 100+ エントリの多音字辞書
- CONTEXT_PINYIN: 前後の文字に基づく読み判定

### 公開 object: `LyricsUtils`

#### 公開 API

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `parseLyrics` | `(lyricsText: String?): Lyrics` | TTML 検出時は変換、それ以外は LRC / Kugou 自動判定。`SyncedLine` リスト + 翻訳 / ローマ字付き |
| `syncedToLrcString` | `(syncedLines: List<SyncedLine>): String` | LRC 形式出力 `[mm:ss.xx]` |
| `plainToString` | `(plainLines: List<String>): String` | プレーンテキスト |
| `toLrcString` | `(lyrics: Lyrics, preferSynced = true): String` | `synced` 優先 / なければ `plain` |
| `pairTranslationLines` (internal) | `(lines: List<SyncedLine>): List<SyncedLine>` | 連続行を翻訳 / ローマ字としてペア化 |
| `stripLrcTimestamps` (internal) | `(value: String): String` | `[mm:ss]` タグ除去 |
| `isTranslationCreditLine` (internal) | `(line): Boolean` | `by :` 形式の翻訳者表記検出 |

#### 内部 Regex / 関数

| 名前 | 用途 |
|------|------|
| `LRC_LINE_REGEX` | `^\\[(\\d{2}):(\\d{2})[.:](\\d{2,3})](.*)$` |
| `LRC_WORD_REGEX` | `<(\\d{2}):(\\d{2})[.:](\\d{2,3})>([^<]*)` |
| `KUGOU_LINE_REGEX` | `^\\[(\\d+),(\\d+)](.*)$` |
| `KUGOU_WORD_PATTERN` | `<(\\d+),(\\d+),(\\d+)>([^<]*)` |
| `stripLeadingLyricsDocumentNoise` (private) | テキスト先頭の BOM / 制御文字削除 |
| `looksLikeTtmlDocument` / `String.startsWithTtmlRoot()` | TTML 検出 |
| `sanitizeLrcLine` (private) | 改行 / キャリッジリターン除去 |
| `stripFormatCharacters` (private) | 制御文字 / `\\u0000` 削除 |
| `looksLikeKugouFormat` (private) | Kugou 形式判定 (カンマ区切りタイムスタンプ) |
| `parseKugouLyrics` (private) | Kugou 拡張 LRC パース |
| `MULTI_LANG_ROMAJI_DISABLED` | フラグ (推定) |

### 公開 @Composable

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `ProviderText` | `(text: String, url: String?, provider: String, accentColor: Color? = null, textAlign: TextAlign? = null, modifier: Modifier = Modifier)` | "by : Provider" 形式でクリック可能リンク表示 |
| `BubblesLine` | `(positionFlow: Flow<Long>, time: Long, nextTime: Long, color: Color, modifier: Modifier = Modifier)` | 音符 / 円の泡が y 振動 + 形態遷移するアニメーション (Path lerp) |

### `BubblesLine` 内部実装メモ

- `rememberInfiniteTransition` + `animateFloat` (0-1f, 1500ms, LinearEasing, RepeatMode.Restart)
- 3 個のバブル (位相 i * 1/3) を y オフセット `sin(progress * 2π) * 8dp`
- 円形 ↔ 音符ベクター (pathData) の形態補間 (`lerpPath`)
- `lerpPath` は PathNode.MoveTo / CurveTo の各座標を lerp
- 円作成: `kappa = 0.552284749831f` (Bezier 円近似定数)

### 関連ファイル

- `TtmlLyricsParser.kt` — TTML 変換
- `data/model/Lyrics.kt` / `SyncedLine` / `SyncedWord` — 出力モデル
- `com.atilika.kuromoji.ipadic` — 日本語形態素解析
- `net.sourceforge.pinyin4j` — 中国語ピンイン

---

## MediaItemBuilder.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: `Song` → Media3 `MediaItem` 構築。

### 公開定数 (`EXTERNAL_EXTRA_*`)

外部コントローラ (Wear / Android Auto / Bluetooth) に楽曲メタデータを Bundle で渡すためのキー定数群:
`FLAG`, `ALBUM`, `DURATION`, `CONTENT_URI`, `ALBUM_ART`, `GENRE`, `TRACK`, `YEAR`, `DATE_ADDED`, `MIME_TYPE`, `BITRATE`, `SAMPLE_RATE`, `FILE_PATH`, `NAVIDROME_ID`。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `build` | `(song: Song): MediaItem` | 標準 `MediaItem` 構築 |
| `buildForExternalController` | `(context, song: Song): MediaItem` | External 用の `extras` Bundle 付き |
| `playbackUri` | `(song): Uri` / `(contentUriString, filePath, mimeType): Uri` | 内部 / 外部 / 直接ファイル URI 選択 |
| `playbackMimeType` | `(internal)` | MIME 選択 |
| `directLocalFileUri` (private) | `(filePath, mimeType): Uri?` | `file://` 直接 |
| `shouldPreferDirectLocalFileUri` | `(filePath, contentUriString, mimeType): Boolean` | ローカル再生向け判定 |
| `artworkUri` | `(rawArtworkUri): Uri?` | アート URI 解決 |
| `externalControllerArtworkUri` | `(context, rawArtworkUri): Uri?` | External 用 (SharedArtworkContentProvider 経由) |
| `buildMediaMetadataForSong` (private) | `(song): MediaMetadata` | Media3 メタデータ |

### 内部実装メモ

- `EXTERNAL_MEDIA_ID_PREFIX = "external:"` (外部ソース曲 ID プレフィックス)
- `DIRECT_FILE_URI_MIME_TYPES` / `DIRECT_FILE_URI_EXTENSIONS` で直接ファイル URI 候補を判定
- `EXTRACTOR_FIRST_MIME_TYPES` は extractor 優先 (FLAC 等)
- `SUPPORTED_INTERNAL_ARTWORK_SCHEMES` / `SUPPORTED_EXTERNAL_ARTWORK_SCHEMES` で URI スキームを許可
- `isInsideAppStorage` で内部ストレージ判定 → FileProvider / 直接 URI 切替
- 外部コントローラ向け Bundle には全メタデータを詰め込み

---

## MediaMetadataRetrieverPool.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: `MediaMetadataRetriever` の作成数追跡 (現状プールは no-op)。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `acquire` | `(internal): MediaMetadataRetriever` | 常に新規作成 |
| `release` | `(internal): Unit` | `retriever.release()` |
| `withRetriever` | `inline <T> (block: (retriever) -> T): T` | acquire / release / finally |
| `clear` | `(): Unit` | no-op (将来用) |
| `poolSize` | `(): Int = 0` | プール未実装 |
| `totalCreated` | `(): Int` | `AtomicInteger` カウント |

### 内部実装メモ

- `AtomicInteger` で累計作成数のみ追跡
- プール化は未実装 (将来拡張用フック)

---

## MediaStorePermissionHelper.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: MediaStore URI 解決、削除 / 書き込み `IntentSender` 作成。

### 公開データクラス

| 名前 | フィールド |
|------|-----------|
| `DeleteRequest` | `intentSender: IntentSender`, `acceptedUris: List<Uri>`, `rejectedUris: List<Uri>` |

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `getMediaStoreUri` | `(context, songId: Long): Uri?` | ボリューム名込みで MediaStore URI 取得 |
| `getMediaStoreUri` | `(songId: Long): Uri?` | Context なし版 (デフォルト) |
| `getAudioMediaStoreUris` | `(context, songId: Long): List<Uri>` | 複数ボリューム候補 |
| `isMediaStoreItemUriString` | `(uriString: String): Boolean` | content://media 判定 |
| `canUseSongIdForMediaStoreRequest` | `(uriString: String): Boolean` | 数字 ID 部分のみ使用可能か |
| `resolveDeleteRequestUri` | `(context, contentUriString): Uri?` | 削除用 URI 解決 |
| `parseMediaStoreItemUri` (private) | `(uriString): Uri?` | 内部 |
| `getMediaStoreUri` | `(context, filePath: String): Uri?` | ファイルパス → MediaStore URI |
| `createWriteRequestIntentSender` | `(context, uris: List<Uri>): IntentSender?` | 書き込み IntentSender |
| `createDeleteRequestIntentSender` | `(context, uris: List<Uri>): IntentSender?` | 削除 IntentSender |
| `createDeleteRequest` | `(context, uris): DeleteRequest?` | 削除リクエスト (許可 / 拒否 URI 振り分け) |
| `createPlatformDeleteRequest` (private) | `(context, uris): IntentSender?` | 内部 |
| `createWriteRequestForSong` | `(context, songId: Long): IntentSender?` | 単曲用 |
| `createDeleteRequestForSong` | `(context, songId: Long): IntentSender?` | 単曲用 |

### 内部実装メモ

- API 29+ で `MediaStore.VOLUME_EXTERNAL` 以外のボリュームも考慮
- `getContentUri(VOLUME_EXTERNAL)` と `Files.getContentUri("external")` の両方サポート
- `MEDIASTORE_AUTHORITY = "media"`

---

## MediaStoreSelectionUtils.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: MediaStore クエリ用 `WHERE` 句 / 選択引数ビルダー。

### 公開 API (file-private constants + function)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `buildLocalAudioSelection` | `(minDurationMs: Int): Pair<String, Array<String>>` | `IS_MUSIC != 0 AND DURATION > minDuration AND (MIME NOT IN (?...) OR (MIME = ? AND DATA NOT LIKE ?))` で MIDI ファイルを除外 |

### 内部実装メモ

- `MIDI_MIME_SELECTION_ARGS`: `audio/midi`, `audio/x-midi`, `audio/sp-midi`, `audio/spmidi`
- `MIDI_EXTENSION_SELECTION_ARGS`: `.mid`, `.midi`
- 拡張子は case-insensitive (LOWER 使用)

---

## NetworkRetryUtils.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: 指数バックオフ付きリトライ。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| (関数) | `suspend <T> (maxAttempts: Int = 3, initialDelayMs: Long = 1000L, backoffMultiplier: Double = 2.0, maxDelayMs: Long = 10000L, block: suspend (attempt: Int) -> T): T` | リトライ。`shouldRetry` で `IOException` / `HttpException` 判定、`CancellationException` は再 throw |
| `Throwable.isRetryableNetworkError` | `(): Boolean` | `IOException` または `HttpException` なら true |

### 内部実装メモ

- バックオフ: `delayMs *= backoffMultiplier`、`min(delayMs, maxDelayMs)`
- 最終試行で失敗 / retryable=false で `throw throwable`

---

## PlaylistCoverColors.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: プレイリストカバー色 luminance 判定。

### 公開 API

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `resolvePlaylistCoverContentColor` | `(colorScheme: ColorScheme, backgroundColor: Color): Color` | background luminance > 0.5 で `onLight`、それ以外 `onDark` から選択 |

---

## QueueUtils.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: Fisher-Yates シャッフル (anchor 固定 / suspending 版)。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `buildAnchoredShuffleQueue` | `(currentQueue: List<Song>, anchorIndex: Int, random: Random = Random.Default): List<Song>` | 現曲を 0 番目に固定し、残りをランダム順 |
| `buildAnchoredShuffleQueueSuspending` | `suspend (currentQueue, anchorIndex, startAtZero: Boolean = true, random: Random = Random.Default): List<Song>` | `yield` 挟み込み版 |
| `generateShuffleOrder` (private) | `(size, anchorIndex, random): IntArray` | 0 番目固定のインデックス順列 |
| `generateShuffleOrderStartAtZero` (private) | `suspend` | `yield` 挟み込み版 |
| `generateShuffleOrderSuspending` (private) | `suspend` | 同上 (旧 API) |

### 内部実装メモ

- `SHUFFLE_YIELD_BATCH = 512` (スワップ何回ごとに `yield`)
- `clampedAnchor = anchorIndex.coerceIn(0, size - 1)`
- `IntArray(size - 1)` を Fisher-Yates → 先頭に anchor を挿入

---

## StorageUtils.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: `StorageManager` で利用可能ストレージ列挙。

### 公開型

| 名前 | 種類 | フィールド |
|------|------|-----------|
| `StorageType` | enum | `INTERNAL`, `EXTERNAL_SD`, `EXTERNAL_USB` |
| `StorageInfo` | data class | `path: File`, `displayName: String`, `storageType: StorageType`, `isRemovable: Boolean` |

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `getAvailableStorages` | `(context): List<StorageInfo>` | `StorageManager.storageVolumes` 列挙 |
| `getSdCardStorage` | `(context): StorageInfo?` | `EXTERNAL_SD` を探す |
| `hasExternalStorage` | `(context): Boolean` | 外部ストレージ有無 |
| `getInternalStorage` | `(): StorageInfo` | `Environment.getExternalStorageDirectory()` ベース |

### 内部実装メモ

- `getVolumePath` (private): `volume.javaClass.getMethod("getPath")` リフレクション
- `determineStorageType` (private): description / isPrimary / removable で判定
- USB には連番で `USB 1`, `USB 2`... 名前を付与

---

## TtmlLyricsParser.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: TTML → 拡張 LRC 変換。

### 公開 API (internal object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `parseToEnhancedLrc` | `(ttmlText: String): String?` | TTML テキストを LRC 文字列に変換 |

### 内部実装メモ

- `DocumentBuilderFactory` で secure processing 設定 (XXE 防御)
- `MAX_TTML_PARAGRAPHS = 5_000` 上限
- `parseTimeExpression`: `HH:MM:SS.fff` / `MM:SS.fff` / `SS.fff` / `SSs` を ms に
- `formatLrcTimestamp`: ms → `[mm:ss.cc]`
- `serializeChildren` / `serializeNode` / `serializeElement` で inline テキスト構築
- `sanitizeTextFragment` / `normalizeInlineText` / `normalizeParagraphBody` で改行 / 余白整理

---

## WavHeader.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: モノラル WAV ヘッダー生成 / 修正。

### 公開 API (class)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `WavHeader(fileSize, subchunk2Size, sampleRate, bitsPerSample, numChannels)` | constructor | ヘッダーオブジェクト生成 |
| `asByteArray()` | `(): ByteArray` | 44 バイト WAV ヘッダー (RIFF / fmt / data チャンク) |
| `updateHeader(file: File)` | `(): Unit` | ファイル末尾から遡って size 情報を書き換え |

### 内部実装メモ

- 44 バイト固定ヘッダー
- ファイルサイズ動的計算 (AudioFileProvider.getWavFile で利用)

---

## ZipShareHelper.kt

**パッケージ**: `com.theveloper.pixelplay.utils`
**役割**: 複数曲を ZIP 圧縮 → `ACTION_SEND` で他アプリに共有。

### 公開 API (object)

| 名前 | シグネチャ | 目的 |
|------|-----------|------|
| `createAndShareZip` | `suspend (context, songs: List<Song>): Intent?` | `cacheDir/shared_zips/PixelPlay_Songs_{ts}.zip` 作成 → `ACTION_SEND` chooser 起動 |
| `calculateTotalSize` | `suspend (songs): Long` | 合計サイズ計算 (ContentResolver open) |
| `isLargeZip` | `(totalSizeBytes: Long): Boolean` | 100MB 超過判定 |
| `formatFileSize` | `(bytes: Long): String` | "1.2 MB" / "3.4 GB" 形式 |
| `cleanupTempZips` | `(context)` | `cacheDir/shared_zips` 配下全削除 |
| `cleanupOldZips` (private) | `(zipDir)` | 1 時間以上前を削除 |
| `shareZipFile` (private) | `(context, zipUri, songCount)` | Intent 起動 |
| `sanitizeFileName` (private) | `(name): String` | ファイル名禁則文字除去 |
| `getFileExtension` (private) | `(path): String` | 拡張子抽出 (`"mp3"` fallback) |

### 内部実装メモ

- `ZIP_CACHE_DIR = "shared_zips"`, `BUFFER_SIZE = 8192`, `MAX_RECOMMENDED_SIZE_BYTES = 100 * 1024 * 1024L`
- File 名重複時は `{base}_{counter}.{ext}` で連番付与
- FileProvider URI で `ACTION_SEND` extra stream に設定

---

## shapes/

| ファイル | 役割 |
|----------|------|
| `OtherShapes.kt` (232 行) | `createHexagonShape()`, `createRoundedTriangleShape(cornerRadius)`, `createSemiCircleShape(cornerRadius)`, `createRoundedHexagonShape(cornerRadius)` |
| `PolygonShape.kt` (59 行) | `PolygonShape(sides, rotation)` — N 角形 |
| `RoundedStarShape.kt` (72 行) | `RoundedStarShape(sides, curve, rotation, iterations)` — 角丸星 (curve=0.09 default) |

### `PolygonShape` 内部実装メモ

- `sides.coerceAtLeast(3)` で 3 以上にクランプ
- `stepCount = 2π / sideCount`
- 中心 `(width/2, height/2)`、半径 `min(w, h) * 0.5`

### `RoundedStarShape` 内部実装メモ

- `iterations` を 360 にクランプ
- `pointAt(t) = (r * (cos(t) * (1 + curve*cos(sides*t))), r * (sin(t) * (1 + curve*cos(sides*t))))`
- `mapRange` で `curve` を 0.0-1.0 → 0.5-1.0 にマッピング (radius に乗算)

### `OtherShapes` 内部実装メモ

- `createHexagonShape`: 6 角形、radius = min/2
- `createRoundedHexagonShape(cornerRadius)`: 6 角形 + 各頂点に arcTo で 60° 弧
- `createRoundedTriangleShape(cornerRadius)`: 3 角形 + quadraticTo で角丸
- `createSemiCircleShape(cornerRadius)`: 180° 弧 + 両端 arcTo
