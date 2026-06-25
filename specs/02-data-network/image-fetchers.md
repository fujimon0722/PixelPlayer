# image-fetchers.md

> Coil 用 `Fetcher` 実装群。アーティスト画像・アルバムアートをカスタム URI スキームで取得。
> カスタムスキーム経由でリポジトリに到達し、必要なヘッダ (Authorization, Cookie) を自動付与する。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/image/
├─ JellyfinCoilFetcher.kt        // 156 lines
├─ LocalArtworkCoilFetcher.kt     // 45 lines
├─ NavidromeCoilFetcher.kt        // 169 lines
└─ TelegramCoilFetcher.kt         // 430 lines (最大)
```

## 共通のキャッシュ戦略

各 Fetcher は:
1. **ディスクキャッシュ**: `cacheDir/<provider>_cover_<id>_<size>.jpg` (Telegram は `<chatId>_<messageId>.jpg`)。
2. **ネガティブキャッシュ (Telegram のみ)**: `telegram_embedded_art_<key>_none` マーカファイル。
3. **失敗ログ抑制**: `recentlyLoggedFailures: ConcurrentHashMap` で 60 秒以内の同一エラー再ログ抑制 (`LOG_FAILURE_COOLDOWN_MS = 60_000L`)。

## 共通の URI スキーム

| スキーム | 形式 | 例 |
| --- | --- | --- |
| `jellyfin` | `jellyfin://<itemId>?size=300` | `jellyfin://abc123def?size=500` |
| `navidrome` | `navidrome://<coverArtId>?size=300` | |
| `telegram_art` | `telegram_art://<chatId>/<messageId>?quality=thumb` | `telegram_art://-1001234567890/123?quality=thumb` |
| `local_artwork` | `local_artwork://song/<songId>` | `local_artwork://song/42` |

> 各スキームは `AppModule` の `ImageLoader` ビルダで `Fetcher.Factory` として登録される (`Fetcher.Factory<Uri>` の `create(data: Uri, ...)` でスキーム判別)。

---

## 1. `LocalArtworkCoilFetcher`

### 役割

ローカル楽曲 (MediaStore 由来) の埋め込みアートを `AlbumArtUtils` 経由で取得。

### 依存

| 依存 | 用途 |
| --- | --- |
| (なし) | `AlbumArtUtils`, `LocalArtworkUri` のみ参照 |

### URI パース

```kotlin
val songId = LocalArtworkUri.parseSongId(uri.toString()) ?: return null
```

### 取得

```kotlin
val cachedFile = AlbumArtUtils.ensureAlbumArtCachedFile(
    options.context,
    songId,
    uri.getQueryParameter("size")?.toIntOrNull() ?: 500
)
```

`AlbumArtUtils` は `MediaMetadataRetriever` で MP3 / M4A / FLAC の埋め込み画像を抽出し、`cacheDir/album_art_<songId>_<size>.jpg` に保存。

### 内部クラス `Factory`

```kotlin
class Factory @Inject constructor() : Fetcher.Factory<Uri> {
    override fun create(data: Uri, options: Options, imageLoader: ImageLoader): Fetcher? {
        if (LocalArtworkUri.isLocalArtworkUri(data.toString())) {
            return LocalArtworkCoilFetcher(data, options)
        }
        return null
    }
}
```

---

## 2. `JellyfinCoilFetcher`

### 役割

Jellyfin `/Items/{id}/Images/Primary?maxWidth=...` を Authorization ヘッダ付きで取得。

### 依存

| 依存 | 用途 |
| --- | --- |
| `JellyfinRepository` | 認証ヘッダ + 画像 URL |
| `OkHttpClient` | HTTP |
| `cacheDir: File` | キャッシュファイル |

### 状態

| フィールド | 説明 |
| --- | --- |
| `recentlyLoggedFailures` | 60 秒クールダウン |
| `LOG_FAILURE_COOLDOWN_MS` | `60_000L` |

### URI パース

```kotlin
val itemId = uri.host ?: uri.path?.removePrefix("/")
val sizeParam = uri.getQueryParameter("size")?.toIntOrNull() ?: 500
```

### 取得フロー

```
1. cachedFile = "jellyfin_cover_${itemId}_$sizeParam.jpg"
2. if cachedFile.exists(): return SourceResult(cachedFile.toPath(), ...)
3. imageUrl = repository.getImageUrl(itemId, sizeParam)
4. authHeader = repository.getAuthorizationHeader()
5. downloadImage(imageUrl, cachedFile, authHeader) → bytes
6. if success: cacheFile に書き込み → SourceResult
7. else: ネガティブキャッシュ無し → null (Coil がフォールバック)
```

### `downloadImage`

