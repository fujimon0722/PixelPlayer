# provider-screens.md

> 外部サービス (gdrive / jellyfin / navidrome / netease / qqmusic / telegram) の Login + Dashboard ViewModel + Screen の詳細仕様。

> **注意**: 各 provider ディレクトリには `auth/` (Login) と `dashboard/` の 2 つのサブディレクトリがあり、それぞれ ViewModel + Screen / Activity の 2 ファイルが対になっている。本 spec は ViewModel を中心に記述。Screen / Activity は UI のため `../05-presentation-ui/` 配下のドキュメントを参照。

---

## 共通パターン

### Login ViewModel の構造

| State | 派生 |
|-------|------|
| `sealed interface XxxLoginState` | `Idle` / `Loading` / `Success(user)` / `Error(message)` |

- `state: StateFlow<XxxLoginState>` を公開
- `fun login(...)` または `fun processCredential(...)` / `fun processCookies(...)` で遷移
- `fun clearError()` / `fun reset()` で Idle に戻す

### Dashboard ViewModel の構造

- 同期対象リスト (Playlists / Folders / Channels) を `StateFlow<List<...>>` で公開 (Repository の `getXxx()` ストリーム)
- `_isSyncing: StateFlow<Boolean>` で同期中フラグ
- `_syncMessage: StateFlow<String?>` で結果メッセージ
- 必要に応じて `_selectedXxxSongs: StateFlow<List<Song>>` で詳細楽曲
- `fun syncAllXxxAndSongs()` / `fun syncXxx(id)` / `fun logout()` を提供

---

## GDrive (Google Drive)

### ディレクトリ

| ファイル | 行数 | 役割 |
|---------|------|------|
| `app/src/main/java/com/theveloper/pixelplay/presentation/gdrive/auth/GDriveLoginViewModel.kt` | 156 | ログインフロー |
| `app/src/main/java/com/theveloper/pixelplay/presentation/gdrive/auth/GDriveLoginActivity.kt` | (Activity) | Google Sign-In UI |
| `app/src/main/java/com/theveloper/pixelplay/presentation/gdrive/dashboard/GDriveDashboardViewModel.kt` | 84 | ダッシュボード |
| `app/src/main/java/com/theveloper/pixelplay/presentation/gdrive/dashboard/GDriveDashboardScreen.kt` | (Screen) | Composable UI |

### `GDriveLoginViewModel`

| 注入 | `repository: GDriveRepository` |
|------|------|

**StateFlow**: `state: StateFlow<GDriveLoginState>`

**`GDriveLoginState` (sealed class)**:
- `Idle` / `Loading` / `LoggedIn(email)` / `FolderSetup(folders, currentPath, isLoading)` / `Success` / `Error(message)`

**`FolderItem` (data class)**: `id` / `name` / `isFolder`

**主要 public 関数**:
- `fun processCredential(idToken: String, serverAuthCode: String?)` — Google Sign-In 完了後の処理
- `fun browseFolders(parentId: String)` — フォルダ一覧取得
- `fun navigateIntoFolder(folder: FolderItem)` / `navigateBack(): Boolean` / `navigateToBreadcrumb(index: Int)` — パンくずナビゲーション
- `fun createMusicFolder()` — 音楽用新規フォルダ作成
- `fun selectFolder(folderId, folderName)` — 同期対象フォルダ選択

**内部実装**:
- 内部 `breadcrumb: MutableList<FolderItem>` で現在のパス階層を保持
- `FolderSetup` ステートで「ログイン後 → フォルダ選択」のウィザードを実現
- `LoggedIn` 検出後自動で `browseFolders("root")` 呼び出し

### `GDriveDashboardViewModel`

| 注入 | `repository: GDriveRepository` |
|------|------|

**StateFlow**:
- `folders: StateFlow<List<GDriveFolderEntity>>` — 同期中フォルダ一覧
- `isSyncing: StateFlow<Boolean>`
- `syncMessage: StateFlow<String?>`
- `isLoggedIn: StateFlow<Boolean>` (Repository 経由)

