# network-lyrics-deezer.md

> 歌詞 (LrcLib) と アーティスト画像 (Deezer) の薄い HTTP クライアント。
> `data/network/lyrics/` と `data/network/deezer/` のみを扱う最小 spec。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/network/
├─ lyrics/
│   ├─ LrcLibApiService.kt     // 42 lines (Retrofit interface)
│   └─ LrcLibResponse.kt       // 16 lines (data class)
└─ deezer/
    ├─ DeezerApiService.kt     // 23 lines (Retrofit interface)
    └─ DeezerModels.kt         // 27 lines (Gson DTOs)
```

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `data/repository/LyricsRepositoryImpl.kt`, `data/repository/ArtistImageRepository.kt`, `di/AppModule.kt` |
| 下流 (依存先) | `retrofit2`, `com.google.gson.annotations.SerializedName` |

---

## 1. LrcLib (歌詞)

### `LrcLibApiService` (Retrofit interface)

| メソッド | HTTP | クエリ / パス | 目的 |
| --- | --- | --- | --- |
| `getLyrics` | `GET /api/get` | `artist_name`, `track_name`, `album_name?`, `duration?`, `format='plain'` | 1 曲ぶんの歌詞を LRC 形式で取得。 |
| `searchLyrics` | `GET /api/search` | `q`, `artist_name?`, `track_name?` | 複数候補を返す検索 (UI からの手動選択用)。 |

レスポンス: `LrcLibResponse` (後述)。

> **Note**: `format='plain'` をクエリに含めることで LRC タイムスタンプ付き歌詞を強制する。

### `LrcLibResponse` (data class)

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `id` | `Int` | LrcLib 内部 ID |
| `trackName` | `String?` | 曲名 |
| `artistName` | `String?` | アーティスト名 |
| `albumName` | `String?` | アルバム名 |
| `duration` | `Double?` | 秒 |
| `instrumental` | `Boolean?` | インスト曲判定 |
| `plainLyrics` | `String?` | タイムスタンプ無しプレーンテキスト |
| `syncedLyrics` | `String?` | LRC タイムスタンプ付き |

> `@SerializedName(...)` で snake_case ↔ camelCase を GSON にマッピングさせる。

### 利用パターン

```
LyricsRepositoryImpl
  → search(query) → searchLyrics("...", ...)
  → 候補表示 → ユーザ選択 → getLyrics(...)
  → syncedLyrics を優先、無ければ plainLyrics
```

---

## 2. Deezer (アーティスト画像)

### `DeezerApiService` (Retrofit interface)

| メソッド | HTTP | クエリ | 目的 |
| --- | --- | --- | --- |
| `searchArtist` | `GET /search/artist` | `q=...` | キーワードで Deezer アーティスト検索。最大件数 / インデックスは固定 (AppModule 側で Retrofit の GSON Converter が整形)。 |

レスポンス: `DeezerSearchResponse` (後述)。

### `DeezerModels` (Gson DTOs)

#### `DeezerSearchResponse`

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `data` | `List<DeezerArtist>` | 検索結果 |

#### `DeezerArtist`

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `id` | `Long` | Deezer アーティスト ID |
| `name` | `String` | アーティスト名 |
| `picture` | `String?` | 小サイズ画像 URL |
| `picture_small` | `String?` | さらに小さい URL |
| `picture_medium` | `String?` | 中サイズ URL |
| `picture_big` | `String?` | 大サイズ URL |
| `picture_xl` | `String?` | 特大サイズ URL |
| `tracklist` | `String?` | Deezer のトラックリスト URL |

> Deezer API は **公開 API で API key 不要** だが、User-Agent などの HTTP ヘッダ制限があるため `AppModule` の OkHttpClient を経由。

---

## 3. 内部実装メモ

### 設計上のポイント

- **Retrofit ベース**: `LrcLibApiService`, `DeezerApiService` は annotation のみ。実装は `retrofit2.Retrofit.create(...)` で動的生成。
- **AppModule での組み立て**: `LrcLibApiService` は `provideLrcLibApiService(...)` 風の `@Provides` (本 spec では確認していないが、DI 経由で取得)。
- **キャッシュ**: Repository 層 (`LyricsRepository`, `ArtistImageRepository`) でメモリ/DB キャッシュされる前提。

### LrcLib エンドポイント

```
GET https://lrclib.net/api/get?artist_name=...&track_name=...
GET https://lrclib.net/api/search?q=...
```

### Deezer エンドポイント

```
GET https://api.deezer.com/search/artist?q=...
```

---

## 4. 関連ファイル

- Lyrics 集約: `data/repository/LyricsRepositoryImpl.kt`
- Artist Image 集約: `data/repository/ArtistImageRepository.kt`
- DI: `di/AppModule.kt`
- Coil Fetcher との接続: [`image-fetchers.md`](./image-fetchers.md)
