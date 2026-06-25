# Media Processing

メディアメタデータ読み取り / アートワーク URI 生成 / ReplayGain / MediaController 生成 / Song メタデータ編集。

## パッケージ

`com.theveloper.pixelplay.data.media`

---

## 依存関係

### 上流
- `data/worker/SyncWorker.kt` — `AudioMetadataReader.read(file)` を `processSongData` で呼び出し
- `data/worker/NavidromeSyncWorker.kt`
- `service/player/DualPlayerEngine.kt` — `ReplayGainManager.gainDbToVolume(...)` でゲイン適用
- `presentation/viewmodel/PlayerViewModel.kt` — `MediaMapper.resolveSongFromMediaItem` をフォールバックに
- `service/MusicService.kt` — `MediaControllerFactory.create(...)` で MediaController 構築
- `presentation/screens/.../SongMetadataEditor.kt` — `SongMetadataEditor` でタグ書き込み

### 下流
- `com.kyant.taglib.TagLib` (JNI)
- `org.jaudiotagger.audio.AudioFileIO` (fallback)
- `coil.imageLoader` / `coil.memory.MemoryCache`
- `androidx.media3.session.MediaController`
- `utils/AlbumArtUtils.kt`
- `utils/LocalArtworkUri.kt`

---

## ファイル一覧

| ファイル | 行 | 役割 |
|---|---|---|
| `AudioMetadataReader.kt` | 265 | TagLib + JAudioTagger フォールバックでタグ読み取り |
| `AudioMetadataUtils.kt` | 144 | 拡張子判定 / 一時ファイル作成 / 画像判定 / MIME 推定 |
| `ImageCacheManager.kt` | 40 | Coil キャッシュ無効化 |
| `MediaControllerFactory.kt` | 21 | `MediaController.Builder.buildAsync()` のラッパ |
| `MediaMapper.kt` | 67 | MediaItem → Song 復元 |
| `ReplayGainManager.kt` | 161 | ReplayGain タグ読み取り + dB → リニア変換 + LRU キャッシュ |
| `SongMetadataEditor.kt` | 59069 (実ファイルは巨大; 5,900 行超) | タグ書き込み (編集 UI) |

---

## `AudioMetadataReader` (`AudioMetadataReader.kt:38`)

`object` (シングルトン)。

### データクラス

| クラス | 行 | フィールド |
|---|---|---|
| `AudioMetadata` | 14 | `title`, `artist`, `albumArtist`, `album`, `genre`, `composer`, `lyrics`, `durationMs`, `trackNumber`, `discNumber`, `year`, `bitrate`, `sampleRate`, `artwork: AudioMetadataArtwork?`, `replayGainTrackGainDb`, `replayGainAlbumGainDb` |
| `AudioMetadataArtwork` | 33 | `bytes: ByteArray`, `mimeType: String?` |

### 公開 API

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `read(context: Context, uri: Uri)` | 51 | `AudioMetadata?` | 一時ファイル経由で `read(file)` に委譲 |
| `read(file: File, readArtwork: Boolean = true)` | 68 | `AudioMetadata?` | TagLib で読み取り、JAudioTagger フォールバック、`METADATA_READ` 計測 |

### 内部実装メモ

- **TagLib** (`com.kyant.taglib.TagLib`): `getAudioProperties(fd)`, `getMetadata(fd, readPictures=false)`, `getPictures(fd)` を使用。
- **キー抽出** (`AudioMetadataReader.kt:85-104`): TITLE / ARTIST / ALBUMARTIST / ALBUMARTIST / BAND / ALBUM / GENRE / COMPOSER / TCOM / LYRICS / UNSYNCEDLYRICS / TRACKNUMBER / TRACK / DISCNUMBER / DISC / DATE / YEAR を順次参照。
- **ReplayGain 抽出** (`AudioMetadataReader.kt:104-111`): `REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_TRACK_GAIN_DB`, `R128_TRACK_GAIN` 等。
- **JAudioTagger フォールバック** (`AudioMetadataReader.kt:130-156`): `title == null || artist == null || (readArtwork && artwork == null)` のいずれかなら実行。`METADATA_FALLBACK_JAUDIOTAGGER` カウンタをインクリメント。
- **VERBOSE = false** (`AudioMetadataReader.kt:49`): 1 行ごとのログをホットパスから除外（コメント参照）。
- **計測**: `finally` 節で `PerformanceMetrics.recordTiming(METADATA_READ, ...)` を発行。