```kotlin
private fun downloadImage(url, cacheFile, authHeader): FetchResult? {
    val request = Request.Builder()
        .url(url)
        .apply { if (authHeader != null) header("Authorization", authHeader) }
        .build()
    okHttpClient.newCall(request).execute().use { response ->
        if (!response.isSuccessful) return null
        val bytes = response.body.bytes()
        cacheFile.writeBytes(bytes)
        return SourceResult(...)
    }
}
```

### `shouldLogFailure(key)`

同一キーのログを 60 秒間隔で抑制。デバッグノイズ軽減。

### 内部クラス `Factory`

```kotlin
class Factory @Inject constructor(
    private val repository: JellyfinRepository,
    private val okHttpClient: OkHttpClient
) : Fetcher.Factory<Uri> {
    private var cacheDir: File? = null
    override fun create(data: Uri, options: Options, imageLoader: ImageLoader): Fetcher? {
        if (data.scheme == "jellyfin") {
            val cache = cacheDir ?: options.context.cacheDir.also { cacheDir = it }
            return JellyfinCoilFetcher(data, repository, okHttpClient, cache)
        }
        return null
    }
}
```

---

## 3. `NavidromeCoilFetcher`

`JellyfinCoilFetcher` とほぼ同じ構造 (`navidrome_cover_<coverArtId>_<size>.jpg`、Subsonic `/getCoverArt.view`、Basic 認証またはトークン無し)。

### 差分

| 項目 | Jellyfin | Navidrome |
| --- | --- | --- |
| 認証ヘッダ | `Authorization: MediaBrowser Client=..., Token=...` | 不要 (URL に `u/t/s/v/c/f` クエリで十分) |
| URL | `/Items/{id}/Images/Primary` | `/rest/getCoverArt.view?id=...&size=...` |
| キャッシュファイル名 | `jellyfin_cover_*` | `navidrome_cover_*` |
| Fetcher スキーム | `jellyfin` | `navidrome` |

---

## 4. `TelegramCoilFetcher`

**最も複雑** (430 行)。複数の取得パスを順に試し、見つからなければ null。

### 依存

| 依存 | 用途 |
| --- | --- |
| `Context` | cacheDir |
| `TelegramRepository` | メッセージ / ファイル取得 |
| `cacheDir: File` | キャッシュ |
| `TelegramCacheManager?` | 失敗ネガティブキャッシュ + キャッシュ管理 |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `recentlyLoggedFailures` | `ConcurrentHashMap<String, Long>` | 60 秒クールダウン |
| `extractionMapMutex` | `Mutex` | per-key mutex マップ用 |
| `extractionLocks` | `ConcurrentHashMap<String, Mutex>` | (chatId, messageId) 単位の抽出ロック |

### URI パース

```kotlin
val chatId = uri.host?.toLongOrNull()
val messageId = uri.pathSegments.firstOrNull()?.toLongOrNull()
val quality = uri.getQueryParameter("quality")  // "thumb" or null
val isThumbnailRequest = quality == "thumb"
```

### 取得優先順位

```
1. tryExtractEmbeddedArtIfSafe(chatId, messageId) → cacheDir/telegram_embedded_art_<key>.jpg
   a. cachedArtFile 存在 → それを返す
   b. noArtMarker 存在 → ネガティブキャッシュ → null
   c. MessageAudio の audio.file.id から音声ファイル取得
   d. MediaMetadataRetriever.setDataSource で埋め込み画像を抽出
   e. 抽出成功 → キャッシュ保存 + _embeddedArtUpdated emit → return path
   f. 抽出失敗 → noArtMarker 作成 → return null
2. (1 失敗時) MessageContent から直接 thumbnail file id 取得
   a. audio.albumCoverThumbnail.file.id
   b. document.thumbnail.file.id
3. (2 失敗時) minithumbnail (TdApi の小さいサムネ) を BitmapFactory.decodeByteArray
4. (2,3 失敗時) downloadWithRetry でフル解像度を downloadFileAwait
   a. 失敗時は refreshMessage で再取得 → 新 file id で再試行
5. 全て失敗 → null
```

### `tryExtractEmbeddedArtIfSafe`

```kotlin
private suspend fun tryExtractEmbeddedArtIfSafe(chatId, messageId): String? {
    val key = "${chatId}_${messageId}"
    val cachedArtFile = File(cacheDir, "telegram_embedded_art_$key.jpg")
    val noArtMarker = File(cacheDir, "telegram_embedded_art_${key}_none")
    
    if (cachedArtFile.exists()) return cachedArtFile.absolutePath
    if (noArtMarker.exists()) return null
    
    // per-key mutex で重複抽出防止
    val lock = extractionMapMutex.withLock {
        extractionLocks.getOrPut(key) { Mutex() }
    }
    return lock.withLock {
        // double-check
        if (cachedArtFile.exists()) return cachedArtFile.absolutePath
        if (noArtMarker.exists()) return null
        
        val message = telegramRepository.getMessage(chatId, messageId) ?: return null
        val audioFileId = extractAudioFileId(message.content) ?: run {
            noArtMarker.createNewFile(); return null
        }
        val audioFile = telegramRepository.getFile(audioFileId)
        val audioFilePath = audioFile?.local?.path
        if (audioFilePath == null || !File(audioFilePath).exists()) {
            noArtMarker.createNewFile(); return null
        }
        
        extractAndCacheEmbeddedArt(audioFilePath, cachedArtFile, noArtMarker)
    }
}
```

