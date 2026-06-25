# Wear アプリエントリポイント — `WearApp` / `AndroidManifest`

## WearApp.kt

**パッケージ**: `com.theveloper.pixelplay`
**役割**: Hilt ベースの `Application`。`BuildConfig.DEBUG` のときだけ `Timber.DebugTree()` を植林。

### 公開 API

| 名前 | 種類 | 説明 |
|------|------|------|
| `WearApp` | class : `Application` (`@HiltAndroidApp`) | アプリの Application クラス |
| `onCreate()` | `override fun` | スーパ呼び出し後、`BuildConfig.DEBUG` 時のみ `Timber.plant(Timber.DebugTree())` |

### 内部実装メモ

- サイズは 15 行の極小クラス。
- 他のロジック (Hilt モジュール初期化、起動時のサービス bind 等) は DI グラフに委譲。
- `Timber` 未使用時の本番ログは出力されない。

### 関連ファイル

- `wear/src/main/AndroidManifest.xml` — `<application android:name=".WearApp">`
- `wear/src/main/java/com/theveloper/pixelplay/di/WearModule.kt` — Hilt

---

## AndroidManifest.xml

**パッケージ**: なし (XML)
**役割**: Wear アプリの権限 / メタデータ / Activity / Service 宣言。

### 主要要素

#### 権限 (`<uses-permission>`)

| 権限 | 説明 |
|------|------|
| `android.permission.READ_MEDIA_AUDIO` | watch 内 MediaStore 音声スキャン (API 33+) |
| `android.permission.READ_EXTERNAL_STORAGE` (`maxSdkVersion=32`) | 旧 API の MediaStore アクセス |
| `android.permission.FOREGROUND_SERVICE` | foreground service 起動 (Wear OS 必須) |
| `android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK` | API 34+ で `mediaPlayback` タイプ指定 |
| `android.permission.POST_NOTIFICATIONS` | メディア通知表示 (API 33+) |
| `android.permission.WAKE_LOCK` | ExoPlayer の wake mode 用 |

#### `<uses-feature>`

| Feature | 説明 |
|---------|------|
| `android.hardware.type.watch` | Wear 専用ビルド |

#### `<application>`

- `android:name=".WearApp"`
- `android:allowBackup="true"`
- `android:icon="@mipmap/ic_launcher"`
- `android:label="@string/app_name"`
- `android:theme="@style/Theme.PixelPlayWear"`

#### `<uses-library>`

- `com.google.android.wearable` (required) — Wearable サポート API

#### メタデータ

- `com.google.android.wearable.standalone = true` — **standalone アプリ** (phone 不要で動作)
- `com.google.android.wearable.notificationBridgeMode = NO_BRIDGING` — 通知は bridge しない

#### `<activity>` — `WearMainActivity`

- `android:name=".presentation.WearMainActivity"`
- `android:exported="true"`
- `android:launchMode="singleTask"`
- `android:taskAffinity=""`
- `android:theme="@style/Theme.PixelPlayWear"`
- intent-filter:
  - `android.intent.action.MAIN` + `LAUNCHER` (ランチャー)
  - `com.google.android.wearable.action.MEDIA_CONTROLS` (システムメディアコントロール)
  - `com.theveloper.pixelplay.action.OPEN_PLAYER` (phone からの自動起動)

#### `<service>` — `WearPlaybackService`

- `android:name=".data.WearPlaybackService"`
- `android:exported="true"`
- `android:foregroundServiceType="mediaPlayback"`
- intent-filter:
  - `androidx.media3.session.MediaSessionService` (Media3 セッション)
  - `android.intent.action.MEDIA_BUTTON` (Bluetooth ヘッドセット等のメディアボタン)

#### `<service>` — `WearDataListenerService`

- `android:name=".data.WearDataListenerService"`
- `android:exported="true"`
- intent-filter (data scheme = `wear`):
  - `com.google.android.gms.wearable.DATA_CHANGED` — `/player_state`
  - `com.google.android.gms.wearable.MESSAGE_RECEIVED` — `/browse_response`, `/playback_result`, `/transfer_metadata`, `/transfer_progress`, `/transfer_request`, `/transfer_cancel`, `/watch_library_query`, `/volume_state`
  - `com.google.android.gms.wearable.CHANNEL_EVENT` — `/transfer_audio`, `/transfer_artwork`

### 内部実装メモ

- standalone = true により watch は phone の companion としてではなく独立して動作。
- `WearPlaybackService` を `mediaPlayback` フォアグラウンドで運用することで、wear OS のプロセス reaping を回避しバックグラウンド再生を持続。
- `WearDataListenerService` は `<service>` として登録され、`WearableListenerService` ベース。OS によって必要に応じて自動生成。

### 関連ファイル

- `wear/src/main/java/com/theveloper/pixelplay/WearApp.kt`
- `wear/src/main/java/com/theveloper/pixelplay/data/WearPlaybackService.kt`
- `wear/src/main/java/com/theveloper/pixelplay/data/WearDataListenerService.kt`
- `wear/src/main/java/com/theveloper/pixelplay/presentation/WearMainActivity.kt`
