# models.md

> `com.theveloper.pixelplay.data.model` パッケージ内の全データ クラス / Enum / Sealed Interface の詳細仕様。
> 20 ファイル。UI / Repository / Service 層から参照される **ドメイン モデル**。

## 一覧

| ファイル | 種別 | クラス / enum |
|----------|------|--------------|
| `Song.kt` | data class | `Song` |
| `PlayList.kt` | data class + enum | `Playlist`, `PlaylistShapeType` |
| `Lyrics.kt` | data class × 3 | `Lyrics`, `SyncedLine`, `SyncedWord` |
| `Transition.kt` | enum × 3 + data class × 3 | `TransitionMode`, `Curve`, `TransitionSource`, `TransitionSettings`, `TransitionResolution`, `TransitionRule` |
| `LibraryModels.kt` | data class × 3 | `Album`, `Artist`, `ArtistRef` |
| `PlayerInfo.kt` | data class × 3 | `QueueItem`, `WidgetThemeColors`, `PlayerInfo` |
| `PlaybackQueueSnapshot.kt` | data class × 2 | `PlaybackQueueItemSnapshot`, `PlaybackQueueSnapshot` |
| `SmartPlaylistRule.kt` | enum | `SmartPlaylistRule` |
| `SearchFilterType.kt` | enum | `SearchFilterType` |
| `SearchHistoryItem.kt` | data class | `SearchHistoryItem` |
| `SearchResultItem.kt` | sealed interface | `SearchResultItem` (+ 4 variants) |
| `SortOption.kt` | sealed class + enum | `SortOption` (40+ objects), `SortDirection` |
| `StorageFilter.kt` | enum | `StorageFilter` |
| `LyricsSourcePreference.kt` | enum | `LyricsSourcePreference` |
| `MusicFolder.kt` | data class | `MusicFolder` |
| `FolderSource.kt` | enum | `FolderSource` |
| `DirectoryItem.kt` | data class | `DirectoryItem` |
| `Genre.kt` | data class | `Genre` |
| `LibraryTabId.kt` | enum + 拡張 | `LibraryTabId`, `String.toLibraryTabIdOrNull()` |
| `SortOptionTest.kt` | テスト | `SortOption.fromStorageKey` の null 防御テスト (コメントアウト) |

---

## 1. `Song.kt`

| パッケージ | `com.theveloper.pixelplay.data.model` |
|-----------|--------------------------------------|
| 役割 | 曲データクラス。全ソース統合の中核 |
| 上流 | Repository, UI, Service, Wear OS 共有 |
| 下流 | `androidx.compose.runtime.Immutable`, `kotlinx.parcelize.Parcelize` |

### `data class Song`

`@Immutable @Parcelize data class`。`id: String` (MediaStore 由来の Long を文字列化、または `telegram_xxx` 等の派生)。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `id` | `String` | — | 一意 ID |
| `title` | `String` | — | 曲タイトル |
| `artist` | `String` | — | 表示用 (主) アーティスト名 |
| `artistId` | `Long` | — | 主アーティスト ID (後方互換) |
| `artists` | `List<ArtistRef>` | `emptyList()` | 全アーティスト (マルチ アーティスト) |
| `album` | `String` | — | アルバム名 |
| `albumId` | `Long` | — | アルバム ID |
| `albumArtist` | `String?` | `null` | TPE2 タグ由来 |
| `path` | `String` | — | ファイル パス (MediaStore 用) |
| `contentUriString` | `String` | — | `content://` または `xxx://` カスタム |
| `albumArtUriString` | `String?` | `null` | ジャケット URI |
| `duration` | `Long` | — | ms |
| `genre` | `String?` | `null` | — |
| `lyrics` | `String?` | `null` | 埋め込み歌詞 |
| `isFavorite` | `Boolean` | `false` | — |
| `trackNumber` | `Int` | `0` | — |
| `discNumber` | `Int?` | `null` | — |
| `year` | `Int` | `0` | — |
| `dateAdded` | `Long` | `0` | epoch ms |
| `dateModified` | `Long` | `0` | epoch ms |
| `mimeType` | `String?` | — | MIME |
| `bitrate` | `Int?` | — | bps |
| `sampleRate` | `Int?` | — | Hz |
| `telegramFileId` | `Int?` | `null` | Telegram ファイル ID |
| `telegramChatId` | `Long?` | `null` | Telegram チャット ID |
| `neteaseId` | `Long?` | `null` | Netease 数字 ID |
| `gdriveFileId` | `String?` | `null` | Google Drive ファイル ID |
| `qqMusicMid` | `String?` | `null` | QQ Music MID |
| `navidromeId` | `String?` | `null` | Navidrome ID |
| `jellyfinId` | `String?` | `null` | Jellyfin ID |

