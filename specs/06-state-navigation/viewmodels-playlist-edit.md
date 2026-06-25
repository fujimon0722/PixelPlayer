# viewmodels-playlist-edit.md

> プレイリスト / 選択 / メタデータ編集 / 楽曲情報 系 StateHolder / ViewModel の詳細仕様。`PlaylistViewModel` (1228 行) / `PlaylistSelectionStateHolder` / `MetadataEditStateHolder` (901 行) / `SongInfoBottomSheetViewModel` (430 行) / `MultiSelectionStateHolder` (505 行) / `SongRemovalStateHolder` (378 行) / `QueueUndoStateHolder` / `PlaylistDismissUndoStateHolder` を扱う。

---

## PlaylistViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlaylistViewModel.kt` (1228 行)
**アノテーション**: `@HiltViewModel`
**役割**: プレイリスト一覧 / 詳細 / 作成 / 削除 / マージ / エクスポート / AI 生成 / 並び替え / スマートプレイリスト / Telegram クラウド可視性の全機能。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Preferences | `playlistPreferencesRepository: PlaylistPreferencesRepository` |
| Repository | `musicRepository: MusicRepository` |
| AI | `dailyMixManager: DailyMixManager` (スマートプレイリスト用) |
| AI | `aiPlaylistGenerator: AiPlaylistGenerator` |
| Util | `m3uManager: M3uManager` |
| Context | `context: Context` (ApplicationContext) |

### `PlaylistUiState` (data class)

| フィールド | 目的 |
|----------|------|
| `playlists: List<Playlist>` | 全プレイリスト |
| `showTelegramCloudPlaylists: Boolean` | Telegram クラウドプレイリスト表示 |
| `telegramTopicDisplayMode: TelegramTopicDisplayMode` | トピック表示モード (CHANNELS_ONLY / CHANNELS_AND_TOPICS / TOPICS_ONLY) |
| `currentPlaylistSongs: List<Song>` | 詳細表示中の楽曲 |
| `currentPlaylistDetails: Playlist?` | 詳細表示中プレイリスト |
| `isLoading: Boolean` | 読込中 |
| `playlistNotFound: Boolean` | 詳細が見つからない |
| `currentPlaylistSortOption: SortOption` | プレイリスト一覧のソート |
| `currentPlaylistSongsSortOption: SortOption` | 詳細内の楽曲ソート |
| `playlistSongsOrderMode: PlaylistSongsOrderMode` | Manual / Sorted |
| `playlistOrderModes: Map<String, PlaylistSongsOrderMode>` | プレイリスト別保存モード |
| `isAiGenerating: Boolean` | AI 生成中 |
| `aiGenerationError: String?` | AI エラー |

### `PlaylistSongsOrderMode` (sealed class)

| 派生 | 用途 |
|------|------|
| `Manual` | ユーザー手動並び替え |
| `Sorted(option: SortOption)` | 自動ソート |

### StateFlow / SharedFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<PlaylistUiState>` | 画面状態 |
| `playlistCreationEvent` | `SharedFlow<Boolean>` | 作成成功通知 (true=作成済) |

### 主要 public 関数

#### 初期化 / 監視

| シグネチャ | 目的 |
|-----------|------|
| `fun sanitizeFileName(name: String): String` (companion) | ファイル名サニタイズ |
| `private fun observePlaylistOrderModes()` | プレイリスト別 order mode 復元 |
| `private fun loadPlaylistsAndInitialSortOption()` | 起動時読込 |
| `fun setTelegramTopicDisplayMode(mode: TelegramTopicDisplayMode)` | トピック表示切替 |
| `private fun observeTelegramCloudPlaylistVisibility()` | 表示フラグ監視 |
| `private fun observeTelegramTopicDisplayMode()` | 表示モード監視 |
| `fun sortPlaylists(sortOption: SortOption)` | プレイリスト一覧ソート |

#### 詳細

| シグネチャ | 目的 |
|-----------|------|
| `fun loadPlaylistDetails(playlistId: String)` | 詳細読込 (Folder pseudo / 通常 / スマート) |
| `fun sortPlaylistSongs(sortOption: SortOption)` | 詳細楽曲ソート (Manual → Sorted 切替) |
| `fun reorderSongsInPlaylist(playlistId, fromIndex, toIndex)` | Manual reorder |

#### 作成 / 削除

