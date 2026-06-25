# viewmodels-support.md

> PlayerViewModel が扱う主要なデータクラスと、`ColorSchemeProcessor` / `DeckController` 等のサポートオブジェクトの詳細仕様。

---

## PlayerUiState

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlayerUiState.kt` (68 行)
**役割**: プレイヤーとライブラリ横断の UI 状態を 1 つの data class に集約。`PlayerViewModel._playerUiState` で使用。

### フィールド一覧 (50+)

#### 再生キュー

| フィールド | 型 | 目的 |
|----------|---|------|
| `currentPlaybackQueue` | `ImmutableList<Song>` | 現在の再生キュー |
| `currentQueueSourceName` | `String` | キュー名 ("All Songs" / "Playlist Name" 等) |

#### ライブラリ / 検索

| フィールド | 型 | 目的 |
|----------|---|------|
| `searchResults` | `ImmutableList<SearchResultItem>` | 検索結果 |
| `musicFolders` | `ImmutableList<MusicFolder>` | 音楽フォルダ |
| `searchHistory` | `ImmutableList<SearchHistoryItem>` | 検索履歴 |
| `searchQuery` | `String` | 検索クエリ (一時) |
| `selectedSearchFilter` | `SearchFilterType` | ALL / SONGS / ALBUMS / ARTISTS / PLAYLISTS |
| `currentStorageFilter` | `StorageFilter` | ALL / OFFLINE / CLOUD |
| `hideLocalMedia` | `Boolean` | ローカル非表示 |
| `isLoadingInitialSongs` | `Boolean` | 初回楽曲読込中 |
| `isLoadingLibrary` | `Boolean` | ライブラリ読込中 |
| `isLoadingLibraryCategories` | `Boolean` | アルバム/アーティスト読込中 |
| `isFiltering` | `Boolean` | フィルタ適用中 |
| `filteredSongs` | `ImmutableList<Song>` | フィルタ後楽曲 |
| `isSyncingLibrary` | `Boolean` | ライブラリ同期中 |

#### 並び替え

| フィールド | 型 | 目的 |
|----------|---|------|
| `currentSongSortOption` | `SortOption` | 楽曲並び順 |
| `currentAlbumSortOption` | `SortOption` | アルバム並び順 |
| `currentArtistSortOption` | `SortOption` | アーティスト並び順 |
| `currentFolderSortOption` | `SortOption` | フォルダ並び順 |
| `currentFavoriteSortOption` | `SortOption` | お気に入り並び順 |
| `isAlbumsListView` | `Boolean` | アルバム一覧モード |

#### AI

| フィールド | 型 | 目的 |
|----------|---|------|
| `showAiPlaylistSheet` | `Boolean` | AI シート表示 |
| `isGeneratingAiPlaylist` | `Boolean` | AI 生成中 |
| `aiStatus` | `String?` | ステータス |
| `aiError` | `String?` | エラー |

#### フォルダ

| フィールド | 型 | 目的 |
|----------|---|------|
| `currentFolder` | `MusicFolder?` | 現在のフォルダ |
| `currentFolderPath` | `String?` | パス |
| `folderSource` | `FolderSource` | INTERNAL / SD_CARD |
| `folderSourceRootPath` | `String` | ストレージのルートパス |
| `isSdCardAvailable` | `Boolean` | SD カード利用可 |
| `isFolderFilterActive` | `Boolean` | フォルダフィルタ |
| `isFoldersPlaylistView` | `Boolean` | Folder pseudo playlist 表示 |
| `folderBackGestureNavigationEnabled` | `Boolean` | バックで親へ |

#### 取り消し (Undo)

| フィールド | 型 | 目的 |
|----------|---|------|
| `showQueueItemUndoBar` | `Boolean` | キュー削除 Undo バー |
| `lastRemovedQueueSong` | `Song?` | 直前削除曲 |
| `lastRemovedQueueIndex` | `Int` | 元の位置 |
| `showDismissUndoBar` | `Boolean` | プレイリスト閉じ Undo バー |
| `dismissedSong` | `Song?` | 閉じた時の曲 |
| `dismissedQueue` | `ImmutableList<Song>` | 閉じた時のキュー |
| `dismissedQueueName` | `String` | キュー名 |
| `dismissedPosition` | `Long` | 再生位置 |
| `undoBarVisibleDuration` | `Long` | Undo バー表示時間 (デフォルト 4000ms) |

#### その他

| フィールド | 型 | 目的 |
|----------|---|------|
| `lavaLampColors` | `ImmutableList<Color>` | Lava Lamp モード色 |
| `preparingSongId` | `String?` | 準備中曲 (UI spinner) |

### 内部実装メモ

- **Immutable 化**: 全 List フィールドが `ImmutableList<>` で、`persistentListOf()` がデフォルト。
- **default 値**: 各フィールドは安全な初期値 (`emptyList()`, `false`, `0L` 等) を持つ。
- **`PlayerViewModel` での update**: `_playerUiState.update { it.copy(...) }` で全フィールドを immutable に更新。

### 関連ファイル

- 書き込み側: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlayerViewModel.kt`