**プロパティ**:
- `userEmail: String?` (Repository から)

**主要 public 関数**:
- `fun syncAllFoldersAndSongs()` — 全フォルダ + 全曲同期
- `fun syncFolder(folderId: String)` — 単体フォルダ同期
- `fun removeFolder(folderId: String)` — フォルダ削除
- `fun logout()` — ログアウト

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/gdrive/GDriveRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/database/GDriveFolderEntity.kt`
- `../02-data-network/gdrive.md` (推定)

---

## Jellyfin

### ディレクトリ

| ファイル | 役割 |
|---------|------|
| `app/src/main/java/com/theveloper/pixelplay/presentation/jellyfin/auth/JellyfinLoginViewModel.kt` | ログイン |
| `app/src/main/java/com/theveloper/pixelplay/presentation/jellyfin/auth/JellyfinLoginActivity.kt` | UI |
| `app/src/main/java/com/theveloper/pixelplay/presentation/jellyfin/dashboard/JellyfinDashboardViewModel.kt` | ダッシュボード |
| `app/src/main/java/com/theveloper/pixelplay/presentation/jellyfin/dashboard/JellyfinDashboardScreen.kt` | UI |

### `JellyfinLoginViewModel`

| 注入 | `repository: JellyfinRepository` |
|------|------|

**StateFlow**: `state: StateFlow<JellyfinLoginState>`

**`JellyfinLoginState` (sealed interface)**:
- `Idle` (data object) / `Loading` (data object) / `Success(username)` (data class) / `Error(message)` (data class)

**主要 public 関数**:
- `fun login(serverUrl: String, username: String, password: String)` — 認証
- `fun clearError()` / `fun reset()` — Idle 復帰

### `JellyfinDashboardViewModel`

| 注入 | `repository: JellyfinRepository` |
|------|------|

**StateFlow**:
- `playlists: StateFlow<List<JellyfinPlaylistEntity>>`
- `isSyncing: StateFlow<Boolean>`
- `syncMessage: StateFlow<String?>`
- `selectedPlaylistSongs: StateFlow<List<Song>>`
- `isLoggedIn: StateFlow<Boolean>`

**プロパティ**:
- `username: String?` / `serverUrl: String?` (Repository から)

**主要 public 関数**:
- `fun syncAllPlaylistsAndSongs()` — 全プレイリスト + 全曲同期
- `fun syncPlaylists()` — プレイリスト一覧のみ同期
- `fun syncPlaylistSongs(playlistId: String)` — 単体プレイリスト楽曲同期
- `fun loadPlaylistSongs(playlistId: String)` — 詳細読込 (DB から)
- `fun deletePlaylist(playlistId: String)` — 削除
- `fun clearSyncMessage()` / `fun logout()`

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/jellyfin/JellyfinRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/database/JellyfinPlaylistEntity.kt`
- `../02-data-network/jellyfin.md` (推定)

---

## Navidrome

### ディレクトリ

| ファイル | 役割 |
|---------|------|
| `app/src/main/java/com/theveloper/pixelplay/presentation/navidrome/auth/NavidromeLoginViewModel.kt` | ログイン |
| `app/src/main/java/com/theveloper/pixelplay/presentation/navidrome/auth/NavidromeLoginActivity.kt` | UI |
| `app/src/main/java/com/theveloper/pixelplay/presentation/navidrome/dashboard/NavidromeDashboardViewModel.kt` | ダッシュボード |
| `app/src/main/java/com/theveloper/pixelplay/presentation/navidrome/dashboard/NavidromeDashboardScreen.kt` | UI |

### `NavidromeLoginViewModel`

`JellyfinLoginViewModel` と同一構造。`login(serverUrl, username, password)` → `Success(username)` / `Error(message)`。

### `NavidromeDashboardViewModel`