### 派生プロパティ

| 名前 | 型 | 説明 |
|------|----|----|
| `displayArtist` | `String` | `artists` が非空なら `sortedByDescending(isPrimary)` → `joinToString(", ")`。空なら `artist` を返す |
| `primaryArtist` | `ArtistRef` | `artists.find { isPrimary }`、無ければ `artists.firstOrNull()`、さらに無ければ `ArtistRef(artistId, artist, isPrimary=true)` |

### `companion object`

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `emptySong()` | `Song` | 全てデフォルトの "空" 曲。`id = "-1"`、`mimeType = "-"` |

---

## 2. `PlayList.kt`

| パッケージ | `com.theveloper.pixelplay.data.model` |
|-----------|--------------------------------------|

### `data class Playlist`

`@Immutable @Serializable data class`。`id: String` (UUID 等)。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `id` | `String` | — | 一意 ID |
| `name` | `String` | — | プレイリスト名 |
| `songIds` | `List<String>` | — | 曲 ID リスト (順序保持) |
| `createdAt` | `Long` | `currentTimeMillis()` | — |
| `lastModified` | `Long` | `currentTimeMillis()` | — |
| `isAiGenerated` | `Boolean` | `false` | AI 生成フラグ |
| `isQueueGenerated` | `Boolean` | `false` | キュー自動生成フラグ |
| `coverImageUri` | `String?` | `null` | カバー画像 |
| `coverColorArgb` | `Int?` | `null` | カバー ARGB |
| `coverIconName` | `String?` | `null` | — |
| `coverShapeType` | `String?` | `null` | "Circle"/"SmoothRect"/"RotatedPill"/"Star" |
| `coverShapeDetail1` | `Float?` | `null` | CornerRadius / StarCurve |
| `coverShapeDetail2` | `Float?` | `null` | Smoothness / StarRotation |
| `coverShapeDetail3` | `Float?` | `null` | StarScale |
| `coverShapeDetail4` | `Float?` | `null` | Star Sides (Int) |
| `source` | `String` | `"LOCAL"` | "LOCAL"/"NETEASE"/"TELEGRAM"/"AI" 等 |

### `enum class PlaylistShapeType`

`Circle`, `SmoothRect`, `RotatedPill`, `Star` の 4 値。

---

## 3. `Lyrics.kt`

| パッケージ | `com.theveloper.pixelplay.data.model` |
|-----------|--------------------------------------|

### `data class Lyrics`

`@Serializable`。歌詞コンテナ。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `plain` | `List<String>?` | `null` | プレーンテキスト行 |
| `synced` | `List<SyncedLine>?` | `null` | 同期歌詞 |
| `areFromRemote` | `Boolean` | `false` | リモート取得フラグ |

### `data class SyncedLine`

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `time` | `Int` | — | 開始時刻 ms |
| `line` | `String` | — | 歌詞テキスト |
| `words` | `List<SyncedWord>?` | `null` | 単語単位タイムスタンプ (オプション) |
| `translation` | `String?` | `null` | 翻訳 (同一 time キーでペアリング) |
| `romanization` | `String?` | `null` | ローマ字 (同一 time キーでペアリング) |

### `data class SyncedWord`

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `time` | `Int` | — | 単語開始時刻 ms |
| `word` | `String` | — | 単語 |
| `startsNewWord` | `Boolean` | `true` | 新単語開始フラグ |

