# Equalizer

Android AudioEffect（Equalizer / BassBoost / Virtualizer / LoudnessEnhancer）のラッパ。ExoPlayer の audioSessionId に紐付けて適用。

## パッケージ

`com.theveloper.pixelplay.data.equalizer`

---

## 依存関係

### 上流
- `service/player/DualPlayerEngine.kt` — `equalizerManager.attachToAudioSession(audioSessionId)` を再生開始 / クロスフェード切替時に呼び出し
- `service/MusicService.kt` — セッション更新時
- `presentation/viewmodel/SettingsViewModel.kt` — `EqualizerPreferencesRepository` 設定値を `EqualizerManager.restoreState(...)` に反映
- `presentation/screens/settings/EqualizerScreen.kt` — バンド変更 / プリセット / BassBoost / Virtualizer / LoudnessEnhancer の UI

### 下流
- `android.media.audiofx.Equalizer`
- `android.media.audiofx.BassBoost`
- `android.media.audiofx.Virtualizer`
- `android.media.audiofx.LoudnessEnhancer`
- `android.media.audiofx.AudioEffect.queryEffects()`
- `data/preferences/EqualizerPreferencesRepository.kt`
- `data/diagnostics/AdvancedPerformanceDiagnostics.kt`

---

## ファイル一覧

| ファイル | 行 | 役割 |
|---|---|---|
| `EqualizerManager.kt` | 715 | 全 AudioEffect のライフサイクル管理 |
| `EqualizerPreset.kt` | 98 | 10 バンドのプリセット定義 (FLAT, ROCK, POP, HIP_HOP, JAZZ, CLASSICAL, ELECTRONIC, BASS_BOOST, TREBLE_BOOST, VOCAL) |

---

## `EqualizerManager` (`EqualizerManager.kt:25`)

`@Singleton` / `@Inject` (引数なし)。

### 定数

| 定数 | 値 | 用途 |
|---|---|---|
| `NUM_BANDS` | `10` | UI バンド数 |
| `MIN_LEVEL` | `-15` | バンド値最小 |
| `MAX_LEVEL` | `15` | バンド値最大 |
| `MAX_LOUDNESS_GAIN_MB` | `1000` | LoudnessEnhancer 最大ゲイン (mB) |

### 内部状態

| フィールド | 行 | 用途 |
|---|---|---|
| `equalizer: Equalizer?` | 35 | AudioEffect |
| `bassBoost: BassBoost?` | 36 | |
| `virtualizer: Virtualizer?` | 37 | |
| `currentAudioSessionId: Int` | 38 | 紐付け中のセッション ID |
| `minEqLevel: Short` | 78 | 端末の最小 mB |
| `maxEqLevel: Short` | 79 | 端末の最大 mB |
| `loudnessEnhancer: LoudnessEnhancer?` | 81 | |
| `isBassBoostSupportedGlobal: Boolean` | 84 | 端末サポート検出 |
| `isVirtualizerSupportedGlobal: Boolean` | 85 | |
| `effectsDisabledForProcess: Boolean` | 86 | エフェクト使用不可フラグ（OEM 対策） |
| `effectsDisableReason: String?` | 87 | |

### StateFlow

| プロパティ | 行 | 型 | デフォルト |
|---|---|---|---|
| `bandLevels` | 51 | `StateFlow<List<Int>>` | `List(10) { 0 }` |
| `isEnabled` | 54 | `StateFlow<Boolean>` | `false` |
| `currentPresetName` | 57 | `StateFlow<String>` | `"flat"` |
| `bassBoostEnabled` | 60 | `StateFlow<Boolean>` | `false` |
| `bassBoostStrength` | 63 | `StateFlow<Int>` | `0` |
| `virtualizerEnabled` | 65 | `StateFlow<Boolean>` | `false` |
| `virtualizerStrength` | 68 | `StateFlow<Int>` | `0` |
| `loudnessEnhancerEnabled` | 71 | `StateFlow<Boolean>` | `false` |
| `loudnessEnhancerStrength` | 74 | `StateFlow<Int>` | `0` |