| 注入 | `repository: NavidromeRepository`, `workManager: WorkManager` |
|------|------|

**StateFlow**:
- `playlists: StateFlow<List<NavidromePlaylistEntity>>`
- `isSyncing: StateFlow<Boolean>`
- `syncProgress: StateFlow<Float?>` — `WorkInfo.progress.getFloat(NavidromeSyncWorker.PROGRESS_VALUE)`
- `syncMessage: StateFlow<String?>`
- `selectedPlaylistSongs: StateFlow<List<Song>>`
- `isLoggedIn: StateFlow<Boolean>`

**プロパティ**:
- `username: String?` / `serverUrl: String?` / `lastSyncTime: Long`

**主要 public 関数**:
- `private fun observeSyncWorker()` — WorkManager `getWorkInfosForUniqueWorkLiveData` を監視
- `fun syncAllPlaylistsAndSongs()` — `WorkManager.enqueueUniqueWork(WORK_NAME_SYNC_ALL, REPLACE, OneTimeWorkRequest)` で Worker 起動
- `fun syncPlaylistSongs(playlistId: String)`
- `fun loadPlaylistSongs(playlistId)`
- `fun deletePlaylist(playlistId)`
- `fun clearSyncMessage()` / `fun logout()`

**定数**:
- `WORK_NAME_SYNC_ALL = "navidrome_sync_all"` (private)

**内部実装**:
- **WorkManager 連携**: バックグラウンドで `NavidromeSyncWorker` が実行され、`setProgress` で進捗を通知
- **進捗取得**: `WorkInfo.progress.getFloat(NavidromeSyncWorker.PROGRESS_VALUE, 0f)` で 0-1 進捗

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/navidrome/NavidromeRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/database/NavidromePlaylistEntity.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/worker/NavidromeSyncWorker.kt`
- `../02-data-network/navidrome.md` (推定)

---

## Netease (网易云音乐)

### ディレクトリ

| ファイル | 役割 |
|---------|------|
| `app/src/main/java/com/theveloper/pixelplay/presentation/netease/auth/NeteaseLoginViewModel.kt` | ログイン |
| `app/src/main/java/com/theveloper/pixelplay/presentation/netease/auth/NeteaseLoginActivity.kt` | UI |
| `app/src/main/java/com/theveloper/pixelplay/presentation/netease/dashboard/NeteaseDashboardViewModel.kt` | ダッシュボード |
| `app/src/main/java/com/theveloper/pixelplay/presentation/netease/dashboard/NeteaseDashboardScreen.kt` | UI |

### `NeteaseLoginViewModel`

| 注入 | `repository: NeteaseRepository` |
|------|------|

**StateFlow**: `state: StateFlow<NeteaseLoginState>`

**`NeteaseLoginState` (sealed class)**:
- `Idle` / `Loading(message)` / `Success(nickname)` / `Error(message)`

**主要 public 関数**:
- `fun clearError()` — エラークリア
- `fun processCookies(cookieJson: String)` — Cookie JSON 文字列でログイン

**定数**:
- `COOKIE_LOGIN_TIMEOUT_MS = 25_000L` — `withTimeout` での Cookie ログインタイムアウト

**内部実装**:
- **Cookie ログイン**: 中国本土向けユーザー向け。`NeteaseRepository.loginWithCookies(cookieJson)` を `withTimeout(25秒)` で呼出。
- **`message` 付き Loading**: ユーザーに進捗を見せる ("Connecting to Netease..." 等)

### `NeteaseDashboardViewModel`

| 注入 | `repository: NeteaseRepository` |
|------|------|

**StateFlow**:
- `playlists: StateFlow<List<NeteasePlaylistEntity>>`
- `isSyncing: StateFlow<Boolean>`
- `syncMessage: StateFlow<String?>`
- `selectedPlaylistSongs: StateFlow<List<Song>>`
- `isLoggedIn: StateFlow<Boolean>`

**プロパティ**:
- `userNickname: String?` / `userAvatar: String?`

**主要 public 関数**:
- `fun syncAllPlaylistsAndSongs()`
- `fun syncPlaylists()` — `repository.syncUserPlaylists()` を呼び
- `fun syncPlaylistSongs(playlistId: Long)` — `playlistId` は `Long`
- `fun loadPlaylistSongs(playlistId: Long)`
- `fun deletePlaylist(playlistId: Long)`
- `fun clearSyncMessage()` / `fun logout()`

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/netease/NeteaseRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/database/NeteasePlaylistEntity.kt`
- `../02-data-network/netease.md` (推定)

