# database-entities.md

> `com.theveloper.pixelplay.data.database` パッケージ内の全 **Entity / 付随データクラス** の詳細仕様。
> 28 の `@Entity` クラス、4 つの Relation / Projection クラス、3 つの変換ヘルパ、1 つの converter 関数を含む。
> DAO は `database-daos.md` を参照。Database 統合仕様は `database-system.md` を参照。

## 一覧

| ファイル | 主要クラス / オブジェクト | テーブル名 |
|----------|------------------------|-----------|
| `SongEntity.kt` | `SourceType`, `SongEntity`, `SongSummary` | `songs` |
| `AlbumEntity.kt` | `AlbumEntity` | `albums` |
| `ArtistEntity.kt` | `ArtistEntity` | `artists` |
| `SongArtistCrossRef.kt` | `SongArtistCrossRef`, `SongWithArtists`, `ArtistWithSongs`, `PrimaryArtistInfo` | `song_artist_cross_ref` |
| `PlaylistEntity.kt` | `PlaylistEntity` | `playlists` |
| `PlaylistSongEntity.kt` | `PlaylistSongEntity` | `playlist_songs` |
| `PlaylistWithSongsEntity.kt` | `PlaylistWithSongsEntity` | (Relation) |
| `AlbumArtThemeEntity.kt` | `StoredColorSchemeValues`, `AlbumArtThemeEntity` | `album_art_themes` |
| `AiCacheEntity.kt` | `AiCacheEntity` | `ai_cache` |
| `AiUsageEntity.kt` | `AiUsageEntity` | `ai_usage` |
| `FavoritesEntity.kt` | `FavoritesEntity` | `favorites` |
| `LyricsEntity.kt` | `LyricsEntity` | `lyrics` |
| `SongEngagementEntity.kt` | `SongEngagementEntity` | `song_engagements` |
| `SearchHistoryEntity.kt` | `SearchHistoryEntity` | `search_history` |
| `TransitionRuleEntity.kt` | `TransitionRuleEntity` | `transition_rules` |
| `SongSearchFtsEntity.kt` | `SongSearchFtsEntity` | `songs_fts` (FTS4 virtual) |
| `TelegramSongEntity.kt` | `TelegramSongEntity` | `telegram_songs` |
| `TelegramChannelEntity.kt` | `TelegramChannelEntity` | `telegram_channels` |
| `TelegramTopicEntity.kt` | `TelegramTopicEntity` | `telegram_topics` |
| `NeteaseSongEntity.kt` | `NeteaseSongEntity` | `netease_songs` |
| `NeteasePlaylistEntity.kt` | `NeteasePlaylistEntity` | `netease_playlists` |
| `GDriveSongEntity.kt` | `GDriveSongEntity` | `gdrive_songs` |
| `GDriveFolderEntity.kt` | `GDriveFolderEntity` | `gdrive_folders` |
| `QqMusicSongEntity.kt` | `QqMusicSongEntity` | `qqmusic_songs` |
| `QqMusicPlaylistEntity.kt` | `QqMusicPlaylistEntity` | `qqmusic_playlists` |
| `NavidromeSongEntity.kt` | `NavidromeSongEntity` | `navidrome_songs` |
| `NavidromePlaylistEntity.kt` | `NavidromePlaylistEntity` | `navidrome_playlists` |
| `JellyfinSongEntity.kt` | `JellyfinSongEntity` | `jellyfin_songs` |
| `JellyfinPlaylistEntity.kt` | `JellyfinPlaylistEntity` | `jellyfin_playlists` |
| `ColorConverters.kt` | `String.toComposeColor()` (拡張関数) | — |
| `FolderSongRow.kt` | `FolderSongRow` | (Projection) |

---

## 1. `SongEntity.kt`

| パッケージ | `com.theveloper.pixelplay.data.database` |
|-----------|----------------------------------------|
| 役割 | 統合 `songs` テーブル + `SourceType` 定数 + Entity↔Model 変換 |
| 上流 | `MusicDao`, `SyncWorker`, Repository 系 |
| 下流 | `Song` モデル, `LocalArtworkUri`, `normalizeMetadataText*` ユーティリティ |

### `object SourceType`

`source_type` 列の整数定数。`LIKE uri.startsWith(...)` チェックより高速な整数比較でソース識別。

| 定数 | 値 | 対応 URI スキーム |
|------|----|------------------|
| `LOCAL` | `0` | デフォルト。`file://`, `content://`, 等 |
| `TELEGRAM` | `1` | `telegram://` |
| `NETEASE` | `2` | `netease://` |
| `GDRIVE` | `3` | `gdrive://` |
| `QQMUSIC` | `4` | `qqmusic://` |
| `NAVIDROME` | `5` | `navidrome://` |
| `JELLYFIN` | `6` | `jellyfin://` |

| メソッド | 戻り値 | 目的 |
|---------|--------|------|
| `fromContentUri(uri: String): Int` | `Int` | URI プレフィックスから対応する `SourceType` を逆引き。`telegram://` で始まれば `TELEGRAM`、等。ローカル URI は `LOCAL` (0) を返す |

### `data class SongEntity`

