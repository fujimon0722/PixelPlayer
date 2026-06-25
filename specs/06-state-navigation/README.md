# 06 — 状態管理 / ViewModel / Navigation / Provider Dashboard

> PixelPlayer のプレゼンテーション層における「状態管理」仕様書。
> Hilt が提供する全 ViewModel / StateHolder / Provider 別 Dashboard ViewModel / Navigation 実装を網羅する。

## 全体アーキテクチャ

```
┌─────────────────────────── presentation 層 ───────────────────────────┐
│                                                                        │
│  Navigation (AppNavigation / Screen / Routes / Transitions)           │
│       │                                                                │
│       ▼                                                                │
│  PlayerViewModel ─── 唯一の「全機能オーケストレータ」 (HiltViewModel)    │
│       │                                                                │
│       ├─ 29 の StateHolder (@Singleton / @ViewModelScoped)             │
│       ├─ 18 のサブ ViewModel (画面単位)                                  │
│       ├─ 1 つの ColorSchemeProcessor (@Singleton)                      │
│       └─ SharedFlow / StateFlow を 各 Composable へ公開                │
│                                                                        │
│  Provider 別 Dashboard: gdrive / jellyfin / navidrome / netease /      │
│  qqmusic / telegram (各 auth ViewModel + dashboard ViewModel)           │
└────────────────────────────────────────────────────────────────────────┘
```

### ファイル分類

| カテゴリ | ファイル数 | 役割 |
|---------|-----------|------|
| コア StateHolder (Singleton) | 25+ | UI 非依存の状態保持。PlayerViewModel が注入して使う |
| 画面単位 ViewModel (HiltViewModel) | 20+ | `hiltViewModel()` で Compose に注入 |
| モデル・State データクラス | 8 | `PlayerUiState`, `StablePlayerState`, `ColorSchemePair`, ... |
| Navigation | 5 | `AppNavigation`, `Screen`, `MainRootRoutes`, `NavControllerExtensions`, `Transitions` |
| Provider Dashboard | 12 | 各 external service 専用の Login + Dashboard ViewModel |
| 補助 | 3 | `AppHaptics`, `GenreIconProvider`, `LibraryTabId` 等 |

## ファイル一覧 (本ディレクトリ)

| ファイル | 主な内容 |
|---------|---------|
| `README.md` | 本ファイル |
| `viewmodels-core.md` | `PlayerViewModel` (3118 行) / `MainViewModel` / `LibraryViewModel` |
| `viewmodels-playback.md` | `PlaybackStateHolder` (1133) / `PlaybackDispatchStateHolder` (1166) / `QueueStateHolder` / `SleepTimerStateHolder` / `LyricsStateHolder` / `TransitionViewModel` |
| `viewmodels-library.md` | `LibraryStateHolder` (615) / `LibraryTabsStateHolder` / `SearchStateHolder` / `AlbumDetailViewModel` / `ArtistDetailViewModel` (305) / `GenreDetailViewModel` (317) |
| `viewmodels-playlist-edit.md` | `PlaylistViewModel` (1228) / `PlaylistSelectionStateHolder` / `MetadataEditStateHolder` (901) / `SongInfoBottomSheetViewModel` (430) / `MultiSelectionStateHolder` (505) / `SongRemovalStateHolder` (378) / `QueueUndoStateHolder` / `PlaylistDismissUndoStateHolder` |
| `viewmodels-settings-stats.md` | `SettingsViewModel` (1490) / `SetupViewModel` (395) / `StatsViewModel` (196) / `EqualizerViewModel` (523) / `DailyMixStateHolder` / `ListeningStatsTracker` (335) |
| `viewmodels-ai-extra.md` | `AiStateHolder` (506) / `CastStateHolder` / `CastRouteStateHolder` / `CastTransferStateHolder` (1598) / `ExternalMediaStateHolder` (367) / `MediaControllerSyncStateHolder` (806) / `FileExplorerStateHolder` (593) / `FolderNavigationStateHolder` / `ConnectivityStateHolder` (592) / `ThemeStateHolder` / `AccountsViewModel` / `ArtistSettingsViewModel` / `DeviceCapabilitiesViewModel` (641) / `MashupViewModel` |
| `viewmodels-support.md` | `PlayerUiState` / `StablePlayerState` / `ColorSchemePair` / `ColorSchemeProcessor` (452) / `LyricsSearchUiState` / `PlayerSheetState` / `exts/DeckController` / `PlayerViewModel` 内部 data class |
| `navigation.md` | `AppNavigation` (573) / `Screen` / `MainRootRoutes` / `NavControllerExtensions` / `Transitions` |
| `provider-screens.md` | gdrive / jellyfin / navidrome / netease / qqmusic / telegram の Login + Dashboard ViewModel + Screen |