---

## QQ Music (QQ音乐)

### ディレクトリ

| ファイル | 役割 |
|---------|------|
| `app/src/main/java/com/theveloper/pixelplay/presentation/qqmusic/auth/QqMusicLoginViewModel.kt` | ログイン |
| `app/src/main/java/com/theveloper/pixelplay/presentation/qqmusic/auth/QqMusicLoginActivity.kt` | UI |
| `app/src/main/java/com/theveloper/pixelplay/presentation/qqmusic/dashboard/QqMusicDashboardViewModel.kt` | ダッシュボード |
| `app/src/main/java/com/theveloper/pixelplay/presentation/qqmusic/dashboard/QqMusicDashboardScreen.kt` | UI |

### `QqMusicLoginViewModel`

`NeteaseLoginViewModel` と同一構造。`processCookies(cookieJson)` でログイン。

### `QqMusicDashboardViewModel`

| 注入 | `repository: QqMusicRepository` |
|------|------|

**StateFlow**:
- `isLoggedIn: StateFlow<Boolean>`
- `loggedOut: StateFlow<Boolean>` — ログアウト完了フラグ
- `playlists: StateFlow<List<QqMusicPlaylistEntity>>`
- `syncState: StateFlow<SyncState>`
- `showSyncTypeDialog: StateFlow<Boolean>` — 同期タイプ選択ダイアログ

**`SyncState` (sealed class)**:
- `Idle` / `Syncing` / `Success(message)` / `Error(message)`

**プロパティ**:
- `nickname: String?` / `avatarUrl: String?`

**主要 public 関数**:
- `fun showSyncTypeDialog()` / `fun hideSyncTypeDialog()`
- `fun syncAll(syncType: QqMusicRepository.PlaylistSyncType = ALL)` — `ALL` / `CREATED` / `COLLECTED` の 3 種
- `fun syncPlaylist(playlistId: Long)`
- `fun deletePlaylist(playlistId: Long)`
- `fun clearSyncState()` / `fun logout()`

