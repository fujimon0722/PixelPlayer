# viewmodels-ai-extra.md

> AI / Cast / 外部メディア / メディアコントローラ同期 / ファイルエクスプローラ / 接続性 / テーマ / アカウント 等の StateHolder / ViewModel の詳細仕様。

---

## AiStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/AiStateHolder.kt` (506 行)
**アノテーション**: `@Singleton`
**役割**: AI プレイリスト生成の UI 状態 + 翻訳。`AiPlaylistGenerator` + `DailyMixManager` への薄いラッパ。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| AI | `aiPlaylistGenerator: AiPlaylistGenerator` |
| Engine | `dailyMixManager: DailyMixManager` |
| Preferences | `playlistPreferencesRepository: PlaylistPreferencesRepository` |
| StateHolder | `dailyMixStateHolder: DailyMixStateHolder` |
| Notification | `notificationManager: AiNotificationManager` |
| AI | `aiHandler: AiHandler` |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `showAiPlaylistSheet` | `StateFlow<Boolean>` | シート表示フラグ |
| `isGeneratingAiPlaylist` | `StateFlow<Boolean>` | 生成中 |
| `aiSuccess` | `StateFlow<Boolean>` | 成功 |
| `aiStatus` | `StateFlow<String?>` | 進行ステータス ("Analyzing your library...") |
| `aiError` | `StateFlow<String?>` | エラー |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(scope, allSongsProvider, favoriteSongIdsProvider, toastEmitter, playSongsCallback, openPlayerSheetCallback)` | 依存注入 |
| `fun showAiPlaylistSheet()` | シート表示 |
| `fun dismissAiPlaylistSheet()` | シート非表示 |
| `fun retryLastPlaylistGeneration()` | 直前プロンプト再生成 |
| `fun clearAiPlaylistError()` | エラークリア |
| `fun generateAiPlaylist(prompt, minLength, maxLength, requestedName?)` | 生成実行 |
| `fun regenerateDailyMixWithPrompt(prompt: String)` | Daily Mix を AI で再構成 |
| `suspend fun translateLyrics(lyricsText: String): Result<String>` | 歌詞翻訳 |
| `fun onCleared()` | クリーンアップ |

### 内部実装メモ

- **Provider パターンの遅延注入**: 7 種類の callback (`allSongsProvider` / `favoriteSongIdsProvider` / `toastEmitter` / `playSongsCallback` / `openPlayerSheetCallback`) を `initialize` で後から設定。これは `PlayerViewModel` の構築時に循環参照を避けるため。
- **プロンプト前処理**: `titleStopWords` セットで "the", "a", "for" 等の単語を除外し、キーワード抽出。
- **候補プール生成**: `dailyMixManager.generateDailyMix(allSongs, favoriteIds, size = desiredSize * 1.5)` で AI へ渡す候補を抽出。
- **エラー解決**: `resolveAiErrorMessage(error)` で `AiProviderException` の `code` / `message` を人間可読メッセージに変換。
- **タイトル自動生成**: `generateShortAiTitle(prompt)` で "AI: ${prompt.take(50)}" のような短いタイトルへ。
- **翻訳プロンプト**: `translateLyrics` は context の言語設定 (`configuration.locales[0].displayLanguage`) をターゲットとして AI に依頼。
- **通知**: `AiNotificationManager` で生成中の進捗をシステム通知 (foreground service) で表示。
- **既存プレイリスト名衝突回避**: `resolveAiPlaylistName` で同名があれば "AI Mix", "AI Mix 2", "AI Mix 3" と suffix 付与。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/ai/AiPlaylistGenerator.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/ai/AiHandler.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/ai/AiNotificationManager.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/ai/AiSystemPromptType.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/ai/provider/AiProviderException.kt`

---

## CastStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/CastStateHolder.kt` (300 行)
**アノテーション**: `@Singleton`
**役割**: Google Cast SDK の MediaRouter + CastSession 統合。`CastPlayer` ブリッジ、リモート再生状態、リモートポジションチック、リモートエラー / バッファリング管理。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Context | `context: Context` (ApplicationContext) |
| System | `mediaRouter: MediaRouter` |

### 主要 StateFlow / プロパティ

