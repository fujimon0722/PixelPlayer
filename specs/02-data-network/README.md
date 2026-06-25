# 02 — Data & Network Integration Layer

> PixelPlayer の **ネットワーク/外部サービス統合層** のメソッドレベル仕様書。
> 対象は `app/src/main/java/com/theveloper/pixelplay/data/` 配下のうち、**クラウド/外部ソース統合・AI 生成・ストリーミング・画像取得** に特化したサブツリー。

## 1. レイヤ責務

| サブツリー | 責務 |
| --- | --- |
| `data/network/` | 外部 HTTP API との 1:1 マッピング（Retrofit / OkHttp）。DTO/モデル変換専用。Repository には依存しない。 |
| `data/<provider>/` (jellyfin / navidrome / netease / qqmusic / gdrive / telegram) | 認証情報管理・同期・ID オフセット解決・Playlists アプリ内統合。`data/network/` 配下の API を束ねる。 |
| `data/stream/` | ローカルループバックプロキシ基盤。クラウド URL を Media3 (ExoPlayer) から直接再生できるよう localhost HTTP へブリッジ。 |
| `data/ai/` | AI プレイリスト生成・モデル列挙・キャッシュ・プロンプト構築・通知。`provider/` で OpenAI 互換 / Gemini を抽象化。 |
| `data/remote/qqmusic/` | QQ Music `musics.fcg` の暗号化・署名 (WebView JS 経由) 専用。 |
| `data/image/` | Coil 用 Fetcher 群 (ローカル / Jellyfin / Navidrome / Telegram)。 |
| `data/github/` | 静的リソース取得 (アナウンスンス・コントリビュータ一覧)。 |

## 2. ファイル一覧とリンク

| Spec ファイル | 主対象ソース | 行数目安 |
| --- | --- | --- |
| [network-jellyfin.md](./network-jellyfin.md) | `data/network/jellyfin/{JellyfinApiService, JellyfinResponseParser}.kt` | 中 |
| [network-navidrome.md](./network-navidrome.md) | `data/network/navidrome/{NavidromeApiService, NavidromeResponseParser}.kt` | 大 |
| [network-netease.md](./network-netease.md) | `data/network/netease/{NeteaseApiService, NeteaseEncryption, CryptoMode}.kt` | 大 |
| [network-qqmusic.md](./network-qqmusic.md) | `data/network/qqmusic/QqMusicApiService.kt` | 中 |
| [network-lyrics-deezer.md](./network-lyrics-deezer.md) | `data/network/lyrics/*`, `data/network/deezer/*` | 小 |
| [ai-system.md](./ai-system.md) | `data/ai/*.kt`, `data/ai/provider/*.kt` | 大 |
| [streaming-jellyfin.md](./streaming-jellyfin.md) | `data/jellyfin/{JellyfinRepository, JellyfinStreamProxy, model/*}.kt` | 大 |
| [streaming-navidrome.md](./streaming-navidrome.md) | `data/navidrome/{NavidromeRepository, NavidromeStreamProxy, model/*}.kt` | 最大 |
| [streaming-netease.md](./streaming-netease.md) | `data/netease/{NeteaseRepository, NeteaseStreamProxy}.kt` | 大 |
| [streaming-qqmusic.md](./streaming-qqmusic.md) | `data/qqmusic/{QqMusicRepository, QqMusicStreamProxy}.kt`, `data/remote/qqmusic/*.kt` | 大 |
| [streaming-gdrive.md](./streaming-gdrive.md) | `data/gdrive/*.kt` | 大 |
| [streaming-telegram.md](./streaming-telegram.md) | `data/telegram/*.kt` | 最大 |
| [streaming-cloud.md](./streaming-cloud.md) | `data/stream/{CloudMusicUtils, CloudStreamProxy, CloudStreamSecurity}.kt` | 中 |
| [image-fetchers.md](./image-fetchers.md) | `data/image/*.kt` | 中 |
| [github-integration.md](./github-integration.md) | `data/github/*.kt` | 小 |

## 3. アーキテクチャ俯瞰

```mermaid
flowchart TB
    UI[Presentation ViewModels]
    subgraph PROVIDERS[Provider Repositories]
        JR[JellyfinRepository]
        NR[NavidromeRepository]
        NER[NeteaseRepository]
        QR[QqMusicRepository]
        GR[GDriveRepository]
        TR[TelegramRepository]
    end
    subgraph NETWORK[data/network/*ApiService]
        JAS[JellyfinApiService]
        NAS[NavidromeApiService]
        NEAS[NeteaseApiService]
        QQAS[QqMusicApiService]
        GDAS[GDriveApiService]
    end
    subgraph ENC[Encryption / Signing]
        NENC[NeteaseEncryption]
        QSEC[QQMusicSecurity]
        QSGN[QQSignGenerator WebView]
        QEI[QQMusicEncryptInterceptor]
    end
    subgraph STREAM[Stream Proxies]
        JSP[JellyfinStreamProxy]
        NSP[NavidromeStreamProxy]
        NESP[NeteaseStreamProxy]
        QSP[QqMusicStreamProxy]
        GSP[GDriveStreamProxy Ktor]
        TSP[TelegramStreamProxy Ktor]
    end
    CSP{{CloudStreamProxy abstract}}
    CSS{{CloudStreamSecurity}}
    UI --> PROVIDERS
    JR --> JAS
    NR --> NAS
    NER --> NEAS
    NER --> NENC
    QR --> QQAS
    QR --> QEI
    QEI --> QSGN
    QEI --> QSEC
    GR --> GDAS
    TR -. TDLib .-> TC[TelegramClientManager]
    JAS -. audio .-> JSP
    NAS -. stream.view .-> NSP
    QR -. cgi .-> QSP
    NER -. song/url .-> NESP
    GR -. drive.google .-> GSP
    TR -. tdlib file .-> TSP
    JSP & NSP & QSP & NESP --> CSP
    GSP & TSP -.direct Ktor.- CSP
    CSP --> CSS
```