### サポート型 (別 spec)

| ファイル | 内容 |
|---------|------|
| `../05-presentation-ui/library-model-stats-utils.md` (将来) | `LibraryTabId` / `RecentlyPlayedSongUi` / `SettingsCategory` / `StatsTimeRangeUi` / `AppHaptics` / `GenreIconProvider` |

## StateHolder 依存関係図 (Mermaid)

```mermaid
graph TD
    PlayerVM[PlayerViewModel<br/>@HiltViewModel] --> PlaybackSH[PlaybackStateHolder<br/>@Singleton]
    PlayerVM --> PlaybackDispatch[PlaybackDispatchStateHolder<br/>@ViewModelScoped]
    PlayerVM --> QueueSH[QueueStateHolder<br/>@Singleton]
    PlayerVM --> LibrarySH[LibraryStateHolder<br/>@Singleton]
    PlayerVM --> SearchSH[SearchStateHolder<br/>@Singleton]
    PlayerVM --> LyricsSH[LyricsStateHolder<br/>@Singleton]
    PlayerVM --> SleepSH[SleepTimerStateHolder<br/>@Singleton]
    PlayerVM --> CastSH[CastStateHolder<br/>@Singleton]
    PlayerVM --> CastTransfer[CastTransferStateHolder<br/>@Singleton]
    PlayerVM --> CastRoute[CastRouteStateHolder<br/>@ViewModelScoped]
    PlayerVM --> MediaCtrl[MediaControllerSyncStateHolder<br/>@ViewModelScoped]
    PlayerVM --> ExtMedia[ExternalMediaStateHolder<br/>@Singleton]
    PlayerVM --> FolderNav[FolderNavigationStateHolder<br/>@ViewModelScoped]
    PlayerVM --> LibraryTabs[LibraryTabsStateHolder<br/>@ViewModelScoped]
    PlayerVM --> DailyMix[DailyMixStateHolder<br/>@Singleton]
    PlayerVM --> AiSH[AiStateHolder<br/>@Singleton]
    PlayerVM --> ThemeSH[ThemeStateHolder<br/>@Singleton]
    PlayerVM --> Connectivity[ConnectivityStateHolder<br/>@Singleton]
    PlayerVM --> MultiSel[MultiSelectionStateHolder<br/>@Singleton]
    PlayerVM --> PlaylistSel[PlaylistSelectionStateHolder<br/>@Singleton]
    PlayerVM --> QueueUndo[QueueUndoStateHolder<br/>@ViewModelScoped]
    PlayerVM --> PlaylistDismiss[PlaylistDismissUndoStateHolder<br/>@ViewModelScoped]
    PlayerVM --> MetadataEdit[MetadataEditStateHolder<br/>@ViewModelScoped]
    PlayerVM --> SongRemoval[SongRemovalStateHolder<br/>@ViewModelScoped]
    PlayerVM --> ThemeRepo[ThemePreferencesRepository]
    PlayerVM --> AiRepo[AiPreferencesRepository]
    PlayerVM --> UserPref[UserPreferencesRepository]
    PlayerVM --> MusicRepo[MusicRepository]
    PlayerVM --> DailyMixMgr[DailyMixManager]
    PlayerVM --> SyncMgr[SyncManager]
    PlayerVM --> DualEngine[DualPlayerEngine]
    PlayerVM --> Listening[ListeningStatsTracker]
    PlayerVM --> CastSession[SessionToken]
    PlayerVM --> MccFactory[MediaControllerFactory]

    %% 内部依存
    PlaybackDispatch --> PlaybackSH
    PlaybackDispatch --> CastSH
    PlaybackDispatch --> CastTransfer
    PlaybackDispatch --> QueueSH
    PlaybackDispatch --> LibrarySH
    PlaybackDispatch --> Connectivity
    PlaybackDispatch --> ExtMedia
    PlaybackDispatch --> ThemeSH
    PlaybackDispatch --> DualEngine
    PlaybackDispatch --> SyncMgr

    MediaCtrl --> PlaybackSH
    MediaCtrl --> CastSH
    MediaCtrl --> LibrarySH
    MediaCtrl --> Connectivity
    MediaCtrl --> LyricsSH
    MediaCtrl --> SleepSH
    MediaCtrl --> PlaybackDispatch
    MediaCtrl --> ThemeSH
    MediaCtrl --> DualEngine
    MediaCtrl --> MusicRepo

    CastTransfer --> CastSH
    CastTransfer --> PlaybackSH
    CastTransfer --> DualEngine
    CastRoute --> CastSH
    CastRoute --> CastTransfer
    SongRemoval --> MetadataEdit
    SongRemoval --> LibrarySH
    SongRemoval --> PlaybackSH
    SongRemoval --> MultiSel
    MetadataEdit --> PlaybackSH
    MetadataEdit --> LibrarySH
    MetadataEdit --> ThemeSH
    MetadataEdit --> MultiSel
    LyricsSH --> SongMetadataEditor
    SleepSH --> EotStateHolder
    ThemeSH --> ColorSchemeProcessor
    ThemeSH --> ThemeRepo
```