| シグネチャ | 目的 |
|-----------|------|
| `fun createPlaylist(name, songIds, coverImageUri?, cropScale, cropPanX, cropPanY, smartRuleKey?)` | 新規作成 + カバー画像処理 |
| `fun deletePlaylist(playlistId: String)` | 単体削除 |
| `fun deletePlaylistsInBatch(playlistIds: List<String>)` | 一括削除 |
| `fun renamePlaylist(playlistId, newName)` | リネーム |
| `fun updatePlaylistParameters(playlistId, name, coverImageUri?, cropScale, cropPanX, cropPanY)` | パラメータ更新 (カバー再生成含む) |

#### 楽曲追加 / 削除

| シグネチャ | 目的 |
|-----------|------|
| `fun addSongsToPlaylist(playlistId, songIdsToAdd)` | 楽曲追加 (重複防止) |
| `fun addOrRemoveSongFromPlaylists(songId, playlistIds)` | トグル操作 |
| `fun addSongsToPlaylists(songIds, playlistIds)` | 一括追加 |
| `fun removeSongFromPlaylist(playlistId, songIdToRemove)` | 1 曲削除 |

#### マージ / エクスポート

| シグネチャ | 目的 |
|-----------|------|
| `fun mergeSelectedPlaylists(playlistIds, newPlaylistName)` | 複数プレイリストをマージ (重複除去) |
| `fun mergePlaylistsIntoOne(playlistIds, newPlaylistName)` | 同上 (内部実装) |
| `suspend fun getPlaylistsWithSongs(playlistIds): List<Pair<Playlist, List<Song>>>` | (Playlist, songs) のリスト |
| `fun shareSelectedPlaylistsAsZip(playlistIds, activity?)` | ZIP にして共有 (FileProvider) |
| `fun exportPlaylistsAsM3u(playlistIds)` | M3U ファイル出力 (外部ストレージへ) |
| `fun importM3u(uri: Uri)` | M3U インポート |
| `fun exportM3u(playlist, uri, context)` | 単体 M3U エクスポート |

#### AI / スマートプレイリスト

| シグネチャ | 目的 |
|-----------|------|
| `fun generateAiPlaylist(prompt, minLength, maxLength)` | AI プレイリスト生成 |
| `fun clearAiError()` | エラークリア |
| `private fun buildSmartPlaylistSongIds(rule, limit): List<String>` | スマートプレイリスト用 ID 解決 |
| `private suspend fun saveCoverImageToInternalStorage(uri, cropScale, cropPanX, cropPanY): String?` | カバー画像保存 |

#### ユーティリティ (private)

| シグネチャ | 目的 |
|-----------|------|
| `private fun isFolderPlaylistId(playlistId: String): Boolean` | `folder_playlist:` prefix 判定 |
| `private fun findFolder(path, folders): MusicFolder?` | BFS でフォルダ検索 |
| `private fun MusicFolder.collectAllSongs(): List<Song>` | 再帰的全曲収集 |
| `private fun applySortToSongs(songs, sortOption): List<Song>` | 共通ソート |
| `private fun sortPlaylistsList(...)` / `sortSongsList(...)` | ソート純粋関数 |
| `private fun decodeOrderMode(value: String): PlaylistSongsOrderMode` | 永続化文字列のデコード |

### 内部実装メモ

- **プレイリスト ID 規則**:
  - 通常: `UUID.randomUUID().toString()`
  - Folder pseudo: `folder_playlist:${Uri.encode(folder.path)}`
- **並び替え永続化**:
  - `playlistOrderModes` Map を `playlistPreferencesRepository` に保存 (プレイリスト別 `MANUAL` / `SORTED:TitleAZ` 等)
  - `MANUAL_ORDER_MODE = "manual"` 定数
- **カバー画像処理**:
  - 元画像を `Bitmap` 化
  - 1024x1024 に `Matrix` で scale + pan 適用
  - `Bitmap.createBitmap` → `JPEG` 圧縮 → `context.filesDir/playlist_cover_<uuid>.jpg`