| 名前 | 型 | 目的 |
|------|---|------|
| `castSession` | `StateFlow<CastSession?>` | 現在の CastSession |
| `castPlayer` | `CastPlayer?` (getter) | `DualPlayerEngine` 用のラッパ |
| `isRemotePlaybackActive` | `StateFlow<Boolean>` | リモート再生中 |
| `isCastConnecting` | `StateFlow<Boolean>` | 接続中 |
| `remotePosition` | `StateFlow<Long>` | リモート再生位置 (ms) |
| `castRoutes` | `StateFlow<List<MediaRouter.RouteInfo>>` | 利用可能 Cast ルート |
| `selectedRoute` | `StateFlow<MediaRouter.RouteInfo?>` | 選択中ルート |
| `routeVolume` | `StateFlow<Int>` | ルートボリューム |
| `isRefreshingRoutes` | `StateFlow<Boolean>` | 再スキャン中 |
| `isRemotelySeeking` | `StateFlow<Boolean>` | リモートシーク中 |
| `pendingCastRouteId` | `String?` (getter) | 接続待機のルート ID |
| `lastRemoteQueue: List<Song>` (var) | 最後のリモートキュー (外部公開) |
| `lastRemoteMediaStatus: MediaStatus?` (var) | 最後のステータス |
| `lastRemoteSongId: String?` (var) | 最後の曲 ID |
| `lastRemoteStreamPosition: Long` (var) | 最後の位置 |
| `lastRemoteRepeatMode: Int` (var) | 最後のリピートモード |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun setCastSession(session: CastSession?)` | セッション登録/解除 |
| `fun setCastPlayer(player: CastPlayer?)` | CastPlayer 登録 |
| `fun setRemotePlaybackActive(active: Boolean)` | リモート再生フラグ |
| `fun setCastConnecting(connecting: Boolean)` | 接続中フラグ |
| `fun setRemotePosition(position: Long)` | 位置更新 |
| `fun setPendingCastRouteId(routeId: String?)` | 接続待機のルート |
| `fun setRemotelySeeking(seeking: Boolean)` | シークフラグ |
| `fun updateLastRemoteState(status, queue, songId, streamPosition, repeatMode, itemId)` | 状態一括更新 |
| `fun setPendingRemoteSong(songId, markedAt)` | ペンディング曲指定 |
| `fun clearRemoteState()` | 全クリア |
| `fun refreshRoutes(scope: CoroutineScope)` | ルート再スキャン (debounce 1.5s) |
| `fun startDiscovery()` | MediaRouter discovery 開始 |
| `fun selectRoute(route: MediaRouter.RouteInfo)` | ルート選択 |
| `fun setRouteVolume(volume: Int)` | ボリューム設定 |
| `fun disconnect()` | 切断 |
| `fun MediaRouter.RouteInfo.isCastRoute(): Boolean` (extension) | Cast ルート判定 |
| `fun buildCastRouteSelector(): MediaRouteSelector` | セレクタ生成 |
| `fun onCleared()` | クリーンアップ |

### 内部実装メモ

- **`sessionManager` lazy**: `CastContext.getSharedInstance(context).sessionManager` を初回使用時に取得。
- **`MediaRouter.Callback`**: `onRouteAdded/Removed/Changed/Selected/Unselected/VolumeChanged` を override し、`_castRoutes` / `_selectedRoute` / `_routeVolume` を更新。
- **debounce 1.5s**: `refreshRoutes` 内で 1.5 秒以内の連続 refresh を無視。
- **route キャッシュ**: `castRoutes` StateFlow にキャスト可能な全ルートを保持。
- **`isCastRoute` 判定**: `route.hasControlCategory(MediaControlIntent.CATEGORY_LIVE_VIDEO) || MediaControlIntent.CATEGORY_REMOTE_PLAYBACK` のいずれかを満たす。
- **`updateLastRemoteState`**: 6 つの引数を一括更新 (アトミック)。
- **`pendingRemoteSong*`**: Cast 上で `loadMedia` 後、ステータスが更新されるまでの「予期曲」を記憶。`CastTransferStateHolder` が `markPendingRemoteSong` で設定。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/service/player/CastPlayer.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/service/cast/CastRemotePlaybackState.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/CastTransferStateHolder.kt`

---

## CastRouteStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/CastRouteStateHolder.kt` (97 行)
**アノテーション**: `@ViewModelScoped` (実際は `@Inject`)
**役割**: Cast ルートの選択 / 切断時の安全制御。`CastStateHolder` + `CastTransferStateHolder` の薄いラッパで、特殊ケース (Switch / Retry / Disconnect) を扱う。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| StateHolder | `castStateHolder: CastStateHolder` |
| StateHolder | `castTransferStateHolder: CastTransferStateHolder` |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun selectRoute(route: MediaRouter.RouteInfo, scope: CoroutineScope)` | ルート選択 (Switch / Retry / 新規 で分岐) |
| `fun disconnect(resetConnecting: Boolean = true)` | 切断 + Transfer back 待機 |
| `fun setRouteVolume(volume: Int)` | ボリューム設定 (CastSession 経由) |
| `fun refreshCastRoutes(scope: CoroutineScope)` | ルート再スキャン |

### 内部実装メモ

- **`isCastRoute` 判定**: 内部で `castStateHolder.run { route.isCastRoute() } && !route.isDefault` でフィルタ。
- **`isSwitchingBetweenRemotes`**: 既に Cast 接続中で、違う Cast ルートを選択した場合 true → 既存セッションを終了し新セッションを開始。
- **`isRetryingFailedSameRoute`**: 直前の Cast セッションが失敗 → 同じルートで再試行する場合 true → `castStateHolder.setPendingCastRouteId(routeId)` で待機 ID 設定。
- **`disconnect` 計測**: `SystemClock.elapsedRealtime()` で所要時間ログ (デバッグ用)。
- **`wasRemote`**: 切断前にリモート再生中だったか。`true` なら切断時にローカルへ transfer back。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/CastStateHolder.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/CastTransferStateHolder.kt`

---

## CastTransferStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/CastTransferStateHolder.kt` (1598 行)
**アノテーション**: `@Singleton`
**役割**: ローカル ↔ Cast 間のキュー / 再生状態 / 曲 / 進捗 / バッファリング の転送と、HTTP サーバー起動 / 停止を統合管理。**コードベース内で最も複雑な StateHolder の一つ**。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| StateHolder | `castStateHolder: CastStateHolder` |
| StateHolder | `playbackStateHolder: PlaybackStateHolder` |
| Engine | `dualPlayerEngine: DualPlayerEngine` |
| Context | `context: Context` (ApplicationContext) |

### 内部 state (var)

| 名前 | 目的 |
|------|------|
| `lastRemoteMediaStatus: MediaStatus?` | 最後のリモートステータス |
| `lastRemoteQueue: List<Song>` | 最後のリモートキュー (公開) |
| `lastRemoteSongId: String?` | 最後の曲 ID |
| `lastRemoteStreamPosition: Long` | 最後の位置 |
| `lastRemoteRepeatMode: Int` | 最後のリピート |
| `lastRemotePlaybackShouldResume: Boolean` | 再開フラグ |
| `lastRemoteItemId: Int?` | 最後の item ID |
| `pendingRemoteSongId: String?` / `pendingRemoteSongMarkedAt: Long` | 予期曲追跡 |
| `pendingMismatchStatusRequestCount: Int` | ステータス再要求カウント |
| `remoteBuffering*` (5 変数) | バッファリング監視 |
| `skipTransferBackOnNextSessionEnd: Boolean` | 次回終了時 transfer back スキップ |

### 主要 public 関数

