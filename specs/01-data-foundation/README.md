# 01 — Data Foundation

> **データレイヤー基盤** の仕様書。`com.theveloper.pixelplay.data.database` および `com.theveloper.pixelplay.data.model` 配下の全 Kotlin ファイルを対象とする。

## ディレクトリ概要

| パッケージ | 役割 |
|------------|------|
| `data.database` | Room データベースの中核。Entity / Dao / Database クラスと外部サービス (Telegram / Netease / QQ / GDrive / Navidrome / Jellyfin) のキャッシュ用 Entity / Dao、AI キャッシュ、歌詞 / お気に入り / 再生統計 / プレイリスト / フォルダ ツリー / 検索履歴 / アルバムアート テーマ をすべて含む。 |
| `data.model` | UI / Repository / Service 層から参照される **ドメイン モデル**。Compose や Wear OS と共有する `Parcelable` / `Serializable` のデータ クラスと、ライブラリ UI のソート・タブ・ジャンル等の enum 群。 |

## このディレクトリの特徴

1. **DB はすべて 1 つの Room データベース (`PixelPlayDatabase`)** に集約。version = 42。Migrations は同ファイル内の companion object に 30 本以上。
2. **`songs` テーブルは全ソース統合テーブル** (LOCAL/TELEGRAM/NETEASE/GDRIVE/QQMUSIC/NAVIDROME/JELLYFIN を `source_type` 列で識別)。
3. **`Song` モデルは Compose / Coroutine / Wear / Cast へ広範囲に共有される** 中心ドメイン オブジェクト。
4. **Local Playlists は中間テーブル `playlist_songs`** による多対多。AI / Queue 生成プレイリストは `is_ai_generated` / `is_queue_generated` フラグで識別。
5. **マルチ アーティスト対応** は `song_artist_cross_ref` (junction) と `songs.artists_json` の二重管理 (前者正本、後者キャッシュ)。

## ファイル一覧

### `data/database/` (46 ファイル)

#### コア DB (Room の中核)

| ファイル | 役割 |
|----------|------|
| `PixelPlayDatabase.kt` | Room DB のエントリ ポイント。29 Entity と 15 DAO、30+ Migration を集約。version=42 |
| `MusicDao.kt` | 最大 DAO (約 1953 行)。Songs / Albums / Artists / Genres / Search / FTS / ページング / 統計 / クラウド ソース別削除 / 多対多を網羅 |
| `SongEntity.kt` | songs テーブル + `SourceType` オブジェクト + Entity↔Model 変換 (`toSong` / `toEntity` / `SongSummary`) |
| `AlbumEntity.kt` | albums テーブル + Entity↔Model 変換。アルバム アート URI の自動マイグレーション |
| `ArtistEntity.kt` | artists テーブル + Entity↔Model 変換。`trackCount` ↔ `songCount` |
| `PlaylistEntity.kt` | playlists テーブル (ユーザー定義プレイリスト) + 変換 |
| `PlaylistSongEntity.kt` | playlist_songs 中間テーブル (PK = playlist_id+song_id) |
| `PlaylistWithSongsEntity.kt` | `PlaylistEntity` + `List<PlaylistSongEntity>` を返す Relation |
| `SongArtistCrossRef.kt` | song_artist_cross_ref 中間テーブル + `SongWithArtists` / `ArtistWithSongs` / `PrimaryArtistInfo` |
| `SongSearchFtsEntity.kt` | FTS4 (`songs_fts`) 仮想テーブル用 Entity。トリガーで songs と同期 |
| `LocalPlaylistDao.kt` | プレイリスト CRUD。`@Transaction` で replace を安全に |
| `ColorConverters.kt` | `String.toComposeColor()` 拡張 (16進 → Compose Color) |
| `FolderSongRow.kt` | フォルダ ツリー構築用の軽量プロジェクション |

#### アルバム アート テーマ

| ファイル | 役割 |
|----------|------|
| `AlbumArtThemeEntity.kt` | `album_art_themes` テーブル (Primary: art URI + palette style)。100 色 (light/dark) を Embed |
| `AlbumArtThemeDao.kt` | 単一取得 / 挿入 / 一括削除 |

#### お気に入り / 歌詞 / 検索履歴 / 統計

