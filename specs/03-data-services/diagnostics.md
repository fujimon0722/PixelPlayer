# Diagnostics

パフォーマンス計測 / フレームストール監視 / デバッグレポート生成。

## パッケージ

`com.theveloper.pixelplay.data.diagnostics`

---

## 依存関係

### 上流
- `presentation/screens/settings/PerformanceReportScreen.kt` — ユーザーがレポート生成ボタン
- `presentation/screens/settings/SettingsScreen.kt` — 詳細診断の有効化トグル
- `service/MusicService.kt`, `service/player/DualPlayerEngine.kt` — 計測値を `PerformanceMetrics` に記録
- `data/equalizer/EqualizerManager.kt` — `attachToAudioSession` のタイム計測
- `data/worker/SyncWorker.kt` — `sync_worker_*` イベント記録
- `data/repository/ArtistImageRepository.kt` — `artist_image_prefetch_*` イベント
- `data/worker/NavidromeSyncWorker.kt` — `navidrome_sync_*` イベント

### 下流
- `data/database/MusicDao.kt` (`getLibraryAudioStats`, `getMimeTypeCounts`)
- `data/preferences/UserPreferencesRepository.kt`, `EqualizerPreferencesRepository.kt`
- `service/player/DualPlayerEngine.kt`
- `utils/AudioMetaUtils.kt`

---

## ファイル一覧

| ファイル | 行 | 役割 |
|---|---|---|
| `AdvancedPerformanceDiagnostics.kt` | 238 | opt-in のイベントレコーダ（120 件まで） |
| `AdvancedPerformanceDiagnosticsController.kt` | 50 | 設定値と内部セッションを橋渡し |
| `MainThreadStallMonitor.kt` | 50 | Choreographer ベースのフレームギャップ監視 |
| `PerformanceMetrics.kt` | 275 | 軽量・常時稼働の集計メトリクス（timings / counters / maxes / offload events） |
| `DebugPerformanceReport.kt` | 352 | JSON / テキスト形式のレポートモデル |
| `DebugPerformanceReportCollector.kt` | 277 | 上記の組み立て |

---

## `AdvancedPerformanceDiagnostics` (`AdvancedPerformanceDiagnostics.kt:12`)

`object` (シングルトン)。`opt-in` で 24 時間のセッションを記録。

### 定数

| 定数 | 値 | 用途 |
|---|---|---|
| `MAX_EVENTS` | 120 | 保持する最大イベント数 |
| `DEFAULT_SESSION_DURATION_MS` | `24 * 60 * 60 * 1000` (24h) | デフォルト有効期間 |
| `FRAME_STALL_THRESHOLD_MS` | `100` | Choreographer のフレームギャップ閾値 |
| `MAX_TRACE_SECTION_CHARS` | 120 | Trace 名の最大長 |
| `MAX_FIELD_CHARS` | 80 | type/name の最大長 |
| `MAX_DETAIL_VALUE_CHARS` | 240 | detail 値の最大長 |
| `MAX_DETAIL_ENTRIES` | 8 | detail の最大エントリ数 |

### `EventTypes` (`AdvancedPerformanceDiagnostics.kt:17`)

| 定数 | 値 |
|---|---|
| `USER_MARK` | `"user_mark"` |
| `FRAME_STALL` | `"frame_stall"` |
| `PLAYBACK` | `"playback"` |
| `AUDIO_EFFECT` | `"audio_effect"` |
| `OFFLOAD` | `"offload"` |
| `WORKER` | `"worker"` |
| `ARTWORK` | `"artwork"` |
| `UI` | `"ui"` |

### `DiagnosticEvent` (`AdvancedPerformanceDiagnostics.kt:28`)

`elapsedRealtimeMs`, `type`, `name`, `details: Map<String, String>`。

### `Snapshot` (`AdvancedPerformanceDiagnostics.kt:35`)

`enabled`, `sessionStartedEpochMs`, `expiresAtEpochMs`, `droppedEventCount`, `events`。

### 公開 API