---

## StablePlayerState

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/StablePlayerState.kt` (22 行)
**役割**: 再生中の「安定スナップショット」。`PlaybackStateHolder.stablePlayerState` で公開。

### フィールド

| フィールド | 型 | デフォルト | 目的 |
|----------|---|---|------|
| `currentSong` | `Song?` | `null` | 現在の曲 |
| `currentMediaItemIndex` | `Int` | `-1` | キュー内位置 |
| `isPlaying` | `Boolean` | `false` | 再生中 |
| `playWhenReady` | `Boolean` | `false` | 再生意図 |
| `totalDuration` | `Long` | `0L` | 曲の総時間 |
| `isShuffleEnabled` | `Boolean` | `false` | シャッフル |
| `isShuffleTransitionInProgress` | `Boolean` | `false` | シャッフル遷移中 |
| `repeatMode` | `Int` | `Player.REPEAT_MODE_OFF` | リピート |
| `isLoadingLyrics` | `Boolean` | `false` | 歌詞読込中 |
| `lyrics` | `Lyrics?` | `null` | 歌詞 |
| `isBuffering` | `Boolean` | `false` | バッファリング |

### 内部実装メモ

- **`@Immutable`**: Compose のリコンポジション最適化用。
- **`Player.REPEAT_MODE_OFF = 0`**: デフォルト。
- **「安定」の意味**: PlayerController の頻繁な状態更新を間引き、UI に対して不必要な再描画を避けるスナップショット。

### 関連ファイル

- 書き込み側: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlaybackStateHolder.kt`

---

## PlayerSheetState

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlayerSheetState.kt` (7 行)
**役割**: Bottom Sheet (プレイヤー) の表示状態 enum。

### 定義

```kotlin
enum class PlayerSheetState {
    COLLAPSED,  // ミニプレイヤー
    EXPANDED,   // フルプレイヤー
    HIDDEN      // 非表示 (Close ボタン)
}
```

### 関連ファイル

- 使用箇所: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlayerViewModel.kt` (`_sheetState`)
- `app/src/main/java/com/theveloper/pixelplay/presentation/components/ScreenWrapper.kt`

---

## ColorSchemePair

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ColorSchemePair.kt` (9 行)
**役割**: アルバムアート由来の Light + Dark Material3 ColorScheme ペア。

### 定義

```kotlin
data class ColorSchemePair(
    val light: ColorScheme,
    val dark: ColorScheme
)
```

### 内部実装メモ

- **Material3 `ColorScheme`**: `androidx.compose.material3.ColorScheme` を使用。
- **永続化**: `AlbumArtThemeDao` で `AlbumArtThemeEntity` に格納 (詳細は `../01-data-foundation/album-art-theme.md` 推定)。

### 関連ファイル

- 生成元: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ColorSchemeProcessor.kt`
- 利用: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ThemeStateHolder.kt`

---

## ColorSchemeProcessor

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ColorSchemeProcessor.kt` (452 行)
**アノテーション**: `@Singleton`
**役割**: アルバムアート URI → Bitmap → Seed Color → Material3 ColorScheme (Light / Dark) の変換。永続化 (Room `AlbumArtThemeDao`) + メモリキャッシュ (LruCache)。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Context | `context: Context` (ApplicationContext) |
| DAO | `albumArtThemeDao: AlbumArtThemeDao` |