`@Entity(tableName = "songs", indices = [...], foreignKeys = [...])`。`@PrimaryKey id: Long`、MediaStore の `_ID` をそのまま使用。クラウド ソース (Telegram 等) は負の Long ID で区別。

#### カラム (合計 25 列)

| カラム名 | 型 | 説明 |
|----------|----|----|
| `id` (PK) | `Long` | MediaStore 由来の主キー (または負の ID でクラウド曲) |
| `title` | `String` | 曲タイトル |
| `artist_name` | `String` | 表示用アーティスト名 (主) |
| `artist_id` | `Long` | 主アーティスト ID (後方互換用) |
| `album_artist` | `String?` | TPE2 タグ由来のアルバム アーティスト |
| `album_name` | `String` | アルバム名 |
| `album_id` | `Long` | アルバム ID (`AlbumEntity.id` FK) |
| `content_uri_string` | `String` | `content://` または `xxx://` カスタム URI |
| `album_art_uri_string` | `String?` | ジャケット画像 URI (local_art scheme / HTTP / Telegram 等) |
| `duration` | `Long` | 再生時間 (ms) |
| `genre` | `String?` | ジャンル (カンマ区切り複合可) |
| `file_path` | `String` | ファイルシステム絶対パス |
| `parent_directory_path` | `String` | 親ディレクトリ (フォルダ フィルタ用) |
| `is_favorite` | `Boolean` | お気に入りフラグ (default `0`) |
| `lyrics` | `String?` | 埋め込み歌詞 (default `null`) |
| `track_number` | `Int` | トラック番号 (default `0`) |
| `disc_number` | `Int?` | ディスク番号 (default `null`) |
| `year` | `Int` | リリース年 (default `0`) |
| `date_added` | `Long` | 追加日時 epoch ms (default `0`、生成時 `currentTimeMillis()`) |
| `mime_type` | `String?` | MIME タイプ |
| `bitrate` | `Int?` | ビットレート (bps) |
| `sample_rate` | `Int?` | サンプリング レート (Hz) |
| `telegram_chat_id` | `Long?` | Telegram チャット ID |
| `telegram_file_id` | `Int?` | Telegram ファイル ID |
| `artists_json` | `String?` | マルチ アーティスト JSON キャッシュ (`[{id,name,primary}, ...]`) |
| `source_type` | `Int` | `SourceType` 定数 (default `0` = LOCAL) |

#### Index / FK

- **Index**: `title`, `album_id`, `artist_id`, `artist_name`, `genre`, `parent_directory_path`, `file_path`, `content_uri_string`, `date_added`, `duration`, `source_type`, `(parent_directory_path, source_type, album_id)`, `(parent_directory_path, source_type, id)`
- **FK** → `albums(id)` ON DELETE CASCADE
- **FK** → `artists(id)` ON DELETE SET NULL

### 変換関数 / 拡張

| 関数 | 戻り値 | 目的 | 呼び出し元 (推測) |
|------|--------|------|-----------------|
| `SongEntity.toSong()` | `Song` | `artists_json` を `List<ArtistRef>` にパースし、`content_uri_string` のスキームから `telegramChatId`/`neteaseId`/`gdriveFileId`/`qqMusicMid`/`navidromeId`/`jellyfinId` を抽出して `Song` モデルを組み立てる。`LocalArtworkUri.resolveSongArtworkUri` でアート URI を解決 | Repository, UI 層 |
| `SongEntity.toSongWithArtistRefs(artists, crossRefs)` | `Song` | junction テーブル経由で取得した `ArtistEntity` + `SongArtistCrossRef` から `ArtistRef` リストを構築。`is_primary DESC` でソート | Repository |
| `Song.toEntity(filePath, parentDir)` | `SongEntity` | モデル → エンティティ。`source_type` は `SourceType.fromContentUri(contentUriString)` で派生 | SyncWorker |
| `Song.toEntityWithoutPaths()` | `SongEntity` | パス不明時のフォールバック。`file_path` / `parent_directory_path` を空文字で生成 | バックアップ / リストア |
| `List<SongEntity>.toSongs()` | `List<Song>` | マップ変換ショートカット | Repository |
| `serializeArtistRefs(artists)` | `String` | `List<ArtistRef>` → JSON 文字列 (`[{id,name,primary}, ...]`)。`artists_json` 列書き込み時に使用 | SyncWorker |
| `parseArtistsJson(json)` (private) | `List<ArtistRef>` | JSON 逆パース。失敗時は空リスト | `toSong()` 内部 |

### `data class SongSummary`

軽量プロジェクション (`MusicDao.getAllLocalSongSummaries()` で使用)。

| カラム | 型 | 備考 |
|--------|----|----|
| `id` | `Long` | MediaStore ID |
| `title` | `String` | — |
| `artist_name` | `String` | `@ColumnInfo` |
| `album_name` | `String` | `@ColumnInfo` |
| `duration` | `Long` | — |

---

## 2. `AlbumEntity.kt`

| パッケージ | `com.theveloper.pixelplay.data.database` |
|-----------|----------------------------------------|
| 役割 | `albums` テーブル + 変換 |
| 上流 | `MusicDao` |
| 下流 | `Album` モデル, `LocalArtworkUri` |

### `data class AlbumEntity`