| API | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `isEnabled` | 57 | `Boolean` (volatile) | 軽量チェック（無効時は即 return） |
| `startSession(startedAtEpochMs, durationMs)` | 60 | `Unit` | イベントをクリアしてセッション開始 |
| `configureSession(enabled, startedAtEpochMs, expiresAtEpochMs, nowEpochMs)` | 74 | `Boolean` (active) | 設定値から内部セッションを構成 |
| `stopSession()` | 103 | `Unit` | 完全停止 |
| `recordEvent(type, name, details, elapsedRealtimeMs, nowEpochMs)` | 109 | `Unit` | イベント記録（無効時は何もしない） |
| `recordEventIfEnabled(type, name, elapsedRealtimeMs, details)` (inline) | 137 | `Unit` | `if (isEnabled) recordEvent(...)` のヘルパ。`details` はラムダで遅延評価 |
| `markLagNow(note, elapsedRealtimeMs, nowEpochMs)` | 156 | `Unit` | USER_MARK を即時記録（手動マーカー） |
| `trace(sectionName, block)` | 173 | `T` | `Trace.beginSection/endSection` のラッパ |
| `traceSuspend(sectionName, block)` | 183 | `T` | suspend 版 |
| `snapshot(nowEpochMs)` | 193 | `Snapshot` | 現在のスナップショットをコピー |
| `resetForTest()` | 208 | `Unit` | テスト専用リセット |

### 内部実装メモ

- `lock` (`AdvancedPerformanceDiagnostics.kt:43`) で `events`, `droppedEventCount`, `sessionStartedEpochMs`, `expiresAtEpochMs`, `enabled` を保護。
- `MAX_EVENTS` を超えると `events.pollFirst()` で先頭を除去し、`droppedEventCount += 1`。
- `sanitizeDetails()` (`AdvancedPerformanceDiagnostics.kt:225`) で `key` / `value` を `take(MAX_FIELD_CHARS)` / `take(MAX_DETAIL_VALUE_CHARS)` にカット。
- `isActiveLocked(nowEpochMs)` (`AdvancedPerformanceDiagnostics.kt:214`) で `enabled && expiresAt > now` を判定し、期限切れなら自動 `clearLocked()`。

---

## `AdvancedPerformanceDiagnosticsController` (`AdvancedPerformanceDiagnosticsController.kt:14`)

`@Singleton` / `@Inject` で `UserPreferencesRepository` を受ける。

| メソッド / フィールド | 行 | 目的 |
|---|---|---|
| `stallMonitor` | 18 | `MainThreadStallMonitor()` インスタンス |
| `observerJob` | 19 | 設定購読コルーチン |
| `expiryJob` | 20 | 期限切れタイマー |
| `start(scope)` | 22 | `disableExpiredAdvancedPerformanceDiagnostics()` 実行 → 設定購読 → `expiryJob` 設定 → メインスレッドで stall 監視 ON/OFF |

`expiryJob` (`AdvancedPerformanceDiagnosticsController.kt:33`) は `delay(expiresAt - now)` して `disableExpiredAdvancedPerformanceDiagnostics()` を呼ぶ。

---

## `MainThreadStallMonitor` (`MainThreadStallMonitor.kt:11`)

| メソッド / フィールド | 行 | 目的 |
|---|---|---|
| `thresholdMs` | 12 | 既定 `100` |
| `running`, `lastFrameNanos` | 15-16 | 状態 |
| `callback` (`Choreographer.FrameCallback`) | 17-35 | `frameTimeNanos` の差が `thresholdMs` を超えると `FRAME_STALL` イベント記録 |
| `start()` | 37 | `Choreographer.getInstance().postFrameCallback(callback)` |
| `stop()` | 44 | `running=false`, `removeFrameCallback` |

メインスレッド confind なので `start/stop` はメインで呼ぶこと（コントローラ側で `withContext(Dispatchers.Main.immediate)` を使用）。

---

## `PerformanceMetrics` (`PerformanceMetrics.kt:26`)

`object` (シングルトン)。**常時稼働・安価・静的呼び出し可能**。