#### 初期化 / 制御

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(scope, getCurrentQueue, updateQueue, getSongsByIdMap, onTransferBackComplete, onSheetVisible, onDisconnect, onCastError, onSongChanged)` | 全 callback 注入 |
| `fun skipNextTransferBack()` | 次回 transfer back をスキップ |
| `suspend fun stopServerAndTransferBack()` | HTTP サーバー停止 + ローカルへ復帰 |
| `fun onCleared()` | クリーンアップ |

#### HTTP サーバー

| シグネチャ | 目的 |
|-----------|------|
| `fun primeHttpServerStart()` | 予防的サーバー起動 |
| `suspend fun ensureHttpServerRunning(castDeviceIpHint: String? = null): Boolean` | サーバー起動確認 |

#### 再生転送

| シグネチャ | 目的 |
|-----------|------|
| `suspend fun playRemoteQueue(songs, startSong, isShuffleEnabled): Boolean` | キューを Cast に転送 + 再生開始 |
| `fun markPendingRemoteSong(song: Song)` | 予期曲指定 |
| `private fun launchAlignToTarget(targetSongId)` | 強制アライン |
| `private suspend fun alignRemotePlaybackToSong(targetSongId)` | リモート再生を target 曲に揃える |

#### バッファリング復旧

| シグネチャ | 目的 |
|-----------|------|
| `private fun updateRemoteBufferingWatchdog(...)` | バッファリング監視 (6s soft / 14s reload / 28s transfer back) |
| `private suspend fun reloadCurrentRemoteItemAfterBuffering(...)` | リモート曲リロード |
| `private fun resetRemoteBufferingWatchdog(...)` | リセット |

### 内部実装メモ

- **バッファリング復旧の 3 段階**:
  1. **Soft recovery (6s)**: 何もしない (一時的なネットワーク遅延を想定)
  2. **Reload (14s)**: 曲をリロード (`mediaStatus.playerState` をチェック後 `load` 再実行)
  3. **Transfer back (28s)**: リモート再生を諦めローカルに戻す
- **`remoteBufferingPositionToleranceMs = 750`**: 進捗があったと判定する最小差分。
- **`Mutex httpServerStartMutex`**: `MediaFileHttpServerService` の起動は排他制御。
- **IP 互換性チェック**: `isServerAddressCompatibleWithCastDevice` で `castDevice` の IP と `MediaFileHttpServerService.serverAddress` のホストを `Inet4Address` 比較。サブネットマスク (`serverPrefixLength`) も考慮。
- **エンドポイント準備確認**: `waitForSongEndpointReady` で HTTP HEAD を試行し、`isSongEndpointReady` で 200 応答を待機。
- **`TransferSnapshot` data class**: transfer back 時のスナップショット (status, queue, songId, position, repeatMode, wasPlaying, isShuffleEnabled)。
- **セッション管理**: `castSessionManagerListener` で `onSessionStarted/Resumed/Ended/Suspended/Starting/StartFailed/Ending/Resuming/ResumeFailed` をハンドル。
- **メディアクライアント コールバック**: `remoteMediaClientCallback` で `onStatusUpdated/MetadataUpdated/QueueStatusUpdated/PreloadStatusUpdated` をハンドル。
- **エラーリカバリ**: `scheduleCastErrorRecoveryIfNeeded` で item error 時に Cast 曲を `seekTo` してリジューム。
- **pending song 解決**: `resolvePendingRemoteSong` で `markPendingRemoteSong` 後 4 秒以内の reported song を採用。
- **`alignRemotePlaybackToSong`**: `repeat(20) { delay(200) }` で 4 秒ポーリング。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/service/http/MediaFileHttpServerService.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/service/http/CastSessionSecurity.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/service/cast/CastRemotePlaybackState.kt`
- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/CastStateHolder.kt`
- `app/src/main/java/com/theveloper/pixelplay/utils/MediaItemBuilder.kt`

---

## ExternalMediaStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ExternalMediaStateHolder.kt` (367 行)
**アノテーション**: `@Singleton`
**役割**: 外部 URI (Share intent 等) から受け取ったメディアを `Song` オブジェクトへ変換。MediaStore メタデータ抽出 + アルバムアート保存 + ファイルキャッシュ。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Context | `context: Context` (ApplicationContext) |
| Media | `audioMetadataReader: AudioMetadataReader` (private 参照) |

### `ExternalSongLoadResult` (data class)

| フィールド | 目的 |
|----------|------|
| `song: Song` | 変換後 Song |
| `relativePath: String?` | MediaStore relative path |
| `bucketId: Long?` | フォルダ ID |
| `displayName: String?` | 表示名 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `suspend fun buildExternalQueue(result, originalUri): List<Song>` | URI から単一曲のキューを生成 + 同フォルダの他の曲を追加 |
| `suspend fun buildExternalSongFromUri(uri: Uri, captureFolderInfo: Boolean = true): ExternalSongLoadResult?` | URI → Song 変換 |
| `private suspend fun loadAdditionalSongsFromFolder(reference, originalUri)` | 同フォルダの他の曲を取得 |
| `private fun resolveDirectFilePath(uri, storeDataPath): String?` | ファイルパス解決 |
| `private fun persistExternalAudioForPlayback(uri: Uri): File?` | キャッシュへ音声ファイル保存 |
| `private fun persistExternalAlbumArt(uri, data, mimeType?): String?` | アルバムアート保存 |

### 内部実装メモ

- **MediaStore クエリ**: `MediaStore.Audio.Media._ID, DISPLAY_NAME, RELATIVE_PATH, BUCKET_ID, DURATION, TITLE, ARTIST, ALBUM, TRACK, YEAR, DATE_ADDED, DATA` を全取得。
- **メタデータ補完**: MediaStore に無い `bitrate` / `sampleRate` などは `AudioMetadataReader.read(context, uri)` で `MediaMetadataRetriever` から抽出。
- **キャッシュ戦略**:
  - `content://` URI の場合: `cacheDir/external_audio/audio_<hash>.<ext>` にコピー
  - `file://` URI の場合: 直接ファイルパス