---

## 4. `Transition.kt`

| パッケージ | `com.theveloper.pixelplay.data.model` |
|-----------|--------------------------------------|

### `enum class TransitionMode`

曲間遷移モード。

| 値 | 説明 |
|----|----|
| `NONE` | 遷移なし |
| `FADE_IN_OUT` | 旧曲完全フェードアウト後に新曲フェードイン |
| `OVERLAP` | 旧曲フェードアウトと新曲フェードインをオーバーラップ |
| `SMOOTH` | S 字カーブで滑らかなフェード |

### `enum class Curve`

音量カーブ。

| 値 | 説明 |
|----|----|
| `LINEAR` | リニア |
| `EXP` | 指数 (急開始、緩終了) |
| `LOG` | 対数 (緩開始、急終了) |
| `S_CURVE` | シグモイド (S 字) |

### `enum class TransitionSource`

| 値 | 説明 |
|----|----|
| `GLOBAL_DEFAULT` | グローバル既定 |
| `PLAYLIST_DEFAULT` | プレイリスト全体既定 |
| `PLAYLIST_SPECIFIC` | 特定トラック ペア |

### `data class TransitionSettings`

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `mode` | `TransitionMode` | `OVERLAP` | — |
| `durationMs` | `Int` | `2000` | 遷移時間 ms |
| `curveIn` | `Curve` | `S_CURVE` | 新曲側 |
| `curveOut` | `Curve` | `S_CURVE` | 旧曲側 |

### `data class TransitionResolution`

| フィールド | 型 | 説明 |
|-----------|----|----|
| `settings` | `TransitionSettings` | 解決された設定 |
| `source` | `TransitionSource` | 解決元 |

### `data class TransitionRule`

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `id` | `Long` | `0` | — |
| `playlistId` | `String` | — | プレイリスト ID |
| `fromTrackId` | `String?` | `null` | 遷移元。null の場合はプレイリスト全体既定 |
| `toTrackId` | `String?` | `null` | 遷移先 |
| `settings` | `TransitionSettings` (`@Embedded`) | — | 設定 |

---

## 5. `LibraryModels.kt`

| パッケージ | `com.theveloper.pixelplay.data.model` |
|-----------|--------------------------------------|

### `data class Album`

`@Immutable @Parcelize`。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `id` | `Long` | — | MediaStore `Albums._ID` |
| `title` | `String` | — | アルバム名 |
| `artist` | `String` | — | 表示用アーティスト |
| `year` | `Int` | — | リリース年 |
| `dateAdded` | `Long` | — | epoch ms |
| `albumArtUriString` | `String?` | — | ジャケット URI |
| `songCount` | `Int` | — | 曲数 |
| `albumArtist` | `String?` | `null` | TPE2 |

| `companion object.empty()` | `Album(id=-1, ...)` | 全デフォルトの空 Album |

### `data class Artist`

`@Immutable @Parcelize`。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `id` | `Long` | — | MediaStore `Artists._ID` |
| `name` | `String` | — | アーティスト名 |
| `songCount` | `Int` | — | 曲数 |
| `imageUrl` | `String?` | `null` | Deezer 等の外部画像 |
| `customImageUri` | `String?` | `null` | ユーザー指定カスタム画像 |

| 派生 | `effectiveImageUrl: String?` | `customImageUri.takeIf { isNotBlank() } ?: imageUrl.takeIf { isNotBlank() }` |
| `companion object.empty()` | `Artist(id=-1, ...)` | 全デフォルト |

### `data class ArtistRef`

`@Immutable @Parcelize`。Song 内のマルチ アーティスト参照。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `id` | `Long` | — | アーティスト ID |
| `name` | `String` | — | アーティスト名 |
| `isPrimary` | `Boolean` | `false` | 主フラグ |

---

## 6. `PlayerInfo.kt`

| パッケージ | `com.theveloper.pixelplay.data.model` |
|-----------|--------------------------------------|

### `data class QueueItem`

`@Serializable`。曲 ID とアート URI の最小ペア。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `id` | `Long` | — | 曲 ID |
| `albumArtUri` | `String?` | `null` | アート URI |

