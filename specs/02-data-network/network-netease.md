# network-netease.md

> Netease Cloud Music (网易云音乐) 暗号化 API クライアント。
> NeriPlayer (GPL-3.0) の NeteaseClient/NeteaseCrypto 設計を踏襲。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/network/netease/
├─ CryptoMode.kt           // 4 つの暗号化モード enum (16 lines)
├─ NeteaseEncryption.kt    // AES / RSA / MD5 ユーティリティ (194 lines)
└─ NeteaseApiService.kt    // OkHttp + CookieJar クライアント (350 lines)
```

## ファイル概要

| ファイル | 行数 | 概要 |
| --- | ---: | --- |
| `CryptoMode.kt` | 16 | `WEAPI` / `EAPI` / `LINUX` / `API` を表す enum。 |
| `NeteaseEncryption.kt` | 194 | AES-CBC/ECB + RSA + MD5 + JSON シリアライザ。 |
| `NeteaseApiService.kt` | 350 | Cookie 管理・4 つの暗号化モード対応のリクエスト実行。 |

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `data/netease/NeteaseRepository.kt`, `data/netease/NeteaseStreamProxy.kt` |
| 下流 (依存先) | `org.json.JSONObject`, `okhttp3.{Cookie, CookieJar, FormBody, OkHttpClient}`, `java.security.{MessageDigest, SecureRandom, KeyFactory}`, `javax.crypto.{Cipher, IvParameterSpec, SecretKeySpec}`, `java.util.UUID` (なし) |

## 1. `CryptoMode`

```kotlin
enum class CryptoMode {
    WEAPI,   // weapi (Web) — Double AES-CBC + RSA
    EAPI,    // eapi (Encrypted API) — AES-ECB + MD5
    LINUX,   // linuxapi — AES-ECB (Linux client 互換)
    API      // /api/* — 平文パラメータ (Cookie のみ)
}
```

> 4 つのモードは公式のエンドポイントごとに要求される暗号方式が分かれるため、URL パスで自動分岐 (`callWeApi` / `callEApi` / `callLinuxApi` / 内部 `request(..., mode = CryptoMode.API)`)。

## 2. `NeteaseEncryption` (`object`)

### 定数

| 定数 | 値 | 役割 |
| --- | --- | --- |
| `BASE62` | `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789` | ランダム鍵生成用 |
| `PRESET_KEY` | `"0CoJUm6Qyw8W8jud"` | WEAPI 第 1 段 AES 鍵 |
| `IV` | `"0102030405060708"` | WEAPI 第 1/2 段共通 IV |
| `LINUX_KEY` | `"rFgB&h#%2?^eDg:Q"` | Linux API AES 鍵 |
| `EAPI_KEY` | `"e82ckenh8dichen8"` | EAPI AES 鍵 |
| `EAPI_FORMAT` | `"%s-36cd479b6b5-%s-36cd479b6b5-%s"` | EAPI メッセージ書式 |
| `EAPI_SALT` | `"nobody%suse%smd5forencrypt"` | EAPI MD5 salt 書式 |
| `PUBLIC_KEY_PEM` | 1024-bit RSA 公開鍵 (PEM) | WEAPI `encSecKey` 生成用 |

### 公開メソッド

| メソッド | シグネチャ | 目的 / 戻り値 |
| --- | --- | --- |
| `md5Hex` | `fun md5Hex(data: String): String` | `MessageDigest("MD5")` を hex 文字列化。EAPI の salt 計算用。 |
| `weApiEncrypt` | `fun weApiEncrypt(payload: Map<String, Any>): Map<String, String>` | Web API 用暗号化。`params` と `encSecKey` を返す。 |
| `eApiEncrypt` | `fun eApiEncrypt(url: String, payload: Map<String, Any>): Map<String, String>` | 暗号化 API 用。`params` (hex 文字列) を返す。 |
| `linuxApiEncrypt` | `fun linuxApiEncrypt(payload: Map<String, Any>): Map<String, String>` | Linux API 用。`eparams` (hex 文字列) を返す。 |

### WEAPI 暗号化フロー (`weApiEncrypt`)

```
1. payload → toJson → json
2. secretKey = randomKey()  // 16 文字の BASE62 ランダム
3. enc1   = AES-CBC(PRESET_KEY, IV).encrypt(json)      → base64
4. params = AES-CBC(secretKey, IV).encrypt(enc1)       → base64
5. encSecKey = RSA(reverse(secretKey))                 → hex
6. return { "params": params, "encSecKey": encSecKey }
```

### EAPI 暗号化フロー (`eApiEncrypt`)

```
1. payload → toJson → data
2. apiUrl = url.replace("/eapi", "/api")
3. message = "${apiUrl}-36cd479b6b5-${data}-36cd479b6b5-${md5("nobody${apiUrl}use${data}md5forencrypt")}"
4. cipher = AES-ECB(EAPI_KEY).encrypt(message)        → hex (uppercase)
5. return { "params": cipher }
```

### Linux API 暗号化フロー (`linuxApiEncrypt`)

```
1. payload → toJson → eparams
2. return { "eparams": AES-ECB(LINUX_KEY).encrypt(eparams) → hex }
```

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `randomKey` | `private fun randomKey(): String` | 16 文字の BASE62 ランダム。 |
| `aesEncrypt` | `private fun aesEncrypt(text, key, iv, mode, format): String` | `AES/CBC/PKCS7Padding` または `AES/ECB/PKCS7Padding`。`format = "base64"` または `"hex"`。 |
| `rsaEncrypt` | `private fun rsaEncrypt(text: String): String` | 公開鍵で `BigInteger.modPow` して 16 進文字列化。先頭の `0x00` バイトを除去。 |
| `toJson` | `private fun toJson(map: Map<String, Any>): String` | Gson を使わず独自実装 (NeriPlayer 流)。 |
| `toJsonValue` | `private fun toJsonValue(v: Any?): String` | 再帰: null / String / Number / Boolean / Map / List。 |
| `jsonQuote` | `private fun jsonQuote(s: String): String` | 制御文字のエスケープ (`\n`/`\r`/`\t`/`\b`/`\f`/`\\`/`\"`/`\uXXXX`)。 |