### private

| 関数 | 行 | 目的 |
|---|---|---|
| `readWithJAudioTagger(file, readArtwork)` | 173 | JAudioTagger フォールバック |
| `extractReplayGainDb(propertyMap, keys)` | 242 | 複数のキーから最初の有効値 |
| `parseReplayGainDb(rawValue)` | 256 | `","` → `"."`, `[dD][bB]` → `""` の正規化 |

---

## `AudioMetadataUtils.kt` (internal 関数)

### 公開関数 (internal)

| 関数 | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `createTempAudioFileFromUri(context, uri)` | 15 | `File?` | `cacheDir` にテンポラリファイル作成、削除フック付き |
| `resolveAudioFileExtension(context, uri)` | 34 | `String` | DISPLAY_NAME → URI path → MIME type → ストリーム MIME → `.mp3` の順で拡張子決定 |
| `isValidImageData(data)` | 122 | `Boolean` | `BitmapFactory.Options.inJustDecodeBounds` で width/height > 0 を検証 |
| `imageExtensionFromMimeType(mimeType)` | 128 | `String?` | jpeg/png/webp/gif → 拡張子 |
| `guessImageMimeType(data)` | 138 | `String?` | `URLConnection.guessContentTypeFromStream` |

### `AUDIO_MIME_OVERRIDES` (`AudioMetadataUtils.kt:95`)

23 個の MIME → 拡張子マッピング (`audio/mp4` → `m4a`, `audio/flac` → `flac` など)。

---

## `ImageCacheManager` (`ImageCacheManager.kt:11`)

`@Singleton` / `@Inject` で `Context` を受ける。

### 公開 API

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `invalidateCoverArtCaches(vararg uriStrings)` | 16 | `Unit` | Coil のメモリ / ディスクキャッシュから特定 URI を削除。9 種類のサイズ suffix (null, 128x128, …, 800x800) を網羅 |

### 内部実装メモ

- `LocalArtworkUri.isLocalArtworkUri(baseUri)` の場合、`LocalArtworkUri.parseSongId(baseUri)?.let { AlbumArtUtils.clearCacheForSong(context, songId) }` でファイルキャッシュも削除。
- `knownSizeSuffixes` を for ループして、各 suffix に対して `MemoryCache.Key("${baseUri}_${suffix}")` / `diskCache.remove(key)` を実行。

---

## `MediaControllerFactory` (`MediaControllerFactory.kt:11`)

`@Singleton` / `@Inject` (引数なし)。

### 公開 API

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `create(context, token, listener)` | 12 | `ListenableFuture<MediaController>` | `MediaController.Builder.buildAsync()` |

---

## `MediaMapper` (`MediaMapper.kt:18`)

`@Singleton` / `@Inject` で `Context` を受ける。

### 公開 API

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `resolveSongFromMediaItem(mediaItem)` | 22 | `Song?` | MediaItem の metadata + extras から部分的な Song を構築 |

### 内部実装メモ

- `MediaItemBuilder.EXTERNAL_EXTRA_CONTENT_URI` / `_ALBUM` / `_DURATION` / `_DATE_ADDED` / `_FILE_PATH` を extras から取得。
- `mediaId` を `Song.id` に設定。
- 一部フィールドは `R.string.common_unknown_*` でフォールバック。
- ファイルパスは `localConfiguration.uri` が `file` スキームの場合のみ使用。

---

## `ReplayGainManager` (`ReplayGainManager.kt:21`)

`@Singleton` / `@Inject` (引数なし)。

### データクラス

| クラス | 行 | フィールド |
|---|---|---|
| `ReplayGainValues` | 51 | `trackGainDb: Float?`, `albumGainDb: Float?` |

### 定数

