# database-daos.md

> `com.theveloper.pixelplay.data.database` パッケージ内の全 DAO の詳細仕様。
> 16 の `@Dao` インターフェース。`MusicDao` は最大 (約 1,952 行 / 100+ メソッド)。

## 一覧

| DAO | ファイル | 行数 (概算) | 主エンティティ |
|-----|----------|------------|---------------|
| `MusicDao` | `MusicDao.kt` | 1,952 | `SongEntity`, `AlbumEntity`, `ArtistEntity`, `SongArtistCrossRef` |
| `AlbumArtThemeDao` | `AlbumArtThemeDao.kt` | 20 | `AlbumArtThemeEntity` |
| `AiCacheDao` | `AiCacheDao.kt` | 21 | `AiCacheEntity` |
| `AiUsageDao` | `AiUsageDao.kt` | 39 | `AiUsageEntity` |
| `EngagementDao` | `EngagementDao.kt` | 75 | `SongEngagementEntity` |
| `FavoritesDao` | `FavoritesDao.kt` | 44 | `FavoritesEntity` |
| `GDriveDao` | `GDriveDao.kt` | 59 | `GDriveSongEntity`, `GDriveFolderEntity` |
| `JellyfinDao` | `JellyfinDao.kt` | 83 | `JellyfinSongEntity`, `JellyfinPlaylistEntity` |
| `LocalPlaylistDao` | `LocalPlaylistDao.kt` | 74 | `PlaylistEntity`, `PlaylistSongEntity`, `PlaylistWithSongsEntity` |
| `LyricsDao` | `LyricsDao.kt` | 37 | `LyricsEntity` |
| `NavidromeDao` | `NavidromeDao.kt` | 83 | `NavidromeSongEntity`, `NavidromePlaylistEntity` |
| `NeteaseDao` | `NeteaseDao.kt` | 63 | `NeteaseSongEntity`, `NeteasePlaylistEntity` |
| `QqMusicDao` | `QqMusicDao.kt` | 51 | `QqMusicSongEntity`, `QqMusicPlaylistEntity` |
| `SearchHistoryDao` | `SearchHistoryDao.kt` | 34 | `SearchHistoryEntity` |
| `TelegramDao` | `TelegramDao.kt` | 92 | `TelegramSongEntity`, `TelegramChannelEntity`, `TelegramTopicEntity` |
| `TransitionDao` | `TransitionDao.kt` | 67 | `TransitionRuleEntity` |

> DAO はすべて `PixelPlayDatabase` (version 42) で abstract 関数として公開される (`database-system.md` 参照)。

---

## 1. `MusicDao` (コア DAO)

`app/src/main/java/com/theveloper/pixelplay/data/database/MusicDao.kt:121`

統合 `songs` / `albums` / `artists` / `song_artist_cross_ref` テーブルへの全 CRUD + 検索 + ページング + 統計 + クラウド ソース別削除を担う最大 DAO。

### 1.1 定数 / Projection

| 名前 | 行 | 種類 | 目的 |
|------|----|----|------|
| `SONG_SEARCH_QUERY_TOKEN_REGEX` | 15 | `Regex` (private) | `[\p{L}\p{N}]+` で Unicode 文字 + 数字のトークン抽出 |
| `EMPTY_SONG_SEARCH_MATCH_QUERY` | 16 | `String` (private const) | `"pixelplayemptyquery*"` — トークン無し時の FTS ダミー クエリ |
| `buildSongTitleSearchMatchQuery(query)` | 18 | private fun | タイトル専用 FTS クエリ構築。最大 6 トークン、各 `title:token*` 形式 |
| `buildSongSearchMatchQuery(query)` | 31 | private fun | タイトル + アーティスト FTS クエリ構築。最大 6 トークン、各 `token*` 形式 |
| `SONG_DETAIL_PROJECTION` | 44 | `String` (private const) | `lyrics` 列を含む 25 列。`COALESCE(song_lyrics.content, songs.lyrics)` で優先 |
| `SONG_LIST_PROJECTION` | 75 | `String` (private const) | `lyrics = NULL` で射影。2MB CursorWindow 制限回避 |
| `DeviceCapabilitySongRow` | 83 | data class | デバイス機能チェック用プロジェクション (filePath, contentUriString, mimeType, duration, bitrate, sampleRate, sourceType) |
| `LibraryAudioStatsRow` | 100 | data class | ライブラリ音声統計 (totalCount, localCount, cloudCount, hiResCount, ultraHiResCount, likelyExpensiveCount, maxBitrate, minSampleRate, maxSampleRate, estMinBytes, estAvgBytes, estMaxBytes) |
| `MimeTypeCountRow` | 115 | data class | MIME タイプ別カウント |
| `SQLITE_MAX_VARIABLE_NUMBER` | 1942 | `Int` const | `999` |
| `CROSS_REF_FIELDS_PER_OBJECT` | 1943 | `Int` const | `3` |
| `CROSS_REF_BATCH_SIZE` | 1944 | `Int` val | `999 / 3 = 333` |
| `SONG_BATCH_SIZE` | 1950 | `Int` const | `500` (インクリメンタル同期用チャンク サイズ) |

### 1.2 挿入 / 更新

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `insertSongsIgnoreConflicts(songs)` | 125 | `List<Long>` | `@Insert IGNORE`。rowid = `-1` のエントリが衝突。`updateSongs` と組み合わせて upsert |
| `updateSongs(songs)` | 128 | `Unit` | `@Update` |
| `insertAlbumsIgnoreConflicts(albums)` | 131 | `List<Long>` | 同上 |
| `updateAlbums(albums)` | 134 | `Unit` | `@Update` |
| `insertArtistsIgnoreConflicts(artists)` | 137 | `List<Long>` | 同上 |
| `updateArtists(artists)` | 140 | `Unit` | `@Update` |
| `getArtistsByIds(artistIds)` | 143 | `List<ArtistEntity>` | `WHERE id IN (:artistIds)` |
| `insertSongs(songs)` | 146 | `Unit` (`@Transaction`) | IGNORE → 衝突行のみ UPDATE に切替 |
| `insertAlbums(albums)` | 159 | `Unit` (`@Transaction`) | 同上 |
| `insertArtists(artists)` | 172 | `Unit` (`@Transaction`) | UPDATE 時に既存行の `imageUrl` / `customImageUri` が null なら保持 |
| `insertMusicData(songs, albums, artists)` | 199 | `Unit` (`@Transaction`) | アーティスト → アルバム → 曲 の順で挿入 |
| `clearAllMusicData()` | 206 | `Unit` (`@Transaction`) | 3 テーブル全削除 |

