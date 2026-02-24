# Bluesky 防災情報配信Bot 開発調査レポート

> **調査日**: 2026-02-24 (JST)
> **確実性**: 中〜高（公式ドキュメント・一次情報ベース）

---

## 【結論】

Bluesky防災情報配信Botの構築は**技術的に十分実現可能**です。AT Protocol APIは無料・登録不要で利用でき、投稿レート制限も防災Bot用途では余裕があります。ただし、緊急性の高い情報をリアルタイムに近い形で配信するには、**情報ソースの監視方式**と**セッション管理**の設計が最も重要なポイントとなります。

---

## 1. Bluesky API / AT Protocol の基本仕様

### 1.1 認証方式

| 項目 | 内容 |
|:---|:---|
| 認証方式 | App Password（推奨）またはOAuth |
| セッション管理 | `accessJwt`（短命）+ `refreshJwt`（長命、50日程度） |
| API Key | **不要**（X/Twitterと異なり開発者登録も不要） |
| SDK | TypeScript (`@atproto/api`), Python (`atproto`) が公式サポート |

**重要**: `createSession`（ログイン）のレート制限が厳しいため（IP単位で制限）、**セッションをファイルやDBに保存し、`refreshSession`で更新する設計が必須**です。毎回ログインし直すと、1日10回程度でロックアウトされるケースが報告されています。

### 1.2 投稿方法

```typescript
// TypeScript SDK
import { BskyAgent } from '@atproto/api';

const agent = new BskyAgent({ service: 'https://bsky.social' });

// セッション復元 or ログイン
const session = readSessionFromDisk();
if (session) {
  agent.resumeSession(session);
} else {
  await agent.login({
    identifier: process.env.BLUESKY_USERNAME!,
    password: process.env.BLUESKY_PASSWORD!,
  });
  writeSessionToDisk(agent.session);
}

// 投稿
await agent.post({ text: "【地震情報】..." });
```

```python
# Python SDK
from atproto import Client

client = Client()
client.login('handle.example.com', 'app-password-here')
client.send_post('【地震情報】...')
```

### 1.3 リッチテキスト・リンク埋め込み

Blueskyでは**URLの自動リンク化が行われない**ため、投稿内にリンクを含める場合は`RichText`で明示的にfacetを指定する必要があります。

```typescript
import { RichText } from '@atproto/api';

const rt = new RichText({
  text: '【気象警報】東京都に大雨警報 詳細: https://www.jma.go.jp/...'
});
await rt.detectFacets(agent);  // URLやメンションを自動検出

await agent.post({
  text: rt.text,
  facets: rt.facets,
});
```

---

## 2. レート制限（Rate Limits）

### 2.1 レコード操作制限（投稿・いいね・リポスト等）

| 制限項目 | 上限値 | ポイント消費 |
|:---|:---|:---|
| **1時間あたり** | 5,000ポイント | - |
| **1日あたり** | 35,000ポイント | - |
| 投稿(create) | 3ポイント/件 | **最大 1,666投稿/時, 11,666投稿/日** |
| いいね/リポスト等 | 1ポイント/件 | - |
| レコード削除 | 1ポイント/件 | - |

### 2.2 HTTP API制限

| 制限項目 | 上限値 | 備考 |
|:---|:---|:---|
| **グローバル制限（全ルート合計）** | 3,000リクエスト/5分 | IP単位 |
| `createSession`（ログイン） | 30回/5分, 300回/日 | **最も厳しい制限** |
| `createRecord`（投稿） | 100回/5分, 5,000回/日 | 十分余裕あり |

### 2.3 防災Bot運用における検証結果

| シナリオ | 想定投稿数 | 制限に対する余裕 |
|:---|:---|:---|
| 通常時（1日10〜30投稿） | 30件/日 | ✅ **余裕大**（上限の0.26%） |
| 大地震発生時（1時間に50投稿） | 50件/時 | ✅ **余裕あり**（上限の3%） |
| 台風接近時（6時間で200投稿） | 200件/6時間 | ✅ **余裕あり**（上限の3.4%） |
| 連続災害（1日300投稿） | 300件/日 | ✅ **余裕あり**（上限の2.6%） |

**結論**: 防災Bot用途では、レート制限が問題になることはほぼありません。ただし、投稿が「スパム的」と判定されるリスクがあるため、以下を推奨します。

- 投稿間隔を**最低10秒以上**空ける
- 同一内容の重複投稿を避ける（重複チェック機構を実装）
- Blueskyコミュニティガイドラインに準拠する
- Botアカウントのプロフィールに**Bot表記と情報ソースを明記**する

### 2.4 セルフホストPDSによるレート制限回避（上級者向け）

自前でPDS（Personal Data Server）をホストすると、`PDS_RATE_LIMITS_ENABLED=false` でレート制限を無効化できます。大量投稿が必要な場合の選択肢ですが、防災Bot程度の投稿量では不要です。

---

## 3. コンテンツ監視（情報ソースの更新検知）

### 3.1 情報ソース別の監視方式