- **アルバムアート**: `cacheDir/external_artwork/art_<hash>.<ext>` に JPEG/PNG で保存。
- **Song ID 規則**:
  - MediaStore に登録済み: `mediaStoreId.toString()`
  - 外部のみ: `"external:${uri}"`
- **Folder 検索**: `loadAdditionalSongsFromFolder` で `BUCKET_ID` または `RELATIVE_PATH` をキーに同フォルダの曲を追加。`startIndex` は URI 自身を指す位置。
- **URI mediaStoreAudioId 抽出**: `Uri.mediaStoreAudioId()` extension で `content://media/external/audio/media/<id>` パターンをパース。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/media/AudioMetadataReader.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/media/resolveAudioFileExtension.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/media/guessImageMimeType.kt`
- 呼び出し元: `PlaybackDispatchStateHolder.playExternalUri`

---

## MediaControllerSyncStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/MediaControllerSyncStateHolder.kt` (806 行)
**アノテーション**: `@ViewModelScoped` (実際は `@Inject`)
**役割**: MediaController の `Player.Listener` 群を設定し、再生状態 / 曲変更 / ポジション / トラック / シャッフル / リピートを `PlayerUiState` / `StablePlayerState` へ反映する中央ブリッジ。

### 注入される依存 (10 個)

| 種類 | フィールド |
|------|----------|
| Repository | `musicRepository: MusicRepository` |
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| Engine | `dualPlayerEngine: DualPlayerEngine` |
| Media | `mediaMapper: MediaMapper` |
| StateHolder | `playbackStateHolder: PlaybackStateHolder` |
| StateHolder | `libraryStateHolder: LibraryStateHolder` |
| StateHolder | `castStateHolder: CastStateHolder` |
| StateHolder | `connectivityStateHolder: ConnectivityStateHolder` |
| StateHolder | `themeStateHolder: ThemeStateHolder` |
| StateHolder | `lyricsStateHolder: LyricsStateHolder` |
| StateHolder | `sleepTimerStateHolder: SleepTimerStateHolder` |
| StateHolder | `playbackDispatchStateHolder: PlaybackDispatchStateHolder` |

### `PlaybackAudioMetadata` (data class)

| フィールド | 目的 |
|----------|------|
| `mediaId: String?` | メディア ID |
| `mimeType: String?` | MIME タイプ |
| `bitrate: Int?` | ビットレート |
| `sampleRate: Int?` | サンプルレート |
| `channelCount: Int?` | チャンネル数 |
| `bitDepth: Int?` | ビット深度 |

### `ControllerSyncCallbacks` (data class)

| 名前 | 目的 |
|------|------|
| `scope: CoroutineScope` | 実行スコープ |
| `getController: () -> MediaController?` | MediaController 取得 |
| `getUiState: () -> PlayerUiState` | UI 状態 |
| `updateUiState: ...` | 状態書込 |
| `showSheet: () -> Unit` | シート展開 |
| `setTrackVolume: (Float) -> Unit` | ボリューム設定 |
| `emitToast: suspend (String) -> Unit` | トースト |
| `showNoInternetDialog: suspend () -> Unit` | オフライン警告 |
| `ensureTelegramObservers: () -> Unit` | Telegram 監視 |
| `cancelSleepTimerForEot: () -> Unit` | EOT タイマーリセット |
| `resetLyricsSearchState: () -> Unit` | 歌詞検索リセット |
| `loadLyricsForCurrentSong: () -> Unit` | 歌詞読込 |
| `toggleShuffle: () -> Unit` | シャッフルトグル |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `playbackAudioMetadata` | `StateFlow<PlaybackAudioMetadata>` | 現在の曲のオーディオメタデータ |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(callbacks: ControllerSyncCallbacks)` | callback 注入 |
| `fun cancelTransitionScheduler()` | クロスフェード Job キャンセル |
| `fun resolveSongFromMediaItem(mediaItem): Song?` | MediaItem → Song 解決 |
| `fun applyPreferredRepeatMode(@Player.RepeatMode mode: Int)` | リピートモード適用 (Cast 中は remote に転送) |
| `fun flushPendingRepeatMode()` | 未適用リピート反映 |
| `fun setupMediaControllerListeners(playerCtrl: MediaController?)` | Listener 登録 (旧 controller は解除) |
| `fun clearMediaControllerPlaybackListeners(controller: MediaController?)` | 旧 controller から除去 |
| `private fun applyInitialControllerState(playerCtrl)` | 接続直後の状態反映 |
| `private fun updateCurrentPlaybackQueueFromPlayer(playerCtrl)` | キュー同期 (Windowed / direct) |
| `private fun refreshPlaybackAudioMetadata(player, tracks)` | メタデータ抽出 (Tracks から) |
| `private fun maybeProbeMissingPlaybackAudioMetadata(player, mediaItem)` | MediaMetadataRetriever で補完 |
| `fun setupVolumeListeners(playerCtrl)` / `setupPlaybackListeners` / `setupTransitionListeners` / `setupMetadataListeners` | 4 種類の Listener セットアップ |
| `fun resetPlaybackAudioMetadata()` | メタデータリセット |

### 内部実装メモ

- **MediaItem → Song 解決**:
  1. `mediaItem.mediaId` から `libraryStateHolder.allSongsById.value[mediaId]` を探す
  2. 見つからない場合、`mediaItem.localConfiguration?.uri` を URI として `MusicDao.getSongByContentUri` でクエリ
- **Windowed Queue**: `dualPlayerEngine.isUsingWindowedQueue()` で 2 つの ExoPlayer (master / window) を切替。`mediaItems` を `currentTimeline.windows` から取得。
- **Audio Metadata 抽出**:
  1. `Tracks.groups.firstOrNull { it.type == C.TRACK_TYPE_AUDIO }?.getTrackFormat(0)` から `mimeType`, `bitrate`, `sampleRate`, `channelCount` 取得
  2. `pcmEncoding` から `extractBitDepthFromPcmEncoding` で bitDepth 算出
- **補完プローブ**: Tracks に bitrate / sampleRate が無い場合のみ `MediaMetadataRetriever` で取得 (1 度だけ)。
- **Transition 時の歌詞クリア**: `EotStateHolder.eotTargetSongId` を参照し、EOT 遷移時に歌詞クリア。
- **Telegram buffering 検知**: 曲変更時に `isOnline` / `telegramFileId` / `isCached` をチェックし、オフラインなら `showNoInternetDialog`。
- **シャッフル / リピート永続化**: 変更検知で `userPreferencesRepository.setShuffleOn` / `setRepeatMode` を即時書込。
- **Repeat Mode 同期**: ユーザーが UI で変更 → `applyPreferredRepeatMode` → MediaController へ → 反映。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/media/MediaMapper.kt`
- `app/src/main/java/com/theveloper/pixelplay/utils/MediaItemBuilder.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/EotStateHolder.kt` (object)
- エンジン: `../04-engine/music-service.md`