### 1.3 一括削除

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `clearAllSongs()` | 214 | `Unit` | `DELETE FROM songs` |
| `clearAllAlbums()` | 217 | `Unit` | `DELETE FROM albums` |
| `clearAllArtists()` | 220 | `Unit` | `DELETE FROM artists` |

### 1.4 インクリメンタル同期

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getAllSongIds()` | 224 | `List<Long>` | 全 ID |
| `getAllMediaStoreSongIds()` | 227 | `List<Long>` | `source_type = 0` (ローカル) のみ |
| `deleteSongsByIds(songIds)` | 230 | `Unit` | `WHERE id IN (:songIds)` |
| `deleteCrossRefsBySongIds(songIds)` | 233 | `Unit` | junction から削除 |
| `deleteFavoritesBySongIds(songIds)` | 236 | `Unit` | favorites から削除 |
| `deleteLyricsBySongIds(songIds)` | 239 | `Unit` | lyrics から削除 |
| `getAllTelegramSongIds()` | 242 | `List<Long>` | `source_type = 1` |
| `getTelegramSongIdsByChatId(chatId)` | 250 | `List<Long>` | `chat_id = :chatId` OR `content_uri_string LIKE 'telegram://' \|\| :chatId \|\| '/%'` |
| `getTelegramSongIdsByTopicId(chatId, threadId)` | 259 | `List<Long>` | `telegram_songs` JOIN で topic 絞り込み |
| `getAllNeteaseSongIds()` | 262 | `List<Long>` | `source_type = 2` |
| `getAllGDriveSongIds()` | 265 | `List<Long>` | `source_type = 3` |
| `getAllQqMusicSongIds()` | 268 | `List<Long>` | `source_type = 4` |
| `getAllNavidromeSongIds()` | 271 | `List<Long>` | `source_type = 5` |
| `getAllJellyfinSongIds()` | 274 | `List<Long>` | `source_type = 6` |
| `deleteSongsAndRelatedData(songIds)` | 277 | `Unit` (`@Transaction`) | `CROSS_REF_BATCH_SIZE` でチャンク分割し、`deleteCrossRefsBySongIds` → `deleteFavoritesBySongIds` → `deleteLyricsBySongIds` → `deleteSongsByIds`。最後に `deleteOrphanedAlbums` / `deleteOrphanedArtists` |
| `clearAllNeteaseSongs()` | 290 | `Unit` (`@Transaction`) | `deleteSongsAndRelatedData(getAllNeteaseSongIds())` |
| `clearAllGDriveSongs()` | 297 | `Unit` (`@Transaction`) | 同上 |
| `clearAllQqMusicSongs()` | 304 | `Unit` (`@Transaction`) | 同上 |
| `clearAllNavidromeSongs()` | 311 | `Unit` (`@Transaction`) | 同上 |
| `clearAllJellyfinSongs()` | 318 | `Unit` (`@Transaction`) | 同上 |
| `clearAllTelegramSongs()` | 325 | `Unit` (`@Transaction`) | 同上 |
| `clearTelegramSongsForChat(chatId)` | 332 | `Unit` (`@Transaction`) | 特定チャットの曲のみ削除 |
| `clearTelegramSongsForTopic(chatId, threadId)` | 339 | `Unit` (`@Transaction`) | 特定トピックの曲のみ削除 |
| `incrementalSyncMusicData(songs, albums, artists, crossRefs, deletedSongIds)` | 350 | `Unit` (`@Transaction`) | 削除 → アーティスト / アルバム upsert → 曲 chunked insert (500 件) → cross-ref 再構築 → 孤立アルバム / アーティスト掃除 |

### 1.5 ディレクトリ ヘルパ

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getDistinctParentDirectories()` | 394 | `List<String>` (suspend) | `SELECT DISTINCT parent_directory_path FROM songs` |
| `getDistinctParentDirectoriesFlow()` | 403 | `Flow<List<String>>` | 同上、リアクティブ |