- **スマートプレイリスト ルール**: `SmartPlaylistRule.fromStorageKey(key)` で文字列をパース。`buildSmartPlaylistSongIds` 内で 6 種類のルール (TOP_PLAYED, RECENTLY_ADDED, RECENTLY_PLAYED, LEAST_PLAYED, FAVORITES, FORGOTTEN_FAVORITES 等) を実装。
- **AI プロンプト**: `prompt` から `playlistName` を `generateShortAiTitle` でキーワード抽出してフォールバック。
- **ZIP 共有**: `java.util.zip.ZipEntry` を順次書き込み、`FileProvider.getUriForFile` で `ACTION_SEND` intent 発行。
- **M3U エクスポート**: `m3uManager.generateM3u(playlist, songs)` で行単位に整形。
- **Folder pseudo playlist**: `Folder.collectAllSongs()` が BFS でサブフォルダ含めて全 Song を再帰取得。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/preferences/PlaylistPreferencesRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/model/Playlist.kt` / `SmartPlaylistRule.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/playlist/M3uManager.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/ai/AiPlaylistGenerator.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/preferences/TelegramTopicDisplayMode.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/PlaylistDetailScreen.kt`

---

## PlaylistSelectionStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlaylistSelectionStateHolder.kt` (144 行)
**アノテーション**: `@Singleton`
**役割**: プレイリスト一覧画面での複数選択モード。`MultiSelectionStateHolder` の Playlist 版。

### 注入される依存

なし (状態のみ)

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `selectedPlaylists` | `StateFlow<List<Playlist>>` | 選択中プレイリスト (順序保持) |
| `selectedPlaylistIds` | `StateFlow<Set<String>>` | ID セット (O(1) 検索用) |
| `isSelectionMode` | `StateFlow<Boolean>` | 選択モード中 |
| `selectedCount` | `StateFlow<Int>` | 選択数 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun toggleSelection(playlist: Playlist)` | 選択/解除 (トグル) |
| `fun selectAll(playlists: List<Playlist>)` | 全選択 |
| `fun clearSelection()` | 全解除 |
| `fun isSelected(playlistId: String): Boolean` | 判定 |
| `fun getSelectionIndex(playlistId: String): Int?` | 順序インデックス取得 |
| `fun removeFromSelection(playlistId: String)` | 強制削除 |
| `private fun updateState(playlists, ids)` | 内部状態同期 |

### 内部実装メモ

- **状態同期パターン**: `_selectedPlaylists` (List) と `_selectedPlaylistIds` (Set) を二重管理し、`updateState` で必ず整合性を保つ。
- **`isSelectionMode`**: `selectedCount > 0` で `true`、`clearSelection` で `false`。
- **`MultiSelectionStateHolder` との関係**: 楽曲の複数選択は別 StateHolder。プレイリストは別管理。両者とも `PlayerViewModel` から `multiSelectionStateHolder` / `playlistSelectionStateHolder` として公開される。
- **順序保持**: List で順序を保持し、`getSelectionIndex` で「何番目に選択されたか」を取得可能 (UI でのバッチ表示順序に使用)。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/MultiSelectionStateHolder.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/model/Playlist.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/PlaylistsScreen.kt` (推定)

---

## MetadataEditStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/MetadataEditStateHolder.kt` (901 行)
**アノテーション**: `@ViewModelScoped` (実際は `@Inject`)
**役割**: 楽曲メタデータ書込 + アルバムアート更新 + 歌詞保存 + MediaStore 権限要求。`SongMetadataEditor` への薄いラッパ + UI 統合層。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Media | `songMetadataEditor: SongMetadataEditor` |
| Repository | `musicRepository: MusicRepository` |
| Cache | `imageCacheManager: ImageCacheManager` |
| StateHolder | `themeStateHolder: ThemeStateHolder` |
| StateHolder | `playbackStateHolder: PlaybackStateHolder` |
| StateHolder | `libraryStateHolder: LibraryStateHolder` |
| StateHolder | `multiSelectionStateHolder: MultiSelectionStateHolder` |
| DAO | `albumArtThemeDao: AlbumArtThemeDao` |

### `MetadataEditCallbacks` (data class)

| 名前 | 目的 |
|------|------|
| `scope: CoroutineScope` | 実行スコープ |
| `getUiState: () -> PlayerUiState` | UI 状態 |
| `updateUiState: ...` | 状態書込 |
| `getSelectedSongForInfo: () -> Song?` | 選択中楽曲 |
| `setSelectedSongForInfo: (Song) -> Unit` | 選択変更 |
| `sendToast: (String) -> Unit` | トースト |
| `reloadLyricsForCurrentSong: () -> Unit` | 歌詞再読込 |