**内部実装**:
- **同期タイプ選択**: ユーザーが「全部 / 自分が作成 / 自分が収集」を選ぶ 2 段階ダイアログ。
- **ログアウト後遷移**: `loggedOut = true` で親画面 (`AccountsScreen`) がログイン画面へ戻す。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/qqmusic/QqMusicRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/database/QqMusicPlaylistEntity.kt`
- `../02-data-network/qqmusic.md` (推定)

---

## Telegram

### ディレクトリ

| ファイル | 役割 |
|---------|------|
| `app/src/main/java/com/theveloper/pixelplay/presentation/telegram/auth/TelegramLoginViewModel.kt` | TDLib ログイン |
| `app/src/main/java/com/theveloper/pixelplay/presentation/telegram/auth/TelegramLoginActivity.kt` | UI |
| `app/src/main/java/com/theveloper/pixelplay/presentation/telegram/dashboard/TelegramDashboardViewModel.kt` | ダッシュボード |
| `app/src/main/java/com/theveloper/pixelplay/presentation/telegram/dashboard/TelegramDashboardScreen.kt` | UI |
| `app/src/main/java/com/theveloper/pixelplay/presentation/telegram/channel/TelegramChannelSearchViewModel.kt` | チャンネル検索 |
| `app/src/main/java/com/theveloper/pixelplay/presentation/telegram/channel/TelegramChannelSearchSheet.kt` | UI |
| `app/src/main/java/com/theveloper/pixelplay/presentation/telegram/channel/TelegramSongItem.kt` | 曲行 Composable |

### `TelegramLoginViewModel`

| 注入 | `telegramRepository: TelegramRepository`, `musicRepository: MusicRepository` |
|------|------|

**StateFlow**:
- `authorizationState: StateFlow<TdApi.AuthorizationState?>` — TDLib 認証状態
- `uiState: StateFlow<TelegramLoginUiState>`

**`TelegramLoginUiState` (data class)**:
- `phoneNumber` / `code` / `password` — 入力値
- `isLoading` / `loadingMessage`
- `inlineError: String?`
- `phoneEditMode: Boolean` — 電話番号編集モード

**SharedFlow**:
- `events: SharedFlow<String>` — エラー / 成功メッセージ
- `playbackRequest: SharedFlow<Song>` — ログインメニューから即時再生する曲

**主要 public 関数**:
- `fun onPhoneNumberChanged(number)` / `onCodeChanged(code)` / `onPasswordChanged(password)` — 入力更新
- `fun clearInlineError()` / `fun enablePhoneEditMode()`
- `fun handleBackNavigation(authState: TdApi.AuthorizationState?): Boolean` — Back キーで前状態へ
- `fun sendPhoneNumber()` / `fun checkCode()` / `fun checkPassword()` — 認証フロー
- `fun downloadAndPlay(song: Song)` — ログインメニューで曲を選んだ際の即時ダウンロード + 再生
- `fun clearData()` — 入力全クリア
- `private fun runAuthAction(action: suspend () -> Unit)` — Loading 状態管理ラッパ
- `private fun observeAuthorizationErrors()` / `observeAuthorizationState()` — TDLib イベント監視
- `private fun normalizePhoneNumber(raw: String): String` — `+` 接頭辞 + 数字のみ
- `private fun isValidPhoneNumber(value: String): Boolean` — 8 桁以上
- `private fun mapThrowableToMessage(error: Throwable): String` — エラー → ユーザー向け文字列
- `private fun mapTdLibError(code: Int, rawMessage: String?): String` — TDLib error code → 文字列

**内部実装**:
- **3 段階認証**: `phoneNumber` → `code` (SMS / Telegram アプリ) → `password` (2 段階認証)
- **TDLib**: `org.drinkless.tdlib.TdApi` を使用。`telegramRepository.authorizationState` を collect して UI を切替。
- **入力検証**: 電話番号は国際形式 (`+14155550123`)、Code は数字のみ、Password は空文字拒否。
- **イベントフロー**: TDLib からの error / state 通知を `observeAuthorizationErrors` / `observeAuthorizationState` で受け、UI へ伝搬。

### `TelegramDashboardViewModel`

| 注入 | `telegramRepository: TelegramRepository`, `musicRepository: MusicRepository`, `connectivityStateHolder: ConnectivityStateHolder` |
|------|------|

**StateFlow**:
- `isOnline: StateFlow<Boolean>` (Connectivity)
- `channels: StateFlow<List<TelegramChannelEntity>>` (DB キャッシュ)
- `topicsMap: StateFlow<Map<Long, List<TelegramTopicEntity>>>` (チャンネル → トピック)
- `isRefreshing: StateFlow<Long?>` — タイムスタンプ (Refresh 中識別)
- `statusMessage: StateFlow<String?>`
- `expandedChannels: StateFlow<Set<Long>>` — 展開中チャンネル

**主要 public 関数**:
- `fun toggleChannelExpanded(chatId: Long)` — チャンネル展開トグル
- `fun refreshChannel(channel: TelegramChannelEntity)` — 単体チャンネル再取得
- `private suspend fun syncFlatChannel(channel)` — フラットチャンネル同期
- `private suspend fun syncForumChannel(channel)` — Forum チャンネル (トピック別) 同期
- `fun removeChannel(chatId: Long)` — 削除
- `fun clearStatus()` / `fun refreshChannels()`

**内部実装**:
- **2 種類のチャンネル**: 
  - Flat: `telegramRepository.getAudioMessages(chatId)` で全曲取得
  - Forum: `telegramRepository.getForumTopics(chatId)` でトピック列挙 → 各トピックで `getAudioMessagesByTopic(chatId, threadId)` を実行
- **`isForum` 判定**: `telegramRepository.isForum(chatId)` で動的判別。
- **Refresh 識別**: `isRefreshing = System.currentTimeMillis()` を発行し、完了時に null へ。`expandedChannels` 内に当該 chatId があれば自動リフレッシュ。
- **オフラインガード**: `isOnline.value` が false なら API 呼び出しをスキップし「No internet」ステータスを表示。

### `TelegramChannelSearchViewModel`

| 注入 | `telegramRepository: TelegramRepository`, `musicRepository: MusicRepository` (推定) |
|------|------|

**StateFlow**:
- `isOnline: StateFlow<Boolean>` (Connectivity)
- `searchQuery: StateFlow<String>`
- `resolvedUsername: StateFlow<String?>` — 抽出されたユーザー名
- `foundChat: StateFlow<TdApi.Chat?>` — 検索結果
- `songs: StateFlow<List<Song>>` — チャンネル内全曲
- `isLoading: StateFlow<Boolean>`
- `statusMessage: StateFlow<String?>`

**SharedFlow**:
- `playbackRequest: SharedFlow<Song>` — 検索結果から再生

**主要 public 関数**:
- `private fun extractUsername(input: String): String` — URL / @username / 生テキスト対応
- `fun onQueryChanged(query: String)` — 検索クエリ更新
- `fun searchChannel()` — `telegramRepository.searchPublicChat(username)` 実行
- `private fun fetchSongs(chatId: Long)` — チャンネルの曲取得 (Forum なら Topic 別)
- `fun downloadAndPlay(song: Song)` — ダウンロードして即時再生
- `fun resetState()` — 全状態リセット

**内部実装**:
- **3 形式 URL 対応**: `https://t.me/<username>`, `@<username>`, `<username>` を正規表現で抽出。
- **Forum 対応**: `telegramRepository.isForum(chatId)` で Forum 判定し、`getForumTopics` → 各トピックの曲を追加。
- **合計曲数表示**: `totalSongs += topicSongs.size` で全トピック合計。