### 1.6 曲クエリ

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getSongs(allowedParentDirs, applyDirectoryFilter)` | 412 | `Flow<List<SongEntity>>` | `SONG_LIST_PROJECTION` 使用。`:applyDirectoryFilter = 0 OR id < 0 OR parent_directory_path IN (:allowedParentDirs)` でクラウド曲 (id<0) は常に含む。`ORDER BY title ASC` |
| `getSongsByIdsListSimple(songIds)` | 418 | `List<SongEntity>` (suspend) | `SONG_LIST_PROJECTION` で ID 検索 |
| `getSongIdByContentUri(contentUri)` | 427 | `Long?` (suspend) | 統合テーブルの ID 解決。クラウド ソースの Song から負の Long を引く |
| `getSongById(songId)` | 436 | `Flow<SongEntity?>` | `SONG_DETAIL_PROJECTION` + `lyrics` LEFT JOIN |
| `getSongByIdOnce(songId)` | 445 | `SongEntity?` (suspend) | 同上、ワンショット |
| `getSongByPath(path)` | 455 | `SongEntity?` (suspend) | `file_path = :path` |
| `getSongsByIds(songIds, allowedParentDirs, applyDirectoryFilter)` | 462 | `Flow<List<SongEntity>>` | `SONG_LIST_PROJECTION` + ディレクトリ フィルタ |
| `getSongsByAlbumId(albumId)` | 469 | `Flow<List<SongEntity>>` | `ORDER BY disc_number ASC, track_number ASC` |
| `getSongsByArtistId(artistId)` | 472 | `Flow<List<SongEntity>>` | `ORDER BY title ASC` |

### 1.7 検索 (FTS + LIKE 二段構え)

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `searchSongsMatch(matchQuery, allowedParentDirs, applyDirectoryFilter)` | 481 | `Flow<List<SongEntity>>` | FTS4 (`songs_fts`) 検索 |
| `searchSongsLike(query, allowedParentDirs, applyDirectoryFilter)` | 493 | `Flow<List<SongEntity>>` | `title`/`artist_name` の `LIKE '%query%'` |
| `searchSongs(query, allowedParentDirs, applyDirectoryFilter)` | 499 | `Flow<List<SongEntity>>` | FTS + LIKE を `combine` + `LinkedHashMap` でマージ (FTS 優先、重複排除) |
| `searchSongsPaginatedMatch(matchQuery, ...)` | 960 | `PagingSource<Int, SongEntity>` | FTS ページング |
| `searchSongsPaginated(query, ...)` | 966 | `PagingSource<Int, SongEntity>` | `buildSongSearchMatchQuery` 経由で上記呼出 |
| `searchSongsLimitedMatch(matchQuery, ..., limit)` | 987 | `Flow<List<SongEntity>>` | FTS + LIMIT |
| `searchSongsLimitedByTitleLike(query, ..., limit)` | 1004 | `Flow<List<SongEntity>>` | title のみ LIKE |
| `searchSongsLimitedLike(query, ..., limit)` | 1021 | `Flow<List<SongEntity>>` | title + artist LIKE |
| `searchSongsLimited(query, ..., limit, titleOnly)` | 1028 | `Flow<List<SongEntity>>` | `searchSongs` の LIMIT 版。`combine` でマージ後に `take(limit)` |
| `getSongsByGenrePaginated(genreName, ...)` | 1074 | `PagingSource<Int, SongEntity>` | ジャンル ページング |

### 1.8 統計 / 集計

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getSongCount()` | 523 | `Flow<Int>` | `SELECT COUNT(*) FROM songs` |
| `getCloudSongCount()` | 526 | `Flow<Int>` | `WHERE source_type != 0` |
| `getSongCountOnce()` | 529 | `Int` (suspend) | ワンショット カウント |
| `getDeviceCapabilitySongRows()` | 542 | `List<DeviceCapabilitySongRow>` | デバイス機能チェック (ExoPlayer / 形式判定) |
| `getLibraryAudioStats()` | 572 | `LibraryAudioStatsRow` | ライブラリ全体統計。hi-res = `sample_rate > 48000`、ultra-hi-res = `>= 176400`。`likelyExpensive` は hi-res + flac/alac/wav/aiff/ape の MIME。バイト推定 = `bitrate × duration / 8000` |
| `getMimeTypeCounts()` | 576 | `List<MimeTypeCountRow>` | MIME 分布 |
| `getRandomSongs(limit, allowedParentDirs, applyDirectoryFilter)` | 588 | `List<SongEntity>` (suspend) | `ORDER BY RANDOM() LIMIT :limit` |
| `getFirstPlayableSong(allowedParentDirs, applyDirectoryFilter)` | 601 | `SongEntity?` | `title COLLATE NOCASE ASC` で先頭 1 件。シャッフル フォールバック |
| `getAllSongs(allowedParentDirs, applyDirectoryFilter)` | 610 | `Flow<List<SongEntity>>` | ディレクトリ フィルタのみ |
| `getDistinctAlbumArtSongs(allowedParentDirs, applyDirectoryFilter)` | 628 | `Flow<List<SongEntity>>` | アート URI ごとに MIN(id) 代表曲 |
| `getHomeMixPreviewSongs(limit, ...)` | 640 | `Flow<List<SongEntity>>` | `ORDER BY date_added DESC, id DESC LIMIT :limit` (Home Mix プレビュー用) |
| `getFolderSongs(allowedParentDirs, applyDirectoryFilter, filterMode)` | 662 | `Flow<List<FolderSongRow>>` | フォルダ ツリー構築用プロジェクション。`filterMode: 0=ALL, 1=OFFLINE, 2=ONLINE` |
| `getSongIdsSorted(allowedParentDirs, applyDirectoryFilter, sortOrder, filterMode)` | 697 | `List<Long>` (suspend) | `SortOption.storageKey` で動的ソート。`CASE WHEN` パターン |
| `getFavoriteSongIdsSorted(...)` | 731 | `List<Long>` (suspend) | お気に入り ID ソート |

### 1.9 ページング ソース (Paging 3)

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getSongsPaginated(allowedParentDirs, applyDirectoryFilter, sortOrder, filterMode)` | 774 | `PagingSource<Int, SongEntity>` | 曲リスト ページング |
| `getSongsPage(...sortOrder, filterMode, limit, offset)` | 812 | `List<SongEntity>` (suspend) | 曲リスト オフセット ページング |
| `getFavoriteSongsPaginated(...sortOrder, filterMode)` | 853 | `PagingSource<Int, SongEntity>` | お気に入り ページング |
| `getFavoriteSongsList(...filterMode)` | 880 | `List<SongEntity>` (suspend) | お気に入り 全件 |
| `getFavoriteSongsPage(...sortOrder, filterMode, limit, offset)` | 915 | `List<SongEntity>` (suspend) | お気に入り オフセット |
| `getFavoriteSongCount(...filterMode)` | 943 | `Flow<Int>` | お気に入り カウント |

### 1.10 アルバム クエリ

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getAlbums(allowedParentDirs, applyDirectoryFilter, filterMode, minTracks)` | 1118 | `Flow<List<AlbumEntity>>` | アルバム取得 (`songs` JOIN で `song_count` 集計、`HAVING COUNT >= :minTracks`) |
| `getAlbumsPaginated(..., sortOrder, minTracks)` | 1174 | `PagingSource<Int, AlbumEntity>` | アルバム ページング |
| `getAlbumsPage(..., sortOrder, minTracks, limit, offset)` | 1232 | `List<AlbumEntity>` (suspend) | アルバム オフセット |
| `getAlbumById(albumId)` | 1261 | `Flow<AlbumEntity?>` | アルバム ID 単体。サブクエリで `song_count` を計算 |
| `searchAlbums(query, minTracks=1)` | 1283 | `Flow<List<AlbumEntity>>` | `title LIKE '%query%'`。曲数フィルタ |
| `getAlbumCount()` | 1286 | `Flow<Int>` | — |
| `getAllAlbumsList(allowedParentDirs, applyDirectoryFilter, minTracks)` | 1315 | `List<AlbumEntity>` (suspend) | 全アルバム ワンショット |
| `getAlbumsByArtistId(artistId)` | 1346 | `Flow<List<AlbumEntity>>` | LEFT JOIN songs、`artist_id = :artistId` |
| `searchAlbums(query, allowedParentDirs, applyDirectoryFilter, minTracks)` | 1375 | `Flow<List<AlbumEntity>>` | タイトル + アーティスト名 LIKE |