## 命名規約

- **`@Singleton`** — DI グラフでアプリ全体 1 インスタンス。MediaController や SharedFlow を持たない純粋な状態保持 / 計算担当。
- **`@ViewModelScoped`** — `PlayerViewModel` のライフサイクル (= Composable Navigation の backstack entry) に紐付く。MediaController / 進行中 Job を保持する。
- **`@HiltViewModel`** — Compose の `hiltViewModel()` で取得する画面単位 ViewModel。`SavedStateHandle` 経由で NavArg を受け取る。
- **`PlayerViewModel`** は 29 の StateHolder を内包する「ファサード」。

## 関連スペック

- データレイヤー Repository / Preferences: `../03-data-services/repositories.md`
- 再生エンジン (`DualPlayerEngine`, `MusicService`): `../04-engine/music-service.md`
- 画面 Composable: `../05-presentation-ui/`
- カラースキーム・Material テーマ: `../07-ui-system/theme.md`

## ホットスポット (最重要 StateHolder)

> 行数が多く、依存が複雑な「要注目」StateHolder の早見表。

| StateHolder | 行数 | DI スコープ | 複雑度 | 重要度 | 解説ファイル |
|-------------|------|------------|--------|--------|------------|
| `PlayerViewModel` | 3118 | `@HiltViewModel` | ★★★★★ | ★★★★★ | `viewmodels-core.md` |
| `CastTransferStateHolder` | 1598 | `@Singleton` | ★★★★★ | ★★★★★ | `viewmodels-ai-extra.md` |
| `SettingsViewModel` | 1490 | `@HiltViewModel` | ★★★★ | ★★★★★ | `viewmodels-settings-stats.md` |
| `PlaylistViewModel` | 1228 | `@HiltViewModel` | ★★★★ | ★★★★ | `viewmodels-playlist-edit.md` |
| `PlaybackDispatchStateHolder` | 1166 | `@ViewModelScoped` | ★★★★★ | ★★★★★ | `viewmodels-playback.md` |
| `PlaybackStateHolder` | 1133 | `@Singleton` | ★★★★ | ★★★★★ | `viewmodels-playback.md` |
| `MetadataEditStateHolder` | 901 | `@ViewModelScoped` | ★★★ | ★★★★ | `viewmodels-playlist-edit.md` |
| `AppNavigation` | 573 | n/a (Composable) | ★★★ | ★★★★ | `navigation.md` |
| `DeviceCapabilitiesViewModel` | 641 | `@HiltViewModel` | ★★★ | ★★ | `viewmodels-ai-extra.md` |
| `MediaControllerSyncStateHolder` | 806 | `@ViewModelScoped` | ★★★★ | ★★★★ | `viewmodels-ai-extra.md` |
| `LibraryStateHolder` | 615 | `@Singleton` | ★★★ | ★★★★ | `viewmodels-library.md` |
| `ConnectivityStateHolder` | 592 | `@Singleton` | ★★★ | ★★★ | `viewmodels-ai-extra.md` |
| `FileExplorerStateHolder` | 593 | `@Inject` (scope) | ★★★ | ★★★ | `viewmodels-ai-extra.md` |
| `ThemeStateHolder` | 319 | `@Singleton` | ★★★ | ★★★ | `viewmodels-ai-extra.md` |
| `ColorSchemeProcessor` | 452 | `@Singleton` | ★★★ | ★★★ | `viewmodels-support.md` |
| `MultiSelectionStateHolder` | 505 | `@Singleton` | ★★★ | ★★★ | `viewmodels-playlist-edit.md` |
| `SongInfoBottomSheetViewModel` | 430 | `@HiltViewModel` | ★★★ | ★★ | `viewmodels-playlist-edit.md` |
| `SetupViewModel` | 395 | `@HiltViewModel` | ★★★ | ★★★ | `viewmodels-settings-stats.md` |
| `SongRemovalStateHolder` | 378 | `@ViewModelScoped` | ★★★ | ★★ | `viewmodels-playlist-edit.md` |
| `EqualizerViewModel` | 523 | `@HiltViewModel` | ★★★ | ★★ | `viewmodels-settings-stats.md` |
| `ExternalMediaStateHolder` | 367 | `@Singleton` | ★★ | ★★ | `viewmodels-ai-extra.md` |
| `GenreDetailViewModel` | 317 | `@HiltViewModel` | ★★★ | ★★ | `viewmodels-library.md` |
| `ArtistDetailViewModel` | 305 | `@HiltViewModel` | ★★★ | ★★ | `viewmodels-library.md` |
| `ListeningStatsTracker` | 335 | `@Singleton` | ★★★ | ★★ | `viewmodels-settings-stats.md` |
| `SearchStateHolder` | 206 | `@Singleton` | ★★ | ★★ | `viewmodels-library.md` |
| `QueueStateHolder` | 289 | `@Singleton` | ★★ | ★★★ | `viewmodels-playback.md` |
| `LyricsStateHolder` | 509 | `@Singleton` | ★★★ | ★★★ | `viewmodels-playback.md` |
| `SleepTimerStateHolder` | 309 | `@Singleton` | ★★ | ★★ | `viewmodels-playback.md` |
| `AiStateHolder` | 506 | `@Singleton` | ★★★ | ★★★ | `viewmodels-ai-extra.md` |
| `CastStateHolder` | 300 | `@Singleton` | ★★★ | ★★★ | `viewmodels-ai-extra.md` |
| `SettingsViewModel` | 1490 | `@HiltViewModel` | ★★★★ | ★★★★★ | `viewmodels-settings-stats.md` |
| `AccountsViewModel` | 278 | `@HiltViewModel` | ★★ | ★★ | `viewmodels-ai-extra.md` |