### 関連ファイル

- `app/src/main/java/com/theveloper/pixelplay/data/telegram/TelegramRepository.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/telegram/TelegramCacheManager.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/telegram/TdlibRequestException.kt`
- `app/src/main/java/com/theveloper/pixelplay/data/database/TelegramChannelEntity.kt` / `TelegramTopicEntity.kt`
- `../02-data-network/telegram.md` (推定)

---

## Provider Screen 一覧 (UI Composable への参照)

| Provider | Login Screen | Dashboard Screen |
|----------|-------------|------------------|
| GDrive | `GDriveLoginActivity` (Activity) | `GDriveDashboardScreen.kt` |
| Jellyfin | `JellyfinLoginActivity` (Activity) | `JellyfinDashboardScreen.kt` |
| Navidrome | `NavidromeLoginActivity` (Activity) | `NavidromeDashboardScreen.kt` |
| Netease | `NeteaseLoginActivity` (Activity) | `NeteaseDashboardScreen.kt` |
| QQ Music | `QqMusicLoginActivity` (Activity) | `QqMusicDashboardScreen.kt` |
| Telegram | `TelegramLoginActivity` (Activity) | `TelegramDashboardScreen.kt` + `TelegramChannelSearchSheet.kt` + `TelegramSongItem.kt` |

> Screen / Activity の詳細は `../05-presentation-ui/` 配下のドキュメントを参照。