### `MetadataEditResult` (data class)

| フィールド | 目的 |
|----------|------|
| `success: Boolean` | 成否 |
| `updatedSong: Song?` | 新しい Song |
| `updatedAlbumArtUri: String?` | 新しいアート URI |
| `parsedLyrics: Lyrics?` | 解析後歌詞 |
| `error: MetadataEditError?` | エラー種別 |
| `errorMessage: String?` | エラー文言 |

### 主要 SharedFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `writePermissionRequest` | `SharedFlow<IntentSender>` | MediaStore 書込権限要求 (System UI 起動) |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun getUserFriendlyErrorMessage(): String` (拡張) | エラー → ユーザー向け文字列 |
| `suspend fun saveMetadata(...): MetadataEditResult` | メタデータ書込 (Cover / Lyrics / Tags 統合) |
| `suspend fun deleteSong(song: Song): Boolean` | 物理削除 (FileDeletionUtils 経由) |
| `fun editSongMetadata(song, fields, cb: MetadataEditCallbacks)` | UI から呼ばれるエントリポイント (権限要求 or 即実行) |
| `fun saveBatchMetadata(songs, fields, cb)` | 一括編集 (Android 11+ の MediaStore バッチ) |
| `fun batchEditGenre(songs, newGenre, cb)` | ジャンル一括 |
| `fun saveLyricsToFile(song, lyrics, preferSynced, cb)` | 歌詞保存 (.lrc ファイル生成) |
| `fun onWritePermissionResult(granted: Boolean, cb: MetadataEditCallbacks)` | 権限結果処理 |
| `fun onCleared()` | クリーンアップ |

### 内部実装メモ

- **権限要求パターン (Android 11+)**:
  1. `editSongMetadata` で `MediaStorePermissionHelper.createWriteRequestForSong` を呼んで `IntentSender` を取得
  2. `_writePermissionRequest.emit(intentSender)` → `PlayerViewModel` が Collect → Activity に渡す
  3. ユーザーが許可 → `onWritePermissionResult(true, cb)` で `pendingMetadataEdit` を実行
- **Pending Edit**: `pendingMetadataEdit` / `pendingBatchMetadataEdit` / `pendingLyricsSave` / `pendingBatchGenreEdit` の 4 種類をフィールドに保持。
- **保存後の処理**:
  1. `LibraryStateHolder.updateSong(updatedSong)` でライブラリ更新
  2. `PlayerUiState.currentPlaybackQueue.replaceSong(updatedSong)` でキュー更新
  3. `MediaController.replaceMediaItem(currentIndex, newMediaItem)` でエンジン更新
  4. `ImageCacheManager` + `AlbumArtThemeDao` のキャッシュ無効化
  5. `ThemeStateHolder.getAlbumColorSchemeFlow(uri)` の `value` を更新
  6. `reloadLyricsForCurrentSong()` で歌詞再読込
- **バッチ編集 (Android 11+ R+)**: `MediaStore.createWriteRequest` で複数の URI に対する許可を 1 度に要求。古い OS では 1 件ずつ。
- **ジャンル一括**: URI をまとめて `createWriteRequestIntentSender(context, uris)` へ。
- **歌詞保存**: `songFile.parentFile/<songFile.nameWithoutExtension>.lrc` に `LyricsUtils.toLrcString(lyrics, preferSynced)` 出力。`MediaStore.Audio.Media` 経由の `createWriteRequest` で権限要求。
- **アルバムアート削除**: `coverArtUpdate.isDeletion = true` の場合、`refreshedAlbumArtUri = null` としてアート除去。
- **カバー画像キャッシュ無効化**: `invalidateCoverArtCaches(vararg uriStrings)` で Coil の `imageLoader.memoryCache` から該当エントリを削除 + `purgeAlbumArtThemes` で DB カラーテーマ削除。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/media/SongMetadataEditor.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/media/CoverArtUpdate.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/media/ImageCacheManager.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/media/MetadataEditError.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/utils/MediaStorePermissionHelper.kt`
- `app/src/main/java/com/theveloper/pixelplay/utils/FileDeletionUtils.kt`

---

## SongInfoBottomSheetViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/SongInfoBottomSheetViewModel.kt` (430 行)
**アノテーション**: `@HiltViewModel`
**役割**: 楽曲情報 BottomSheet。音声メタデータ取得 + Wear OS 連携 + 着信音設定 + アーティスト解決。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Service | `wearPhoneTransferSender: WearPhoneTransferSender` |
| State | `transferStateStore: PhoneWatchTransferStateStore` |
| DAO | `musicDao: MusicDao` |