## StateFlow / SharedFlow 命名パターン

| パターン | 例 | 説明 |
|---------|----|------|
| `_xxx` | `_playerUiState` | private な `MutableStateFlow` |
| `xxx` | `playerUiState` | public な `StateFlow` (read-only) |
| `_xxxEvents` | `_toastEvents` | private な `MutableSharedFlow` |
| `xxxEvents` | `toastEvents` | public な `SharedFlow` |
| `xxxFlow` | `songsPagingFlow` | `Flow` 公開 (Paging 等) |
| `combine` 結果 | `fullPlayerSlice` | 複数 Flow の合成結果 |

## ViewModel 間イベント集約パターン

`PlayerViewModel` は StateHolder の `initialize(scope, callbacks)` パターンで callback 群を注入し、内部で StateHolder 間を調整する。

```kotlin
// 典型的パターン
class PlayerViewModel {
    fun init {
        playbackStateHolder.initialize(viewModelScope, PlaybackStateHolderCallbacks(
            onShuffleChanged = { /* ... */ },
            onRepeatChanged = { /* ... */ },
            ...
        ))
        playbackDispatchStateHolder.initialize(PlaybackDispatchCallbacks(
            getController = { mediaController },
            getUiState = { _playerUiState.value },
            updateUiState = { transform -> _playerUiState.update(transform) },
            ...
        ))
    }
}
```