### `Timings` (`PerformanceMetrics.kt:29`)

| 定数 | 用途 |
|---|---|
| `FULL_SCAN` | フルスキャン全体 |
| `METADATA_READ` | タグ読み取り |
| `ARTWORK_EXTRACT` | 埋め込みアート抽出 |
| `ARTWORK_DECODE` | アートデコード |
| `PLAYBACK_PREPARE` | 再生準備 |
| `AUDIO_DECODER_INIT` | デコーダ初期化 |
| `TRANSITION` | トランジション |
| `WIDGET_UPDATE` | ウィジェット更新 |
| `MEDIASESSION_ITEM_BUILD` | MediaSession アイテム構築 |

### `Counters` (`PerformanceMetrics.kt:41`)

`SCAN_RUNS`, `SONGS_SCANNED`, `METADATA_FALLBACK_JAUDIOTAGGER`, `ARTWORK_CACHE_HIT`, `ARTWORK_CACHE_MISS`, `ARTWORK_EXTRACTED_FRESH`, `ARTWORK_LARGE`, `OFFLOAD_FALLBACKS`, `MULTICHANNEL_PLAYBACKS`。

### `Maxes` (`PerformanceMetrics.kt:53`)

`ARTWORK_BYTES`, `DECODED_ARTWORK_WIDTH`, `DECODED_ARTWORK_HEIGHT`, `PLAYBACK_CHANNEL_COUNT`, `PLAYBACK_SAMPLE_RATE`, `PLAYBACK_PCM_ENCODING`。

`LARGE_ARTWORK_BYTES = 1_000_000L` (`PerformanceMetrics.kt:63`) で `ARTWORK_LARGE` カウンタを更新。

### `TimingStat` (`PerformanceMetrics.kt:68`)

`count`, `sumMs`, `minMs`, `maxMs`, `lastMs` を synchronized で記録。`snapshot()` で `TimingSnapshot` を返す。

### 公開 API

| API | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `recordTiming(name, durationMs)` | 148 | `Unit` | タイミング記録。`name in advancedTimelineTimingNames` なら `AdvancedPerformanceDiagnostics.recordEvent` も発行 |
| `time(name, block)` (inline) | 168 | `T` | block の wall time を `recordTiming` で記録 |
| `increment(name, delta=1)` | 177 | `Unit` | カウンタを +delta |
| `recordMax(name, value)` | 182 | `Unit` | CAS で最大値を更新 |
| `recordEmbeddedArtwork(bytes)` | 192 | `Unit` | `recordMax(ARTWORK_BYTES)` + `>= LARGE_ARTWORK_BYTES` なら `increment(ARTWORK_LARGE)` |
| `recordDecodedArtworkDimensions(width, height)` | 203 | `Unit` | 幅 / 高さの max を更新 |
| `recordPlaybackFormat(channelCount, sampleRate, pcmEncoding)` | 209 | `Unit` | playback 系の max + `channelCount > 2` で `increment(MULTICHANNEL_PLAYBACKS)` |
| `recordOffloadFallback(reason, elapsedRealtimeMs)` | 216 | `Unit` | `OFFLOAD_FALLBACKS++` + `AdvancedPerformanceDiagnostics.OFFLOAD` イベント + リングバッファ (max 20 件) に追加 |
| `recordControllerConnected(packageName, isAndroidAuto, isWear, elapsedRealtimeMs)` | 231 | `Unit` | `controllers` Map に追加（重複は無視） |
| `setWidgetActive(active)` | 243 | `Unit` | `widgetActive` フラグ更新 |
| `snapshot()` | 249 | `Snapshot` | すべてのメトリクスを集約 |
| `resetForTest()` | 267 | `Unit` | 全消去 |

### データクラス