---

## FileExplorerStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/FileExplorerStateHolder.kt` (593 行)
**アノテーション**: `@Inject` (コンストラクタで scope / context を受け取る)
**役割**: 設定画面・セットアップ画面のファイルエクスプローラ。MediaStore インデックス + File System の統合ナビゲーション。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| Scope | `scope: CoroutineScope` (コンストラクタ) |
| Context | `context: Context` (コンストラクタ) |

### `DirectoryEntry` (data class)

| フィールド | 目的 |
|----------|------|
| `file: File` | ファイル |
| `directAudioCount: Int` | 直下の曲数 |
| `totalAudioCount: Int` | 配下すべての曲数 |
| `canonicalPath: String` | 正規化パス |
| `displayName: String?` | 表示名 |
| `isBlocked: Boolean` | ブロック状態 |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `availableStorages` | `StateFlow<List<StorageInfo>>` | 内部 / SDカード / USB |
| `selectedStorageIndex` | `StateFlow<Int>` | 選択中ストレージ |
| `currentPath` | `StateFlow<File>` | 現在パス |
| `allowedDirectories` | `StateFlow<Set<String>>` | 許可ディレクトリ |
| `blockedDirectories` | `StateFlow<Set<String>>` | ブロックディレクトリ |
| `isLoading` | `StateFlow<Boolean>` | 読込中 |
| `isPrimingExplorer` | `StateFlow<Boolean>` | 初期 priming 中 |
| `isExplorerReady` | `StateFlow<Boolean>` | 準備完了 |
| `isCurrentDirectoryResolved` | `StateFlow<Boolean>` | 現在ディレクトリ解決済み |
| `currentDirectoryChildren` | `StateFlow<List<DirectoryEntry>>` | 子ディレクトリ一覧 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun refreshAvailableStorages()` | ストレージ列挙更新 |
| `fun selectStorage(index: Int)` | ストレージ切替 |
| `fun refreshCurrentDirectory(): Job` | 現在ディレクトリ再読込 |
| `fun loadDirectory(file: File, updatePath: Boolean = true, forceRefresh: Boolean = false): Job` | ディレクトリ移動 |
| `fun primeExplorerRoot(): Job?` | ルート priming |
| `fun openExplorerRoot(): Job` | ルート移動 |
| `fun navigateUp()` | 親へ |
| `suspend fun toggleDirectoryAllowed(file: File)` | 許可/ブロック トグル |
| `fun isAtRoot(): Boolean` | ルート判定 |
| `fun rootDirectory(): File` | ルート取得 |

### 内部実装メモ

- **MediaStore インデックス**: 起動時に `buildMediaStoreDirectoryIndex` で全楽曲をスキャンし、`childrenByParent` (親→子) / `directAudioCountByPath` / `totalAudioCountByPath` の 3 つの Map を作成。
- **ハイブリッド取得**:
  1. `listImmediateDirectoryEntries` で MediaStore から該当ディレクトリの子を取得
  2. `enrichDirectoryEntries` で File System 側 (サブフォルダ含む) と merge
- **キャッシュ**: `directoryChildrenCache: ConcurrentHashMap<String, List<RawDirectoryEntry>>` で親パス毎の結果をキャッシュ。
- **Prefetch**: `prefetchChildDirectories` で現在のディレクトリの子 (最大 8) を先読み。
- **Mutex**: `loadMutex` (ディレクトリロード) と `mediaStoreIndexMutex` (インデックス構築) で並行実行を制御。
- **ストレージ**: `StorageUtils.getAvailableStorages(context)` で内部 / SDカード / USB を列挙。
- **Block 判定**: `DirectoryRuleResolver(allowed, blocked)` で `isAllowed` / `isBlocked` を解決。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/utils/StorageUtils.kt`
- `app/src/main/java/com/theveloper/pixelplay/utils/StorageInfo.kt`
- `app/src/main/java/com/theveloper/pixelplay/utils/DirectoryRuleResolver.kt`
- 画面: `SettingsScreen.kt` / `SetupScreen.kt`

---

## FolderNavigationStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/FolderNavigationStateHolder.kt` (129 行)
**アノテーション**: `@ViewModelScoped` (実際は `@Inject`)
**役割**: フォルダツリー内ナビゲーション。フォルダ階層の上 / 下移動、Storage Source (Internal / SD Card) 切替。

### 注入される依存

なし (callback で `PlayerViewModel` から提供)

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun setFoldersPlaylistViewState(updateUiState, isFoldersPlaylistView: Boolean)` | Folder Playlist View 切替 |
| `fun navigateToFolder(path: String?, updateUiState, currentStorageFilter, getUiState)` | フォルダ移動 |
| `fun navigateBackFolder(updateUiState, getUiState)` | 親フォルダへ |
| `fun hydrateCurrentFolderSongsIfNeeded(updateUiState, getUiState, scope, hydrateSongs: suspend (List<Song>) -> List<Song>)` | フォルダ内楽曲の水和 |
| `private fun findFolder(path: String?, folders: List<MusicFolder>): MusicFolder?` | BFS で検索 |

### 内部実装メモ

- **`folderSourceRootPath`**: Internal Storage の絶対パス or SD Card パス。`Environment.getExternalStorageDirectory()` 由来。
- **Parent ガード**: 親パスが `rootCanonicalPath` 外に出る場合は `navigateUp` を無視 (Storage Source 切替防止)。
- **Hydration**: Telegram 曲など DB から補完が必要な曲に対し `hydrateSongs` callback で `PlaybackDispatchStateHolder.hydrateSongsIfNeeded` を呼び出す。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/PlayerUiState.kt` (`currentFolder`, `folderSource`, `folderSourceRootPath`, `isSdCardAvailable`)
- `app/src/main/java/com/theveloper/pixelplay/data/model/MusicFolder.kt`