`equals` / `hashCode` を `id` + `albumArtUri` で自前実装。

### `data class WidgetThemeColors`

`@Serializable`。ウィジェット表示用ライト / ダーク カラー ペア。

| ライト グループ | フィールド | 型 |
|---------------|----------|----|
| Surface | `lightSurfaceContainer`, `lightSurfaceContainerLowest`, `lightSurfaceContainerLow`, `lightSurfaceContainerHigh`, `lightSurfaceContainerHighest` | `Int` |
| Text | `lightTitle`, `lightArtist` | `Int` |
| PlayPause | `lightPlayPauseBackground`, `lightPlayPauseIcon` | `Int` |
| Prev/Next | `lightPrevNextBackground`, `lightPrevNextIcon` | `Int` |

ダーク グループも同名で `dark` プレフィックス。

### `data class PlayerInfo`

`@Serializable`。Now Playing + Wear OS / Glance 用スナップショット。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `songTitle` | `String` | `""` | — |
| `artistName` | `String` | `""` | — |
| `isPlaying` | `Boolean` | `false` | — |
| `albumArtUri` | `String?` | `null` | — |
| `albumArtBitmapData` | `ByteArray?` | `null` | アート バイト配列 |
| `currentPositionMs` | `Long` | `0L` | 再生位置 |
| `totalDurationMs` | `Long` | `0L` | 総再生時間 |
| `isFavorite` | `Boolean` | `false` | — |
| `lyrics` | `Lyrics?` | `null` | — |
| `isLoadingLyrics` | `Boolean` | `false` | — |
| `queue` | `List<QueueItem>` | `emptyList()` | — |
| `themeColors` | `WidgetThemeColors?` | `null` | — |
| `isShuffleEnabled` | `Boolean` | `false` | — |
| `repeatMode` | `Int` | `0` | 0=OFF, 1=ONE, 2=ALL |
| `wearThemePalette` | `WearThemePalette?` | `null` | Wear 用 |
| `wearQueueRevision` | `String` | `""` | Wear キュー改訂番号 |

> `equals` / `hashCode` を `ByteArray` の `contentEquals` / `contentHashCode` で自前実装 (デフォルトの参照比較を回避)。

---

## 7. `PlaybackQueueSnapshot.kt`

| パッケージ | `com.theveloper.pixelplay.data.model` |
|-----------|--------------------------------------|

### `data class PlaybackQueueItemSnapshot`

`@Serializable`。Media3 / キュー永続化用の最小スナップショット。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `mediaId` | `String` | — | Media3 mediaId |
| `uri` | `String` | — | 再生 URI |
| `title` | `String?` | `null` | — |
| `artist` | `String?` | `null` | — |
| `albumTitle` | `String?` | `null` | — |
| `artworkUri` | `String?` | `null` | — |
| `durationMs` | `Long?` | `null` | — |

### `data class PlaybackQueueSnapshot`

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `items` | `List<PlaybackQueueItemSnapshot>` | — | キュー全件 |
| `currentMediaId` | `String?` | `null` | 現在再生中 |
| `currentIndex` | `Int` | `0` | キュー位置 |
| `currentPositionMs` | `Long` | `0L` | 再生位置 |
| `playWhenReady` | `Boolean` | `false` | — |
| `repeatMode` | `Int` | `0` | 0=OFF, 1=ONE, 2=ALL |
| `shuffleEnabled` | `Boolean` | `false` | — |
| `savedAtEpochMs` | `Long` | `currentTimeMillis()` | 永続化時刻 |

---

## 8. `SmartPlaylistRule.kt`

### `enum class SmartPlaylistRule`

`@Immutable`。スマート プレイリスト ルール。永続化キー + 表示名 + 説明。

