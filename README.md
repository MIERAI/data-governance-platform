# Data Governance Platform

企業データ資産の全体像を可視化し、分級管理・アクセス制御・ライフサイクル管理を統合的に提供するデータガバナンスプラットフォーム。

## Background

AI活用が企業運営に深く浸透するにつれ、「データをどう管理し、誰がどの範囲でアクセスでき、どう保護するか」が経営レベルの課題になっている。

現状の課題：

- **データの全体像が見えない**: どのデータがどこにあり、誰が管理し、誰が使っているかを一元的に把握する手段がない
- **権限管理が属人的**: データアクセスの許可が口頭ベースで行われ、監査証跡がない
- **データの価値が定量化されていない**: どのデータが重要で、どこに投資すべきかの判断基準がない
- **コンプライアンス対応がリアクティブ**: 個人情報保護法等の要件に対して、事後対応になりがち

## Goals

1. 企業内の全データ資産を一元管理する **Data Catalog** を構築する
2. データの **分級体系**（公開/内部/機密/極秘）と、各レベルに応じたアクセス制御を定義・運用する
3. データの **血縁関係（Lineage）** を追跡し、影響範囲分析を可能にする
4. データ資産の **価値評価フレームワーク** を構築し、投資判断に使えるROI指標を提供する

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                   Data Governance Platform                       │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │  Governance   │  │    Data      │  │    Access Control     │  │
│  │   Console     │  │   Catalog    │  │    & Audit            │  │
│  │  (Web UI)     │  │              │  │                       │  │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬───────────┘  │
│         │                 │                       │              │
│         ▼                 ▼                       ▼              │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Core Services                            │ │
│  │                                                             │ │
│  │  ┌───────────┐ ┌───────────┐ ┌──────────┐ ┌─────────────┐ │ │
│  │  │ Metadata  │ │  Lineage  │ │  Policy  │ │    Asset    │ │ │
│  │  │  Service  │ │  Service  │ │  Engine  │ │  Valuation  │ │ │
│  │  └─────┬─────┘ └─────┬─────┘ └────┬─────┘ └──────┬──────┘ │ │
│  └────────┼─────────────┼────────────┼───────────────┼────────┘ │
│           │             │            │               │           │
│  ┌────────▼─────────────▼────────────▼───────────────▼────────┐ │
│  │                   Metadata Store                            │ │
│  │              (PostgreSQL / Graph DB)                        │ │
│  └────────────────────────┬───────────────────────────────────┘ │
└───────────────────────────┼─────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
        ┌─────▼────┐ ┌─────▼─────┐ ┌─────▼──────┐
        │   cdc-   │ │   mcp-    │ │  External  │
        │lakehouse │ │   data-   │ │  Systems   │
        │          │ │  gateway  │ │            │
        └──────────┘ └───────────┘ └────────────┘
```

### Components

| Component | Role | Description |
|:---|:---|:---|
| Governance Console | 管理UI | データ資産の検索・閲覧・分級設定・ポリシー管理 |
| Data Catalog | 資産目録 | 全データソースのメタデータ（スキーマ・オーナー・説明・品質スコア）を集中管理 |
| Metadata Service | メタデータ収集 | 各データソースからスキーマ・統計情報を自動収集 |
| Lineage Service | 血縁追跡 | データの流れ（source → transform → destination）を記録・可視化 |
| Policy Engine | ポリシー実行 | データ分級に基づくアクセス制御ルールの定義・自動適用 |
| Asset Valuation | 価値評価 | 使用頻度・ビジネス影響度・代替コストに基づくデータ資産の定量評価 |
| Access Control & Audit | 権限・監査 | 誰が・いつ・何にアクセスしたかの記録と監査レポート |

### Data Lineage Example

```
PostgreSQL (orders)
       │
       ▼ CDC
cdc-lakehouse (Bronze: raw_orders)
       │
       ▼ Spark ETL
cdc-lakehouse (Silver: silver_orders)
       │
       ├──▶ cdc-lakehouse (Gold: daily_summary)
       │
       └──▶ mcp-data-gateway (API: /api/v1/orders)
                   │
                   └──▶ AI Agent (MCP経由)
```

この血縁関係が可視化されることで、「ordersテーブルのスキーマを変更したら何が壊れるか」が事前に把握できる。

## Governance Framework

### Data Classification

| Level | Label | Example | Access Rule |
|:---|:---|:---|:---|
| L1 | 公開 | 商品カテゴリマスタ | 全社アクセス可 |
| L2 | 内部 | 売上日次サマリ | 部門内 + 申請ベース |
| L3 | 機密 | 顧客個人情報 | 権限者のみ + 暗号化必須 |
| L4 | 極秘 | 経営戦略データ | 名指し許可 + 監査ログ必須 |

### Data Lifecycle

```
作成(Create) → 登録(Register) → 利用(Use) → 保守(Maintain) → 棚卸(Review) → 廃棄(Retire)
    │              │              │             │               │              │
    ▼              ▼              ▼             ▼               ▼              ▼
 スキーマ定義   Catalog登録   アクセスログ   品質モニタリング   価値再評価   安全な削除
 オーナー指定   分級設定      権限チェック   鮮度チェック      利用状況確認   証跡保存
```

## Relationship to Other Projects

```
Phase 1: cdc-lakehouse             → データの生産（CDC + Lakehouse）
Phase 2: mcp-data-gateway          → データの消費（Contract + API + MCP）
Phase 3: data-governance-platform  → データの管理（治理 + 資産化）  ← This
```

本プロジェクトは Phase 1・Phase 2 の上位レイヤーとして、全データ資産を横断的に管理する。

- `cdc-lakehouse` のメタデータ（テーブル定義・品質指標・処理履歴）を Catalog に取り込む
- `mcp-data-gateway` のアクセスログ・Data Contract をポリシーエンジンと連携させる
- 両プロジェクトの Lineage 情報を統合し、エンドツーエンドの血縁関係を可視化する

## Planned Milestones

| Milestone | Description | Status |
|:---|:---|:---|
| M1 | Data Catalog設計 + 既存データソースのメタデータ自動収集 | Not Started |
| M2 | データ分級体系の定義 + Policy Engine基盤 | Not Started |
| M3 | Lineage追跡（cdc-lakehouse → mcp-data-gateway の流れ） | Not Started |
| M4 | Governance Console (Web UI) + 監査レポート | Not Started |
| M5 | 資産価値評価フレームワーク + ROI分析ダッシュボード | Not Started |

## Tech Stack (候補)

- **Catalog Engine**: Apache Atlas / Amundsen / DataHub / Custom
- **Metadata Store**: PostgreSQL + Neo4j (Lineage Graph)
- **Policy Engine**: Open Policy Agent (OPA)
- **Web UI**: React / Next.js
- **Auth**: OAuth 2.0 + RBAC
- **Monitoring**: Prometheus + Grafana
- **CI/CD**: GitHub Actions

## License

See [LICENSE](./LICENSE).