| ファイル | 役割 |
|----------|------|
| `FavoritesEntity.kt` | `favorites` テーブル (PK = songId)。`songs.is_favorite` をトリガーで同期 |
| `FavoritesDao.kt` | `Flow<List<Long>>` でお気に入り ID を提供。`replaceAll` トランザクション |
| `SongEngagementEntity.kt` | `song_engagements` テーブル (再生回数 / 累計再生時間 / 最終再生時刻) |
| `EngagementDao.kt` | 再生カウントの atomic increment (`recordPlay`)、Top / Recent クエリ |
| `LyricsEntity.kt` | `lyrics` テーブル (PK = songId)。`isSynced` / `source` |
| `LyricsDao.kt` | 歌詞 CRUD。`getSongIdsWithLyrics` でバルク判定 |
| `SearchHistoryEntity.kt` | `search_history` テーブル。`SearchHistoryItem` (Model) との変換 |
| `SearchHistoryDao.kt` | 検索履歴の recent / clear / replaceAll |

#### トランジション (曲間クロスフェード)

| ファイル | 役割 |
|----------|------|
| `TransitionRuleEntity.kt` | `transition_rules` テーブル (PK = playlistId+fromTrackId+toTrackId)。`TransitionSettings` を Embed |
| `TransitionDao.kt` | プレイリスト単位 / 特定トラックペア / デフォルトの 3 階層ルックアップ |

#### AI キャッシュ / 使用量

| ファイル | 役割 |
|----------|------|
| `AiCacheEntity.kt` | `ai_cache` テーブル (PK = SHA-256 ハッシュ)。プロンプト応答のキャッシュ |
| `AiCacheDao.kt` | キャッシュ取得 / TTL 掃除 / 全消去 |
| `AiUsageEntity.kt` | `ai_usage` テーブル (PK = autoincrement)。トークン使用量記録 |
| `AiUsageDao.kt` | 集計 (`SUM`) を `Flow` で提供 |

#### 外部サービス: Telegram

| ファイル | 役割 |
|----------|------|
| `TelegramSongEntity.kt` | `telegram_songs` テーブル (PK = "chatId_messageId") + `toSong` 変換 |
| `TelegramChannelEntity.kt` | `telegram_channels` テーブル (PK = chatId) |
| `TelegramTopicEntity.kt` | `telegram_topics` テーブル (PK = "chatId_threadId") — フォーラム対応 |
| `TelegramDao.kt` | Songs / Channels / Topics の全 CRUD + Topic 別削除 |

#### 外部サービス: 各国クラウド音楽

| ファイル | 役割 |
|----------|------|
| `NeteaseSongEntity.kt` / `NeteasePlaylistEntity.kt` / `NeteaseDao.kt` | 网易云音乐 (Netease Cloud Music) キャッシュ |
| `QqMusicSongEntity.kt` / `QqMusicPlaylistEntity.kt` / `QqMusicDao.kt` | QQ 音楽 キャッシュ |
| `GDriveSongEntity.kt` / `GDriveFolderEntity.kt` / `GDriveDao.kt` | Google Drive の音楽ファイル キャッシュ (folder 階層対応) |
| `NavidromeSongEntity.kt` / `NavidromePlaylistEntity.kt` / `NavidromeDao.kt` | Navidrome / Subsonic API キャッシュ (playlist_id を含む) |
| `JellyfinSongEntity.kt` / `JellyfinPlaylistEntity.kt` / `JellyfinDao.kt` | Jellyfin メディア サーバー キャッシュ |

> 外部サービス固有の Entity / DAO は統合 `songs` テーブルとは **別テーブル** に格納される。
> 同じ `Song` モデルに正規化されるが、検索や `deleteByIds` の操作対象は統合 `MusicDao` の `songs` 側。

### `data/model/` (20 ファイル)

#### コア ドメインモデル

| ファイル | 役割 |
|----------|------|
| `Song.kt` | 曲データクラス (`@Parcelize`)。全ソース統合の中核。`displayArtist` / `primaryArtist` 派生プロパティ |
| `PlayList.kt` | プレイリスト データクラス + `PlaylistShapeType` enum |
| `LibraryModels.kt` | `Album` / `Artist` / `ArtistRef` (`@Parcelize` Parcelable) |
| `PlayerInfo.kt` | Now Playing + Wear / Glance 用のスナップショット (`QueueItem` / `WidgetThemeColors` / `PlayerInfo`) |
| `Transition.kt` | 曲間フェード設定 (`TransitionMode` / `Curve` / `TransitionSource` / `TransitionSettings` / `TransitionResolution` / `TransitionRule`) |
| `Lyrics.kt` | 歌詞モデル (`Lyrics` / `SyncedLine` / `SyncedWord`) |
| `LyricsSourcePreference.kt` | 歌詞取得優先度の enum (`API_FIRST` / `EMBEDDED_FIRST` / `LOCAL_FIRST`) |

#### ライブラリ UI 補助