| 値 | `storageKey` | `title` | `subtitle` |
|----|-------------|---------|-----------|
| `TOP_PLAYED` | `"top_played"` | "Top Played" | "Your most played tracks." |
| `RECENTLY_PLAYED` | `"recently_played"` | "Recently Played" | "Songs you listened to most recently." |
| `FORGOTTEN_FAVORITES` | `"forgotten_favorites"` | "Forgotten Favorites" | "Favorite tracks you haven't played in a while." |
| `NEW_GEMS` | `"new_gems"` | "New Gems" | "Recently added tracks with low play counts." |

| `companion fromStorageKey(key)` | `SmartPlaylistRule?` | `storageKey` で逆引き。null/未一致なら `null` |

---

## 9. `SearchFilterType.kt`

### `enum class SearchFilterType`

`@Immutable`。検索フィルタ種別。

| 値 | 意味 |
|----|----|
| `ALL` | 全て |
| `SONGS` | 曲のみ |
| `ALBUMS` | アルバムのみ |
| `ARTISTS` | アーティストのみ |
| `PLAYLISTS` | プレイリストのみ |

---

## 10. `SearchHistoryItem.kt`

### `data class SearchHistoryItem`

`@Immutable`。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `id` | `Long?` | `null` | DB id (新規作成時は null) |
| `query` | `String` | — | 検索クエリ |
| `timestamp` | `Long` | — | 検索時刻 epoch ms |

---

## 11. `SearchResultItem.kt`

### `sealed interface SearchResultItem`

`@Immutable`。検索結果の統一型 (4 バリアント)。

| バリアント | フィールド |
|----------|----------|
| `SongItem` | `song: Song` |
| `AlbumItem` | `album: Album` |
| `ArtistItem` | `artist: Artist` |
| `PlaylistItem` | `playlist: Playlist` |

---

## 12. `SortOption.kt`

| パッケージ | `com.theveloper.pixelplay.data.model` |
|-----------|--------------------------------------|
| 行数 | 542 |
| 役割 | ライブラリ UI のソート オプション定義 (sealed class + 40+ object) |

### `enum class SortDirection`

`Ascending` / `Descending`。

### `sealed class SortOption`

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `storageKey` | `String` | — | 永続化キー (例: `"song_title_az"`) |
| `displayName` | `String` | — | 表示名 (英語フォールバック) |
| `displayNameRes` | `Int` (`@StringRes`) | — | 文字列リソース ID |
| `methodLabel` | `String` | `displayName` | ソート手法ラベル |
| `methodLabelRes` | `Int` (`@StringRes`) | `displayNameRes` | 手法ラベル リソース |
| `methodKey` | `String` | `storageKey` | グルーピング用キー (例: `"song_title"`) |
| `direction` | `SortDirection?` | `null` | 並び方向 |

#### 派生

| 名前 | 型 | 説明 |
|------|----|----|
| `canFlipDirection` | `Boolean` | `direction != null && flipDirection().storageKey != storageKey` |
| `methodOption()` | `SortOption` | `defaultOptionByMethodKey[methodKey]` (グルーピングの代表) |
| `resolveForDirection(targetDirection)` | `SortOption` | 指定方向に一致するオプション、無ければ `methodOption()` |
| `flipDirection()` | `SortOption` | `direction` を反転したオプション |

### 定義済み `object` 一覧 (40+)

| カテゴリ | object (storageKey) |
|---------|--------------------|
| **Song** | `SongDefaultOrder`, `SongTitleAZ`, `SongTitleZA`, `SongArtist`, `SongArtistDesc`, `SongAlbum`, `SongAlbumDesc`, `SongDateAdded`, `SongDateAddedAsc`, `SongDuration`, `SongDurationAsc` |
| **Album** | `AlbumTitleAZ`, `AlbumTitleZA`, `AlbumArtist`, `AlbumArtistDesc`, `AlbumReleaseYear`, `AlbumReleaseYearAsc`, `AlbumDateAdded`, `AlbumSizeAsc`, `AlbumSizeDesc` |
| **Artist** | `ArtistNameAZ`, `ArtistNameZA`, `ArtistNumSongsDesc`, `ArtistNumSongsAsc` |
| **Playlist** | `PlaylistNameAZ`, `PlaylistNameZA`, `PlaylistDateCreated`, `PlaylistDateCreatedAsc` |
| **Liked** | `LikedSongTitleAZ`, `LikedSongTitleZA`, `LikedSongArtist`, `LikedSongArtistDesc`, `LikedSongAlbum`, `LikedSongAlbumDesc`, `LikedSongDateLiked`, `LikedSongDateLikedAsc` |
| **Folder** | `FolderNameAZ`, `FolderNameZA`, `FolderSongCountAsc`, `FolderSongCountDesc`, `FolderSubdirCountAsc`, `FolderSubdirCountDesc` |