### 1.11 アーティスト クエリ

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getArtists(allowedParentDirs, applyDirectoryFilter)` | 1389 | `Flow<List<ArtistEntity>>` | `songs.artist_id` JOIN。distinct |
| `getArtistsPaginated(...filterMode, sortOrder)` | 1421 | `PagingSource<Int, ArtistEntity>` | junction (`song_artist_cross_ref`) 経由で曲数を集計 |
| `getArtistsPage(...filterMode, sortOrder, limit, offset)` | 1456 | `List<ArtistEntity>` (suspend) | 上記 オフセット版 |
| `getAllArtistsRaw()` | 1469 | `Flow<List<ArtistEntity>>` | junction 経由せず、`artists` テーブル全件 |
| `getArtistById(artistId)` | 1472 | `Flow<ArtistEntity?>` | — |
| `searchArtists(query)` | 1475 | `Flow<List<ArtistEntity>>` | `name LIKE '%query%'` |
| `getArtistCount()` | 1478 | `Flow<Int>` | — |
| `getAllArtistsList(allowedParentDirs, applyDirectoryFilter)` | 1487 | `List<ArtistEntity>` (suspend) | JOIN 経由のワンショット |
| `getAllArtistsListRaw()` | 1496 | `List<ArtistEntity>` (suspend) | junction 経由しない |
| `searchArtists(query, allowedParentDirs, applyDirectoryFilter)` | 1509 | `Flow<List<ArtistEntity>>` | junction 経由 |
| `getArtistImageUrl(artistId)` | 1517 | `String?` | `SELECT image_url FROM artists WHERE id = :artistId` |
| `getArtistImageUrlByNormalizedName(name)` | 1520 | `String?` | `LOWER(TRIM(name)) = LOWER(TRIM(:name))` |
| `updateArtistImageUrl(artistId, imageUrl)` | 1523 | `Unit` | — |
| `getArtistIdByName(name)` | 1526 | `Long?` | 完全一致 |
| `getArtistIdByNormalizedName(name)` | 1529 | `Long?` | 大文字小文字 / 空白無視 |
| `getMaxArtistId()` | 1532 | `Long?` | `SELECT MAX(id) FROM artists` |
| `updateArtistCustomImage(artistId, uri)` | 1536 | `Unit` | `custom_image_uri` 更新 |
| `getArtistCustomImage(artistId)` | 1539 | `String?` | — |

### 1.12 ジャンル クエリ

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getSongsByGenre(genreName, allowedParentDirs, applyDirectoryFilter)` | 1549 | `Flow<List<SongEntity>>` | `genre LIKE :genreName` |
| `getSongsByGenreContaining(genreName, genrePrefix, ...)` | 1573 | `Flow<List<SongEntity>>` | カンマ区切り複合ジャンルの構成要素マッチ (6 つの LIKE パターン) |
| `getSongsWithNullGenre(...)` | 1590 | `Flow<List<SongEntity>>` | `genre IS NULL OR genre = ''` |
| `getUniqueGenres()` | 1597 | `Flow<List<String>>` | 全曲から DISTINCT |
| `getUniqueGenres(allowedParentDirs, applyDirectoryFilter)` | 1605 | `Flow<List<String>>` | ディレクトリ フィルタ適用 |
| `hasUnknownGenre(allowedParentDirs, applyDirectoryFilter)` | 1617 | `Flow<Boolean>` | "Unknown Genre" バッジ表示用 |