### 内部状態

| 名前 | 目的 |
|------|------|
| `memoryCache: LruCache<String, ColorSchemePair>` | メモリキャッシュ (容量 20) |
| `processingMutex: Mutex` | 並行処理制御 |
| `inProgressUris: MutableSet<String>` | 処理中 URI 追跡 |
| `requestChannel: Channel<String>` | 処理要求キュー (capacity 32, DROP_OLDEST) |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `suspend fun getOrGenerateColorScheme(albumArtUri, paletteStyle, colorAccuracyLevel): ColorSchemePair?` | メインエントリ (DB / メモリ / 生成 / キャッシュ) |
| `suspend fun getPreviewColorScheme(uri, paletteStyle, colorAccuracyLevel): ColorSchemePair?` | 一時プレビュー (永続化しない) |
| `private suspend fun generateAndCacheColorScheme(uri, paletteStyle, colorAccuracyLevel): ColorSchemePair` | 内部生成 |
| `private suspend fun loadBitmapForColorExtraction(uri, skipCache): Bitmap?` | Coil で Bitmap 取得 |
| `suspend fun markProcessing(uri: String): Boolean` | 処理開始マーク (重複防止) |
| `suspend fun markComplete(uri: String)` | 処理完了マーク |
| `fun clearMemoryCache()` | メモリキャッシュ全消去 |
| `fun evictFromCache(uri: String)` | 特定 URI 削除 |
| `suspend fun invalidateScheme(uri: String)` | メモリ + DB 削除 |
| `private fun mapColorSchemePairToEntity(uri, schemePair, paletteStyle, accuracy): AlbumArtThemeEntity` | 永続化用変換 |
| `private fun mapEntityToColorSchemePair(entity): ColorSchemePair` | DB → ColorSchemePair |
| `private suspend fun loadCachedColorScheme(uri, paletteStyle, accuracy): AlbumArtThemeEntity?` | DB キャッシュ取得 |
| `private fun buildCacheKey(uri, paletteStyle, accuracy): String` | キャッシュキー生成 |

### 内部実装メモ

- **キャッシュキー**: `"$CACHE_KEY_SEPARATOR$uri|$paletteStyle|$accuracy"` (e.g. `|content://media/audio/123|VIBRANT|50`)。`CACHE_KEY_SEPARATOR = "|"`, `CACHE_ALGORITHM_VERSION = "algo_v7"`。
- **生成パイプライン**:
  1. `memoryCache.get(cacheKey)` → ヒットなら返却
  2. `loadCachedColorScheme` で DB から取得 → あればメモリキャッシュに格納
  3. `markProcessing` で重複防止
  4. `loadBitmapForColorExtraction` で Bitmap 取得
  5. `extractSeedColor(bitmap, ...)` でシード色抽出
  6. `generateColorSchemeFromSeed(seed, isDark = true/false)` で ColorScheme 2 セット生成
  7. DB へ永続化
  8. メモリキャッシュへ格納
  9. `markComplete`