#### A. 気象庁防災情報XML（最重要ソース）

| 方式 | 概要 | 遅延 | コスト |
|:---|:---|:---|:---|
| **PULL型（Atom Feed）** | 気象庁公式の高頻度フィードをポーリング | 約1〜2分 | 無料 |
| **DMDATA.JP（WebSocket）** | 気象業務支援センター経由のリアルタイム配信 | **数秒以内** | 有料（個人プラン有） |

**PULL型フィード一覧（気象庁公式）**:

| フィード名 | URL | 更新頻度 |
|:---|:---|:---|
| 高頻度・定時 | `https://www.data.jma.go.jp/developer/xml/feed/regular.xml` | 毎分 |
| 高頻度・随時 | `https://www.data.jma.go.jp/developer/xml/feed/extra.xml` | 毎分 |
| **高頻度・地震火山** | `https://www.data.jma.go.jp/developer/xml/feed/eqvol.xml` | **毎分（地震速報はこちら）** |
| 高頻度・その他 | `https://www.data.jma.go.jp/developer/xml/feed/other.xml` | 毎分 |
| 長期（定時） | `https://www.data.jma.go.jp/developer/xml/feed/regular_l.xml` | 1時間 |
| 長期（随時） | `https://www.data.jma.go.jp/developer/xml/feed/extra_l.xml` | 1時間 |
| 長期（地震火山） | `https://www.data.jma.go.jp/developer/xml/feed/eqvol_l.xml` | 1時間 |

**推奨アーキテクチャ（PULL型）**:

```
気象庁 Atom Feed (毎分更新)
    ↓ ポーリング（30秒〜1分間隔）
Bot Server (Node.js / Python)
    ↓ XMLパース → 新着チェック（既報IDと比較）
    ↓ テキスト整形
Bluesky API → 投稿
```

**DMDATA.JP（リアルタイム配信サービス）**:

緊急地震速報など秒単位の速報性が必要な場合は、DMDATA.JPの利用を推奨します。WebSocket経由でリアルタイム配信を受けられ、気象庁防災情報XMLフォーマットで提供されます。

| プラン | 月額 | 特徴 |
|:---|:---|:---|
| 個人プラン | 要確認 | 地震・津波情報を含む基本セット |
| 法人プラン | 要確認 | 全電文対応 |

#### B. ウェブサイト変更監視（RSS非対応サイト向け）

| ツール/方式 | 概要 | 適用場面 |
|:---|:---|:---|
| **changedetection.io** | セルフホスト型Webページ変更検知。Webhook連携可能 | RSS非対応の公的機関サイト |
| **RSS.app** | 任意のWebページからRSSフィードを生成するSaaS | SNSアカウントの投稿監視 |
| **自前スクレイピング** | Puppeteer/Playwrightで定期巡回 | カスタム要件がある場合 |

#### C. SNSアカウント監視

| プラットフォーム | 監視方法 |
|:---|:---|
| Bluesky | AT Protocol API (`getAuthorFeed`) でポーリング |
| Mastodon | ActivityPub / RSS フィード（`/users/{id}.rss`） |
| X (旧Twitter) | API有料化により困難（月額$100〜） |

### 3.2 推奨監視間隔

| 情報種別 | 推奨間隔 | 理由 |
|:---|:---|:---|
| 緊急地震速報 | **30秒以下**（WebSocket推奨） | 人命に関わる最速情報 |
| 地震情報・津波警報 | 30秒〜1分 | 気象庁フィードの更新頻度に合わせる |
| 気象警報・注意報 | 1〜3分 | 発表頻度が比較的低い |
| 台風情報 | 5〜10分 | 更新間隔が数時間単位 |
| ニュース速報 | 1〜5分 | RSS/Atom フィード監視 |

---

## 4. 推奨システムアーキテクチャ

### 4.1 構成図