### 1.13 アルバム アート / 統合

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getAllUniqueAlbumArtUrisFromSongs()` | 1625 | `Flow<List<String>>` | テーマ プリロード用 |
| `deleteOrphanedAlbums()` | 1628 | `Unit` | `songs.album_id` で参照されない album を削除 |
| `deleteOrphanedArtists()` | 1631 | `Unit` | `song_artist_cross_ref` で参照されない artist を削除 |

### 1.14 お気に入り / メタデータ更新

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `setFavoriteStatus(songId, isFavorite)` | 1635 | `Unit` | `songs.is_favorite` 更新 |
| `getFavoriteStatus(songId)` | 1638 | `Boolean?` | — |
| `toggleFavoriteStatus(songId)` | 1642 | `Boolean` (`@Transaction`) | 現在の状態を反転、新状態を返す |
| `updateSongMetadata(songId, title, artist, artistId, artistsJson, album, albumArtist, genre, trackNumber, discNumber)` | 1662 | `Unit` | 10 列 UPDATE |
| `updateSongMetadataAndArtistLinks(songId, ..., artistsToEnsure, crossRefs)` | 1676 | `Unit` (`@Transaction`) | アーティスト upsert → メタデータ UPDATE → 既存 cross_ref 全削除 → 新規 cross_ref チャンク insert → 孤立 artist 掃除 |
| `updateSongAlbumArt(songId, albumArtUri)` | 1718 | `Unit` | — |
| `updateLyrics(songId, lyrics)` | 1721 | `Unit` | `songs.lyrics` 直接書き込み |
| `resetLyrics(songId)` | 1724 | `Unit` | `songs.lyrics = NULL` |
| `resetAllLyrics()` | 1727 | `Unit` | 全曲の lyrics を NULL |
| `getAllSongsList()` | 1730 | `List<SongEntity>` (suspend) | 全曲 `SONG_LIST_PROJECTION` |
| `getAllLocalSongSummaries()` | 1733 | `List<SongSummary>` (suspend) | 軽量プロジェクション (`source_type = 0` のみ) |
| `getAlbumArtUriById(id)` | 1736 | `String?` | — |
| `deleteById(id)` | 1739 | `Unit` | `DELETE FROM songs WHERE id = :id` |
| `getAudioMetadataById(id)` | 1748 | `AudioMeta?` | `mimeType`, `bitrate`, `sampleRate` のみ |

### 1.15 Song-Artist Cross Ref (Junction)

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `insertSongArtistCrossRefs(crossRefs)` | 1753 | `Unit` | `@Insert IGNORE` |
| `getAllSongArtistCrossRefs()` | 1756 | `Flow<List<SongArtistCrossRef>>` | 全 junction |
| `getAllSongArtistCrossRefsList()` | 1759 | `List<SongArtistCrossRef>` (suspend) | ワンショット |
| `clearAllSongArtistCrossRefs()` | 1762 | `Unit` | — |
| `deleteCrossRefsForSong(songId)` | 1765 | `Unit` | — |
| `deleteCrossRefsForArtist(artistId)` | 1768 | `Unit` | — |
| `getArtistsForSong(songId)` | 1779 | `Flow<List<ArtistEntity>>` | junction 経由、`is_primary DESC, name ASC` |
| `getArtistsForSongList(songId)` | 1790 | `List<ArtistEntity>` (suspend) | ワンショット |
| `getSongsForArtist(artistId)` | 1801 | `Flow<List<SongEntity>>` | 逆方向 |
| `getSongsForArtistList(artistId)` | 1812 | `List<SongEntity>` (suspend) | 逆方向 ワンショット |
| `getCrossRefsForSong(songId)` | 1818 | `List<SongArtistCrossRef>` (suspend) | junction レコードそのもの |
| `getPrimaryArtistForSong(songId)` | 1829 | `PrimaryArtistInfo?` | `is_primary = 1` 1 件 |
| `getSongCountForArtist(artistId)` | 1835 | `Int` (suspend) | junction 経由カウント |
| `getArtistsWithSongCounts()` | 1848 | `Flow<List<ArtistEntity>>` | 全アーティスト + junction 経由カウント |
| `getArtistsWithSongCountsFiltered(allowedParentDirs, applyDirectoryFilter, filterMode)` | 1874 | `Flow<List<ArtistEntity>>` | ディレクトリ + ソース フィルタ適用 |
| `clearAllMusicDataWithCrossRefs()` | 1884 | `Unit` (`@Transaction`) | junction → 曲 → アルバム → アーティスト の順で全削除 |
| `insertMusicDataWithCrossRefs(songs, albums, artists, crossRefs)` | 1896 | `Unit` (`@Transaction`) | junction を `CROSS_REF_BATCH_SIZE` チャンクで挿入 |
| `rebuildMusicDataWithCrossRefs(songs, albums, artists, crossRefs)` | 1913 | `Unit` (`@Transaction`) | 全削除 → 全再挿入 |

### 1.16 内部実装メモ (MusicDao)

- **FTS + LIKE 二段構え**: `searchSongs` 系は `combine` で 2 つの Flow をマージ。`LinkedHashMap.putIfAbsent` で FTS 結果を優先しつつ LIKE 結果を追加。重複 ID は排除
- **CursorWindow 制限回避**: `SONG_LIST_PROJECTION` は `lyrics = NULL` で射影。Now Playing のみ `SONG_DETAIL_PROJECTION`
- **ディレクトリ フィルタ**: 共通パターン `(:applyDirectoryFilter = 0 OR id < 0 OR parent_directory_path IN (:allowedParentDirs))` — `id < 0` はクラウド曲 (Telegram 等) を常に含める特例
- **ソース フィルタ**: `filterMode: 0=ALL, 1=OFFLINE (source_type=0), 2=ONLINE (source_type!=0)`。`StorageFilter.value` に対応
- **動的ソート**: `sortOrder: String` パラメータ + `CASE WHEN` で `SortOption.storageKey` を SQL の ORDER BY に展開
- **多段トランザクション**: `clearAllMusicDataWithCrossRefs` のように FK CASCADE に頼らず明示的に削除順を制御

---

## 2. `AlbumArtThemeDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/AlbumArtThemeDao.kt:9`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `insertTheme(theme)` | 11 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `getThemeByUriAndStyle(uriString, paletteStyle)` | 16 | `AlbumArtThemeEntity?` (suspend) | `(albumArtUriString, paletteStyle)` で取得 |
| `deleteThemesByUris(uriStrings)` | 19 | `Unit` (suspend) | `WHERE albumArtUriString IN (:uriStrings)` |

---

## 3. `AiCacheDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/AiCacheDao.kt:9`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `insert(cache)` | 11 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `getCache(hash)` | 14 | `AiCacheEntity?` (suspend) | `promptHash = :hash` |
| `clearOldCache(olderThanTimestamp)` | 17 | `Unit` (suspend) | `timestamp < :olderThanTimestamp` で TTL 掃除 |
| `clearAllCache()` | 20 | `Unit` (suspend) | 全削除 |

---

## 4. `AiUsageDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/AiUsageDao.kt:9`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getAllUsagesOnce()` | 11 | `List<AiUsageEntity>` (suspend) | 全件 |
| `getUsageCount()` | 14 | `Int` (suspend) | `COUNT(*)` |
| `insertUsage(usage)` | 17 | `Unit` (suspend) | `@Insert` |
| `insertAll(usages)` | 20 | `Unit` (suspend) | `@Insert` バルク |
| `getRecentUsages(limit)` | 23 | `Flow<List<AiUsageEntity>>` | `ORDER BY timestamp DESC LIMIT :limit` |
| `getTotalPromptTokens()` | 26 | `Flow<Int?>` | `SUM(promptTokens)` |
| `getTotalOutputTokens()` | 29 | `Flow<Int?>` | 同上 |
| `getTotalThoughtTokens()` | 32 | `Flow<Int?>` | 同上 |
| `clearUsage()` | 35 | `Unit` (suspend) | 全削除 |
| `clearAll()` | 38 | `Unit` (suspend) | 全削除 (clearUsage のエイリアス) |

---

## 5. `EngagementDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/EngagementDao.kt:15`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `upsertEngagement(engagement)` | 18 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `upsertEngagements(engagements)` | 21 | `Unit` (suspend) | バルク |
| `getEngagement(songId)` | 24 | `SongEngagementEntity?` (suspend) | — |
| `getAllEngagements()` | 27 | `List<SongEngagementEntity>` (suspend) | 全件 |
| `getAllEngagementsFlow()` | 30 | `Flow<List<SongEngagementEntity>>` | リアクティブ |
| `getPlayCount(songId)` | 33 | `Int?` (suspend) | `SELECT play_count` |
| `deleteEngagement(songId)` | 36 | `Unit` (suspend) | — |
| `deleteOrphanedEngagements()` | 39 | `Unit` (suspend) | `songs` テーブルに存在しない `song_id` の engagement を削除 |
| `clearAllEngagements()` | 42 | `Unit` (suspend) | 全削除 |
| `recordPlay(songId, durationMs, timestamp)` | 56 | `Unit` (suspend) | `INSERT ... ON CONFLICT DO UPDATE SET play_count + 1, total_play_duration_ms + :durationMs` のアトミック increment |
| `getTopPlayedSongs(limit)` | 62 | `List<SongEngagementEntity>` (suspend) | `ORDER BY play_count DESC LIMIT :limit` |
| `getRecentlyPlayedSongs(limit)` | 68 | `List<SongEngagementEntity>` (suspend) | `last_played_timestamp > 0 ORDER BY ... DESC LIMIT :limit` |
| `replaceAll(engagements)` | 71 | `Unit` (`@Transaction`) | `clearAllEngagements` → バルク insert |