- **`AlbumArtPaletteStyle`**: VIBRANT / MUTED / PASTEL / EARTHY / CUSTOM。`CUSTOM` はシード色をそのまま使用。
- **Local Artwork URI**: `LocalArtworkUri.isLocalArtworkUri(uri)` でローカル判定 → ディスクキャッシュ無効化 (毎回 Bitmap 再読込)。
- **Color accuracy**: 1-100 のサンプリング精度。`AlbumArtColorAccuracy.clamp` で正規化。
- **永続化戦略**: 同じ (uri, paletteStyle, accuracy) に対して 1 度だけ生成 → 永続化。次回以降は DB から即時取得。
- **MP3 埋め込みアート**: ファイルパスから直接 Bitmap 取得 (Coil バイパス)。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/database/AlbumArtThemeDao.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/database/AlbumArtThemeEntity.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/database/StoredColorSchemeValues.kt`
- カラーパレット生成: `app/src/main/java/com/theveloper/pixelplay/ui/theme/extractSeedColor.kt` / `generateColorSchemeFromSeed.kt`
- `app/src/main/java/com/theveloper/pixelplay/utils/LocalArtworkUri.kt`

---

## LyricsSearchUiState

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/LyricsSearchUiState.kt` (14 行)
**役割**: 歌詞検索 UI の状態。`LyricsStateHolder.searchUiState` で公開。

### 派生 (sealed interface)

| 派生 | 目的 |
|------|------|
| `Idle` (object) | 初期状態 |
| `Loading` (object) | 検索中 |
| `PickResult(query, results: List<LyricsSearchResult>)` | 候補表示 |
| `Success(lyrics: Lyrics)` | 取得成功 |
| `NotFound(message, allowManualSearch: Boolean = true)` | 見つからず |
| `Error(message, query?)` | エラー |

### 関連ファイル

- 書き込み側: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/LyricsStateHolder.kt`
- 画面: 歌詞検索 BottomSheet (推定 `LyricsSearchSheet.kt` 等)

---

## DeckController

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel.exts`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/exts/DeckController.kt` (153 行)
**アノテーション**: なし (Plain class)
**役割**: DJ マッシュアップ用の独立 ExoPlayer ラッパ。HiRes / Offload 無効化など特別な設定。

### コンストラクタ

```kotlin
class DeckController(
    private val context: Context
)
```

### 内部状態

| 名前 | 目的 |
|------|------|
| `player: ExoPlayer?` | 構築された ExoPlayer (loadSong で生成) |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun loadSong(songUri: Uri)` | ExoPlayer 構築 + 曲ロード |
| `fun playPause()` | 再生/一時停止 |
| `fun seek(progress: Float)` | 0-1 でシーク |
| `fun setSpeed(speed: Float)` | 0.5-2.0 で速度設定 |
| `fun nudge(amountMs: Long)` | ±ms 微小シーク |
| `fun setDeckVolume(deckVolume: Float)` | 0-1 ボリューム |
| `fun getProgress(): Float` | 0-1 進捗 |
| `fun release()` | ExoPlayer 解放 |
| `private fun buildSafePlayer(): ExoPlayer` | カスタムレンダラ付き ExoPlayer |

### 内部実装メモ

- **カスタム `DefaultRenderersFactory`**:
  - `buildAudioSink` で `HiResSampleRateCapAudioProcessor` + `SurroundDownmixProcessor` を挿入
  - ビデオ / テキスト / カメラモーションレンダラは無効化 (音楽専用)
- **Offload 無効化**: `AudioOffloadPreferences.Builder().setEnabled(false)` で常にソフトウェア処理。
- **再生属性**: `AudioAttributes.Builder().setUsage(C.USAGE_MEDIA).setContentType(C.AUDIO_CONTENT_TYPE_MUSIC).build()`。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/service/player/HiResSampleRateCapAudioProcessor.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/service/player/SurroundDownmixProcessor.kt`
- 使用: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/MashupViewModel.kt`

---

## PlayerUiState 更新ヘルパー (exts 内)