`@Entity(tableName = "albums", indices = [title, artist_id, artist_name, album_artist])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `Long` | MediaStore `Albums._ID` |
| `title` | `String` | アルバム名 |
| `artist_name` | `String` | アルバム アーティスト名 (表示用) |
| `artist_id` | `Long` | 主アーティスト ID |
| `album_art_uri_string` | `String?` | ジャケット URI |
| `song_count` | `Int` | 曲数 (ライブラリ クエリで COUNT) |
| `date_added` | `Long` | 追加日時 |
| `year` | `Int` | リリース年 |
| `album_artist` | `String?` | TPE2 タグ由来 (default `null`) |

### 変換関数

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `AlbumEntity.toAlbum()` | `Album` | `albumArtUriString` が Beta 6 の揮発性 FileProvider URI の場合、`LocalArtworkUri.parseSongIdFromVolatileArtworkUri` で song_id を抽出し `pixelplay_local_art://<song_id>` に再マップ。`albumArtist` が空白文字のみの場合は `null` に正規化 |
| `List<AlbumEntity>.toAlbums()` | `List<Album>` | マップ変換 |
| `Album.toEntity(artistIdForAlbum)` | `AlbumEntity` | モデル → エンティティ。`Album` モデル自体は `artistId` を持たないため、外部から ID を注入する必要あり |

---

## 3. `ArtistEntity.kt`

| パッケージ | `com.theveloper.pixelplay.data.database` |
|-----------|----------------------------------------|
| 役割 | `artists` テーブル + 変換 |
| 上流 | `MusicDao` |
| 下流 | `Artist` モデル |

### `data class ArtistEntity`

`@Entity(tableName = "artists", indices = [name])`。`@PrimaryKey id: Long`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `Long` | MediaStore `Artists._ID` |
| `name` | `String` | アーティスト名 |
| `track_count` | `Int` | 曲数 (ライブラリ クエリで COUNT) |
| `image_url` | `String?` | Deezer 等の外部画像 URL |
| `custom_image_uri` | `String?` | ユーザー指定カスタム画像 URI |

### 変換関数

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `ArtistEntity.toArtist()` | `Artist` | `trackCount` → `songCount` にリネーム |
| `List<ArtistEntity>.toArtists()` | `List<Artist>` | マップ変換 |
| `Artist.toEntity()` | `ArtistEntity` | モデル → エンティティ |

---

## 4. `SongArtistCrossRef.kt`

| パッケージ | `com.theveloper.pixelplay.data.database` |
|-----------|----------------------------------------|
| 役割 | 多対多 junction + Relation クラス |
| 上流 | `MusicDao` |
| 下流 | `Song`, `ArtistEntity` |

### `data class SongArtistCrossRef`

`@Entity(tableName = "song_artist_cross_ref", primaryKeys = ["song_id", "artist_id"])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `song_id` (PK) | `Long` | `songs.id` FK → CASCADE |
| `artist_id` (PK) | `Long` | `artists.id` FK → CASCADE |
| `is_primary` | `Boolean` | 主フラグ (default `0`) |

Index: `song_id`, `artist_id`, `is_primary`。

### `data class SongWithArtists`

`@Relation` 経由で song + 全アーティストを取得。`MusicDao.getArtistsForSong` 系で直接 `List<ArtistEntity>` を取りに行く方がコストが低いため、使用頻度は低い。

| フィールド | 型 | 備考 |
|-----------|----|----|
| `song` | `SongEntity` (`@Embedded`) | — |
| `artists` | `List<ArtistEntity>` (`@Relation` + `Junction`) | `SongArtistCrossRef` 経由 |

### `data class ArtistWithSongs`

逆方向の Relation。

| フィールド | 型 | 備考 |
|-----------|----|----|
| `artist` | `ArtistEntity` (`@Embedded`) | — |
| `songs` | `List<SongEntity>` (`@Relation`) | `SongArtistCrossRef` 経由 |

### `data class PrimaryArtistInfo`

軽量プロジェクション (`MusicDao.getPrimaryArtistForSong` で使用)。

| フィールド | 型 | 備考 |
|-----------|----|----|
| `artist_id` | `Long` | `@ColumnInfo("artist_id")` |
| `artistName` | `String` | `artists.name` をそのまま射影 |

---

## 5. `PlaylistEntity.kt`

| パッケージ | `com.theveloper.pixelplay.data.database` |
|-----------|----------------------------------------|
| 役割 | ユーザー定義プレイリスト |
| 上流 | `LocalPlaylistDao` |
| 下流 | `Playlist` モデル |

### `data class PlaylistEntity`

`@Entity(tableName = "playlists", indices = [last_modified])`。`@PrimaryKey id: String` (UUID 等の文字列)。

| カラム | 型 | デフォルト | 説明 |
|--------|----|----------|------|
| `id` (PK) | `String` | — | UUID 等 |
| `name` | `String` | — | プレイリスト名 |
| `created_at` | `Long` | `currentTimeMillis()` | 作成日時 |
| `last_modified` | `Long` | `currentTimeMillis()` | 最終更新日時 |
| `is_ai_generated` | `Boolean` | `false` | AI 生成フラグ |
| `is_queue_generated` | `Boolean` | `false` | キュー自動生成フラグ |
| `cover_image_uri` | `String?` | `null` | カバー画像 URI |
| `cover_color_argb` | `Int?` | `null` | カバー ARGB カラー |
| `cover_icon_name` | `String?` | `null` | カバー アイコン名 |
| `cover_shape_type` | `String?` | `null` | "Circle"/"SmoothRect"/"RotatedPill"/"Star" |
| `cover_shape_detail_1` | `Float?` | `null` | CornerRadius / StarCurve |
| `cover_shape_detail_2` | `Float?` | `null` | Smoothness / StarRotation |
| `cover_shape_detail_3` | `Float?` | `null` | StarScale |
| `cover_shape_detail_4` | `Float?` | `null` | Star Sides (Int) |
| `source` | `String` | `"LOCAL"` | "LOCAL" / "NETEASE" / "TELEGRAM" / "AI" 等 |

### 変換関数

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `PlaylistEntity.toPlaylist(songIds: List<String>)` | `Playlist` | `songIds` は呼び出し側で別途取得 (`playlist_songs` から) |
| `Playlist.toEntity()` | `PlaylistEntity` | モデル → エンティティ。`songIds` は保存されない (別途 `playlist_songs` テーブルで管理) |

---

## 6. `PlaylistSongEntity.kt`

| パッケージ | `com.theveloper.pixelplay.data.database` |
|-----------|----------------------------------------|
| 役割 | playlist_songs 中間テーブル |
| 上流 | `LocalPlaylistDao` |

### `data class PlaylistSongEntity`

`@Entity(tableName = "playlist_songs", primaryKeys = ["playlist_id", "song_id"])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `playlist_id` (PK) | `String` | `playlists.id` (FK なし — 整合性は Repository 層で担保) |
| `song_id` (PK) | `String` | `songs.id` の文字列表現 |
| `sort_order` | `Int` | 表示順序 |