### 公開 API

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `isAttached` (val) | 40 | `Boolean` | `equalizer != null && currentAudioSessionId != 0` |
| `hasAnyEnabledEffects` (val) | 43 | `Boolean` | 4 つのいずれかが ON |
| `attachToAudioSession(audioSessionId)` | 134 | `suspend Unit` | 4 つの AudioEffect を生成・初期化。`AdvancedPerformanceDiagnostics.traceSuspend` 経由 |
| `attachToAudioSessionIfNeeded(audioSessionId)` | 360 | `suspend Unit` | 全エフェクト無効ならスキップ |
| `setEnabled(enabled)` | 375 | `Unit` | EQ ON/OFF |
| `setBandLevel(bandIndex, level)` | 391 | `Unit` | 個別バンド変更（手動変更で `custom` に切替） |
| `applyPreset(preset)` | 408 | `Unit` | プリセット適用 |
| `setBassBoostEnabled(enabled)` | 418 | `Unit` | サポート外なら強制 OFF |
| `setBassBoostStrength(strength)` | 436 | `Unit` | 0..1000 にクランプ |
| `setVirtualizerEnabled(enabled)` | 457 | `Unit` | サポート外なら強制 OFF |
| `setVirtualizerStrength(strength)` | 475 | `Unit` | 0..1000 にクランプ |
| `setLoudnessEnhancerEnabled(enabled)` | 496 | `Unit` | ON/OFF |
| `setLoudnessEnhancerStrength(strength)` | 510 | `Unit` | 0..1000 にクランプ |
| `restoreState(...)` | 525 | `Unit` | 設定値から全状態を復元（attach 前 / 後の両方で安全） |
| `getBandFrequencies()` | 674 | `List<Int>` | 端末のバンド中心周波数 (Hz) |
| `isBassBoostSupported()` | 684 | `Boolean` | 端末サポート |
| `isVirtualizerSupported()` | 689 | `Boolean` | 端末サポート |
| `isLoudnessEnhancerSupported()` | 694 | `Boolean` | LE インスタンスがあるか、API 19+ |
| `release()` | 699 | `Unit` | 全 AudioEffect を release |

### 内部実装メモ

#### デバイスサポート検出 (`checkDeviceSupport`)

`EqualizerManager.kt:93` で `AudioEffect.queryEffects()` を呼び出し、BassBoost / Virtualizer の有無をキャッシュ。

#### エフェクト使用不能処理 (`markBassBoostUnavailable` / `markVirtualizerUnavailable`)

OEM 対策。サポートフラグを false にし、インスタンスを release。

#### `attachToAudioSessionInternal` (`EqualizerManager.kt:144`)

1. 早期 return 条件:
   - `effectsDisabledForProcess == true`
   - `audioSessionId == 0`
   - `currentAudioSessionId == audioSessionId && equalizer != null`（既に attach 済み）
2. `release()` でクリーンアップ。
3. `Equalizer(0, audioSessionId)` で生成し、`bandLevelRange` から `minEqLevel/maxEqLevel` を取得。
4. **BassBoost / Virtualizer は 3 回リトライ** (`EqualizerManager.kt:240-298`) — 初期化失敗時に `kotlinx.coroutines.delay(300)` で 300ms 待機して再試行。すべて失敗なら `markBassBoostUnavailable` / `markVirtualizerUnavailable`。
5. **LoudnessEnhancer** (`EqualizerManager.kt:302-319`) は失敗しても例外は出さず `null` のまま。
6. `applyBandLevels(_bandLevels.value)` で現在のバンド値を反映。
7. `AdvancedPerformanceDiagnostics.recordEventIfEnabled(AUDIO_EFFECT, "equalizer_attach_*")` で詳細イベント記録。

#### バンド値マッピング (`applyBandLevels`)

- 端末バンド数 >= UI バンド数 → 直接マッピング
- 端末バンド数 < UI バンド数 → UI バンドの平均を端末バンドへ

これにより 5 バンド端末でも 10 バンド UI が破綻しない。

#### `applyBandLevelDirect` (`EqualizerManager.kt:646`)

normalized level (-15..15) → millibel に変換: `minEqLevel + (level + 15) * range / 30`。

#### `restoreState` (`EqualizerManager.kt:525`)

8 つの引数（enabled, presetName, customBands, BB/V/LE の enabled+strength）を受け取り、`_isEnabled` / `_bandLevels` を更新。`equalizer != null` なら実際の AudioEffect にも適用。**attach 前でも安全なように StateFlow のみ更新する設計**。

#### `release` (`EqualizerManager.kt:699`)

`try-catch` で全 AudioEffect を release。失敗しても続行し、変数を null に戻す。`currentAudioSessionId = 0`。

---