| クラス | 行 | フィールド |
|---|---|---|
| `TimingSnapshot` | 98 | `count`, `minMs`, `avgMs`, `maxMs`, `lastMs` |
| `OffloadEvent` | 106 | `elapsedRealtimeMs`, `reason` |
| `ControllerInfo` | 111 | `packageName`, `isAndroidAuto`, `isWear`, `firstSeenElapsedMs` |
| `Snapshot` | 118 | `timings`, `counters`, `maxes`, `offloadEvents`, `controllers`, `widgetActive` |

### 内部実装メモ

- すべて `ConcurrentHashMap` + `AtomicLong` で並行安全。
- `advancedTimelineTimingNames = {FULL_SCAN, ARTWORK_EXTRACT, ARTWORK_DECODE, PLAYBACK_PREPARE, AUDIO_DECODER_INIT, TRANSITION, MEDIASESSION_ITEM_BUILD}` (`PerformanceMetrics.kt:132-140`) は `AdvancedPerformanceDiagnostics` 有効時にのみイベントとして転送される。
- `recordOffloadFallback` のリングバッファは `MAX_OFFLOAD_EVENTS = 20`。

---

## `DebugPerformanceReport` (`DebugPerformanceReport.kt:22`)

`@Serializable` な純粋データクラス。Android 依存なし。

### メインクラス

```kotlin
@Serializable
data class DebugPerformanceReport(
    val schemaVersion: Int = SCHEMA_VERSION, // 2
    val generatedAtIso: String,
    val device: DeviceSection,
    val app: AppSection,
    val library: LibrarySection,
    val hiRes: HiResSection,
    val artwork: ArtworkSection,
    val playback: PlaybackSection,
    val controllers: ControllerSection,
    val timings: Map<String, ReportTiming>,
    val offloadEvents: List<OffloadEventEntry>,
    val advancedDiagnostics: AdvancedDiagnosticsSection = AdvancedDiagnosticsSection(),
    val notes: List<String>
)
```

### 派生 / メソッド

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `toJson()` | 37 | `String` | `kotlinx.serialization.encodeToString` |
| `toPlainText()` | 40 | `String` | 人間可読なレポート |

### サブセクション

| クラス | 行 | フィールド |
|---|---|---|
| `DeviceSection` | 221 | manufacturer, model, brand, device, androidVersion, sdkInt, supportedAbis, memoryClassMb, isLowRamDevice, totalRamBytes, availableRamBytes |
| `AppSection` | 236 | versionName, versionCode, buildType, applicationId |
| `LibrarySection` | 244 | totalSongs, localSongs, cloudSongs, mimeCounts, maxBitrate, minSampleRate, maxSampleRate, maxObservedChannels, maxObservedPcmEncoding, estFileSizeMinBytes, estFileSizeAvgBytes, estFileSizeMaxBytes |
| `HiResSection` | 260 | hiResCount, ultraHiResCount, losslessCodecCount, likelyExpensiveToDecodeCount, multichannelPlaybackObservations, largeArtworkObservations |
| `ArtworkSection` | 270 | embeddedArtSeen, maxEmbeddedBytes, maxDecodedWidth, maxDecodedHeight, cacheHits, cacheMisses, freshExtractions, extractionTiming, decodeTiming |
| `PlaybackSection` | 283 | currentMime, sampleRate, bitrate, channelCount, pcmEncoding, decoderName, decoderHardware, audioOffloadEnabled, offloadFallbackCount, hiFiModeEnabled, crossfadeEnabled, crossfadeDurationMs, equalizerEnabled, replayGainEnabled, replayGainAlbumMode |
| `ControllerSection` | 302 | widgetActive, wearActive, androidAutoActive, connectedControllers |
| `ConnectedController` | 310 | packageName, isAndroidAuto, isWear |
| `ReportTiming` | 317 | count, minMs, avgMs, maxMs, lastMs + `format()` |
| `OffloadEventEntry` | 331 | elapsedRealtimeMs, reason |
| `AdvancedDiagnosticsSection` | 337 | enabled, sessionStartedIso, expiresAtIso, eventCount, droppedEventCount, events |
| `AdvancedDiagnosticEventEntry` | 347 | elapsedRealtimeMs, type, name, details |

### ヘルパ関数