---

## 6. `FavoritesDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/FavoritesDao.kt:12`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `setFavorite(favorite)` | 14 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `insertAll(favorites)` | 17 | `Unit` (suspend) | バルク |
| `removeFavorite(songId)` | 20 | `Unit` (suspend) | `DELETE FROM favorites WHERE songId = :songId` |
| `isFavorite(songId)` | 23 | `Boolean?` (suspend) | `SELECT isFavorite` |
| `getFavoriteSongIdsRaw()` | 26 | `Flow<List<Long>>` | `isFavorite = 1 ORDER BY songId` |
| `getFavoriteSongIds()` | 28 | `Flow<List<Long>>` | 上記に `.distinctUntilChanged()` を付けた派生 |
| `getFavoriteSongIdsOnce()` | 31 | `List<Long>` (suspend) | ワンショット |
| `getAllFavoritesOnce()` | 34 | `List<FavoritesEntity>` (suspend) | 全件 |
| `clearAll()` | 37 | `Unit` (suspend) | 全削除 |
| `replaceAll(favorites)` | 40 | `Unit` (`@Transaction`) | `clearAll` → バルク insert |

---

## 7. `GDriveDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/GDriveDao.kt:10`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getAllGDriveSongs()` | 15 | `Flow<List<GDriveSongEntity>>` | `ORDER BY date_added DESC` |
| `getAllGDriveSongsList()` | 18 | `List<GDriveSongEntity>` (suspend) | ワンショット |
| `getSongsByFolder(folderId)` | 21 | `Flow<List<GDriveSongEntity>>` | `folder_id = :folderId ORDER BY title ASC` |
| `searchSongs(query)` | 24 | `Flow<List<GDriveSongEntity>>` | `title/artist LIKE '%query%'` |
| `getSongsByIds(ids)` | 27 | `Flow<List<GDriveSongEntity>>` | — |
| `insertSongs(songs)` | 30 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `deleteSong(songId)` | 33 | `Unit` (suspend) | — |
| `deleteSongsByFolder(folderId)` | 36 | `Unit` (suspend) | — |
| `insertFolder(folder)` | 41 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `getAllFolders()` | 44 | `Flow<List<GDriveFolderEntity>>` | `ORDER BY name ASC` |
| `getAllFoldersList()` | 47 | `List<GDriveFolderEntity>` (suspend) | ワンショット |
| `deleteFolder(folderId)` | 50 | `Unit` (suspend) | — |
| `clearAllSongs()` | 55 | `Unit` (suspend) | 全削除 |
| `clearAllFolders()` | 58 | `Unit` (suspend) | 全削除 |

---

## 8. `JellyfinDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/JellyfinDao.kt:10`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getAllJellyfinSongs()` | 15 | `Flow<List<JellyfinSongEntity>>` | — |
| `getAllJellyfinSongsList()` | 18 | `List<JellyfinSongEntity>` (suspend) | — |
| `getSongsByPlaylist(playlistId)` | 21 | `Flow<List<JellyfinSongEntity>>` | `playlist_id = :playlistId ORDER BY date_added DESC` |
| `searchSongs(query)` | 24 | `Flow<List<JellyfinSongEntity>>` | LIKE 検索 |
| `getSongsByIds(ids)` | 27 | `Flow<List<JellyfinSongEntity>>` | — |
| `getSongByJellyfinId(jellyfinId)` | 30 | `JellyfinSongEntity?` (suspend) | `LIMIT 1` |
| `insertSongs(songs)` | 33 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `insertSong(song)` | 36 | `Unit` (suspend) | 単体 |
| `deleteSong(songId)` | 39 | `Unit` (suspend) | — |
| `deleteSongsByPlaylist(playlistId)` | 42 | `Unit` (suspend) | — |
| `insertPlaylist(playlist)` | 47 | `Unit` (suspend) | — |
| `insertPlaylists(playlists)` | 50 | `Unit` (suspend) | バルク |
| `getAllPlaylists()` | 53 | `Flow<List<JellyfinPlaylistEntity>>` | `ORDER BY name ASC` |
| `getAllPlaylistsList()` | 56 | `List<JellyfinPlaylistEntity>` (suspend) | — |
| `getPlaylistById(playlistId)` | 59 | `JellyfinPlaylistEntity?` (suspend) | `LIMIT 1` |
| `deletePlaylist(playlistId)` | 62 | `Unit` (suspend) | — |
| `clearSongsByPlaylist(playlistId)` | 65 | `Unit` (suspend) | プレイリスト内の曲のみクリア |
| `getPlaylistCount()` | 68 | `Int` (suspend) | — |
| `getAllDistinctJellyfinIds()` | 71 | `List<String>` (suspend) | `SELECT DISTINCT jellyfin_id` |
| `clearLibrarySongs()` | 74 | `Unit` (suspend) | `playlist_id = '__library__'` の曲のみ削除 |
| `clearAllSongs()` | 79 | `Unit` (suspend) | 全削除 |
| `clearAllPlaylists()` | 82 | `Unit` (suspend) | 全削除 |

---

## 9. `LocalPlaylistDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/LocalPlaylistDao.kt:12`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `observePlaylistsWithSongs()` | 15 | `Flow<List<PlaylistWithSongsEntity>>` | `@Transaction @Query` `ORDER BY last_modified DESC` |
| `observePlaylistWithSongs(playlistId)` | 19 | `Flow<PlaylistWithSongsEntity?>` | `@Transaction` |
| `observePlaylistSongs(playlistId)` | 22 | `Flow<List<PlaylistSongEntity>>` | `ORDER BY sort_order ASC` |
| `getPlaylistById(playlistId)` | 25 | `PlaylistEntity?` (suspend) | `LIMIT 1` |
| `getPlaylistCount()` | 28 | `Int` (suspend) | `COUNT(*)` |
| `upsertPlaylist(entity)` | 31 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `updatePlaylist(entity)` | 34 | `Unit` (suspend) | `@Update` |
| `deletePlaylist(playlistId)` | 37 | `Unit` (suspend) | — |
| `upsertPlaylistSongs(entities)` | 40 | `Unit` (suspend) | `@Insert(REPLACE)` バルク |
| `clearPlaylistSongs(playlistId)` | 43 | `Unit` (suspend) | — |
| `clearAllPlaylistSongs()` | 46 | `Unit` (suspend) | 全削除 |
| `clearAllPlaylists()` | 49 | `Unit` (suspend) | 全削除 |
| `replacePlaylistSongs(playlistId, songIds)` | 52 | `Unit` (`@Transaction`) | `clearPlaylistSongs` → `mapIndexed` で `PlaylistSongEntity(sortOrder = index)` を生成 → upsert |
| `replaceAllPlaylistsTransactional(playlists)` | 66 | `Unit` (`@Transaction`) | 全削除 → 各プレイリストを upsert + replacePlaylistSongs |

