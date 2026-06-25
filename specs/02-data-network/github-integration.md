# github-integration.md

> GitHub-hosted 静的アセット取得 (Play Store 案内・コントリビュータ一覧)。
> 公開 API で API キー不要。

## パッケージ

```
app/src/main/java/com/theveloper/pixelplay/data/github/
├─ GitHubAnnouncementPropertiesService.kt   // 99 lines
└─ GitHubContributorService.kt              // 62 lines
```

## 依存関係

| 方向 | ファイル |
| --- | --- |
| 上流 (呼び出し元) | `presentation/screens/AboutScreen.kt`, `di/AppModule.kt`, `MainActivity.kt` |
| 下流 (依存先) | `java.net.HttpURLConnection`, `kotlinx.serialization.Serializable`, `kotlinx.serialization.json.Json` |

---

## 1. `GitHubAnnouncementPropertiesService`

### 役割

`https://raw.githubusercontent.com/.../playstore_announcement.properties` を取得し、`PlayStoreAnnouncementRemoteConfig` を返す。

### 内部データクラス

```kotlin
data class PlayStoreAnnouncementRemoteConfig(
    val enabled: Boolean = false,
    val playStoreUrl: String? = null,
    val title: String? = null,
    val body: String? = null,
    val primaryActionLabel: String? = null,
    val dismissActionLabel: String? = null,
    val linkPendingMessage: String? = null,
)
```

### 公開 API

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `fetchPlayStoreAnnouncement` | `suspend fun fetchPlayStoreAnnouncement(refreshAfterDays: Float = 1f, lastFetchedAt: Long): Result<PlayStoreAnnouncementRemoteConfig>` | キャッシュ判定 + HTTP 取得 + `Properties` パース。 |

### 動作

```
1. now = System.currentTimeMillis()
2. refreshAfterMs = (refreshAfterDays * 24 * 60 * 60 * 1000).toLong()
3. if lastFetchedAt > 0 && now - lastFetchedAt < refreshAfterMs: Result.failure
4. rawUrl = "https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<path>"
5. URL(rawUrl).openConnection() as HttpURLConnection
6. connectTimeout = readTimeout = 15_000ms
7. User-Agent: "PixelPlayer-Android"
8. responseCode に応じて:
   200: inputStream から text 取得 → Properties().load(StringReader(text))
       → config = PlayStoreAnnouncementRemoteConfig(
           enabled = props.booleanFlag("enabled"),
           playStoreUrl = props.stringValue("play_store_url"),
           title = props.stringValue("title"),
           ...
       )
       → Result.success(config)
   403: Result.failure(HttpException)
   その他: Result.failure
9. catch (e: Exception) → Result.failure(e)
```

### 非公開ヘルパ

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `Properties.stringValue(key)` | `private fun Properties.stringValue(key): String?` | 空文字を null に正規化。 |
| `Properties.booleanFlag(key)` | `private fun Properties.booleanFlag(key): Boolean` | `"true"`, `"1"`, `"yes"` を true 判定。 |

### 内部実装メモ

- **`.properties` 形式**: Java の標準 Properties ファイル。例:
  ```properties
  enabled=true
  play_store_url=https://play.google.com/...
  title=Now available on Play Store!
  body=Download the official release.
  primary_action_label=Open Play Store
  dismiss_action_label=Maybe later
  link_pending_message=Opening Play Store...
  ```
- **キャッシュ層は呼び出し元**: `MainActivity` などで `lastFetchedAt` を SharedPreferences に保存し、`refreshAfterDays` 間隔で再フェッチ。

---

## 2. `GitHubContributorService`

### 役割

GitHub REST API からコントリビュータ一覧 (`/repos/{owner}/{repo}/contributors`) を取得。

### 内部データクラス

```kotlin
@Serializable
data class GitHubContributor(
    val login: String,
    @SerialName("avatar_url") val avatar_url: String,
    @SerialName("html_url") val html_url: String,
    val contributions: Int,
    val type: String = "User"
)
```

### 状態

| フィールド | 説明 |
| --- | --- |
| `json` | `Json { ignoreUnknownKeys = true }` |

### 公開 API

| メソッド | シグネチャ | 目的 |
| --- | --- | --- |
| `fetchContributors` | `suspend fun fetchContributors(owner: String, repo: String): Result<List<GitHubContributor>>` | `https://api.github.com/repos/<owner>/<repo>/contributors` を取得 + `contributions` 降順ソート。 |

### 動作

```
1. URL("https://api.github.com/repos/$owner/$repo/contributors").openConnection() as HttpURLConnection
2. requestMethod = "GET"
3. setRequestProperty("Accept", "application/vnd.github+json")
4. setRequestProperty("User-Agent", "PixelPlayer-Android")  // GitHub API は UA 必須
5. responseCode == 200:
   text = inputStream.bufferedReader().readText()
   contributors = json.decodeFromString<List<GitHubContributor>>(text)
   sorted = contributors.sortedByDescending { it.contributions }
   Result.success(sorted)
6. else: Result.failure(HttpException(...))
```

### エラーハンドリング

| ステータス | 挙動 |
| --- | --- |
| 200 | 成功 |
| 403 | レート制限。`Result.failure`。 |
| 404 | リポジトリ不存在。`Result.failure`。 |
| その他 | `Result.failure`。 |

### 内部実装メモ

- **User-Agent 必須**: GitHub API は `User-Agent` ヘッダ無しリクエストを拒否。
- **匿名コントリビュータ**: `type` フィールドは `User` (アカウント有) / `Bot` を区別。`Anonymous` (login=null) は API から除外されるため通常現れない。
- **キャッシュ層は呼び出し元**: `AboutScreen.kt` で `LaunchedEffect` 内で呼び、`viewModel` でメモリ保持。

---

## 3. 利用パターン

### About 画面 (`presentation/screens/AboutScreen.kt`)

```kotlin
val config by produceState<PlayStoreAnnouncementRemoteConfig?>(initialValue = null) {
    val lastFetchedAt = prefs.getLong("last_announcement_fetch", 0L)
    value = announcementService.fetchPlayStoreAnnouncement(
        refreshAfterDays = 1f,
        lastFetchedAt = lastFetchedAt
    ).getOrNull()
    if (value != null) {
        prefs.edit().putLong("last_announcement_fetch", System.currentTimeMillis()).apply()
    }
}

if (config?.enabled == true) {
    AnnouncementCard(
        title = config?.title ?: "",
        body = config?.body ?: "",
        onPrimary = { openUrl(config?.playStoreUrl) },
        primaryLabel = config?.primaryActionLabel ?: "Open"
    )
}
```

### コントリビュータ一覧

```kotlin
val contributors by produceState<List<GitHubContributor>>(emptyList()) {
    value = contributorService.fetchContributors("theveloper", "PixelPlayer").getOrDefault(emptyList())
}

LazyColumn {
    items(contributors) { contributor ->
        ContributorRow(contributor)
    }
}
```

---

## 4. 関連ファイル

- UI: `presentation/screens/AboutScreen.kt`
- DI: `di/AppModule.kt`
- SharedPreferences: `data/preferences/UserPreferencesRepository.kt`