| 定数 | 値 |
|---|---|
| `DEFAULT_PRE_AMP_DB` | `0.0f` |
| `TRACK_GAIN_KEYS` | `REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_TRACK_GAIN_DB`, `R128_TRACK_GAIN` |
| `ALBUM_GAIN_KEYS` | `REPLAYGAIN_ALBUM_GAIN`, `REPLAYGAIN_ALBUM_GAIN_DB`, `R128_ALBUM_GAIN` |

### LRU キャッシュ

`cache: LinkedHashMap<String, ReplayGainValues?>(64, 0.75f, true)` — 200 件超で eldest 削除 (`ReplayGainManager.kt:26-28`)。

### 公開 API

| メソッド | 行 | 戻り値 | 目的 |
|---|---|---|---|
| `getCachedReplayGain(filePath)` | 64 | `ReplayGainValues?` | キャッシュ参照のみ (IO なし) |
| `readReplayGain(filePath)` | 69 | `ReplayGainValues?` | TagLib で読み取り、LRU キャッシュ保存 |
| `gainDbToVolume(gainDb, preAmpDb)` | 113 | `Float` | `10.pow((gainDb+preAmp) / 20)` を `[0, 2]` にクランプ |
| `getVolumeMultiplier(values, useAlbumGain, preAmpDb)` | 123 | `Float` | useAlbumGain ? album : track のフォールバック付きゲインを volume に変換 |

### 内部実装メモ

- `parseGainString(raw)` (`ReplayGainManager.kt:155`) は `[dD][bB]` を削除してから `toFloatOrNull`。
- ゲインが負（loud tracks）の場合、volume < 1。`coerceIn(0f, 2f)` で最大 +6 dB まで許容。
- `PlaybackActivityTracker.isPlaybackActive` には依存しない（純粋関数）。

---

## `SongMetadataEditor.kt`

> **警告**: 5,900 行超の巨大ファイル。UI とタグ書き込みロジックが混在しているため、ここでは主要 API のみ列挙する。詳細が必要なら追加スペック参照。

### 概要

- SongEntity のタイトル / アーティスト / アルバム / ジャンル / トラック番号 / ディスク番号 / 年 / 歌詞 などのタグをファイルへ書き戻す編集 UI。
- 内部で `org.jaudiotagger` を直接使用して ID3 / Vorbis / MP4 タグを書き込み。

---

## 内部実装メモ (横断)

### TagLib → JAudioTagger フォールバック戦略

`AudioMetadataReader.kt:130` で判定：
1. TagLib で `title`, `artist`, `album` のいずれかが `null`
2. `readArtwork && artwork == null`

いずれかに該当したら `readWithJAudioTagger(...)` を実行。`PerformanceMetrics.Counters.METADATA_FALLBACK_JAUDIOTAGGER` でフォールバックヒット数を計測。

### ReplayGain の 2 段フォールバック

`ReplayGainManager.getVolumeMultiplier`:
- `useAlbumGain == true`: `albumGainDb ?: trackGainDb`
- `useAlbumGain == false`: `trackGainDb ?: albumGainDb`

最終的に `gainDbToVolume` で `[0, 2]` にクランプ。

### Coil キャッシュ無効化の網羅性

`ImageCacheManager.invalidateCoverArtCaches` は 9 種類のサイズ suffix (null, 128x128, 150x150, 168x168, 256x256, 300x300, 512x512, 600x600, 800x800) に対してメモリ + ディスク両方のエントリを削除。Coil がサイズごとに別キーを生成するため網羅が必要。

### MediaItem → Song の fallback パス

`MediaMapper.resolveSongFromMediaItem` は部分的な `Song` のみ生成するため、UI 側では「ID で `MusicRepository.getSong(id)` を試行 → 失敗時に MediaItem から復元」という順序で扱うことが期待される（コメント `MediaMapper.kt:14-17`）。

---

## 関連ファイル

- 上位: `data/worker/SyncWorker.kt`, `service/MusicService.kt`, `service/player/DualPlayerEngine.kt`
- 下位: `com.kyant.taglib:TagLib`, `org.jaudiotagger:jaudiotagger`, `androidx.media3:media3-session`
- 関連: [`repositories.md`](./repositories.md), [`equalizer.md`](./equalizer.md), [`diagnostics.md`](./diagnostics.md)