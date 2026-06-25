# streaming-telegram.md

> Telegram 経由の楽曲再生 (TDLib + Ktor CIO Stream Proxy) 仕様。
> **本プロジェクト最大のサブシステムの一つ** (合計約 1,700 行)。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/telegram/
├─ TelegramClientManager.kt        // TDLib ラッパ (240 lines)
├─ TelegramCacheManager.kt         // ファイル / 埋め込みアートキャッシュ (280 lines)
├─ TelegramRepository.kt           // 楽曲同期 + 再生 (840 lines)
└─ TelegramStreamProxy.kt          // Ktor CIO プロキシ (360 lines)
```

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `presentation/telegram/*`, `data/repository/MusicRepositoryImpl.kt`, `data/service/MusicService.kt` (再生) |
| 下流 (依存先) | `org.drinkless.tdlib:TdApi` + `Client`, `data/database/{TelegramDao, LocalPlaylistDao}`, `data/preferences/PlaylistPreferencesRepository`, `data/stream/CloudStreamSecurity`, `io.ktor:ktor-server-cio`, `okhttp3.OkHttpClient`, `android.media.MediaMetadataRetriever` |

---

## 1. `TelegramClientManager` (`@Singleton`)

### 役割

TDLib の `Client` ライフサイクル管理。**認証ステートマシン** と **更新/エラーフロー** を Kotlin Flow で公開。

### 依存

| 依存 | 用途 |
| --- | --- |
| `@ApplicationContext Context` | `tdlib` / `tdlib_files` ディレクトリ |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `_authorizationState` | `MutableStateFlow<TdApi.AuthorizationState?>` | 認証ステート |
| `authorizationState` | `StateFlow<TdApi.AuthorizationState?>` | 公開 |
| `_updates` | `MutableSharedFlow<TdApi.Object>` (extraBufferCapacity=64) | TDLib からの全更新 |
| `updates` | `SharedFlow<TdApi.Object>` | 公開 |
| `_errors` | `MutableSharedFlow<TdApi.Error>` (extraBufferCapacity=16) | TDLib エラー |
| `errors` | `SharedFlow<TdApi.Error>` | 公開 |
| `client` | `var Client?` | TDLib クライアント |
| `recreateClientAfterClose` | `var Boolean` | close 後の自動再作成 |

### 認証ステート

| `TdApi.AuthorizationState` | 次の操作 |
| --- | --- |
| `WaitTdlibParameters` | `initializeClient()` 呼び出し |
| `WaitPhoneNumber` | `sendPhoneNumber(phone)` |
| `WaitCode` | `checkAuthenticationCode(code)` |
| `WaitPassword` | `checkAuthenticationPassword(pw)` |
| `Ready` | 認証完了 |

### 公開 API

#### 認証

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `sendPhoneNumber` | `fun sendPhoneNumber(phoneNumber: String)` | `PhoneNumberAuthenticationSettings()` を作成 → `SetAuthenticationPhoneNumber`。 |
| `checkAuthenticationCode` | `fun checkAuthenticationCode(code: String)` | `CheckAuthenticationCode`。 |
| `checkAuthenticationPassword` | `fun checkAuthenticationPassword(password: String)` | `CheckAuthenticationPassword`。 |
| `logout` | `fun logout()` | `LogOut`。 |

#### ライフサイクル

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `closeClient` | `fun closeClient(recreate: Boolean = false)` | TDLib クライアントを閉じ、必要なら再作成をスケジュール。 |
| `isReady` | `fun isReady(): Boolean` | `authorizationState is TdApi.AuthorizationStateReady`。 |
| `awaitReady` | `suspend fun awaitReady(timeoutMs = 30_000L): Boolean` | 30 秒タイムアウト。 |

#### 通信

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `sendRequest<T>` | `inline fun <reified T : TdApi.Object> sendRequest(query: TdApi.Function<T>): T?` | `withTimeoutOrNull(20_000L)` でラップ。`TdApi.Error` を `_errors` に通知。 |

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `initializeClient` | `private fun initializeClient()` | `TdApi.SetTdlibParameters` を初回に投げ、`Client` を作成。 |
| `onAuthorizationStateUpdated` | `private fun onAuthorizationStateUpdated(authState: TdApi.AuthorizationState)` | 状態変化時に `_authorizationState` を更新。 |
| `updateHandler` | `Client.ResultHandler` | 全ての更新を `_updates` に emit。 |
| `defaultHandler` | `Client.ResultHandler` | エラー時は `_errors` に通知。 |
| `reportTdError` | `private fun reportTdError(error: TdApi.Error)` | Timber でログ + `_errors` に emit。 |

### TDLib パラメータ

```kotlin
TdApi.SetTdlibParameters().apply {
    databaseDirectory = File(context.filesDir, "tdlib").absolutePath
    filesDirectory = File(context.filesDir, "tdlib_files").absolutePath
    useMessageDatabase = true
    useSecretChats = false
    apiId = BuildConfig.TELEGRAM_API_ID
    apiHash = BuildConfig.TELEGRAM_API_HASH
    systemLanguageCode = "en"
    deviceModel = "PixelPlayer"
    systemVersion = "Android ${Build.VERSION.RELEASE}"
    applicationVersion = BuildConfig.VERSION_NAME
    // ...
}
```

> `BuildConfig.TELEGRAM_API_ID` / `TELEGRAM_API_HASH` は `local.properties` 由来。

### 内部例外

```kotlin
class TdlibRequestException(val code: Int, ...) : Exception(message)
```

---

## 2. `TelegramCacheManager` (`@Singleton`)

### 役割

再生中のファイル管理 + 埋め込みアート抽出キャッシュ + 失敗時ネガティブキャッシュ + tdlib キャッシュクリア。

### 依存

| 依存 | 用途 |
| --- | --- |
| `TelegramClientManager` | `DeleteFile` / `StorageStatistics` |
| `@ApplicationContext Context` | `cacheDir` |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `scope` | `CoroutineScope` | `SupervisorJob + Dispatchers.IO` |
| `activeFileId` | `var Int?` | 現在再生中の file id |
| `recentlyPlayedFileIds` | `ConcurrentHashMap.newKeySet<Int>()` | 最近再生した file id (重複ダウンロード防止) |
| `maxEmbeddedArtCacheSize` | `50L * 1024 * 1024` (50 MB) | 埋め込みアートのディスク上限 |
| `_embeddedArtUpdated` | `MutableSharedFlow<String>` (extraBufferCapacity=8) | 抽出完了通知 |
| `embeddedArtUpdated` | `SharedFlow<String>` | Coil Fetcher が監視 |
| `failedArtCache` | `ConcurrentHashMap<String, Long>` | 失敗ネガティブキャッシュ |
| `FAILED_ART_EXPIRY_MS` | `5 * 60 * 1000L` (5 分) | ネガティブキャッシュ TTL |
| `audioFileHistory` | `Collections.synchronizedList(LinkedList<Int>())` | 履歴 (最大 5 件) |
| `HISTORY_CACHE_LIMIT` | `5` | 履歴最大件数 |

### 公開 API

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `notifyEmbeddedArtExtracted` | `fun notifyEmbeddedArtExtracted(chatId, messageId)` | `"telegram_art:<chatId>/<messageId>"` URI を `_embeddedArtUpdated` に emit。 |
| `isArtFailed` | `fun isArtFailed(chatId, messageId): Boolean` | ネガティブキャッシュ判定 (5 分以内なら true)。 |
| `markArtFailed` | `fun markArtFailed(chatId, messageId)` | ネガティブキャッシュ登録。 |
| `setActivePlayback` | `fun setActivePlayback(fileId: Int?)` | アクティブファイル設定。古い履歴は `DeleteFile` で削除。 |
| `onPlaybackStopped` | `fun onPlaybackStopped()` | `activeFileId` を解放。 |
| `cleanupAudioFile` | `private fun cleanupAudioFile(fileId)` | `TdApi.DeleteFile` 送信。 |
| `clearEmbeddedArtCache` | `fun clearEmbeddedArtCache()` | `cacheDir/telegram_embedded_art_*.jpg` を全削除。 |
| `trimEmbeddedArtCache` | `fun trimEmbeddedArtCache()` | 合計サイズが 50 MB を超える場合、古い順に削除。 |
| `clearTdLibCache` | `suspend fun clearTdLibCache()` | `OptimizeStorage` で全消去。 |
| `clearAllCache` | `suspend fun clearAllCache()` | `clearEmbeddedArtCache` + `clearTdLibCache`。 |

### `setActivePlayback` のロジック

```
1. fileId を activeFileId に設定
2. audioFileHistory に追加 (重複時はスキップ)
3. HISTORY_CACHE_LIMIT を超えたら古い順に cleanupAudioFile
4. recentlyPlayedFileIds に追加
```

---

## 3. `TelegramRepository` (`@Singleton`)

### 依存

| 依存 | 用途 |
| --- | --- |
| `TelegramClientManager` | TDLib |
| `TelegramDao` | DAO |
| `PlaylistPreferencesRepository` | アプリ内プレイリスト |

### 定数

| 定数 | 値 | 用途 |
| --- | --- | --- |
| `AUTH_REQUEST_TIMEOUT_MS` | `20_000L` | |
| `TELEGRAM_PLAYLIST_PREFIX` | `"telegram_channel:"` | 統合プレイリスト ID |
| `TELEGRAM_TOPIC_PLAYLIST_PREFIX` | `"telegram_topic:"` | 統合トピックプレイリスト ID |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `authorizationState` | `Flow<TdApi.AuthorizationState?>` | `clientManager.authorizationState` |
| `authErrors` | `SharedFlow<TdApi.Error>` | `clientManager.errors` |
| `resolvedPathCache` | `ConcurrentHashMap<Int, String>` | fileId → ローカルパス |
| `uriResolutionCache` | `ConcurrentHashMap<String, Pair<Int, Long>>` | URI → (fileId, messageId) |
| `repositoryScope` | `CoroutineScope` | |
| `activeDownloads` | `ConcurrentHashMap<Int, Deferred<String?>>` | fileId 単位の多重ダウンロード集約 |
| `downloadSemaphore` | `Semaphore(4)` | 同時ダウンロード 4 件 |
| `_downloadCompleted` | `MutableSharedFlow<Int>` | download 完了通知 |
| `downloadCompleted` | `SharedFlow<Int>` | |
| `_songFileUpdated` | `MutableSharedFlow<String>` | song file 更新通知 |
| `songFileUpdated` | `SharedFlow<String>` | |

### 認証情報管理

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `clearMemoryCache` | `fun clearMemoryCache()` | `resolvedPathCache`, `uriResolutionCache`, `activeDownloads` をクリア。 |
| `isReady` | `fun isReady(): Boolean` | `clientManager.isReady()` |
| `awaitReady` | `suspend fun awaitReady(timeoutMs = 30_000L): Boolean` | 委譲。 |
| `sendPhoneNumber` | `fun sendPhoneNumber(phone)` | `clientManager.sendPhoneNumber` |
| `sendPhoneNumberAwait` | `suspend fun sendPhoneNumberAwait(phone, timeoutMs = 20_000L): Boolean` | 完了待機版。 |
| `checkAuthenticationCode` | `fun checkAuthenticationCode(code)` | |
| `checkAuthenticationCodeAwait` | `suspend fun checkAuthenticationCodeAwait(...)` | |
| `checkAuthenticationPassword` | `fun checkAuthenticationPassword(password)` | |
| `checkAuthenticationPasswordAwait` | `suspend fun checkAuthenticationPasswordAwait(...)` | |
| `logout` | `fun logout()` | `clientManager.logout()` + `clearMemoryCache` |

### チャット / メッセージ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `searchPublicChat` | `suspend fun searchPublicChat(username): TdApi.Chat?` | `@username` を公開チャット検索。 |
| `isForum` | `suspend fun isForum(chatId): Boolean` | `chat.type is TdApi.ChatTypeSupergroup && supergroup.isForum`。 |
| `getForumTopics` | `suspend fun getForumTopics(chatId): List<TelegramTopicEntity>` | `GetForumTopics` をページング。**リフレクションで threadId 抽出** (TdApi バージョン差吸収)。 |
| `getAudioMessagesByTopic` | `suspend fun getAudioMessagesByTopic(chatId, threadId): List<Song>` | `SearchChatMessages` をページング (batchSize=100)。topicId/messageThreadId をリフレクション設定。 |
| `getAudioMessages` | `suspend fun getAudioMessages(chatId): List<Song>` | チャンネル全体から音声メッセージ取得。 |
| `mapMessageToSong` | `private suspend fun mapMessageToSong(message: TdApi.Message): Song?` | `MessageAudio` または `MessageDocument (audio/*)` を Song に変換。 |
| `getMessage` | `suspend fun getMessage(chatId, messageId): TdApi.Message?` | `GetMessage`。 |

### ファイルダウンロード

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `downloadFile` | `suspend fun downloadFile(fileId, priority=1): TdApi.File?` | `DownloadFile` を 1 度投げる (ポーリングなし)。 |
| `getFile` | `suspend fun getFile(fileId): TdApi.File?` | `GetFile`。 |
| `isFileCached` | `suspend fun isFileCached(fileId): Boolean` | `getFile(fileId).local.isDownloadingCompleted` |
| `downloadFileAwait` | `suspend fun downloadFileAwait(fileId, priority=1): String?` | **完了待機 + 多重集約**。15 秒タイムアウト + 60 秒 path wait。 |

### `downloadFileAwait` のロジック

```
1. existingJob = activeDownloads[fileId]
2. if existingJob: return existingJob.await()
3. newJob = repositoryScope.async(start = LAZY) {
     currentFile = getFile(fileId)
     if (currentFile.local.isDownloadingCompleted && currentFile.local.path != null):
       return local.path
     isSmallFile = size == 0L || size < 1MB
     resultFile = withTimeout(15_000L) {
       // small: 単発 DownloadFile + 60 秒 path wait
       // large: DownloadFile を投げ、updates から UpdateFile を filter して completed を待つ
     }
   }
4. activeDownloads[fileId] = newJob
5. val path = withTimeoutOrNull(60_000L) { newJob.await() }
6. activeDownloads.remove(fileId)
7. return path
```

### URI 解決

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `resolveTelegramUri` | `suspend fun resolveTelegramUri(uriString): Pair<Int, Long>?` | `telegram://<chatId>/<messageId>` を (fileId, messageId) に変換。`getMessage` で内容取得。 |
| `preResolveTelegramUri` | `fun preResolveTelegramUri(uriString)` | バックグラウンドで uriResolutionCache にシード。 |

### プレイリスト統合

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getAppPlaylistIdForChannel` | `private fun getAppPlaylistIdForChannel(chatId): String` | `"telegram_channel:$chatId"` |
| `getAppPlaylistIdForTopic` | `private fun getAppPlaylistIdForTopic(chatId, threadId): String` | `"telegram_topic:${chatId}_${threadId}"` |
| `toUnifiedTelegramSongId` | `private fun toUnifiedTelegramSongId(telegramSongId: String): Long` | `-telegramSongId.hashCode().absoluteValue` (負値で他の Song ID と衝突しない)。 |
| `updateAppPlaylistForTelegramChannel` | `suspend fun updateAppPlaylistForTelegramChannel(chatId, telegramEntities)` | アプリ内プレイリスト upsert |
| `updateAppPlaylistForTopic` | `suspend fun updateAppPlaylistForTopic(chatId, threadId, telegramEntities)` | |
| `upsertPlaylist` | `private suspend fun upsertPlaylist(...)` | `playlistPreferencesRepository` の upsert ヘルパ |
| `deleteAppPlaylistForTelegramChannel` | `suspend fun deleteAppPlaylistForTelegramChannel(chatId)` | |
| `deleteAppPlaylistForTopic` | `suspend fun deleteAppPlaylistForTopic(chatId, threadId)` | |
| `deleteAllTopicPlaylistsForChannel` | `suspend fun deleteAllTopicPlaylistsForChannel(chatId)` | `$TELEGRAM_TOPIC_PLAYLIST_PREFIX${chatId}_` で全削除 |

### アート warm-up

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `warmUpArtworkForSongs` | `fun warmUpArtworkForSongs(songs: List<Song>)` | songs の埋め込みアートを非同期でプリフェッチ。 |
| `warmUpArtwork` | `private suspend fun warmUpArtwork(chatId, messageId)` | `TdApi.Message` 取得 → サムネ fileId 抽出 → ダウンロード開始。 |
| `extractArtworkFileId` | `private fun extractArtworkFileId(content: TdApi.MessageContent?): Int?` | `audio.albumCoverThumbnail?.file?.id` または `document.thumbnail?.file?.id`。 |

### 内部ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `persistSongFilePathIfNeeded` | `private suspend fun persistSongFilePathIfNeeded(fileId, path?)` | `dao.getSongByFileId(fileId)?.filePath = path` を更新。 |

---

## 4. `TelegramStreamProxy` (`@Singleton`)

**`CloudStreamProxy` を継承しない** (ファイルベースのため range/stall 制御が必要)。

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `server` | `var EmbeddedServer<...>` | Ktor サーバ |
| `proxyScope` | `CoroutineScope` | `SupervisorJob + Dispatchers.IO` |
| `startJob` | `var Job?` | 起動ジョブ |
| `actualPort` | `var Int` | ポート |

### 公開 API

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `startIfNeeded` | `fun startIfNeeded()` | |
| `start` | `fun start()` | `ServerSocket(0)` で空きポート取得 |
| `stop` | `fun stop()` | サーバ停止 + scope cancel |
| `getProxyUrl` | `fun getProxyUrl(fileId, knownSize=0): String` | `"http://127.0.0.1:<port>/telegram/<fileId>?size=<knownSize>"` |
| `isReady` | `fun isReady(): Boolean` | `actualPort > 0` |
| `awaitReady` | `suspend fun awaitReady(timeoutMs = 10_000L): Boolean` | 50ms ポーリング |
| `ensureReady` | `suspend fun ensureReady(timeoutMs = 10_000L): Boolean` | 未起動なら起動 |

### 非公開

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `createServer` | `private fun createServer(port): EmbeddedServer<...>` | Ktor ルーティング |
| `connector` | `private fun connector(builder: EngineConnectorBuilder.() -> Unit) {}` | ダミー |

### Ktor ルート内部

```
GET /telegram/{fileId}?size=...
1. ready = telegramRepository.awaitReady(10_000L)
2. fileInfo = telegramRepository.downloadFile(fileId, 1)
3. path wait (最大 ? 秒)
4. knownSize = params["size"]?.toLongOrNull() (Content-Length 推測)
5. file = File(path)
6. file wait (サイズが knownSize になるまで)
7. rangeValidation = CloudStreamSecurity.validateRangeHeader(call.request.headers["Range"])
8. isRangeRequest = rangeValidation.normalizedHeader != null
9. start = 0; end = expectedSize - 1 (or Long.MAX_VALUE - 1)
10. contentLength = end - start + 1
11. respondBytesWriter で Content-Range / Accept-Ranges を維持しつつ転送
12. RandomAccessFile で読み、stall 検出 (downloadedPrefixSize) でバックオフ
```

### Range + stall 制御

- `cachedDownloadedPrefixSize = fileInfo.local.downloadedPrefixSize`
- 毎チャンク読み込み前に `updatedInfo = telegramRepository.getFile(fileId)` で進行状況更新。
- 進行していない場合 stallDelayMs を 50ms → 100ms → ... → 400ms まで増やす (`maxStallDelayMs = 400`)。
- バッファ読み込みサイズ = `min(buffer.size, min(remaining, remainingValid))`。

---

## 5. 内部実装メモ

### TDLib バージョン互換性

`getForumTopics` はリフレクションで `info.javaClass.declaredFields` を舐めて `threadId` 相当を探す (TdApi バージョン差吸収)。優先順位: `threadId`, `topicId`, `messageThreadId` → その他 Long フィールド。

### 同時ダウンロード 4

```kotlin
private val downloadSemaphore = Semaphore(4)
```

> Media3 ExoPlayer の先読み用 + Coil Fetcher のサムネイル用 + ユーザ手動再生の余裕。

### URI キャッシュ

`uriResolutionCache: ConcurrentHashMap<String, Pair<Int, Long>>` で `telegram://<chatId>/<messageId>` → (fileId, messageId) を 1 度だけ解決してキャッシュ。

### `downloadFileAwait` の 15 秒 vs 60 秒の使い分け

- **15 秒**: 小ファイル (≤1MB) は DownloadFile 投げっぱなしで完了。
- **60 秒**: 大ファイルは `UpdateFile` 監視しつつ `local.isDownloadingCompleted` 待ち。

### history based cleanup

`audioFileHistory` で最近再生した 5 件の file id のみキャッシュ保持。古いものは `DeleteFile` で解放。

---

## 6. 関連ファイル

- TDLib 認証 UI: `presentation/telegram/auth/TelegramLoginViewModel.kt`
- ダッシュボード UI: `presentation/telegram/dashboard/TelegramDashboardViewModel.kt`
- チャンネル検索 UI: `presentation/telegram/channel/TelegramChannelSearchViewModel.kt`
- Coil Fetcher (埋め込みアート抽出): [`image-fetchers.md`](./image-fetchers.md) §4
- DB: [`../01-data-foundation/database.md`](../01-data-foundation/database.md)