### `SongLocationInfo` (data class)

| フィールド | 目的 |
|----------|------|
| `label: String` | "File Path" 等 |
| `value: String` | 実値 |
| `isCloud: Boolean` | クラウドソースか |

### `ToneTarget` (enum)

| 値 | 用途 |
|----|------|
| `RINGTONE` | 着信音 |
| `NOTIFICATION` | 通知音 |
| `ALARM` | アラーム音 |

### `ToneActionResult` (sealed interface)

| 派生 | 用途 |
|------|------|
| `Success(message)` | 成功 |
| `NeedsSystemWritePermission(message)` | WRITE_SETTINGS 権限必要 |
| `Error(message)` | エラー |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `audioMeta` | `StateFlow<AudioMeta?>` | 音声メタデータ (bitrate, sample rate, etc.) |
| `resolvedArtists` | `StateFlow<List<Artist>>` | 解決済みアーティスト |
| `isPixelPlayWatchAvailable` | `StateFlow<Boolean>` | 接続中の PixelPlay Watch がある |
| `isWatchAvailabilityResolved` | `StateFlow<Boolean>` | 検出完了 |
| `watchTransfers` | `StateFlow<Map<String, PhoneWatchTransferState>>` | Watch 転送状態 |
| `watchSongIds` | `StateFlow<Set<String>>` | Watch にある曲 ID |
| `isSendingToWatch` | `StateFlow<Boolean>` | 転送中 |
| `activeWatchTransfer` | `StateFlow<PhoneWatchTransferState?>` | 進行中転送 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun loadArtistsForSong(song: Song)` | アーティスト ID → エンティティ解決 |
| `fun loadAudioMeta(song: Song)` | 音声メタデータ読込 (AudioMetaUtils 経由) |
| `fun getSongLocationInfo(song: Song): SongLocationInfo` | "File Path" / "Telegram" 等のラベル + 値 |
| `fun refreshWatchAvailability()` | Watch ノード再検出 |
| `fun isLocalSongForWatchTransfer(song: Song): Boolean` | Watch 転送可能か (content URI 判定) |
| `fun sendSongToWatch(song, onComplete: (String) -> Unit)` | Watch への曲転送要求 |
| `fun cancelWatchTransfer(requestId: String)` | 転送キャンセル |
| `fun isSongSavedOnAllReachableWatches(songId: String): Boolean` | 全 Watch に保存済み判定 |
| `fun isSongEditable(song: Song): Boolean` | 編集可能判定 (file exists チェック) |
| `fun hasSystemWritePermission(): Boolean` | WRITE_SETTINGS 権限 |
| `fun createSystemWriteSettingsIntent(): Intent` | 設定画面 Intent |
| `fun setSongAsTone(song, target, onComplete: (ToneActionResult) -> Unit)` | 着信音/通知音/アラーム設定 |
| `private fun setSongAsToneInternal(song, target): ToneActionResult` | 内部実装 (MediaStore + RingtoneManager) |

### 内部実装メモ

- **音声メタデータ**: `AudioMetaUtils.getAudioMetadata(song)` で `MediaMetadataRetriever` を起動 (IO スレッド)。
- **Watch 連携**:
  - `transferStateStore` は Singleton で全画面共有
  - `wearPhoneTransferSender.isPixelPlayWatchAvailable()` でクライアント検知
  - `requestSongTransfer(song.id, song.title)` で `Telegram` のような chat-to-chat メッセージングプロトコル経由で送信
  - 状態は `PhoneWatchTransferState` (QUEUED / IN_PROGRESS / COMPLETED / FAILED / CANCELED) で管理
- **着信音設定**:
  1. `MediaStore.Audio.Media._ID` 経由で URI 解決
  2. `RingtoneManager.setActualDefaultRingtoneUri` で登録
  3. `WRITE_SETTINGS` 権限が無い場合 `NeedsSystemWritePermission` を返却
- **クラウド判定**: `contentUriString` のスキームで判定 (`telegram:` / `gdrive:` / `jellyfin:` / `navidrome:` / `netease:` / `qqmusic:`)。
- **アーティスト解決**: `song.artists: List<ArtistRef>` から `MusicDao.getArtistsByIds(ids)` で名前解決。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/service/wear/WearPhoneTransferSender.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/service/wear/PhoneWatchTransferStateStore.kt`
- `app/src/main/java/com/theveloper/pixelplay/utils/AudioMetaUtils.kt`
- `app/src/main/java/com/theveloper/pixelplay/utils/AudioMeta.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/database/MusicDao.kt`
- 共有モジュール: `../08-shared-module.md` (`WearTransferProgress`)