### `companion object`

| 名前 | 型 | 説明 |
|------|----|----|
| `SONGS` | `List<SortOption>` | 曲用リスト |
| `ALBUMS` | `List<SortOption>` | アルバム用 |
| `ARTISTS` | `List<SortOption>` | アーティスト用 |
| `PLAYLISTS` | `List<SortOption>` | プレイリスト用 |
| `FOLDERS` | `List<SortOption>` | フォルダ用 |
| `LIKED` | `List<SortOption>` | お気に入り用 |
| `ALL` (private) | `List<SortOption>` | 全結合 |
| `defaultOptionByMethodKey` (private) | `Map<String, SortOption>` | `methodKey` → 代表オプション |
| `optionByMethodAndDirection` (private) | `Map<Pair<String, SortDirection>, SortOption>` | キー + 方向 → オプション |
| `fromStorageKey(rawValue, allowed, fallback)` | `SortOption` | `storageKey` で逆引き。レガシー `displayName` フォールバック、空 / 未一致なら `fallback` |

---

## 13. `StorageFilter.kt`

### `enum class StorageFilter(val value: Int)`

| 値 | `value` | 意味 |
|----|--------|----|
| `ALL` | `0` | 全曲 |
| `OFFLINE` | `1` | ローカルのみ (`source_type = 0`) |
| `ONLINE` | `2` | クラウド ソースのみ (`source_type != 0`) |

> `MusicDao` の `filterMode` パラメータに直接渡される。

---

## 14. `LyricsSourcePreference.kt`

### `enum class LyricsSourcePreference(val displayName: String)`

歌詞取得の優先順位ポリシー。

| 値 | `displayName` | 説明 |
|----|--------------|----|
| `API_FIRST` | "Online First" | API → 埋め込み → ローカル `.lrc` |
| `EMBEDDED_FIRST` | "Embedded First" | 埋め込み → API → ローカル `.lrc` |
| `LOCAL_FIRST` | "Local First" | ローカル `.lrc` → 埋め込み → API |

### `companion object`

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `fromOrdinal(ordinal)` | `LyricsSourcePreference` | ordinal → 値。範囲外なら `EMBEDDED_FIRST` |
| `fromName(name)` | `LyricsSourcePreference` | name → 値。null / 未一致なら `EMBEDDED_FIRST` |

---

## 15. `MusicFolder.kt`

### `data class MusicFolder`

階層フォルダ ツリー用。再帰的に song / subFolder を持つ。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `path` | `String` | — | フォルダ パス |
| `name` | `String` | — | フォルダ名 |
| `songs` | `ImmutableList<Song>` | `persistentListOf()` | 直下の曲 |
| `subFolders` | `ImmutableList<MusicFolder>` | `persistentListOf()` | サブフォルダ |

| 派生 | 型 | 説明 |
|------|----|----|
| `totalSongCount` | `Int` | `songs.size + subFolders.sumOf { totalSongCount }` |
| `totalSubFolderCount` | `Int` | `subFolders.size + subFolders.sumOf { totalSubFolderCount }` |

> `kotlinx.collections.immutable` (PersistentList) を使用。

---

## 16. `FolderSource.kt`

### `enum class FolderSource(val storageKey: String, val displayName: String)`

| 値 | `storageKey` | `displayName` |
|----|-------------|--------------|
| `INTERNAL` | `"internal"` | "Internal Storage" |
| `SD_CARD` | `"sd_card"` | "SD Card" |

### `companion object.fromStorageKey(rawValue)`

`storageKey` で逆引き、null / 未一致なら `INTERNAL`。