### RSA 暗号化の詳細

- **対象**: `secretKey.reversed()` を `BigInteger(1, bytes)` で数値化 → `modPow(publicExponent, modulus)`。
- **結果**: バイト列 (長さ = modulus bit 長 / 8) → 先頭 `0x00` バイトを除去 → 16 進文字列 (lower)。
- **lgtm 警告抑制**: `AES/CBC/PKCS7Padding` と `AES/ECB/PKCS7Padding` は Netease 公開 API の wire format 要件のため `@SuppressLint("GetInstance")` で警告抑制。

## 3. `NeteaseApiService`

### 定数

| 定数 | 値 |
| --- | --- |
| `TAG` | `"NeteaseApi"` |

### 状態

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `cookieStore` | `MutableMap<String, MutableList<Cookie>>` | ホスト別インメモリ CookieJar。 |
| `persistedCookies` | `@Volatile Map<String, String>` | EncryptedSharedPreferences から復元した永続クッキー (`MUSIC_U` 等)。 |
| `okHttpClient` | `OkHttpClient` | `cookieJar` で上記ストアを使用。15s/15s タイムアウト。 |

### 公開 API メソッド

#### 認証情報管理

| メソッド | シグネチャ | 目的 / 戻り値 |
| --- | --- | --- |
| `hasLogin` | `fun hasLogin(): Boolean` | `persistedCookies["MUSIC_U"]` が空文字でなければログイン済み。 |
| `setPersistedCookies` | `fun setPersistedCookies(cookies: Map<String, String>)` | `os=pc`, `appver=8.10.35` を自動補完し、`music.163.com` と `interface.music.163.com` の CookieJar にシード。 |
| `getCookies` | `fun getCookies(): Map<String, String>` | 現在の CookieJar のスナップショット。 |
| `logout` | `fun logout()` | `cookieStore.clear()` + `persistedCookies = emptyMap()`。 |
| `ensureWeapiSession` | `fun ensureWeapiSession()` | `https://music.163.com/` を `CryptoMode.API` で GET し、`__csrf` Cookie を取得。 |

#### 低レベルリクエスト

| メソッド | シグネチャ | 目的 / 戻り値 |
| --- | --- | --- |
| `request` | `fun request(url, params: Map<String, Any>, mode = CryptoMode.WEAPI, method = "POST", usePersistedCookies = true): String` | コアメソッド。`mode` に応じて暗号化パラメータを生成し、POST/GET を実行。`csrf_token` は WEAPI 時に自動付与。 |
| `callWeApi` | `fun callWeApi(path, params, usePersistedCookies = true): String` | `https://music.163.com/weapi{path}` を WEAPI で叩くショートカット。 |
| `callEApi` | `fun callEApi(path, params, usePersistedCookies = true): String` | `https://music.163.com/eapi{path}` を EAPI で叩くショートカット。 |
| `callLinuxApi` | (Repository 経由、`NeteaseApiService` には存在しない) — 内部的に `request(mode = CryptoMode.LINUX)` を使う |