## `EqualizerPreset` (`EqualizerPreset.kt:13`)

`@Serializable` / `@Immutable` data class。

### フィールド

| フィールド | 型 | 用途 |
|---|---|---|
| `name` | `String` | 内部識別子 |
| `displayName` | `String` | UI 表示名 |
| `bandLevels` | `List<Int>` | 10 バンド値 (-15..15) |
| `isCustom` | `Boolean = false` | カスタムプリセットフラグ |

### 既定プリセット

| 名前 | displayName | バンド値 | 用途 |
|---|---|---|---|
| `FLAT` | FLAT | `0,0,0,0,0,0,0,0,0,0` | 無加工 |
| `ROCK` | ROCK | `5,4,3,1,-1,-1,1,3,4,5` | 低音強 / 中音カット / 高音強 |
| `POP` | POP | `-1,2,4,5,5,4,2,1,2,2` | ボーカルフォーカス |
| `HIP_HOP` | HIP HOP | `6,8,4,1,-1,-1,1,1,3,4` | 重サブベース |
| `JAZZ` | JAZZ | `3,2,1,2,-1,-1,0,2,3,4` | 暖色 |
| `CLASSICAL` | CLASSICAL | `4,3,2,1,-1,-1,0,2,4,4` | バランス V 字 |
| `ELECTRONIC` | ELECTRONIC | `5,6,2,0,-1,1,0,2,6,7` | パンチ低音 / 鋭い高音 |
| `BASS_BOOST` | BASS BOOST | `7,9,6,3,0,0,0,0,0,0` | 低音ブースト |
| `TREBLE_BOOST` | TREBLE BOOST | `0,0,0,0,0,1,3,6,8,9` | 高音ブースト |
| `VOCAL` | VOCAL | `-3,-2,-1,2,5,6,5,3,1,0` | 中音域ブースト |

### `BAND_FREQUENCIES` (`EqualizerPreset.kt:21`)

`["31Hz", "62Hz", "125Hz", "250Hz", "500Hz", "1kHz", "2kHz", "4kHz", "8kHz", "16kHz"]` — UI 表示用。

### companion

| API | 行 | 目的 |
|---|---|---|
| `custom(bandLevels)` | 83 | カスタムプリセット生成 |
| `ALL_PRESETS` | 90 | 10 個のリスト |
| `fromName(name)` | 94 | 名前 → プリセット（無ければ FLAT） |

---

## 内部実装メモ (横断)

### クロスフェード互換性

> 効果（EQ / BassBoost / Virtualizer / LE）は AudioSession に紐付くため、プレイヤーインスタンスに紐付かない。
> クロスフェードで ExoPlayer インスタンスが切り替わっても、同じ `audioSessionId` であればエフェクトは維持される（コメント `EqualizerManager.kt:21-22`）。

### 端末サポート不在時の挙動

- **BassBoost / Virtualizer**: `markBassBoostUnavailable` / `markVirtualizerUnavailable` で `isBassBoostSupportedGlobal = false` にし、`setBassBoostEnabled(true)` 等の呼び出しは no-op。
- **Equalizer**: 生成時の例外で `effectsDisabledForProcess = true` にし、以後 `attachToAudioSession(...)` をスキップ。
- **LoudnessEnhancer**: 例外時 `null` のまま、`isLoudnessEnhancerSupported` は `loudnessEnhancer != null || SDK_INT >= KITKAT` で判定。

### トレースとイベント記録

- `attachToAudioSession` は `AdvancedPerformanceDiagnostics.traceSuspend("Equalizer.attachToAudioSession")` で囲まれ、`equalizer_attach_start/success/skipped/failed` の各イベントが `AUDIO_EFFECT` 種で記録される。
- 失敗時は `effectsDisableReason` を `error` 詳細として保存。

### `setBandLevel` の自動 `custom` 切替

`EqualizerManager.kt:402` で `setBandLevel` を呼ぶと `_currentPresetName.value = "custom"` に強制設定。これにより UI は「ユーザーが手動で弄った」ことを検知できる。

---

## 関連ファイル

- 上位: `service/player/DualPlayerEngine.kt`, `service/MusicService.kt`, `presentation/viewmodel/SettingsViewModel.kt`
- 下位: `android.media.audiofx.*`
- 関連: [`preferences.md`](./preferences.md), [`diagnostics.md`](./diagnostics.md), [`media-processing.md`](./media-processing.md)