---

## ConnectivityStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ConnectivityStateHolder.kt` (592 行)
**アノテーション**: `@Singleton`
**役割**: Wi-Fi / Bluetooth / モバイルネットワークの統合状態監視。`ConnectivityManager` + `WifiManager` + `BluetoothManager` + `AudioDeviceCallback` を Bridge。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Context | `context: Context` (ApplicationContext) |

### `BluetoothAudioDeviceState` (data class)

| フィールド | 目的 |
|----------|------|
| `name: String` | デバイス名 |
| `address: String?` | MAC アドレス |
| `isConnected: Boolean` | 接続中か |
| `batteryPercent: Int?` | バッテリー残量 (リフレクション) |

### 主要 StateFlow / SharedFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `isWifiEnabled` | `StateFlow<Boolean>` | Wi-Fi ON |
| `isWifiRadioOn` | `StateFlow<Boolean>` | Wi-Fi Radio ON |
| `wifiName` | `StateFlow<String?>` | SSID |
| `isOnline` | `StateFlow<Boolean>` | インターネット接続あり |
| `isBluetoothEnabled` | `StateFlow<Boolean>` | Bluetooth ON |
| `bluetoothName` | `StateFlow<String?>` | BT アダプタ名 |
| `bluetoothAudioDeviceStates` | `StateFlow<List<BluetoothAudioDeviceState>>` | 接続中/検出 BT オーディオ |
| `bluetoothAudioDevices` | `StateFlow<List<String>>` | BT オーディオ名 (シンプル) |
| `offlinePlaybackBlocked` | `SharedFlow<Unit>` | オフライン再生阻止イベント |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun triggerOfflineBlockedEvent()` | オフライン再生阻止イベント発行 |
| `fun refreshLocalConnectionInfo(refreshBluetoothDevices: Boolean = false)` | 全情報更新 |
| `fun initialize()` | BroadcastReceiver / NetworkCallback 登録 |
| `fun onCleared()` | 解除 |

### 内部実装メモ

- **`ConnectivityManager.NetworkCallback`**: `onAvailable` / `onCapabilitiesChanged` / `onLost` で `_isOnline` を更新。`NET_CAPABILITY_INTERNET` + `NET_CAPABILITY_VALIDATED` で実インターネット判定。
- **Wi-Fi 監視**: `WIFI_STATE_CHANGED` BroadcastReceiver で `_isWifiRadioOn` を更新。`WifiManager.connectionInfo.ssid` で SSID 取得 (API 26+ で位置情報権限要)。
- **Bluetooth 監視**:
  - `BluetoothA2dp.ACTION_CONNECTION_STATE_CHANGED` で接続検出
  - `BluetoothHeadset.ACTION_CONNECTION_STATE_CHANGED` で通話検出
  - `ACTION_FOUND` / `ACTION_DISCOVERY_FINISHED` でペアリング検出
- **`AudioDeviceCallback`**: `onAudioDevicesAdded/Removed` で BT デバイスの追加/削除を検出。
- **バッテリー取得**: `BluetoothDevice::class.java.methods.firstOrNull { it.name == "getBatteryLevel" }` のリフレクション。
- **権限**: Android 12+ で `BLUETOOTH_CONNECT` / `BLUETOOTH_SCAN`、Android 11 以下で `ACCESS_FINE_LOCATION` が必要。`hasBluetoothConnectPermission` / `hasBluetoothScanPermission` / `hasFineLocationPermission` で判定。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/preferences/ConnectivityPreferences.kt` (推定)
- システム API: `ConnectivityManager`, `WifiManager`, `BluetoothManager`, `BluetoothAdapter`

---

## ThemeStateHolder

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ThemeStateHolder.kt` (319 行)
**アノテーション**: `@Singleton`
**役割**: アルバムアート / プレイヤー色テーマ (アルバムアート / Lava Lamp / デフォルト) の動的 ColorScheme 生成とキャッシュ管理。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Process | `colorSchemeProcessor: ColorSchemeProcessor` |
| Preferences | `themePreferencesRepository: ThemePreferencesRepository` |

### 主要 StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `currentAlbumArtColorSchemePair` | `StateFlow<ColorSchemePair?>` | 現在のアルバムアート ColorScheme |
| `currentAlbumArtUri` | `StateFlow<String?>` | 現在のアルバムアート URI |
| `lavaLampColors` | `StateFlow<ImmutableList<Color>>` | Lava Lamp モードの色 |
| `activePlayerColorSchemePair` | `StateFlow<ColorSchemePair?>` | プレイヤー画面用 (active) |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun initialize(scope: CoroutineScope)` | スコープ注入 + `currentSong` 監視開始 |
| `suspend fun extractAndGenerateColorScheme(uri, currentSongUri, isPreload)` | アルバムアート → ColorScheme |
| `private fun updateLavaLampColors(schemePair: ColorSchemePair?)` | Lava Lamp 用色更新 |
| `fun getAlbumColorSchemeFlow(uriString): StateFlow<ColorSchemePair?>` | アルバム別色 (キャッシュ) |
| `fun ensureAlbumColorScheme(uriString: String)` | 色生成をスケジュール |
| `suspend fun getOrGenerateColorScheme(uriString): ColorSchemePair?` | suspend 取得 |
| `suspend fun forceRegenerateColorScheme(uriString, regenerateAllStyles, paletteStyle, accuracy)` | 強制再生成 |
| `fun trimMemory(level: Int)` | メモリトリム |
| `fun onCleared()` | クリーンアップ |

