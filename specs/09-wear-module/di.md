# wear/di — Hilt モジュール

## WearModule.kt

**パッケージ**: `com.theveloper.pixelplay.di`
**役割**: Wear アプリ用の Hilt 依存提供。Wearable クライアント (Data/Message/Node/Channel) と Room データベースを Singleton として公開。

### 依存関係

- **上流**: `WearApp` (HiltAndroidApp)
- **下流**: `androidx.room.Room`, `com.google.android.gms.wearable.Wearable`

### 公開 API (object @Module @InstallIn(SingletonComponent::class))

| 関数 | 戻り値 | 目的 | 依存 |
|------|--------|------|------|
| `provideDataClient(application: Application)` | `DataClient` | `Wearable.getDataClient(application)` を Singleton 提供 | 受信: `WearDataListenerService` |
| `provideMessageClient(application: Application)` | `MessageClient` | `Wearable.getMessageClient(application)` | 送信: 全 Repository, 受信: `WearDataListenerService` |
| `provideNodeClient(application: Application)` | `NodeClient` | `Wearable.getNodeClient(application)` | 送信: `WearPlaybackController`, `WearLibraryRepository`, `WearTransferRepository`, `WearFavoriteSyncRepository` |
| `provideChannelClient(application: Application)` | `ChannelClient` | `Wearable.getChannelClient(application)` | 受信: `WearTransferRepository` |
| `provideWearMusicDatabase(application: Application)` | `WearMusicDatabase` | `Room.databaseBuilder(...).build()` (DB名 `wear_music.db`) | `LocalSongDao` |
| `provideLocalSongDao(database: WearMusicDatabase)` | `LocalSongDao` | DAO 提供 | Repository 群 |

### 内部実装メモ

- すべて `@Provides @Singleton` で application lifecycle 維持。
- 他の Repository クラス (`@Singleton` + `@Inject constructor`) は Hilt 自動コンストラクタ。
- データベース名は `"wear_music.db"`。

### 呼び出し元 (依存するクラス)

- `WearPlaybackController` — MessageClient, NodeClient
- `WearLibraryRepository` — 内部で `Wearable.getMessageClient(application)` も使う (DI 経由でなく遅延取得)
- `WearTransferRepository` — MessageClient, NodeClient, ChannelClient
- `WearFavoriteSyncRepository` — MessageClient, NodeClient
- `WearLocalPlayerRepository` — Application, LocalSongDao
- `WearDataListenerService` — lateinit var で Repository 注入 (要確認)

### 関連ファイル

- `wear/src/main/java/com/theveloper/pixelplay/WearApp.kt` — `@HiltAndroidApp`
- `wear/src/main/java/com/theveloper/pixelplay/data/local/WearMusicDatabase.kt`