---

## MultiSelectionStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/MultiSelectionStateHolder.kt` (505 行)
**アノテーション**: `@Singleton`
**役割**: 楽曲 / アルバム / ジャンルの複数選択モード。選択楽曲に対するバッチ操作 (再生 / キュー追加 / お気に入り / ZIP 共有) を提供。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |
| Context | `context: Context` |

### `SelectionActionCallbacks` (data class)

| 名前 | 目的 |
|------|------|
| `scope: CoroutineScope` | 実行スコープ |
| `playSongs: (songs, startSong, queueName) -> Unit` | 再生 |
| `addSongToQueue: (Song) -> Unit` | キュー末尾追加 |
| `addSongNextToQueue: (Song) -> Unit` | 次に追加 |
| `showSheet: () -> Unit` | プレイヤーシート展開 |
| `emitToast: suspend (String) -> Unit` | トースト |
| `favoriteSongIds: () -> Set<String>` | お気に入り ID 取得 |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `selectedSongs` | `StateFlow<List<Song>>` | 選択楽曲 (順序保持) |
| `selectedSongIds` | `StateFlow<Set<String>>` | ID セット |
| `isSelectionMode` | `StateFlow<Boolean>` | 選択モード |
| `selectedCount` | `StateFlow<Int>` | 選択数 |

### 主要 public 関数

#### 選択操作

| シグネチャ | 目的 |
|-----------|------|
| `fun toggleSelection(song: Song)` | 1 曲トグル |
| `fun selectAll(songs: List<Song>)` | 全選択 (songs が現在の表示ソース) |
| `fun clearSelection()` | 全解除 |
| `fun isSelected(songId: String): Boolean` | 判定 |
| `fun getSelectionIndex(songId: String): Int?` | 順序インデックス |
| `fun removeFromSelection(songId: String)` | 強制削除 |
| `private fun updateState(songs, ids)` | 内部同期 |

#### バッチ操作

| シグネチャ | 目的 |
|-----------|------|
| `fun playSelectedSongs(songs: List<Song>, callbacks)` | 選択楽曲を再生 (1 曲目を startSong) |
| `fun addSelectedToQueue(songs, callbacks)` | キュー末尾追加 |
| `fun addSelectedAsNext(songs, callbacks)` | 次の曲として追加 |
| `fun playSelectedAlbums(albums, callbacks)` | アルバム一括再生 |
| `fun addSelectedAlbumsAsNext(albums, callbacks)` | アルバムを次に追加 |
| `fun addSelectedAlbumsToQueue(albums, callbacks)` | アルバムをキュー末尾 |
| `fun likeSelectedSongs(songs, callbacks)` | いいね |
| `fun unlikeSelectedSongs(songs, callbacks)` | いいね解除 |
| `fun shareSelectedAsZip(songs, callbacks)` | ZIP 共有 |
| `fun playSelectedGenres(genres, callbacks)` | ジャンル一括再生 |
| `fun addSelectedGenresToQueue(genres, callbacks)` | ジャンル → キュー |
| `fun addSelectedGenresAsNext(genres, callbacks)` | ジャンル → 次に |

#### 内部

| シグネチャ | 目的 |
|-----------|------|
| `suspend fun getSongsForGenres(genres): List<Song>` | ジャンル → 楽曲 |
| `suspend fun getSongsForAlbums(albums): List<Song>` | アルバム → 楽曲 |
| `private suspend fun resolveSelectedAlbumSongs(albums): ResolvedAlbumSelection` | アルバム + 楽曲 + trim フラグ |
| `private fun sortSongsForAlbumSelection(songs): List<Song>` | アルバム選曲時のソート |
| `private fun launchAlbumSelectionAction(...)` | アルバムバッチアクション |
| `private fun launchGenreSelectionAction(...)` | ジャンルバッチアクション |

### 内部実装メモ