Index: `(playlist_id, sort_order)`, `song_id`。

---

## 7. `PlaylistWithSongsEntity.kt`

| パッケージ | `com.theveloper.pixelplay.data.database` |
|-----------|----------------------------------------|
| 役割 | `PlaylistEntity` + `List<PlaylistSongEntity>` の Relation |
| 上流 | `LocalPlaylistDao.observePlaylistsWithSongs` |

| フィールド | 型 | 備考 |
|-----------|----|----|
| `playlist` | `PlaylistEntity` (`@Embedded`) | — |
| `songs` | `List<PlaylistSongEntity>` (`@Relation`) | `entityColumn = "playlist_id"`、`entity = PlaylistSongEntity::class` |

---

## 8. `AlbumArtThemeEntity.kt`

| パッケージ | `com.theveloper.pixelplay.data.database` |
|-----------|----------------------------------------|
| 役割 | アルバム アートから抽出した Material3 カラー スキーム永続化 |
| 上流 | `AlbumArtThemeDao` |

### `data class StoredColorSchemeValues`

50 個の `String` カラー トークン (16進 `0xAARRGGBB` 文字列)。light / dark 両スキーム分を Embed。

| グループ | トークン (16進 String) |
|---------|-----------------------|
| Primary | `primary`, `onPrimary`, `primaryContainer`, `onPrimaryContainer`, `inversePrimary` |
| Secondary | `secondary`, `onSecondary`, `secondaryContainer`, `onSecondaryContainer` |
| Tertiary | `tertiary`, `onTertiary`, `tertiaryContainer`, `onTertiaryContainer` |
| Surface | `surface`, `onSurface`, `surfaceVariant`, `onSurfaceVariant`, `surfaceBright`, `surfaceDim`, `surfaceContainer`, `surfaceContainerHigh`, `surfaceContainerHighest`, `surfaceContainerLow`, `surfaceContainerLowest`, `surfaceTint`, `inverseSurface`, `inverseOnSurface`, `scrim`, `outline`, `outlineVariant` |
| Background | `background`, `onBackground` |
| Error | `error`, `onError`, `errorContainer`, `onErrorContainer` |
| Fixed | `primaryFixed`, `primaryFixedDim`, `onPrimaryFixed`, `onPrimaryFixedVariant`, `secondaryFixed`, `secondaryFixedDim`, `onSecondaryFixed`, `onSecondaryFixedVariant`, `tertiaryFixed`, `tertiaryFixedDim`, `onTertiaryFixed`, `onTertiaryFixedVariant` |

### `data class AlbumArtThemeEntity`