| ファイル | 役割 |
|----------|------|
| `SortOption.kt` | ソート オプションの sealed class。Songs / Albums / Artists / Playlists / Folders / Liked 別に 38 オブジェクト。`fromStorageKey` で永続化復元 |
| `LibraryTabId.kt` | ライブラリ タブ識別 enum (`SONGS` / `ALBUMS` / `ARTISTS` / `PLAYLISTS` / `FOLDERS` / `LIKED`) + 既定 Sort |
| `SmartPlaylistRule.kt` | スマート プレイリスト enum (`TOP_PLAYED` / `RECENTLY_PLAYED` / `FORGOTTEN_FAVORITES` / `NEW_GEMS`) |
| `Genre.kt` | ジャンル データクラス (アイコン + カラー hex) |
| `SearchFilterType.kt` | 検索フィルタ enum (`ALL` / `SONGS` / `ALBUMS` / `ARTISTS` / `PLAYLISTS`) |
| `SearchHistoryItem.kt` | 検索履歴アイテム |
| `SearchResultItem.kt` | 検索結果の sealed interface (`SongItem` / `AlbumItem` / `ArtistItem` / `PlaylistItem`) |

#### ファイル / フォルダ管理

| ファイル | 役割 |
|----------|------|
| `DirectoryItem.kt` | ディレクトリ選択 UI 用 (許可 / 不許可 フラグ) |
| `FolderSource.kt` | フォルダ ソース enum (`INTERNAL` / `SD_CARD`) |
| `MusicFolder.kt` | 階層フォルダ ツリー用 (再帰的な `totalSongCount` / `totalSubFolderCount`) |

#### 再生 / 状態

| ファイル | 役割 |
|----------|------|
| `PlaybackQueueSnapshot.kt` | 再生キューの永続化用スナップショット (`PlaybackQueueItemSnapshot` + メタ) |
| `StorageFilter.kt` | ストレージ フィルタ enum (`ALL` / `OFFLINE` / `ONLINE`) |
| `SortOptionTest.kt` | `SortOption.fromStorageKey` の null 防御 テスト (コメントアウト中) |

## 仕様書ファイル

| ファイル | 内容 |
|----------|------|
| `database.md` | DB レイヤーの統合仕様。Entity / DAO / Database の主要メソッド (MusicDao は代表的に抜粋) |
| `models.md` | 全 model クラス / enum の統合仕様 |

## 上流 / 下流の代表依存

```
UI / ViewModel
  ↓ Song, Album, Artist, Playlist, Lyrics, ...
Repository (data/repository/*)
  ↓
DAO (data/database/*Dao)
  ↓
Entity (data/database/*Entity)
  ↓
Room → SQLite
```

- **上流 (呼ばれる側)**: 基本的に Repository 層 (`data/repository/MusicRepositoryImpl.kt`, `data/repository/PlaylistRepositoryImpl.kt` 等)、Worker (`data/worker/SyncWorker.kt`)、DI (`di/AppModule.kt`)。
- **下流 (呼ぶ側)**: AndroidX Room, Compose, kotlinx-serialization, kotlinx-parcelize, Gson (一部 Entity の SerializedName)。

## 関連スペック

- `../03-data-services/` — Repository / Backup / Preferences / Worker / Diagnostics がここに属する
- `../02-data-network/` — 外部サービス クライアント (Jellyfin / Navidrome / Netease / QQ / GDrive / Telegram)
- `../04-engine/` — `MusicService` が `Song` を `MediaItem` に変換して再生エンジンに渡す
- `../08-shared-module.md` — `PlayerInfo` / `QueueItem` は `shared/` 経由で Wear OS と共有される

## 既知の制約 / 注意点

- `MusicDao` の `searchSongs*` 系クエリは **FTS + LIKE の二段構え**。FTS で取りこぼすタイトルを LIKE で補完する設計
- 大きな曲リストを `Flow` で購読する際、`lyrics` 列が 2MB CursorWindow を超える問題を避けるため、`SONG_LIST_PROJECTION` は lyrics を `NULL` に射影する (詳細は `MusicDao.kt:75`)
- `MusicDao.kt:1942` `CROSS_REF_BATCH_SIZE = 999 / 3 = 333` で SQLite 変数上限を回避
- `PixelPlayDatabase.kt:182` の `MIGRATION_15_16` は `album_art_themes` を DROP して 100 列を再生成する破壊的マイグレーション
- `SongEntity.toSong()` (`SongEntity.kt:154`) は `contentUriString` のスキームから `telegram_*` / `netease_*` / `gdrive_*` / `qqmusic_*` / `navidrome_*` / `jellyfin_*` をパースしてモデル フィールドに詰め替える
- `AlbumArtThemeEntity` の 100 列は Compose Material3 の全カラー ロール (light/dark 両方)