| 関数 | 行 | 目的 |
|---|---|---|
| `bytes(value: Long)` | 182 | 単位付きフォーマット |
| `pcmEncodingLabel(encoding: Int)` | 189 | Media3 ENCODING_PCM_* 値 → ラベル |

`SCHEMA_VERSION = 2` (`DebugPerformanceReport.kt:172`)。

---

## `DebugPerformanceReportCollector` (`DebugPerformanceReportCollector.kt:33`)

`@Singleton` / `@Inject` で `Context`, `MusicDao`, `DualPlayerEngine`, `UserPreferencesRepository`, `EqualizerPreferencesRepository` を受ける。

### 公開 API

| API | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `generate()` | 52 | `DebugPerformanceReport` | レポート生成 |

### 内部実装メモ

- **エンジン状態の取得は Main で** (`DebugPerformanceReportCollector.kt:56`): ExoPlayer が thread-confined なので、`engine.currentAudioFormatSnapshot()`, `activeDecoderInfo.value`, `isAudioOffloadEnabled` はメインスレッドで取得。
- **IO 集計** (`DebugPerformanceReportCollector.kt:66`): `PerformanceMetrics.snapshot()` + `musicDao.getLibraryAudioStats()` + `musicDao.getMimeTypeCounts()` を `withContext(Dispatchers.IO)` 内で並列でなく順次実行。
- **lossless フォーマット判定** (`DebugPerformanceReportCollector.kt:42, 73-75`): `setOf("flac", "alac", "wav", "aiff")` を `AudioMetaUtils.mimeTypeToFormat(...)` で判定。
- **advancedDiagnostics セクション** (`DebugPerformanceReportCollector.kt:238-255`): `AdvancedPerformanceDiagnostics.snapshot()` を `AdvancedDiagnosticsSection` にマッピング。
- **notes** (`DebugPerformanceReportCollector.kt:257-265`): レポート末尾の定型的注釈。

### 補助 private

- `collectDevice()`, `collectApp()`, `collectLibrary()`, `collectHiRes()`, `collectArtwork()`, `collectPlayback()`, `collectControllers()`, `collectAdvancedDiagnostics()`, `buildNotes()`, `PerformanceMetrics.TimingSnapshot.toReportTiming()`, `isoNow()`, `isoFromEpochMs()` (`DebugPerformanceReportCollector.kt:123-276`)

---

## 内部実装メモ（横断）

### 「オプトイン 24h」設計

- 通常ユーザーは `AdvancedPerformanceDiagnostics.isEnabled = false` で、追加メモリ / CPU コストなし。
- 有効化は `UserPreferencesRepository.setAdvancedPerformanceDiagnosticsEnabled(true)` で 24 時間のセッションが切れるまで `recordEvent` が動作。
- `disableExpiredAdvancedPerformanceDiagnostics` が期限切れを自動クリア。

### 「常時稼働メトリクス」設計

- `PerformanceMetrics` は通常ユーザーでも常時動作（ただし書き込みは O(1) かつ揮発性）。
- `recordTiming` / `increment` 等の呼び出しコストを最小化するため、`computeIfAbsent` で初回のみ Map エントリ作成。
- `recordMax` は CAS ループで最大値を更新。

### レポート生成の安全策

- プレイヤースレッドは Main で読み出し（ExoPlayer の thread confinement 回避）。
- レポートには **ファイルパス・タイトル・アーティストなどのユーザー固有情報は含めない**（コメント `DebugPerformanceReport.kt:17-19`）。
- 端末スペック・件数・タイミング集計値のみ。

---

## 関連ファイル

- 上位: `presentation/screens/settings/PerformanceReportScreen.kt`, `service/MusicService.kt`, `service/player/DualPlayerEngine.kt`
- 下位: `data/database/MusicDao.kt` (`getLibraryAudioStats`, `getMimeTypeCounts`), `data/preferences/UserPreferencesRepository.kt`
- 関連: [`workers.md`](./workers.md), [`equalizer.md`](./equalizer.md), [`repositories.md`](./repositories.md)