`@Entity(tableName = "album_art_themes", indices = [(albumArtUriString, paletteStyle)])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `albumArtUriString` (PK) | `String` | アルバム アート URI |
| `paletteStyle` | `String` | パレット スタイル ("Vibrant" 等) |
| `lightThemeValues` | `StoredColorSchemeValues` (`@Embedded(prefix = "light_")`) | ライト モード用 50 トークン → `light_primary`, `light_onPrimary`, ... |
| `darkThemeValues` | `StoredColorSchemeValues` (`@Embedded(prefix = "dark_")`) | ダーク モード用 50 トークン → `dark_primary`, `dark_onPrimary`, ... |

> 注: `MIGRATION_15_16` で 50 色 → 100 色に破壊的マイグレ (`database-system.md` 参照)。

---

## 9. `AiCacheEntity.kt`

`@Entity(tableName = "ai_cache")`。プロンプト応答キャッシュ (SHA-256 → JSON)。

| カラム | 型 | 説明 |
|--------|----|----|
| `promptHash` (PK) | `String` | SHA-256 ハッシュ |
| `responseJson` | `String` | レスポンス JSON 文字列 |
| `timestamp` | `Long` | キャッシュ作成日時 |

---

## 10. `AiUsageEntity.kt`

`@Entity(tableName = "ai_usage")`。AI 使用量ログ。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK, autoGenerate) | `Long` | — |
| `timestamp` | `Long` | リクエスト時刻 epoch ms |
| `provider` | `String` | プロバイダ名 |
| `model` | `String` | モデル名 |
| `promptType` | `String` | プロンプト種別 |
| `promptTokens` | `Int` | プロンプト トークン数 |
| `outputTokens` | `Int` | 出力トークン数 |
| `thoughtTokens` | `Int` | 推論トークン数 |

---

## 11. `FavoritesEntity.kt`

`@Entity(tableName = "favorites", indices = [timestamp])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `songId` (PK) | `Long` | `songs.id` |
| `isFavorite` | `Boolean` | フラグ (default `true`、事実上一意) |
| `timestamp` | `Long` | お気に入り追加日時 (default `currentTimeMillis()`) |

> `@SerializedName` 多数 (`song_id`/`songId` 両対応等) — Gson JSON バックアップ互換用。
> お気に入りの正本は `songs.is_favorite` 列で、`favorites` テーブルは timestamp 管理用に並走 (`database-system.md` の `installFavoriteSyncTriggers` 参照)。

---

## 12. `LyricsEntity.kt`

`@Entity(tableName = "lyrics")`。

| カラム | 型 | 説明 |
|--------|----|----|
| `songId` (PK) | `Long` | `songs.id` |
| `content` | `String` | 歌詞本文 |
| `isSynced` | `Boolean` | LRC 同期歌詞かどうか (default `false`) |
| `source` | `String?` | "local" / "remote" / "embedded" |

> `lyrics` テーブルと `songs.lyrics` 列は別管理。`MusicDao.getSongById*` で `COALESCE(song_lyrics.content, songs.lyrics)` の優先順。

---

## 13. `SongEngagementEntity.kt`

`@Entity(tableName = "song_engagements", indices = [play_count])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `song_id` (PK) | `String` | `songs.id` の文字列表現 |
| `play_count` | `Int` | 累積再生回数 (default `0`) |
| `total_play_duration_ms` | `Long` | 累積再生時間 ms (default `0`) |
| `last_played_timestamp` | `Long` | 最終再生時刻 epoch ms (default `0`) |

> Gson `@SerializedName` のエイリアス多数 (`songId`/`song_id`、`score`/`plays` 等) で JSON マイグレーション対応。

---

## 14. `SearchHistoryEntity.kt`

`@Entity(tableName = "search_history")`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK, autoGenerate) | `Long` | — |
| `query` | `String` | 検索クエリ |
| `timestamp` | `Long` | 検索時刻 |

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `SearchHistoryEntity.toSearchHistoryItem()` | `SearchHistoryItem` | Entity → モデル |
| `SearchHistoryItem.toEntity()` | `SearchHistoryEntity` | モデル → Entity (id が null なら `0` で autoGenerate) |

---

## 15. `TransitionRuleEntity.kt`

`@Entity(tableName = "transition_rules", indices = [(playlistId, fromTrackId, toTrackId, unique = true)])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK, autoGenerate) | `Long` | — |
| `playlistId` | `String` | プレイリスト ID (GLOBAL は `"__default__"` 等と推測) |
| `fromTrackId` | `String?` | 遷移元トラック ID。null の場合はプレイリスト全体デフォルト |
| `toTrackId` | `String?` | 遷移先トラック ID。null の場合はプレイリスト全体デフォルト |
| `settings` | `TransitionSettings` (`@Embedded`) | モード / デュレーション / フェード カーブ |

> `@SerializedName` 多数で `fromSongId`/`toSongId` 等のレガシー名も許容。
> 同じ `(playlistId, fromTrackId, toTrackId)` トリプルに対して 1 レコードのみ存在可能 (unique index)。

---

## 16. `SongSearchFtsEntity.kt`

`@Fts4(tokenizer = "unicode61") @Entity(tableName = "songs_fts")`。SQLite FTS4 仮想テーブル用。

| カラム | 型 | 説明 |
|--------|----|----|
| `rowid` (PK) | `Long` | FTS 内部 rowid (`songs.id` と一致) |
| `title` | `String` | FTS インデックス対象 |
| `artist_name` | `String` | FTS インデックス対象 |

> トリガーで `songs` テーブルと自動的に同期 (`database-system.md` の `installSongsSearchSyncTriggers` 参照)。

---

## 17. `TelegramSongEntity.kt`