### `extractAndCacheEmbeddedArt`

```kotlin
private fun extractAndCacheEmbeddedArt(audioPath, cachedArtFile, noArtMarker): String? {
    val retriever = MediaMetadataRetriever()
    retriever.setDataSource(audioPath)
    val embeddedPicture = retriever.embeddedPicture
    retriever.release()
    if (embeddedPicture == null) {
        noArtMarker.createNewFile(); return null
    }
    // サイズデコード + BitmapFactory.Options.inSampleSize でダウンサンプル
    FileOutputStream(cachedArtFile).use { it.write(embeddedPicture) }
    telegramCacheManager?.notifyEmbeddedArtExtracted(chatId, messageId)
    return cachedArtFile.absolutePath
}
```

### `downloadWithRetry`

```kotlin
private suspend fun downloadWithRetry(initialFileId, chatId, messageId): String? {
    val path = telegramRepository.downloadFileAwait(initialFileId, 1)
    if (path != null) return path
    val refreshedMessage = telegramRepository.refreshMessage(chatId, messageId)
    val refreshedFileId = extractFileIdFromContent(refreshedMessage?.content) ?: return null
    return telegramRepository.downloadFileAwait(refreshedFileId, 1)
}
```

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `extractMinithumbnail` | `private fun extractMinithumbnail(content: TdApi.MessageContent): ByteArray?` | `content.audio?.minithumbnail?.data` または `content.document?.minithumbnail?.data` |
| `extractAudioFileId` | `private fun extractAudioFileId(content: TdApi.MessageContent?): Int?` | `audio.audio?.id?.value` または `document.document?.id?.value` |
| `extractThumbnailFileId` | `private suspend fun extractThumbnailFileId(chatId, messageId): Int?` | `Message` 取得 → `extractFileIdFromContent` |
| `extractFileIdFromContent` | `private fun extractFileIdFromContent(content: TdApi.MessageContent?): Int?` | `audio.albumCoverThumbnail?.file?.id` または `document.thumbnail?.file?.id` |
| `shouldLogFailure` | `private fun shouldLogFailure(key): Boolean` | 60 秒クールダウン |

### 内部クラス `Factory`

```kotlin
class Factory @Inject constructor(
    private val telegramRepository: TelegramRepository,
    private val telegramCacheManager: TelegramCacheManager
) : Fetcher.Factory<Uri> {
    private var cacheDir: File? = null
    override fun create(data: Uri, options: Options, imageLoader: ImageLoader): Fetcher? {
        if (data.scheme == "telegram_art") {
            val cache = cacheDir ?: options.context.cacheDir.also { cacheDir = it }
            return TelegramCoilFetcher(
                options.context, data, telegramRepository, cache, telegramCacheManager
            )
        }
        return null
    }
}
```

---

## 5. 内部実装メモ

### Coil `Fetcher` の責務

| メソッド | 責務 |
| --- | --- |
| `fetch(): FetchResult?` | データ取得 → `SourceResult` / `DrawableResult` 返却。null なら Coil が次 Fetcher を試行。 |
| `data` | 任意の型 (本実装は全て `Uri`) |

### キャッシュファイルの TTL

キャッシュファイルは明示的な TTL を持たず、永続化される。`AppCacheManager` 等で明示的に削除しない限り残り続ける。

### Telegram の三重キャッシュ戦略

1. **埋め込みアート抽出**: 1 度成功したら `telegram_embedded_art_<key>.jpg` を再利用。
2. **minithumbnail**: TdApi が返す低解像度 (例: 96x96) を即座にデコード。
3. **フルダウンロード**: 上記 2 つが無い場合の最終手段 (帯域コスト大)。

### 失敗抑制パターン

`shouldLogFailure` で同じ file id のログを 60 秒に 1 回に抑制。デバッグ時にログスパムを防ぐ。

---

## 6. 関連ファイル

- 認証ヘッダ供給元: 各 Repository (`JellyfinRepository.getAuthorizationHeader` 等)
- Coil 統合: `di/AppModule.kt` の `ImageLoader` 設定
- キャッシュ戦略: [`streaming-telegram.md`](./streaming-telegram.md) §2