- **`MAX_ALBUM_BATCH_SELECTION = 6`**: 一度にバッチ処理できるアルバム数の上限。`resolveSelectedAlbumSongs` で `albums.take(6)`、`wasTrimmed = albums.size > 6` で通知。
- **アルバム楽曲のソート**: `discNumber → trackNumber → title`。
- **ZIP 共有**: `ZipShareHelper.createAndShareZip(context, songs)` で FileProvider 経由で Intent 発行。
- **`likeSelectedSongs` / `unlikeSelectedSongs`**: 現在の `favoriteSongIds()` 状態を読み取って、差分のみ更新。トーストメッセージは変更数に応じて動的生成 ("Liked 5 songs" / "Already liked" / "Unliked 3 songs" / "Removed from favorites")。
- **シャッフル / ソート永続化との相互作用**: 選択モードは表示中のソースに対する選択であり、楽曲自体の表示順序には影響しない。
- **`_isSelectionMode`**: `selectedCount > 0` で `true` へ。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/utils/ZipShareHelper.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/model/Album.kt` / `Genre.kt`
- 画面: ライブラリ / アルバム / プレイリスト / ジャンル / お気に入り 各一覧

---

## SongRemovalStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/SongRemovalStateHolder.kt` (378 行)
**アノテーション**: `@ViewModelScoped` (実際は `@Inject`)
**役割**: 楽曲の「ライブラリから削除」と「デバイスから物理削除」の両方を管理。`MediaStore` 削除権限要求と確認ダイアログを統合。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |
| StateHolder | `metadataEditStateHolder: MetadataEditStateHolder` |
| Preferences | `playlistPreferencesRepository: PlaylistPreferencesRepository` |
| StateHolder | `libraryStateHolder: LibraryStateHolder` |
| StateHolder | `playbackStateHolder: PlaybackStateHolder` |
| StateHolder | `multiSelectionStateHolder: MultiSelectionStateHolder` |
| Context | `context: Context` (ApplicationContext) |

### `SongRemovalCallbacks` (data class)

| 名前 | 目的 |
|------|------|
| `scope: CoroutineScope` | 実行スコープ |
| `sendToast: (String) -> Unit` | トースト |
| `removeFromMediaControllerQueue: (String) -> Unit` | キューから除去 |
| `removeSong: suspend (Song) -> Unit` | ライブラリから除去 |

### 主要 SharedFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `deletePermissionRequest` | `SharedFlow<IntentSender>` | MediaStore 削除権限要求 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `suspend fun showDeleteConfirmation(activity: Activity, song: Song): Boolean` | 確認ダイアログ (suspend) |
| `suspend fun deleteSongFile(song: Song): Boolean` | 物理ファイル削除 |
| `suspend fun removeSongFromLibrary(song: Song)` | ライブラリ + プレイリスト + キュー除去 |
| `fun deleteSelectedFromDevice(songs: List<Song>, onComplete: () -> Unit)` | 選択楽曲を一括削除 |
| `fun deleteFromDevice(activity, song, onComplete)` | 単体削除 (権限要求含む) |
| `fun onDeletePermissionResult(granted: Boolean, cb: SongRemovalCallbacks)` | 権限結果処理 |
| `private suspend fun showMultiDeleteConfirmation(activity, count): Boolean` | 複数削除確認 |

### 内部実装メモ

- **2 段階削除**:
  1. **ライブラリから削除**: `LibraryStateHolder.removeSong` + プレイリストから除去 + キューから除去 (`removeFromMediaControllerQueue`)
  2. **物理削除**: `FileDeletionUtils.deleteFile` でファイルシステムから削除 + DB の `MusicDao` から除去