`@Entity(tableName = "telegram_songs", indices = [chat_id, message_id, file_id, (chat_id, message_id), thread_id])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `String` | `"chatId_messageId"` 形式 |
| `chat_id` | `Long` | Telegram チャット ID |
| `message_id` | `Long` | メッセージ ID |
| `file_id` | `Int` | Telegram ファイル参照 ID |
| `title` | `String` | 曲タイトル |
| `artist` | `String` | アーティスト名 |
| `duration` | `Long` | 再生時間 ms |
| `file_path` | `String` | ローカル ダウンロード済みパス (未 DL なら空文字) |
| `mime_type` | `String` | MIME |
| `date_added` | `Long` | 追加日時 |
| `album_art_uri_string` | `String?` | アート URI (省略時は `telegram_art://chatId/messageId`) |
| `thread_id` | `Long?` | フォーラム トピック ID (null = 通常チャンネル) |

### ヘルパ関数

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `TelegramSongEntity.resolveAlbumArtUri()` | `String?` | ローカル ファイルがある場合は `?v=<lastModified>` を付与したキャッシュ バスター URI を返す |
| `TelegramSongEntity.toSong(channelTitle, topicName)` | `Song` | `Song` モデルへ変換。`channelTitle`/`topicName` から合成アルバム ラベルを作成 (`<channel>/<topic>` または `Telegram Stream`)。負の Long で合成 artist_id / album_id を生成 |
| `Song.toTelegramEntity()` | `TelegramSongEntity?` | 逆変換。`telegramChatId` / `telegramFileId` が null なら null 返却 |
| `Song.toTelegramEntityWithThread(threadId)` | `TelegramSongEntity?` | 上記に `threadId` を付与した copy |

---

## 18. `TelegramChannelEntity.kt`

`@Entity(tableName = "telegram_channels")`。

| カラム | 型 | 説明 |
|--------|----|----|
| `chat_id` (PK) | `Long` | Telegram チャット ID |
| `title` | `String` | チャンネル / グループ名 |
| `username` | `String?` | Telegram ユーザー名 |
| `song_count` | `Int` | 曲数 (default `0`) |
| `last_sync_time` | `Long` | 最終同期時刻 (default `0`) |
| `photo_path` | `String?` | プロフィール画像キャッシュ パス |

---

## 19. `TelegramTopicEntity.kt`

`@Entity(tableName = "telegram_topics", indices = [chat_id])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `String` | `"chatId_threadId"` 形式 |
| `chat_id` | `Long` | 所属チャンネル ID |
| `thread_id` | `Long` | トピック ID |
| `name` | `String` | トピック名 |
| `song_count` | `Int` | 曲数 (default `0`) |
| `last_sync_time` | `Long` | 最終同期時刻 (default `0`) |
| `icon_emoji` | `String?` | アイコン絵文字 (例: `"🎵"`) |

---

## 20. `NeteaseSongEntity.kt`

`@Entity(tableName = "netease_songs", indices = [netease_id, playlist_id, (playlist_id, date_added)])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `String` | `"<playlistId>_<neteaseId>"` 複合 |
| `netease_id` | `Long` | Netease 数字 ID |
| `playlist_id` | `Long` | Netease プレイリスト ID |
| `title`, `artist`, `album` | `String` | — |
| `album_id` | `Long` | アルバム ID |
| `duration` | `Long` | ms |
| `album_art_url` | `String?` | HTTP カバー URL |
| `mime_type` | `String` | — |
| `bitrate` | `Int?` | — |
| `date_added` | `Long` | — |

### 変換関数

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `NeteaseSongEntity.toSong()` | `Song` | `id = "netease_$id"`、`contentUriString = "netease://$neteaseId"`、`neteaseId` を保持 |
| `Song.toNeteaseEntity(playlistId)` | `NeteaseSongEntity` | 逆変換。`id = "${playlistId}_${neteaseId}"` |

---

## 21. `NeteasePlaylistEntity.kt`

`@Entity(tableName = "netease_playlists")`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `Long` | Netease プレイリスト ID |
| `name` | `String` | プレイリスト名 |
| `cover_url` | `String?` | HTTP カバー URL |
| `song_count` | `Int` | — |
| `last_sync_time` | `Long` | — |

---

## 22. `GDriveSongEntity.kt`