---

## 10. `LyricsDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/LyricsDao.kt:10`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `insert(lyrics)` | 12 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `insertAll(lyrics)` | 15 | `Unit` (suspend) | バルク |
| `getLyrics(songId)` | 18 | `LyricsEntity?` (suspend) | — |
| `deleteLyrics(songId)` | 21 | `Unit` (suspend) | — |
| `deleteAll()` | 24 | `Unit` (suspend) | 全削除 |
| `getAll()` | 27 | `List<LyricsEntity>` (suspend) | 全件 |
| `getSongIdsWithLyrics(songIds)` | 30 | `List<Long>` (suspend) | `songId IN (:songIds) AND content != ''` — バルク判定 |
| `replaceAll(lyrics)` | 33 | `Unit` (`@Transaction`) | `deleteAll` → バルク insert |

---

## 11. `NavidromeDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/NavidromeDao.kt:13`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getAllNavidromeSongs()` | 18 | `Flow<List<NavidromeSongEntity>>` | — |
| `getAllNavidromeSongsList()` | 21 | `List<NavidromeSongEntity>` (suspend) | — |
| `getSongsByPlaylist(playlistId)` | 24 | `Flow<List<NavidromeSongEntity>>` | — |
| `searchSongs(query)` | 27 | `Flow<List<NavidromeSongEntity>>` | LIKE |
| `getSongsByIds(ids)` | 30 | `Flow<List<NavidromeSongEntity>>` | — |
| `getSongByNavidromeId(navidromeId)` | 33 | `NavidromeSongEntity?` (suspend) | `LIMIT 1` |
| `insertSongs(songs)` | 36 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `insertSong(song)` | 39 | `Unit` (suspend) | 単体 |
| `deleteSong(songId)` | 42 | `Unit` (suspend) | — |
| `deleteSongsByPlaylist(playlistId)` | 45 | `Unit` (suspend) | — |
| `insertPlaylist(playlist)` | 50 | `Unit` (suspend) | — |
| `insertPlaylists(playlists)` | 53 | `Unit` (suspend) | バルク |
| `getAllPlaylists()` | 56 | `Flow<List<NavidromePlaylistEntity>>` | — |
| `getAllPlaylistsList()` | 59 | `List<NavidromePlaylistEntity>` (suspend) | — |
| `getPlaylistById(playlistId)` | 62 | `NavidromePlaylistEntity?` (suspend) | — |
| `deletePlaylist(playlistId)` | 65 | `Unit` (suspend) | — |
| `getPlaylistCount()` | 68 | `Int` (suspend) | `COUNT(*)` |
| `getAllDistinctNavidromeIds()` | 71 | `List<String>` (suspend) | `SELECT DISTINCT navidrome_id` |
| `clearLibrarySongs()` | 74 | `Unit` (suspend) | `playlist_id = '__library__'` の曲のみ削除 |
| `clearAllSongs()` | 79 | `Unit` (suspend) | 全削除 |
| `clearAllPlaylists()` | 82 | `Unit` (suspend) | 全削除 |

---

## 12. `NeteaseDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/NeteaseDao.kt:10`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getAllNeteaseSongs()` | 15 | `Flow<List<NeteaseSongEntity>>` | — |
| `getAllNeteaseSongsList()` | 18 | `List<NeteaseSongEntity>` (suspend) | — |
| `getNeteaseCount()` | 22 | `Int` (suspend) | `COUNT(*)` — 軽量ガード チェック |
| `getSongsByPlaylist(playlistId)` | 25 | `Flow<List<NeteaseSongEntity>>` | — |
| `searchSongs(query)` | 28 | `Flow<List<NeteaseSongEntity>>` | LIKE |
| `getSongsByIds(ids)` | 31 | `Flow<List<NeteaseSongEntity>>` | — |
| `insertSongs(songs)` | 34 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `deleteSong(songId)` | 37 | `Unit` (suspend) | — |
| `deleteSongsByPlaylist(playlistId)` | 40 | `Unit` (suspend) | — |
| `insertPlaylist(playlist)` | 45 | `Unit` (suspend) | — |
| `getAllPlaylists()` | 48 | `Flow<List<NeteasePlaylistEntity>>` | — |
| `getAllPlaylistsList()` | 51 | `List<NeteasePlaylistEntity>` (suspend) | — |
| `deletePlaylist(playlistId)` | 54 | `Unit` (suspend) | — |
| `clearAllSongs()` | 59 | `Unit` (suspend) | 全削除 |
| `clearAllPlaylists()` | 62 | `Unit` (suspend) | 全削除 |

---

## 13. `QqMusicDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/QqMusicDao.kt:10`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getAllQqMusicSongs()` | 15 | `Flow<List<QqMusicSongEntity>>` | — |
| `getAllQqMusicSongsList()` | 18 | `List<QqMusicSongEntity>` (suspend) | — |
| `getSongsByPlaylist(playlistId)` | 21 | `Flow<List<QqMusicSongEntity>>` | — |
| `searchSongs(query)` | 24 | `Flow<List<QqMusicSongEntity>>` | LIKE |
| `getSongsByIds(ids)` | 27 | `Flow<List<QqMusicSongEntity>>` | — |
| `insertSongs(songs)` | 30 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `deleteSong(songId)` | 33 | `Unit` (suspend) | — |
| `deleteSongsByPlaylist(playlistId)` | 36 | `Unit` (suspend) | — |
| `insertPlaylist(playlist)` | 41 | `Unit` (suspend) | — |
| `getAllPlaylists()` | 44 | `Flow<List<QqMusicPlaylistEntity>>` | — |
| `getAllPlaylistsList()` | 47 | `List<QqMusicPlaylistEntity>` (suspend) | — |
| `deletePlaylist(playlistId)` | 50 | `Unit` (suspend) | — |