## Coroutine 起動コンテキスト

| Context | 用途 |
|---------|------|
| `viewModelScope` (PlayerViewModel) | トップレベル ViewModel のデフォルト |
| `StateHolder.scope` (PlayerViewModel から注入) | 進行中 Job のキャンセル可能化 |
| `persistenceScope` (ListeningStatsTracker) | `SupervisorJob() + Dispatchers.IO` — 永続化専用 |
| `Dispatchers.Default` | CPU バウンド処理 (ソート / 検索) |
| `Dispatchers.IO` | ファイル / ネットワーク I/O |
| `Dispatchers.Main` (implicit) | StateFlow / SharedFlow 発行 |

## 既知の落とし穴

1. **`PlayerViewModel` の循環依存**: 一部の StateHolder は `PlayerViewModel` の構築時に完全な依存を注入できない (e.g. `AiStateHolder` は `allSongsProvider` 等の遅延 callback を持つ)。`initialize` パターンで解決。
2. **CastTransferStateHolder の 2 系統状態管理**: `castStateHolder` 側 (`lastRemoteQueue` / `lastRemoteSongId`) と `castTransferStateHolder` 側 (`lastRemoteQueue` / `lastRemoteStreamPosition`) で重複管理しているが、責務が違う (前者は表示、後者は transfer back 用)。
3. **PlaylistViewModel の Folder pseudo playlist**: `folder_playlist:` プレフィックスで通常 Playlist と区別するが、内部で `MusicFolder.collectAllSongs()` を BFS で実行するため、大量フォルダで遅延の可能性。
4. **MetadataEditStateHolder の Android 11+ 制約**: `MediaStore.createWriteRequest` が必須。古い OS では 1 件ずつ処理。
5. **ColorSchemeProcessor のキャッシュキー衝突**: `CACHE_ALGORITHM_VERSION` を上げると全キャッシュが無効化される。`"algo_v7"` は `Color` → `StoredColorSchemeValues` のシリアライズ形式変更に対応。

## テスト戦略のヒント

| テスト対象 | 推奨手法 |
|----------|---------|
| `PlaybackStateHolder` | `runTest { ... }` + `TestScope` で position 解決ロジック |
| `QueueStateHolder` | シャッフル結果のテスト (Fisher-Yates) |
| `SearchStateHolder` | `debounce` の `TestScope.advanceTimeBy()` で検証 |
| `CastTransferStateHolder` | 状態遷移の状態機械テスト |
| `LibraryStateHolder` | Paging フローの Snapshot |
| `PlayerViewModel` | 統合テスト (DI 全体) — 重いため避けたい |
| `ColorSchemeProcessor` | `Robolectric` で Bitmap 生成 |

## 既知の TODO / 推定 (要確認)

- 以下のファイルは `outline` ベースで作成しており、本文を `read` していないため、推測を含む部分がある:
  - `MultiSelectionStateHolder.kt` — 一部 API (liked / zip share) は推測
  - `SongRemovalStateHolder.kt` — `CompletableDeferred` を使った suspend ダイアログパターンは推定
  - `MashupViewModel.kt` — `DeckController` の `getProgress` 等の内部実装は推定
  - `ColorSchemeProcessor.kt` — `AlbumArtPaletteStyle.CUSTOM` の挙動は推定
- 本 spec のソースコード参照は outline ベースのものが大半。実コードと乖離している場合は spec を更新する必要あり。