`@Entity(tableName = "gdrive_songs", indices = [drive_file_id, folder_id, (folder_id, date_added)])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `String` | `"<folderId>_<driveFileId>"` 複合 |
| `drive_file_id` | `String` | Google Drive ファイル ID |
| `folder_id` | `String` | 所属 Drive フォルダ ID |
| `title`, `artist`, `album` | `String` | — |
| `album_id` | `Long` | — |
| `duration` | `Long` | ms |
| `album_art_url` | `String?` | HTTP カバー URL |
| `mime_type` | `String` | — |
| `bitrate` | `Int?` | — |
| `file_size` | `Long` | バイト サイズ |
| `date_added` | `Long` | — |
| `date_modified` | `Long` | — |

### 変換関数

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `GDriveSongEntity.toSong()` | `Song` | `id = "gdrive_$driveFileId"`、`contentUriString = "gdrive://$driveFileId"`、`genre = "Google Drive"`、負の Long で合成 artist_id / album_id |
| `Song.toGDriveEntity(folderId)` | `GDriveSongEntity` | 逆変換 |

---

## 23. `GDriveFolderEntity.kt`

`@Entity(tableName = "gdrive_folders")`。

| カラム | 型 | デフォルト | 説明 |
|--------|----|----------|------|
| `id` (PK) | `String` | — | Drive フォルダ ID |
| `name` | `String` | — | フォルダ名 |
| `song_count` | `Int` | `0` | 曲数 |
| `last_sync_time` | `Long` | `0` | 最終同期時刻 |

---

## 24. `QqMusicSongEntity.kt`

`@Entity(tableName = "qqmusic_songs")`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `String` | `"<playlistId>_<songMid>"` 複合 |
| `song_mid` | `String` | QQ Music の song mid 識別子 |
| `playlist_id` | `Long` | 所属プレイリスト ID |
| `title`, `artist`, `album` | `String` | — |
| `album_mid` | `String?` | アルバム mid |
| `duration` | `Long` | ms |
| `album_art_url` | `String?` | HTTP カバー URL |
| `mime_type` | `String` | — |
| `bitrate` | `Int?` | — |
| `date_added` | `Long` | — |

### 変換関数

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `QqMusicSongEntity.toSong()` | `Song` | `id = "qqmusic_$id"`、`contentUriString = "qqmusic://$songMid"`、`qqMusicMid` 保持。`artistId` / `albumId` は `-1L` |
| `Song.toQqMusicEntity(playlistId)` | `QqMusicSongEntity` | 逆変換 |

---

## 25. `QqMusicPlaylistEntity.kt`

`@Entity(tableName = "qqmusic_playlists")`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `Long` | QQ プレイリスト ID |
| `name` | `String` | — |
| `cover_url` | `String?` | HTTP カバー |
| `song_count` | `Int` | — |
| `last_sync_time` | `Long` | — |

---

## 26. `NavidromeSongEntity.kt`

`@Entity(tableName = "navidrome_songs", indices = [navidrome_id, playlist_id, (playlist_id, date_added)])`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `String` | `"<playlistId>_<navidromeId>"` 複合 |
| `navidrome_id` | `String` | Subsonic サーバ上の曲 ID |
| `playlist_id` | `String` | 所属プレイリスト ID (ライブラリは `"__library__"` 予約) |
| `title`, `artist`, `album` | `String` | — |
| `artist_id` | `String?` | Subsonic アーティスト ID |
| `album_id` | `String?` | Subsonic アルバム ID |
| `cover_art_id` | `String?` | カバー アート ID |
| `duration` | `Long` | ms |
| `track_number` | `Int` | — |
| `disc_number` | `Int` | — |
| `year` | `Int` | — |
| `genre` | `String?` | — |
| `bitRate` | `Int?` | kbps (Subsonic 単位) |
| `mime_type` | `String?` | — |
| `suffix` | `String?` | "mp3"/"flac" 等 |
| `path` | `String` | サーバ上のパス |
| `date_added` | `Long` | — |

### 変換関数

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `NavidromeSongEntity.toSong()` | `Song` | `id = "navidrome_$id"`、`contentUriString = "navidrome://$navidromeId"`、`albumArtUriString = "navidrome_cover://$coverArtId"`、`bitrate = bitRate * 1000` (bps 変換) |
| `NavidromeSong.toEntity(playlistId)` | `NavidromeSongEntity` | API モデル → Entity。`dateAdded = currentTimeMillis()` |

> `NavidromeSong` は `data/navidrome/model/NavidromeSong` からの依存。

---

## 27. `NavidromePlaylistEntity.kt`

`@Entity(tableName = "navidrome_playlists")`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `String` | Subsonic プレイリスト ID |
| `name` | `String` | — |
| `comment` | `String?` | — |
| `owner` | `String?` | — |
| `cover_art_id` | `String?` | — |
| `song_count` | `Int` | — |
| `duration` | `Long` | ms |
| `public` | `Boolean` | 公開フラグ |
| `last_sync_time` | `Long` | — |

---

## 28. `JellyfinSongEntity.kt`

`@Entity(tableName = "jellyfin_songs", indices = [jellyfin_id, playlist_id, (playlist_id, date_added)])`。Navidrome と類似スキーマ (文字列 ID ベース)。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `String` | `"<playlistId>_<jellyfinId>"` 複合 |
| `jellyfin_id` | `String` | Jellyfin item ID |
| `playlist_id` | `String` | 所属プレイリスト ID (`"__library__"` 予約) |
| `title`, `artist`, `album` | `String` | — |
| `artist_id` | `String?` | — |
| `album_id` | `String?` | — |
| `duration` | `Long` | ms |
| `track_number` | `Int` | — |
| `disc_number` | `Int` | — |
| `year` | `Int` | — |
| `genre` | `String?` | — |
| `bitRate` | `Int?` | kbps |
| `mime_type` | `String?` | — |
| `path` | `String` | — |
| `date_added` | `Long` | — |

### 変換関数

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `JellyfinSongEntity.toSong()` | `Song` | `id = "jellyfin_$id"`、`contentUriString = "jellyfin://$jellyfinId"`、`albumArtUriString = "jellyfin_cover://$jellyfinId"`、`bitrate = bitRate * 1000` |
| `JellyfinSong.toEntity(playlistId)` | `JellyfinSongEntity` | API モデル → Entity。`dateAdded = currentTimeMillis()` |

> `JellyfinSong` は `data/jellyfin/model/JellyfinSong` からの依存。

---

## 29. `JellyfinPlaylistEntity.kt`

`@Entity(tableName = "jellyfin_playlists")`。

| カラム | 型 | 説明 |
|--------|----|----|
| `id` (PK) | `String` | Jellyfin playlist ID |
| `name` | `String` | — |
| `song_count` | `Int` | — |
| `duration` | `Long` | ms |
| `last_sync_time` | `Long` | — |

---

## 30. `ColorConverters.kt`

`@TypeConverter` ではなく `String` 拡張関数として実装。

| 関数 | 戻り値 | 目的 |
|------|--------|------|
| `String.toComposeColor(): Color` | `androidx.compose.ui.graphics.Color` | 16進 `#RRGGBB` または `#AARRGGBB` を `Color` にパース。`IllegalArgumentException` 発生時は `Color.Black` フォールバック。`androidx.core.graphics.toColorInt` を使用 |