---

## 14. `SearchHistoryDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/SearchHistoryDao.kt:10`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `insert(item)` | 12 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `insertAll(items)` | 15 | `Unit` (suspend) | バルク |
| `getRecentSearches(limit)` | 18 | `List<SearchHistoryEntity>` (suspend) | `ORDER BY timestamp DESC LIMIT :limit` |
| `deleteByQuery(query)` | 21 | `Unit` (suspend) | `WHERE query = :query` |
| `clearAll()` | 24 | `Unit` (suspend) | 全削除 |
| `getAll()` | 27 | `List<SearchHistoryEntity>` (suspend) | 全件 |
| `replaceAll(items)` | 30 | `Unit` (`@Transaction`) | `clearAll` → バルク insert |

---

## 15. `TelegramDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/TelegramDao.kt:11`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `getAllTelegramSongs()` | 13 | `Flow<List<TelegramSongEntity>>` | `ORDER BY date_added DESC` |
| `searchSongs(query)` | 16 | `Flow<List<TelegramSongEntity>>` | LIKE |
| `getSongsByIds(ids)` | 19 | `Flow<List<TelegramSongEntity>>` | — |
| `getSongsByChatId(chatId)` | 22 | `List<TelegramSongEntity>` (suspend) | — |
| `getSongsByTopicId(chatId, threadId)` | 25 | `List<TelegramSongEntity>` (suspend) | — |
| `getSongByFileId(fileId)` | 28 | `TelegramSongEntity?` (suspend) | `LIMIT 1` |
| `insertSongs(songs)` | 31 | `Unit` (suspend) | `@Insert(REPLACE)` |
| `deleteSong(id)` | 34 | `Unit` (suspend) | — |
| `insertChannel(channel)` | 37 | `Unit` (suspend) | — |
| `getAllChannels()` | 40 | `Flow<List<TelegramChannelEntity>>` | `ORDER BY title ASC` |
| `deleteChannel(chatId)` | 43 | `Unit` (suspend) | — |
| `deleteSongsByChatId(chatId)` | 46 | `Unit` (suspend) | — |
| `deleteSongsByTopicId(chatId, threadId)` | 49 | `Unit` (suspend) | — |
| `clearAll()` | 52 | `Unit` (`@Transaction`) | clearAllSongs + clearAllChannels + clearAllTopics |
| `clearAllSongs()` | 59 | `Unit` (suspend) | — |
| `clearAllChannels()` | 62 | `Unit` (suspend) | — |
| `clearAllTopics()` | 65 | `Unit` (suspend) | — |
| `insertTopic(topic)` | 70 | `Unit` (suspend) | — |
| `insertTopics(topics)` | 73 | `Unit` (suspend) | バルク |
| `getTopicsByChannel(chatId)` | 76 | `Flow<List<TelegramTopicEntity>>` | `ORDER BY name ASC` |
| `getTopicsByChannelOnce(chatId)` | 79 | `List<TelegramTopicEntity>` (suspend) | — |
| `getTopicById(id)` | 82 | `TelegramTopicEntity?` (suspend) | — |
| `deleteTopicsByChannel(chatId)` | 85 | `Unit` (suspend) | — |
| `getAllTopics()` | 88 | `Flow<List<TelegramTopicEntity>>` | — |
| `deleteTopic(id)` | 91 | `Unit` (suspend) | — |

---

## 16. `TransitionDao`

`app/src/main/java/com/theveloper/pixelplay/data/database/TransitionDao.kt:13`

| メソッド | 行 | 戻り値 | 目的 |
|---------|----|--------|------|
| `setRule(rule)` | 19 | `Unit` (suspend) | `@Upsert` |
| `setRules(rules)` | 22 | `Unit` (suspend) | バルク |
| `getPlaylistDefaultRule(playlistId)` | 29 | `Flow<TransitionRuleEntity?>` | `fromTrackId IS NULL AND toTrackId IS NULL` |
| `getSpecificRule(playlistId, fromTrackId, toTrackId)` | 35 | `Flow<TransitionRuleEntity?>` | 特定トラック ペア |
| `getAllRulesForPlaylist(playlistId)` | 42 | `Flow<List<TransitionRuleEntity>>` | デフォルト + 個別両方 |
| `deleteRule(ruleId)` | 48 | `Unit` (suspend) | PK 削除 |
| `deletePlaylistDefaultRule(playlistId)` | 54 | `Unit` (suspend) | デフォルトのみ削除 |
| `getAllRulesOnce()` | 57 | `List<TransitionRuleEntity>` (suspend) | 全件 |
| `clearAllRules()` | 60 | `Unit` (suspend) | 全削除 |
| `replaceAllRules(rules)` | 63 | `Unit` (`@Transaction`) | `clearAllRules` → バルク setRules |

---

## 内部実装メモ

### 共通パターン

- **`@Insert(REPLACE)` + `@Update` の使い分け**: `REPLACE` は id 衝突時に DELETE + INSERT 副作用がある。junctions や派生テーブルでは `IGNORE` を使って個別 UPDATE に切替える
- **`@Insert` の戻り値 `List<Long>`**: 各 row の rowid、衝突時は `-1L` (IGNORE 時)
- **`Flow<>` vs `suspend fun`**: UI 監視は `Flow`、ワンショット取得は `suspend fun` (Repository が `first()` などで消費)
- **`@Transaction`**: 複数 DAO 呼出を 1 トランザクションにまとめる (Room が自動生成)
- **`@Upsert`**: `setRule` 系で使用。Room 2.5+ で利用可能。INSERT と UPDATE を自動判定

### Repository 層への示唆 (推測)

- `MusicDao` の多すぎるオーバーロード → Repository で意味のある単位 (例: `searchSongsInDirectory(...)`) にラップ
- `clearAll<X>Songs()` 系 → 「クラウド ソース ログアウト」時の完全クリア API
- `incrementalSyncMusicData` → SyncWorker のメイン エントリ ポイント

---

## 関連ファイル

- Entity 層: [`database-entities.md`](./database-entities.md)
- DB 統合 + Migration: [`database-system.md`](./database-system.md)
- モデル層: [`models.md`](./models.md)