**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlayerUiState.kt` 内 / `exts/` 内

### `replaceSong(updatedSong: Song)` (extension)

| シグネチャ | 戻り値 | 目的 |
|-----------|--------|------|
| `private fun ImmutableList<Song>.replaceSong(updatedSong: Song): ImmutableList<Song>` | 新しい ImmutableList | キュー内の曲を更新 |

### `removeSongById(songId: String)` (extension, private)

| シグネチャ | 戻り値 | 目的 |
|-----------|--------|------|
| `private fun ImmutableList<Song>.removeSongById(songId: String): ImmutableList<Song>` | 新しい ImmutableList | ID で削除 |

### `moveSong(fromIndex: Int, toIndex: Int)` (extension, private)

| シグネチャ | 戻り値 | 目的 |
|-----------|--------|------|
| `private fun ImmutableList<Song>.moveSong(fromIndex: Int, toIndex: Int): ImmutableList<Song>` | 新しい ImmutableList | 並び替え |

### `moveQueueIndex(index: Int, fromIndex: Int, toIndex: Int): Int` (private)

| シグネチャ | 戻り値 | 目的 |
|-----------|--------|------|
| `private fun moveQueueIndex(index: Int, fromIndex: Int, toIndex: Int): Int` | 新しい index | シャッフルトグル時のインデックス再計算 |

### `List<Song>.toPlaybackQueue()` (internal)

| シグネチャ | 戻り値 | 目的 |
|-----------|--------|------|
| `internal fun List<Song>.toPlaybackQueue(): ImmutableList<Song>` | `ImmutableList<Song>` | 通常リストを再生キュー形式に変換 |

### `ImmutableList<Song>.asPersistentPlaybackQueue()` (internal)

| シグネチャ | 戻り値 | 目的 |
|-----------|--------|------|
| `internal fun ImmutableList<Song>.asPersistentPlaybackQueue(): PersistentList<Song>` | `PersistentList<Song>` | kotlinx.collections.immutable の Persistent へ変換 |

### 内部実装メモ

- すべて `PlayerViewModel.kt` の `private` / `internal` extension 関数。
- ImmutableList 操作で常に新インスタンスを返すため、`StateFlow.update { it.copy(currentPlaybackQueue = newQueue) }` で安全。

---

## ListeningStatsTracker (再掲)

詳細は `viewmodels-settings-stats.md` の § ListeningStatsTracker を参照。

---

## `PlayerViewModel` の内部 data class

### `AiUiSnapshot` (private)

| フィールド | 目的 |
|----------|------|
| `showAiPlaylistSheet: Boolean` | AI シート |
| `isGeneratingAiPlaylist: Boolean` | AI 生成中 |
| `aiStatus: String?` | ステータス |
| `aiError: String?` | エラー |

### `SortOptionsSnapshot` (private)

| フィールド | 目的 |
|----------|------|
| `songSort: SortOption` | 楽曲並び順 |
| `albumSort: SortOption` | アルバム並び順 |
| `artistSort: SortOption` | アーティスト並び順 |
| `folderSort: SortOption` | フォルダ並び順 |
| `favoriteSort: SortOption` | お気に入り並び順 |

### `FullPlayerSlice`

| フィールド | 目的 |
|----------|------|
| `currentSongArtists: List<Artist>` | 曲のアーティスト |
| `lyricsSyncOffset: Int` | オフセット |
| `albumArtQuality: AlbumArtQuality` | アート品質 |
| `audioMetadata: PlaybackAudioMetadata` | 音声メタ |
| `showPlayerFileInfo: Boolean` | ファイル情報表示 |
| `immersiveLyricsEnabled: Boolean` | 没入歌詞 |
| `immersiveLyricsTimeout: Long` | タイムアウト |
| `isImmersiveTemporarilyDisabled: Boolean` | 一時無効 |
| `isRemotePlaybackActive: Boolean` | Cast |
| `selectedRouteName: String?` | ルート名 |
| `isBluetoothEnabled: Boolean` | BT |
| `bluetoothName: String?` | BT 名 |

### `PlayerConfigSlice`

| フィールド | 目的 |
|----------|------|
| `navBarCornerRadius: Int` | 角丸 |
| `navBarStyle: String` | スタイル |
| `carouselStyle: String` | カルーセル |
| `fullPlayerLoadingTweaks: FullPlayerLoadingTweaks` | 遅延設定 |
| `tapBackgroundClosesPlayer: Boolean` | タップで閉じる |
| `useSmoothCorners: Boolean` | スムーズコーナー |
| `playerThemePreference: String` | プレイヤー色テーマ |