> `AlbumArtThemeEntity.StoredColorSchemeValues` の各トークンを Compose の `ColorScheme` に組み立てる UI 層から呼び出される。

---

## 31. `FolderSongRow.kt`

軽量プロジェクション。`MusicDao.getFolderSongs` でフォルダ ツリー構築時に使用。

| フィールド | 型 | 備考 |
|-----------|----|----|
| `id` | `Long` | `@ColumnInfo("id")` |
| `parent_directory_path` | `String` | `@ColumnInfo("parent_directory_path")` |
| `title` | `String` | — |
| `album_art_uri_string` | `String?` | — |

---

## 内部実装メモ

### マルチ アーティストの二重管理

- 正本: `song_artist_cross_ref` (junction、`is_primary` フラグ付き)
- キャッシュ: `songs.artists_json` (`[{id, name, primary}, ...]`)
- `SongEntity.toSong()` はまず `artists_json` をパース (`parseArtistsJson`)
- `MusicDao.toSongWithArtistRefs(artists, crossRefs)` は junction から `ArtistRef` を組み立て

### Source Type 識別

- 整数 (`source_type`) による高速判定を推奨 (`SourceType` 参照)
- レガシー対応: `SongEntity.toSong()` は `contentUriString.startsWith("telegram://")` 等の文字列パースでも同等情報を `Song` モデルに詰める

### アルバム アートの揮発性 URI マイグレーション

- Beta 6 で FileProvider URI (`content://.../cache/song_art_<id>.jpg`) が `album_art_uri_string` に保存された
- v0.7 のアート書き換え後、キャッシュ ファイルが消えて 404 するようになった
- `AlbumEntity.toAlbum()` で揮発性 URI を検出し、`pixelplay_local_art://<song_id>` に再マップ
- `SongEntity.toSong()` も `LocalArtworkUri.resolveSongArtworkUri` で同様にリマップ

### Lyrics の二段管理

- `songs.lyrics` 列 (埋め込みタグ由来)
- `lyrics` テーブル (外部取得 / ユーザー編集由来)
- `MusicDao.getSongById*` で `COALESCE(song_lyrics.content, songs.lyrics)` の優先順位で取得

### Favorite の二段管理

- `songs.is_favorite` (再生時に高速アクセス)
- `favorites` テーブル (タイムスタンプ管理 + バックアップ互換)
- トリガー (`PixelPlayDatabase.installFavoriteSyncTriggers`) で双方向同期

### FTS4 仮想テーブル

- `songs_fts` は `songs` とトリガー (`installSongsSearchSyncTriggers`) で同期
- `unicode61` トークナイザ使用
- `MusicDao.searchSongs*` 系は FTS + LIKE の二段構え (`buildSongSearchMatchQuery` でトークン抽出)

### 外部サービス テーブルは songs と別管理

- Telegram / Netease / GDrive / QQ / Navidrome / Jellyfin はそれぞれ専用テーブル
- 同じ `Song` モデルに正規化されるが、検索対象は統合 `songs` テーブル側 (MusicDao で取り込み)
- `MusicDao.clearAll<Source>Songs()` で `source_type` 別に一括削除 → 関連 (cross_ref / favorites / lyrics) も連動削除 (`deleteSongsAndRelatedData`)

### Junction テーブルのバッチ サイズ

- `MusicDao.CROSS_REF_BATCH_SIZE = 999 / 3 = 333`
- SQLite の変数上限 (デフォルト 999) を 3 列 (`song_id`/`artist_id`/`is_primary`) で割った値
- バルク インサートは `chunked(CROSS_REF_BATCH_SIZE)` で分割

### Song バッチ サイズ

- `MusicDao.SONG_BATCH_SIZE = 500`
- インクリメンタル同期でチャンク書き込み、並行読み取りを許可

---

## 関連ファイル

- DAO 層: [`database-daos.md`](./database-daos.md)
- DB 統合 + Migration: [`database-system.md`](./database-system.md)
- モデル層: [`models.md`](./models.md)
- 上流 Repository: `../../03-data-services/` (推測)
- 下流 モデル: `../model/Song.kt`, `../model/Album.kt`, `../model/Artist.kt`, `../model/Playlist.kt`, `../model/Lyrics.kt`, `../model/Transition.kt`