---

## 17. `DirectoryItem.kt`

### `data class DirectoryItem`

`@Immutable`。ディレクトリ選択 UI 用。

| フィールド | 型 | 説明 |
|-----------|----|----|
| `path` | `String` | 絶対パス |
| `isAllowed` | `Boolean` | ライブラリに含めるか |

| 派生 | 型 | 説明 |
|------|----|----|
| `displayName` | `String` | `File(path).name.ifEmpty { path }` (ルートなら path) |

---

## 18. `Genre.kt`

### `data class Genre`

`@Immutable`。ジャンル エントリ。

| フィールド | 型 | デフォルト | 説明 |
|-----------|----|----------|------|
| `id` | `String` | — | ジャンル ID |
| `name` | `String` | — | 表示名 |
| `iconResId` | `Int?` | `null` | Material アイコン drawable |
| `lightColorHex` | `String?` | `null` | ライト カラー 16進 |
| `onLightColorHex` | `String?` | `null` | ライト on カラー 16進 |
| `darkColorHex` | `String?` | `null` | ダーク カラー 16進 |
| `onDarkColorHex` | `String?` | `null` | ダーク on カラー 16進 |

---

## 19. `LibraryTabId.kt`

### `enum class LibraryTabId(val storageKey, val title, @StringRes val titleRes, val defaultSort)`

`@Immutable`。ライブラリ画面タブ識別。

| 値 | `storageKey` | `defaultSort` |
|----|-------------|--------------|
| `SONGS` | `"SONGS"` | `SortOption.SongTitleAZ` |
| `ALBUMS` | `"ALBUMS"` | `SortOption.AlbumTitleAZ` |
| `ARTISTS` | `"ARTIST"` | `SortOption.ArtistNameAZ` |
| `PLAYLISTS` | `"PLAYLISTS"` | `SortOption.PlaylistNameAZ` |
| `FOLDERS` | `"FOLDERS"` | `SortOption.FolderNameAZ` |
| `LIKED` | `"LIKED"` | `SortOption.LikedSongDateLiked` |

### `companion object.fromStorageKey(key)`

`storageKey` で逆引き、未一致なら `SONGS`。

### `String.toLibraryTabIdOrNull()` (拡張関数)

`storageKey == this` で `LibraryTabId` を返す。`String` レイヤ (SharedPreferences 等) から変換用。

---

## 20. `SortOptionTest.kt`

`SortOption.fromStorageKey` の null 防御テスト。コメントアウト中 (推測 — `*.kt` ファイルだが内容は不明)。

> 実装コードには影響しない。テスト用に分離されたファイル。

---

## 内部実装メモ

### Parcelize + Compose Immutable

- データクラスは原則 `@Parcelize @Immutable` を併用
- Wear OS 共有用 (`shared/`) に `PlayerInfo`, `QueueItem`, `WidgetThemeColors` は `@Serializable` も付与

### `Song` の source 識別

- `id` プレフィックスではなく、`contentUriString.startsWith("telegram://")` 等の文字列マッチでソース識別
- `SourceType.fromContentUri` の方が高速だが、`Song` モデルには `source_type` 列がない (派生情報のため)

### 多言語 / 文字列リソース

- `SortOption.displayNameRes` / `LibraryTabId.titleRes` は `@StringRes` Int で `R.string.*` を参照
- 英語 `displayName` はフォールバック

### `equals` / `hashCode` の自前実装が必要な型

- `PlayerInfo` (`ByteArray` を `contentEquals` / `contentHashCode`)
- `QueueItem` (デフォルトが参照比較)

---

## 関連ファイル

- Entity 層: [`database-entities.md`](./database-entities.md)
- DAO 層: [`database-daos.md`](./database-daos.md)
- DB 統合 + Migration: [`database-system.md`](./database-system.md)
- 上位ディレクトリ README: [`README.md`](./README.md)
- 上流: Repository (推測: `data/repository/*`)
- 下流: Compose UI (`presentation/`), Service (`service/MusicService.kt`), Wear OS (`wear/`), Cast