### 内部実装メモ

- **個別アルバム色キャッシュ**: `individualAlbumColorSchemes: LinkedHashMap<String, MutableStateFlow<ColorSchemePair?>>` で LRU 風に保持 (`removeEldestEntry` で上限管理)。
- **pendingAlbumColorSchemeTargets**: 同じ URI に対する複数回 `ensureAlbumColorScheme` を 1 つの Job に集約 (重複処理防止)。
- **`colorSchemeProcessor` 委譲**: 実際の Bitmap → Seed Color → Material3 ColorScheme 変換は `ColorSchemeProcessor` に集約。
- **Lava Lamp モード**: `ThemePreference.LAVA_LAMP` の場合、`schemePair.dark` から抽出した色リストを `lavaLampColors` に格納。Compose 側の `LavaLampBackground` で使用。
- **アルバムアート URI 変更検知**: `currentSong.albumArtUriString` の `distinctUntilChanged` で、変更時にのみ ColorScheme 再生成。
- **Trim Memory**: `level >= TRIM_MEMORY_UI_HIDDEN` で個別アルバム色キャッシュを全削除 (再起動時に再生成)。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ColorSchemeProcessor.kt` (`viewmodels-support.md`)
- `app/src/main/java/com/theveloper/pixelplay/data/preferences/ThemePreferencesRepository.kt`
- カラーパレット: `app/src/main/java/com/theveloper/pixelplay/ui/theme/`

---

## AccountsViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/AccountsViewModel.kt` (278 行)
**アノテーション**: `@HiltViewModel`
**役割**: 接続中外部サービス一覧 (Telegram / GDrive / Netease / QQ / Navidrome / Jellyfin) を集約表示。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Repository | `telegramRepository: TelegramRepository` |
| Repository | `musicRepository: MusicRepository` |
| Repository | `gdriveRepository: GDriveRepository` |
| Repository | `neteaseRepository: NeteaseRepository` |
| Repository | `qqMusicRepository: QqMusicRepository` |
| Repository | `navidromeRepository: NavidromeRepository` |
| Repository | `jellyfinRepository: JellyfinRepository` |

### `ExternalServiceAccount` (enum)

| 値 |
|----|
| `TELEGRAM` |
| `GDRIVE` |
| `NETEASE` |
| `QQ_MUSIC` |
| `NAVIDROME` |
| `JELLYFIN` |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<AccountsUiState>` | 画面状態 (connected + disconnected) |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun logout(service: ExternalServiceAccount)` | 該当サービスをログアウト |
| `private fun formatCount(count, singular, plural): String` | "3 channels" / "1 channel" 整形 |

### 内部実装メモ

- **6 つの独立 StateFlow**: `telegramStateFlow` / `gDriveStateFlow` / `neteaseStateFlow` / `qqMusicStateFlow` / `navidromeStateFlow` / `jellyfinStateFlow` を `combine` して `uiState` へ。
- **ログアウト中の視覚化**: `loggingOutServices: Set<ExternalServiceAccount>` を MutableStateFlow で保持し、UI でスピナー表示。
- **Account 表示モデル**: `ExternalAccountUiModel(service, title, accountLabel, syncedContentLabel, isLoggingOut)`。
- **キャッシュ無効化**: ログアウト時に `musicRepository` の関連キャッシュをクリア。

### 関連ファイル

- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/AccountsScreen.kt`
- 個別 Repository: `../02-data-network/` 配下

---

## ArtistSettingsViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/ArtistSettingsViewModel.kt` (158 行)
**アノテーション**: `@HiltViewModel`
**役割**: アーティスト名分割 (区切り文字) 設定画面。`SyncManager` 連携で再スキャン。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| Worker | `syncManager: SyncManager` |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<ArtistSettingsUiState>` | 画面状態 |
| `isSyncing` | `StateFlow<Boolean>` | 再スキャン中 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun setGroupByAlbumArtist(enabled: Boolean)` | アルバムアーティストでグループ化 |
| `fun setArtistDelimiters(delimiters: List<String>)` | 区切り文字設定 |
| `fun addDelimiter(delimiter: String): Boolean` | 追加 (重複防止) |
| `fun removeDelimiter(delimiter: String)` | 削除 |
| `fun resetDelimitersToDefault()` | デフォルト復元 |
| `fun addWordDelimiter` / `removeWordDelimiter` / `resetWordDelimitersToDefault` | 単語区切り |
| `fun setExtractArtistsFromTitle(enabled: Boolean)` | タイトルから抽出 |
| `fun rescanLibrary()` | 再スキャン起動 |

### 内部実装メモ

- **2 種類の区切り文字**:
  - `artistDelimiters` — アーティスト名分割 (例: "Artist1; Artist2" → 2 名に分割)
  - `wordDelimiters` — 単語分割 (例: "AC/DC" → "AC" + "DC")
- **再スキャン必要フラグ**: 設定変更時に `rescanRequired = true` にして、UI で「再スキャンが必要」と表示。

### 関連ファイル

- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/ArtistSettingsScreen.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/preferences/UserPreferencesRepository.kt`

---

## DeviceCapabilitiesViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/DeviceCapabilitiesViewModel.kt` (641 行)
**アノテーション**: `@HiltViewModel`
**役割**: デバイス性能・ストレージ・再生互換性レポート画面。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Engine | `engine: DualPlayerEngine` |
| Preferences | `userPreferencesRepository: UserPreferencesRepository` |
| DAO | `musicDao: MusicDao` |
| Diagnostics | `reportCollector: DebugPerformanceReportCollector` |

### データクラス (10 個)