- **現在再生中曲の保護**: `deletableSongs = songs.filter { it.id != currentSongId }` で再生中の曲を除外。`skippedCount` で件数通知。
- **権限要求**: `MediaStore.createDeleteRequest(uris)` の `IntentSender` を `_deletePermissionRequest.emit()` で公開。
- **Pending State**: `pendingBatchDeleteSongs` / `pendingBatchDeleteSkippedCount` / `pendingBatchDeleteOnComplete` / `pendingDeleteSong` / `pendingDeleteCallback` を保持。
- **確認ダイアログ**: `MaterialAlertDialogBuilder` で構築。`CompletableDeferred<Boolean>` で結果を suspend ポイントへ返す。
- **失敗時**: `deleteSongFile` が false を返したら `LibraryStateHolder.removeSong` のみ実行 (ライブラリからは除去するがファイルは残す)。
- **`FileDeletionUtils.getFileInfo`**: 削除前にファイル存在 / サイズ / 書き込み権限を確認。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/utils/FileDeletionUtils.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/utils/MediaStorePermissionHelper.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/database/MusicDao.kt`
- 画面: 楽曲コンテキストメニュー / 選択時バッチメニュー

---

## QueueUndoStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/QueueUndoStateHolder.kt` (102 行)
**アノテーション**: `@ViewModelScoped` (実際は `@Inject`)
**役割**: キューから曲を削除した際の 1 回限りの取り消し (Undo) 状態管理。

### 注入される依存

なし (callback で PlayerViewModel から提供される)

### 内部 state (var)

| 名前 | 目的 |
|------|------|
| `queueItemUndoTimerJob: Job?` | 5 秒タイマー |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun removeSongFromQueue(mediaController, getUiState, updateUiState, songId, sendToast)` | 削除 + Undo バー表示 |
| `fun undoRemoveSongFromQueue(mediaController, getUiState, updateUiState, sendToast)` | 復元 (元のインデックスに挿入) |
| `fun hideQueueItemUndoBar(updateUiState, sendToast)` | バー手動閉じ |
| `fun onCleared()` | Job キャンセル |

### 内部実装メモ

- **削除手順**:
  1. `currentQueue.indexOfFirst { it.id == songId }` で位置特定
  2. `controller.removeMediaItem(indexToRemove)` でエンジンから除去
  3. `lastRemovedQueueSong` / `lastRemovedQueueIndex` を UI State に保存
  4. `showQueueItemUndoBar = true` で Undo Snackbar 表示
  5. 5 秒タイマーで自動非表示
- **復元手順**:
  1. `controller.mediaItemCount` を超えない位置に `controller.addMediaItem(index, mediaItem)`
  2. UI State の `lastRemovedQueue*` を null クリア
  3. Undo バー非表示
- **5 秒タイマー**: `delay(5000)` → `hideQueueItemUndoBar`。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlayerUiState.kt` (`lastRemovedQueueSong`, `lastRemovedQueueIndex`, `showQueueItemUndoBar`)

---

## PlaylistDismissUndoStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlaylistDismissUndoStateHolder.kt` (165 行)
**アノテーション**: `@ViewModelScoped` (実際は `@Inject`)
**役割**: プレイリスト / アルバム画面を閉じてキュー再生中の状態から「元画面 + 元曲 + 元位置」に戻す取り消し。

### 注入される依存

なし (callback 経由)

### 内部 state (var)

| 名前 | 目的 |
|------|------|
| `dismissUndoTimerJob: Job?` | 5 秒タイマー |
| `undoObserverJob: Job?` | 曲遷移監視 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun dismissPlaylistAndShowUndo(scope, currentScreen, currentSong, currentQueue, currentPosition, queueName, hideCurrentScreen, showUndoBar)` | 画面閉じ + Undo 表示 |
| `fun hideDismissUndoBar(hideUndoBar, scope, showToast)` | Undo バー閉じる |
| `fun observeUndoStateAgainstPlayback(...)` | 曲自動切替で Undo 状態リセット |
| `fun undoDismissPlaylist(...)` | 元画面復元 + 元曲 + 元位置 |
| `fun onCleared()` | クリーンアップ |

### 内部実装メモ

- **保存される状態**:
  - `dismissedSong: Song?` (現在の曲)
  - `dismissedQueue: ImmutableList<Song>` (現在のキュー全体)
  - `dismissedQueueName: String` ("Playlist Name" 等)
  - `dismissedPosition: Long` (現在の再生位置 ms)
- **Undo 動作**:
  1. `playSongs(songs, startSong, queueName)` でキュー復元
  2. `seekTo(position)` で位置復元
  3. 閉じた画面を再表示
- **曲自動切替時の自動失効**: `stablePlayerState` を監視し、曲が変わったら Undo を無効化。
- **5 秒タイマー**: 同じく自動失効。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlayerUiState.kt` (`dismissedSong`, `dismissedQueue`, `dismissedPosition`, `showDismissUndoBar`)
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/PlaylistDetailScreen.kt` / `AlbumDetailScreen.kt`