## 4. ID 空間とオフセット戦略

クラウド楽曲をローカル DB の `Song.id` (`Long`) に統合するため、各プロバイダは固定オフセットを採用。
`data/database/` のエンティティ生成側で参照される。

| Provider | Song offset | Album offset | Artist offset |
| --- | --- | --- | --- |
| Jellyfin | `12_000_000_000_000L` | `13_000_000_000_000L` | `14_000_000_000_000L` |
| Navidrome | `9_000_000_000_000L` | `10_000_000_000_000L` | `11_000_000_000_000L` |
| Netease | `3_000_000_000_000L` | `4_000_000_000_000L` | `5_000_000_000_000L` |
| QQ Music | `6_000_000_000_000L` | `7_000_000_000_000L` | `8_000_000_000_000L` |
| Google Drive | `6_000_000_000_000L` | `7_000_000_000_000L` | `8_000_000_000_000L` |
| Telegram | `-X.hashCode()` (負値) | n/a | n/a |

> GDrive と QQMusic が同じ Song offset を共有するのは、ID が一意である限り衝突しない設計 (QQMusic の Song は `songMid` の `hashCode().absoluteValue`、GDrive は `driveFileId` の `hashCode().absoluteValue`)。

## 5. 主要横断コンセプト

| 概念 | 場所 | 役割 |
| --- | --- | --- |
| `CloudStreamProxy<K>` | `data/stream/CloudStreamProxy.kt` | Ktor CIO で localhost:<port>/{prefix}/{id} を待ち受ける抽象基底。各プロバイダの Proxy が継承。 |
| `CloudStreamSecurity` | `data/stream/CloudStreamSecurity.kt` | Range ヘッダ検証・SSRF 対策 (private/CGNAT/IPv6 ローカル禁止) |
| `CloudMusicUtils` | `data/stream/CloudMusicUtils.kt` | クロスプロバイダ共通 JSON クッキー復元・アーティスト名分割 |
| `AiClient` | `data/ai/provider/AiClient.kt` | OpenAI 互換 / Gemini を統一するインターフェース |
| `AiProviderSupport` | `data/ai/provider/AiProviderSupport.kt` | フォールバックチェーン・例外ラッパー・モデル復旧 |
| `BulkSyncResult` | `data/stream/CloudMusicUtils.kt` | プレイリスト楽曲同期の結果集計 (Repository 横断で再宣言) |

## 6. 呼び出し元（被依存）

主要コンシューマ:
- `presentation/jellyfin/*`, `presentation/navidrome/*`, `presentation/netease/*`, `presentation/qqmusic/*`, `presentation/gdrive/*`, `presentation/telegram/*` — ログイン画面・ダッシュボード ViewModel
- `data/repository/MusicRepositoryImpl.kt`, `MusicService.kt`, `DualPlayerEngine.kt` — 再生時に Stream Proxy を解決して Media3 に渡す
- `data/worker/SyncWorker.kt`, `NavidromeSyncWorker.kt`, `AiWorker.kt` — バックグラウンド同期 (WorkManager)
- `data/image/*Fetcher.kt` — Coil 統合
- `presentation/viewmodel/AiStateHolder.kt`, `PlaylistViewModel.kt`, `AboutScreen.kt` — AI プレイリスト生成 UI

## 7. 関連 spec へのリンク

- データモデル/DB スキーマ: [`../01-data-foundation/database.md`](../01-data-foundation/database.md) (要作成)
- 再生エンジン統合: [`../04-engine/`](../04-engine/) (要作成)
- ViewModel/UI: [`../05-presentation-ui/`](../05-presentation-ui/) (要作成)

## 8. 凡例

- 相対パスは全てリポジトリルート (`/home/fujimon/source/PixelPlayer/`) 起点。
- `L<行番号>` はファイル内の行番号参照。
- ⚠️ 推測を含む箇所は明示し、`【推測】` と注記。
- `🎵` ミュージック、`🤖` AI、`🔐` 暗号化、`🌐` ネットワーク、`📡` ストリーミングを表章として用いている (装飾目的、ファイル出力では ASCII fallback)。