#### ドメインショートカット (一部抜粋)

| メソッド | シグネチャ | 暗号化モード | 目的 |
| --- | --- | --- | --- |
| `sendCaptcha` | `fun sendCaptcha(phone, ctcode = 86): String` | EAPI | 携帯認証 SMS 送信 (`/eapi/captcha/sent`)。 |
| `loginByCaptcha` | `fun loginByCaptcha(phone, captcha, ctcode = 86): String` | EAPI | キャプチャコードでログイン。 |
| `getCurrentUserAccount` | `fun getCurrentUserAccount(): String` | EAPI | `/eapi/nuser/account/get`。 |
| `getCurrentUserId` | `fun getCurrentUserId(): Long` | (内部) | `getCurrentUserAccount()` から userId 抽出。 |
| `getUserPlaylists` | `fun getUserPlaylists(userId, offset = 0, limit = 50): String` | WEAPI | `/weapi/user/playlist`。 |
| `getPlaylistDetail` | `fun getPlaylistDetail(playlistId): String` | WEAPI | `/weapi/v6/playlist/detail`。 |
| `getSongDetails` | `fun getSongDetails(songIds: List<Long>): String` | EAPI | `/eapi/v3/song/detail`。 |
| `getSongDownloadUrl` | `fun getSongDownloadUrl(songId, level = "exhigh"): String` | EAPI | `/eapi/song/enhance/download/url`。`lossless`/`jyeffect` → `encodeType=flac`、それ以外 → `mp3`。失敗時に quality フォールバック。 |
| `searchSongs` | `fun searchSongs(keyword, limit = 30, offset = 0): String` | WEAPI | `/weapi/search/get`。 |
| `getLyrics` | `fun getLyrics(songId): String` | EAPI | `/eapi/song/lyric`。 |

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `saveFromResponse` | `override fun saveFromResponse(url: HttpUrl, cookies: List<Cookie>)` | CookieJar: 同じ名前の古い Cookie を置換。 |
| `loadForRequest` | `override fun loadForRequest(url: HttpUrl): List<Cookie>` | ホスト名で lookup。 |
| `seedCookieJarFromPersisted` | `private fun seedCookieJarFromPersisted(host: String)` | `persistedCookies` から指定ホスト向けに Cookie を再構築。 |
| `getCookie` | `private fun getCookie(name: String): String?` | 全ホスト横断で名前検索。 |
| `buildPersistedCookieHeader` | `private fun buildPersistedCookieHeader(): String?` | `"k1=v1; k2=v2"` 形式に整形。空なら `null`。 |

### User-Agent

固定文字列:
```
Mozilla/5.0 (Linux; Android 14; PixelPlayer) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Mobile Safari/537.36
```
Referer: `https://music.163.com`

## 4. 内部実装メモ

### クッキー運用

- CookieJar は **インメモリのみ**。アプリ再起動後は `setPersistedCookies` で復元が必要 (Repository の `loginWithCookies` から呼ばれる)。
- `MUSIC_U` が空 = 未ログイン。
- `__csrf` は WEAPI で必須。`ensureWeapiSession` で取得するか、前回ログイン時の永続値を使う。

### レート制限対策 (Repository 側)

`NeteaseApiService` 自体にレート制御はないが、`NeteaseRepository` 側で:
- `songUrlRequestCooldownMs = 1500` ms/曲
- `globalSongUrlRequestIntervalMs = 1100` ms (全曲)
- `inFlightSongUrlRequests` で同一曲多重リクエストを 1 つに集約

詳細は [`streaming-netease.md`](./streaming-netease.md) 参照。

### 暗号化モード選択の実用的指針

| 用途 | モード | 理由 |
| --- | --- | --- |
| 検索・プレイリスト取得 | WEAPI | 一般公開 API。 |
| 楽曲 URL 取得・歌詞・ログイン | EAPI | 認証必須エンドポイント。 |
| Linux クライアント互換 | LINUX | 一部の隠しエンドポイント用 (現状未使用)。 |
| `https://music.163.com/` の HTML 取得 | API | 暗号化なし、`__csrf` Cookie 取得目的。 |

## 5. 関連ファイル

- 上位層: [`streaming-netease.md`](./streaming-netease.md)
- 認証/クッキー: `data/netease/NeteaseRepository.kt` の `loginWithCookies()`, `initFromSavedCookies()`
- 暗号化関連: 同ディレクトリの `NeteaseEncryption.kt` / `CryptoMode.kt`
- セキュリティ: [`streaming-cloud.md`](./streaming-cloud.md)
