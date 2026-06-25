# network-qqmusic.md

> QQ Music (QQ 音乐) API クライアント。
> `musics.fcg` 系エンドポイントへ未署名リクエストを投げる薄い層。
> 暗号化・署名は `data/remote/qqmusic/{QQSignGenerator, QQMusicSecurity, QQMusicEncryptInterceptor}` で行う (別 spec: [`streaming-qqmusic.md`](./streaming-qqmusic.md))。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/network/qqmusic/
└─ QqMusicApiService.kt   // 336 lines
```

## ファイル概要

| ファイル | 行数 | 概要 |
| --- | ---: | --- |
| `QqMusicApiService.kt` | 336 | QQ Music の cookie ベース API + `musics.fcg` への HTTP ラッパ。 |

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `data/qqmusic/QqMusicRepository.kt`, `data/qqmusic/QqMusicStreamProxy.kt` |
| 下流 (依存先) | `okhttp3.{Cookie, CookieJar, OkHttpClient, Request}`, `org.json.JSONObject`, `java.util.zip.InflaterInputStream` (deflate 展開) |

## 1. `QqMusicApiService`

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `cookieStore` | `MutableMap<String, MutableList<Cookie>>` | ホスト別インメモリ CookieJar。 |
| `persistedCookies` | `@Volatile Map<String, String>` | 永続クッキー (`uin`, `qm_keyst`, `euin`, `psrf_qqaccess_token`, `p_skey`, `skey` 等)。 |
| `okHttpClient` | `OkHttpClient` | 注入されたベース OkHttp を `newBuilder()` で複製。 |

### 認証情報管理

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `hasLogin` | `fun hasLogin(): Boolean` | `qm_keyst` または `psrf_qqaccess_token` が空文字でなければログイン済み。 |
| `setPersistedCookies` | `fun setPersistedCookies(cookies: Map<String, String>)` | 永続クッキーをメモリに反映し、`y.qq.com` / `c.y.qq.com` / `u.y.qq.com` / `music.qq.com` のホスト CookieJar にシード。 |
| `logout` | `fun logout()` | クッキー全消去。 |

### GTK 計算

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `getGTK` | `private fun getGTK(): Long` | QQ の GTK (g_tk) アルゴリズム: `skey = persistedCookies["p_skey"] ?: persistedCookies["skey"] ?: ""` を使い、`hash = 5381; for (c in skey) hash = (hash << 5) + hash + c` (符号なし 32-bit) → Long。 |

> GTK は各種 API リクエスト URL の `g_tk=` パラメータに必要な CSRF 風トークン。

### 公開 API メソッド

| メソッド | シグネチャ | 暗号化 | 目的 / 戻り値 |
| --- | --- | --- | --- |
| `getUserPlaylists` | `suspend fun getUserPlaylists(start=0, count=100): String` | (なし) | `cdlist` (收藏歌单) 取得。`g_tk` + `uin` パラメータ。 |
| `getUserCreatedPlaylists` | `suspend fun getUserCreatedPlaylists(start=0, size=100): String` | (なし) | `disslist` (创建歌单) 取得。`uin` で本人作成分のみ。 |
| `getUserAvatarUrl` | `fun getUserAvatarUrl(): String?` | (なし) | 永続クッキーの `psrf_qqaccess_token` 等から組み立てた静的 URL。 |
| `getUserProfile` | `suspend fun getUserProfile(): String` | (なし) | `/splcloud/fcgi-bin/getuserinfo.fcg`。 |
| `getPlaylistDetail` | `suspend fun getPlaylistDetail(dissid, songBegin=0, songNum=30, ownerUin?, uin?, format='json'): String` | (なし) | `/cgi-bin/musicu.fcg` (`cgi_musics_rsp Playlist`)。 |
| `getSongDownloadUrl` | `suspend fun getSongDownloadUrl(songMid, songtype=0, filename?): String` | (なし) | **未署名 POST**。Repository 側で `QQMusicEncryptInterceptor` を経た OkHttpClient が `musics.fcg` を叩く想定。 |

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `extractUin` | `private fun extractUin(): String` | `persistedCookies["uin"]` から取得 (未ログインなら "0")。 |
| `extractKeyst` | `private fun extractKeyst(): String` | `persistedCookies["qm_keyst"]` から取得。 |
| `makeGetRequest` | `private fun makeGetRequest(url: String): String` | GET 実行。`response.body.bytes()` を返す。 |
| `makePostRequest` | `private fun makePostRequest(url: String, payload: JSONObject): String` | POST 実行。`Content-Type: application/x-www-form-urlencoded` (URL 構築は Repository 側で `param` をエンコードしたクエリ文字列を渡す運用)。 |
| `decompressIfNeeded` | `private fun decompressIfNeeded(data: ByteArray): String` | レスポンス先頭バイトが `0x78 0x9C` / `0x78 0x01` / `0x78 0xDA` (zlib) なら `InflaterInputStream` で展開。それ以外は UTF-8 デコード。 |
| `buildPersistedCookieHeader` | `private fun buildPersistedCookieHeader(): String?` | `uin=...; qm_keyst=...; ...` 形式に整形。 |
| `seedCookieJar` | `private fun seedCookieJar(host: String)` | ホスト別に永続クッキーを CookieJar に投入。 |
| `logCookieKeyDiagnostics` | `private fun logCookieKeyDiagnostics(stage: String)` | 必須キー (`uin`, `qm_keyst`, `euin`, `psrf_qqaccess_token`) の有無を Timber で警告出力。 |

### CookieJar 実装 (`saveFromResponse` / `loadForRequest`)

Netease と同じパターン:
- `saveFromResponse`: 同名 Cookie を置換。
- `loadForRequest`: ホスト別 Cookie を返却。

## 2. 内部実装メモ

### `musics.fcg` の特殊性

`getSongDownloadUrl` は **API 仕様上は暗号化必須**:
- リクエスト: `sign` クエリ + AES-GCM 暗号化 Base64 body (`ag-1` encoding)
- レスポンス: `vm_new.js` の `__cgiDecrypt` で復号、または XOR fallback

しかし本ファイル単体では署名・暗号化を実装していない (インターセプタ層に委譲)。つまり:
- 本サービスから `getSongDownloadUrl` を呼ぶ = 平文 HTTP リクエストを送る。
- Repository 層 (`QqMusicRepository.requestPurl`) で、専用の `OkHttpClient` (Interceptor 付き) を別途組み立てて、それ経由で `getSongDownloadUrl` を呼ぶ。

### Deflate レスポンス

QQ Music は稀に zlib 圧縮レスポンスを返す。`decompressIfNeeded` でマジックバイトを見て自動展開。

### 必須クッキー

| キー | 用途 |
| --- | --- |
| `uin` | QQ 番号 (ログイン対象ユーザ識別) |
| `qm_keyst` | QQ Music 認証トークン (モバイルクライアント由来) |
| `euin` | Encrypted UIN |
| `psrf_qqaccess_token` | Web 用 access_token |
| `p_skey` / `skey` | GTK 計算の種 |

## 3. 関連ファイル

- 上位層: [`streaming-qqmusic.md`](./streaming-qqmusic.md) — 暗号化・署名 (WebView JS) を含む完全なストリーミング仕様
- 暗号化: `data/remote/qqmusic/{QQMusicSecurity.kt, QQMusicEncryptInterceptor.kt, QQSignGenerator.kt}`
- Cookie: `data/preferences/EncryptedSharedPreferences` (`QqMusicRepository` 経由)
- セキュリティ: [`streaming-cloud.md`](./streaming-cloud.md)
