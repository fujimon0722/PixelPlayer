# PixelPlayer コードベース仕様書 (Specs)

> **このディレクトリは個人用のドキュメントです。** 上流への PR には含めません。
> 各エージェントが書いた PixelPlayer の全モジュール詳細仕様。

## 構成

| # | ファイル/ディレクトリ | 内容 |
|---|---|---|
| 00 | `00-architecture.md` | 全体アーキテクチャ・モジュール境界・主要データフロー |
| 01 | `01-data-foundation/` | データレイヤー基盤: DB / Entity / DAO / Domain Model |
| 02 | `02-data-network/` | ネットワーク層 + 外部サービス統合 (Jellyfin / Navidrome / Netease / QQ / GDrive / Telegram / AI / Stream) |
| 03 | `03-data-services/` | データレイヤー応用: Repository / Backup / Preferences / Worker / Diagnostics / Media / Equalizer / Stats |
| 04 | `04-engine/` | 再生エンジン: `MusicService` / 内部 Player / Wear / Cast / DI / Application 起動 |
| 05 | `05-presentation-ui/` | プレゼンテーション: Screen / Component / BottomSheet |
| 06 | `06-state-navigation/` | 状態管理: ViewModel / StateHolder / Navigation / Provider 別 Dashboard |
| 07 | `07-ui-system/` | UI 基盤: テーマ / カラー / シェイプ / Glance Widget |
| 08 | `08-shared-module.md` | 共有モジュール (`shared/`) — phone↔wear 通信用モデル |
| 09 | `09-wear-module.md` | Wear OS アプリ全体 |
| 10 | `10-utils.md` | 共通ユーティリティ (`utils/`) |
| 99 | `99-correlation-diagrams.md` | 相関図・依存関係図・コールグラフ |

## 仕様書フォーマット

各モジュール (`*.kt`) の仕様は以下の構成:

```markdown
## <ファイル名>

**パッケージ**: `com.theveloper.pixelplay.<dir>...`
**役割**: 1〜2 行で要約
**依存**: 上流 (呼ばれる) / 下流 (呼ぶ) への矢印

### クラス / オブジェクト / Enum
| 名前 | 種類 | 説明 |
|------|------|------|
| ... | class / object / enum / sealed / interface | 役割 |

### public API (主要メソッド)
| シグネチャ | 戻り値 | 目的 | 呼び出し元 |
|------------|--------|------|-----------|
| `fun foo(a: A): R` | `R` | 説明 | `Bar.kt:42` |

### 内部実装メモ
- 特筆すべきロジック・境界条件・設計判断

### 関連ファイル
- 依存 / 逆依存
```

## 読み方

1. まず `00-architecture.md` で全体像を掴む
2. 個別モジュール詳細は各セクションの `README.md` を読む
3. メソッド単位の詳細は各 `*.md` ファイルを参照
4. クラス間の相関は `99-correlation-diagrams.md`

## パス表記ルール

- ソースコードへの参照は **リポジトリルートからの相対パス** で表記する
  - 例: `app/src/main/java/com/theveloper/pixelplay/data/database/MusicDao.kt:42`
  - 例: `service/MusicService.kt`
- スペック内 (`specs/`) ファイル間の相互参照も同じく **相対パス**
  - 例: `../02-data-network/qqmusic.md` (← スペック内)
  - 例: `00-architecture.md` (← 同じスペックディレクトリ内)
- 絶対パス・フルパス表記は禁止
- クラス・メソッドの参照は Kotlin の完全修飾を避け、ファイルパス + シンボル名で参照
  - 例: `MusicService.kt` の `MusicService.playSong()`

## 編集方針

- 各エージェントが担当ディレクトリを解析して書き出す
- メソッドはシグネチャ + 1〜3 行の説明を必ず付ける
- private / 内部関数は必要に応じて省略可
- 既存コメント・命名から意味を抽出、推測は明示する