```
┌─────────────────────────────────────────────────────┐
│                   情報ソース層                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐ │
│  │気象庁XML  │ │DMDATA.JP │ │ニュースRSS/SNS     │ │
│  │(PULL型)   │ │(WebSocket)│ │(Atom/RSS/API)      │ │
│  └─────┬────┘ └─────┬────┘ └──────────┬───────────┘ │
└────────┼────────────┼──────────────────┼─────────────┘
         ↓            ↓                  ↓
┌─────────────────────────────────────────────────────┐
│                   監視・処理層                        │
│  ┌──────────────────────────────────────────────┐   │
│  │           Bot Server (Node.js / Python)       │   │
│  │  ┌────────────┐ ┌──────────┐ ┌────────────┐  │   │
│  │  │ Poller/    │ │ Parser / │ │ Dedup /    │  │   │
│  │  │ Listener   │ │ Formatter│ │ Filter     │  │   │
│  │  └────────────┘ └──────────┘ └────────────┘  │   │
│  └──────────────────────────────────────────────┘   │
│  ┌──────────────┐ ┌──────────────┐                  │
│  │ Session Store│ │ Posted ID DB │                  │
│  │ (JWT保存)    │ │ (重複排除)    │                  │
│  └──────────────┘ └──────────────┘                  │
└────────────────────────┬────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│                   配信層                             │
│  ┌──────────────────────────────────────────────┐   │
│  │           Bluesky AT Protocol API             │   │
│  │  createRecord → app.bsky.feed.post            │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### 4.2 技術スタック候補

| レイヤー | 選択肢 | 備考 |
|:---|:---|:---|
| 言語 | **TypeScript (Node.js)** or **Python** | 公式SDK対応。TypeScriptはkazahanaとの親和性◎ |
| スケジューラ | node-cron / GitHub Actions / systemd timer | 定期ポーリング用 |
| WebSocket | ws (Node.js) / websockets (Python) | DMDATA.JP接続用 |
| データ保存 | SQLite / Redis / JSON file | セッション管理・重複排除ID保存 |
| ホスティング | VPS (低コスト) / AWS Lambda / Fly.io / Railway | 常時稼働が必要ならVPS推奨 |
| CI/CD | GitHub Actions | デプロイ・定期実行兼用可 |

### 4.3 重要な実装ポイント

1. **セッション永続化**: `accessJwt`をファイルまたはDBに保存し、期限切れ時のみ`refreshSession`を実行
2. **重複投稿防止**: 気象庁XMLのentry IDを保存し、処理済みIDをスキップ
3. **エラーハンドリング**: HTTP 429（レート制限）を受けた場合のExponential Backoff
4. **投稿フォーマット**: 300文字制限（Blueskyの文字数制限）を考慮した情報の要約
5. **リンクカード埋め込み**: External embedで気象庁ページへのリンクカードを表示

---

## 5. デプロイ先の比較

| プラットフォーム | 月額 | 常時稼働 | WebSocket | 適性 |
|:---|:---|:---|:---|:---|
| **VPS (ConoHa, さくら等)** | ¥500〜 | ✅ | ✅ | ◎ 最も安定 |
| **Fly.io** | 無料枠あり | ✅ | ✅ | ○ 小規模Bot向き |
| **Railway** | 無料枠あり | ✅ | ✅ | ○ |
| **GitHub Actions** | 無料 | ❌（定期実行のみ） | ❌ | △ ポーリング間隔5分が最短 |
| **AWS Lambda** | ほぼ無料 | ❌（イベント駆動） | △ | △ WebSocket不向き |
| **Google Apps Script** | 無料 | ❌（トリガー最短1分） | ❌ | △ 簡易版なら可 |

**推奨**: 緊急性の高い防災情報を扱うなら、**VPS + systemdサービス**または**Fly.io**での常時稼働が最も適しています。

---

## 6. Bluesky公式のBotガイドライン

Bluesky公式ドキュメントに記載されているBotに関するルールは以下のとおりです。

- 定期的に投稿する自動Botは**歓迎**されている
- 他ユーザーとインタラクション（いいね、リポスト、リプライ）する場合は、**ユーザーがBotをタグ付けした場合のみ**
- レート制限を尊重すること
- スパム的な振る舞いはコミュニティガイドライン違反
- Botアカウントであることを**プロフィールに明記**することを推奨

---

## 【注意点・例外】

- 気象庁防災情報XMLの**PUSH型配信は2020年9月に終了**しており、現在はPULL型のみ
- DMDATA.JPは有料サービスのため、コストとのトレードオフを検討する必要あり
- Blueskyの投稿文字数上限は**300文字**（grapheme単位）のため、警報情報の要約ロジックが必要
- レート制限値は**今後変更される可能性**がある（公式ドキュメントに明記）
- セルフホストPDSは運用コスト・複雑性が高いため、通常のBot開発では不要

---

## 【出典】

| 情報 | URL |
|:---|:---|
| Bluesky公式 Rate Limits | https://docs.bsky.app/docs/advanced-guides/rate-limits |
| Bluesky公式 Bot テンプレート | https://docs.bsky.app/docs/starter-templates/bots |
| Bluesky公式 投稿API解説 | https://docs.bsky.app/blog/create-post |
| Bluesky公式 APIドキュメント | https://docs.bsky.app/ |
| Bluesky Rate Limits ブログ | https://docs.bsky.app/blog/rate-limits-pds-v3 |
| 気象庁防災情報XMLフォーマット | https://xml.kishou.go.jp/ |
| 気象庁 PULL型フィード一覧 | https://www.data.jma.go.jp/developer/xml/feed/ |
| DMDATA.JP（リアルタイム配信） | https://dmdata.jp/ |
| changedetection.io | https://github.com/dgtlmoon/changedetection.io |
| GitHub: bsky-bot テンプレート | https://github.com/philnash/bsky-bot |
| AT Protocol GitHub Discussions (Rate Limit) | https://github.com/bluesky-social/atproto/discussions/697 |
| 気象庁XMLの扱い方（JX通信社） | https://tech.jxpress.net/entry/2024/07/29/090000 |
| 気象庁XML + GAS → Slack通知 実装例 | https://zenn.dev/medirom_tech/articles/19f7b720e808d0 |