| 名前 | フィールド |
|------|----------|
| `CodecInfo` | name, supportedTypes, isHardwareAccelerated, maxSupportedInstances |
| `AudioOutputInfo` | name, category |
| `AudioCapabilities` | outputSampleRate, outputFramesPerBuffer, isLowLatencySupported, isProAudioSupported, isPcmFloatSupported, offloadSupportedFormats, outputRoutes, supportedCodecs |
| `FormatSupportInfo` | label, mimeType, isDecoderAvailable, isHardwareAccelerated, isOffloadSupported, librarySongCount |
| `LocalMusicStorageSummary` | localSongCount, cloudSongCount, knownLocalFileCount, unavailableLocalFileCount, localMusicBytes, deviceAvailableBytes, deviceTotalBytes |
| `PlaybackCompatibilitySummary` | supportedLibrarySongCount, unsupportedLibrarySongCount, unknownFormatSongCount, unsupportedFormats, localHiResSongCount, resampledLocalSongCount, maxLocalSampleRate, maxLocalBitrate |
| `MemorySummary` | availableRamBytes, totalRamBytes, memoryClassMb, isLowRamDevice, isSystemLowMemory |
| `ExoPlayerInfo` | version, renderers, decoderCounters |
| `DeviceCapabilitiesState` | 上記を束ねる 60+ フィールド |
| `AudioFormatCandidate` (private) | label, mimeType, offloadEncoding |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `state` | `StateFlow<DeviceCapabilitiesState>` | 画面状態 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun generatePerformanceReport()` | パフォーマンスレポート生成 |
| `fun setAdvancedPerformanceDiagnosticsEnabled(enabled: Boolean)` | 高度診断モード切替 |
| `fun markLagNow()` | ラグマーカー (デバッグ用) |

### 内部実装メモ

- **Codec 列挙**: `MediaCodecList(ALL_CODECS)` で全コーデックを列挙。`Build.VERSION.SDK_INT >= Q` でハードウェア判定。
- **PCM bit depth**: `AudioFormat.ENCODING_PCM_16BIT` / `24BIT` / `FLOAT` から 16 / 24 / 32 bit にマッピング。
- **Offload サポート**: `AudioManager.getOffloadSupportedFormats` + 個別の `AudioFormat.Builder().setEncoding(...)` でテスト。
- **Storage 計測**: `StatFs(Environment.getExternalStorageDirectory().path)` + 各曲の `File.length()` の合計。
- **ライブラリ互換性**: ライブラリ全曲と `supportedCodecs` を突き合わせ、サポート / 非サポート / 未知の 3 分類。
- **Hi-Res 判定**: `sampleRate > 48_000` で Hi-Res カウント。
- **メモリトリム**: `TRIM_MEMORY_RUNNING_LOW` 等を `ComponentCallbacks2` 経由で監視。
- **デバッグレポート**: `DebugPerformanceReportCollector.generate()` で問題発生時のダンプ文字列を生成し、`state.performanceReport` に保持。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/service/player/ActiveDecoderInfo.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/service/player/HiFiCapabilityChecker.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/diagnostics/DebugPerformanceReportCollector.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/diagnostics/AdvancedPerformanceDiagnostics.kt`
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/DeviceCapabilitiesScreen.kt`

---

## MashupViewModel

**パッケージ**: `com.theveloper.pixelplay.presentation.viewmodel`
**ファイル**: `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/MashupViewModel.kt` (139 行)
**アノテーション**: `@HiltViewModel`
**役割**: DJ マッシュアップ画面。2 つの独立 ExoPlayer (Deck) を制御し、クロスフェーダー / ボリューム / 速度 / シークを管理。

### 注入される依存

| 種類 | フィールド |
|------|----------|
| Application | `application: Application` |
| Repository | `musicRepository: MusicRepository` |

### `DeckState` (data class)

| フィールド | 目的 |
|----------|------|
| `song: Song?` | ロード中の曲 |
| `isPlaying: Boolean` | 再生中 |
| `progress: Float` | 進捗 (0-1) |
| `volume: Float` | ボリューム (0-1) |
| `speed: Float` | 速度 (0.5-2.0) |
| `stemWaveforms: Map<String, List<Int>>` | ステム波形 |

### `MashupUiState` (data class)

| フィールド | 目的 |
|----------|------|
| `deck1` / `deck2` | 2 デッキ |
| `crossfaderValue: Float` | -1 (deck1) 〜 +1 (deck2) |
| `allSongs: List<Song>` | 曲選択候補 |
| `showSongPickerForDeck: Int?` | 1 or 2 = ピッカー表示先 |

### StateFlow

| 名前 | 型 | 目的 |
|------|---|------|
| `uiState` | `StateFlow<MashupUiState>` | 画面状態 |

### 主要 public 関数

| シグネチャ | 目的 |
|-----------|------|
| `fun loadSong(deck: Int, song: Song)` | デッキに曲ロード |
| `fun playPause(deck: Int)` | 再生/一時停止 |
| `fun seek(deck: Int, progress: Float)` | シーク |
| `fun nudge(deck: Int, amountMs: Long)` | 微小シーク |
| `fun setVolume(deck: Int, volume: Float)` | 個別ボリューム |
| `fun setSpeed(deck: Int, speed: Float)` | 速度 (0.5-2.0) |
| `fun onCrossfaderChange(value: Float)` | クロスフェーダー値変更 |
| `fun openSongPicker(deck: Int)` / `closeSongPicker()` | 曲選択 |
| `fun onCleared()` | デッキ解放 |

### 内部実装メモ

- **2 つの独立 `ExoPlayer`**: `deck1Controller` / `deck2Controller` を `lateinit var` で保持。`DeckController` が `ExoPlayer` を構築 (HiRes 対応 / Offload 無効化)。
- **クロスフェーダー計算**: `vol1 = (1 - (value+1)/2)` / `vol2 = (value+1)/2` で -1 → deck1 のみ、+1 → deck2 のみ。
- **Progress Job**: 100ms 毎に `controller.getProgress()` を呼び `_uiState` を更新。
- **ステム波形**: 別 API (推定) で取得、未使用なら空 Map。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/presentation/viewmodel/exts/DeckController.kt` (`viewmodels-support.md`)
- 画面: `app/src/main/java/com/theveloper/pixelplay/presentation/screens/MashupScreen.kt`
