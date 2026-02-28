# CLAUDE.md — BSAF プロジェクト概要

> **Last updated:** 2026-02-24
> **Status:** 仕様 v1 確定、実装着手前

---

## プロジェクト概要

**BSAF（Bluesky Structured Alert Feed）** は、Bluesky上の構造化情報Botのためのオープン規約（コンベンション）です。

Bot側が構造化メタデータ（AT Protocolの `tags` フィールド）付きで投稿し、クライアント側でユーザー設定に基づきフィルタリングする、というハイブリッドアーキテクチャを採用しています。

## 背景

- **kazahana**（https://github.com/osprey74/kazahana）は11言語対応のBlueskyデスクトップクライアント（Tauri v2 + React + TypeScript）
- 当初は日本向け防災機能として構想されたが、国際化の課題（各国のデータソースが異なる）を踏まえ、汎用的な構造化フィードフィルタ規約に昇華した
- kazahanaはBSAFのリファレンスクライアントとして、タグベースのフィルタリング機能を実装予定

## 核となる設計思想

1. **関心の分離:**
   - Bot = データ収集・投稿（国・情報源ごとに独立）
   - 規約 = 共通タグ形式（本リポジトリ）
   - クライアント = ユーザー設定に基づくフィルタリング（kazahana等）

2. **コンテンツ非依存の設計:**
   - BSAFは災害情報に限定しない **universal bot parsing system**
   - ニュース、スポーツ、交通情報など、構造化フィルタリングが有益なあらゆるBotに適用可能
   - コアタグ名は意図的に汎用名（`type`, `value`, `target`）を採用し、特定ドメインに依存しない

---

## リポジトリ構成

```
bsaf-protocol/
├── CLAUDE.md          ← 本ファイル（Claude Code向けプロジェクト概要）
├── README.md          ← リポジトリのメインドキュメント
├── LICENSE            ← CC BY 4.0（仕様書）+ MIT（コード）のデュアルライセンス
├── docs/
│   ├── bsaf-spec.md       ← BSAF仕様書（英語版、正式版）
│   ├── bsaf-spec-ja.md    ← BSAF仕様書（日本語版）
│   ├── bsaf-jma-bot-spec.md ← JMA Bot実装仕様
│   └── ...
├── presets/
│   └── jp/            ← 日本向けフィルタプリセット（JSON）
├── examples/          ← Bot・クライアント実装のサンプルコード
├── src/               ← バリデーションツール等の共通ユーティリティ
└── .github/           ← Issue/PRテンプレート等
```

---

## BSAF タグ仕様 v1（確定版）

### コアタグ構成（6必須 + 2予約）

| スロット | タグ | 説明 | 例 | 必須 |
|:--------|:-----|:-----|:---|:-----|
| 1 | `bsaf:v1` | プロトコル識別子 | — | ✅ |
| 2 | `type:{種別}` | 情報カテゴリ | `type:earthquake` | ✅ |
| 3 | `value:{規模}` | 重み・規模・程度 | `value:5+` | ✅ |
| 4 | `time:{ISO8601}` | ソースイベント発生時刻（UTC） | `time:2026-02-15T02:52:00Z` | ✅ |
| 5 | `target:{対象}` | 対象・受け手（誰に向けた情報か） | `target:jp-kanto` | ✅ |
| 6 | `source:{情報源}` | 発表元機関・データソース | `source:jma` | ✅ |
| 7 | （予約1） | 将来の拡張用 | — | 任意 |
| 8 | （予約2） | 将来の拡張用 | — | 任意 |

### 制約

- AT Protocolの制限: **最大8タグ、合計640バイト**
- タグは `app.bsky.feed.post` の `tags` フィールドに格納（UIに表示されない機械可読メタデータ）
- タグ値は小文字英数字とハイフンのみ使用（例: `weather-warning`, `5+`, `6-`）

### ⚠️ 旧タグ体系からの変更点（2026-02-24 改訂）

以下の旧タグは **廃止** されました。コード中に残っている場合は新タグに置き換えてください：

| 旧タグ | → 新タグ | 変更理由 |
|:------|:--------|:---------|
| `severity:` | `value:` | ドメイン非依存化。災害の「深刻度」だけでなく、スポーツの「試合状況」等にも適用可能 |
| `country:` | `target:` に統合 | 地理的制限を排除 |
| `region:` | `target:` に統合 | `target:jp-kanto` のようにプレフィックス形式で統合 |
| `subregion:` | 廃止 | `target:` に統合 |
| `scope:` | 廃止 | 不要と判断 |
| （なし） | `time:` 新設 | 重複検知のキーとして必須。ソースイベントの発生時刻 |
| `source:`（任意） | `source:`（必須） | 出典の保証のため必須に昇格 |

### タグ名の設計原則

タグ名は **完全にドメイン非依存** であることが求められる：

- `type` = 何の情報か（完全に汎用）
- `value` = どの程度か（完全に汎用）
- `target` = 誰に向けた情報か（完全に汎用）

`area` ではなく `target` を採用した理由：
- `area` は地理的制限を暗示し、スポーツ（`target:npb-giants`）やニュース（`target:us-politics`）で不自然
- `target` は「この情報の対象は誰か」を汎用的に表現できる

### 投稿例

```json
{
  "$type": "app.bsky.feed.post",
  "text": "🔴 地震情報\n2月15日 11:52 — 茨城県南部\nM5.2（深さ約50km）\n最大震度：5強\n津波の心配なし\n(情報源: 気象庁)",
  "tags": [
    "bsaf:v1",
    "type:earthquake",
    "value:5+",
    "time:2026-02-15T02:52:00Z",
    "target:jp-kanto",
    "source:jma"
  ],
  "langs": ["ja"],
  "createdAt": "2026-02-15T02:55:54Z"
}
```

### 非防災ドメインでの使用例

```json
// スポーツBot
"tags": ["bsaf:v1", "type:baseball", "value:final", "time:2026-03-01T12:00:00Z", "target:npb-giants", "source:espn"]

// ニュースBot
"tags": ["bsaf:v1", "type:breaking", "value:high", "time:2026-03-01T15:30:00Z", "target:us-politics", "source:ap"]
```

---

## Bot定義JSON

### 概要

全BSAFボットは **Bot定義JSONファイル** を公開しなければならない（MUST）。このファイルは以下の役割を果たす：

- Bot識別・データソース情報の提供
- クライアントUIの動的生成に使用するフィルタオプション定義
- `self_url` によるバージョン管理・自動更新

### スキーマ概要

```json
{
  "bsaf_schema": "1.0",
  "updated_at": "2026-02-24T00:00:00Z",
  "self_url": "https://example.com/bsaf-jma-bot.json",
  "bot": {
    "handle": "jma-alert.bsky.social",
    "did": "did:plc:xxxxx",
    "name": "Japan Disaster Alerts (Unofficial)",
    "description": "Disaster alerts based on JMA public data",
    "source": "Japan Meteorological Agency — Disaster Prevention XML",
    "source_url": "https://www.data.jma.go.jp/developer/xml/feed/"
  },
  "filters": [
    {
      "tag": "type",
      "label": "Alert Type",
      "options": [
        { "value": "earthquake", "label": "Earthquake" },
        { "value": "tsunami", "label": "Tsunami" }
      ]
    },
    {
      "tag": "value",
      "label": "Weight",
      "options": [
        { "value": "3", "label": "Seismic 3" },
        { "value": "5+", "label": "Seismic 5 Upper" }
      ]
    },
    {
      "tag": "target",
      "label": "Region",
      "options": [
        { "value": "jp-hokkaido", "label": "Hokkaido" },
        { "value": "jp-kanto", "label": "Kanto" }
      ]
    }
  ]
}
```

### 更新メカニズム

クライアントは `self_url` を定期的（推奨: 1日1回）にフェッチし、`updated_at` を比較して更新を検知する。

---

## 重要な規約

### ソース出典の二重保証（MUST）

全BSAF投稿は以下の **両方** を含まなければならない：
1. `source:` タグ — 機械可読な出典情報
2. 投稿本文中の `(情報源: {名称})` または `(Source: {name})` — 人間可読な出典情報

非BSAFクライアントのユーザーにも出典が見えることを保証するため。

### 同一ソースBot間の相互運用性要件（MUST）

既存のBSAF Botと **同一の情報ソース** を扱う新しいBotを開発する場合：
- 既存Botが公開しているBot定義JSONの `value` および `target` の値を **そのまま使用しなければならない**
- `value:shindo-5-upper` vs `value:5+`、`target:jp-kantou` vs `target:jp-kanto` のような代替値は不可
- この規約はクライアント側の重複検知が正しく機能するために不可欠

### 重複検知ロジック

クライアント側で `type` + `value` + `time` + `target` が一致する異なるBotからの投稿を重複と判定する。
- `time` タグはBotの投稿時刻ではなく **ソースイベントの発生時刻** を使用するため、同一イベントは同一 `time` 値になる
- 重複投稿は折りたたみ表示（「他N件のBotも報告」）し、冗長性は維持する

---

## JMA（気象庁）データソース情報

bsaf-jma-bot のデータソースとなる気象庁フィード情報。

### フィード構成

| フィード | 内容 | 更新間隔 |
|:---------|:-----|:---------|
| `eqvol.xml` | 地震・火山 | 毎分（高頻度） |
| `extra.xml` | 気象警報・随時情報 | 毎分（高頻度） |
| `regular.xml` | 定時予報 | 毎時（長期） |
| `other.xml` | その他（海上警報等） | 毎時（長期） |

### 技術的制約

- 推奨ポーリング間隔: 30〜60秒
- CDNキャッシュ: 新データ反映まで毎分約20秒のラグ
- ダウンロード上限: 10GB/日（超過でIPブロック）
- HTTPS必須（2022年よりHSTS）
- 形式: RFC 4287 Atom XML（エントリ内の `link href` に詳細XMLのURL）

### ⚠️ 2026年度 電文体系変更

- FY2026（出水期）に新電文が追加予定
- 新電文: 集約通報、気象防災速報
- 移行期間: 大半は2年間（〜2028年）
- 即時廃止: 全般台風情報（移行期間なし）

---

## ホスティング想定（Bot側）

リファレンスBotは **Fly.io** の東京リージョン（nrt）でホスティングを想定。

| 構成 | 月額概算 |
|:-----|:---------|
| 最小: shared-cpu-1x / 256MB + 1GB Volume + 1GB通信 | 約$2.21（約¥330） |
| 推奨: shared-cpu-1x / 512MB + 専用IPv4 + 3GB Volume + 5GB通信 | 約$5.97（約¥900） |

24/7稼働、秒単位課金、Docker対応、WebSocket対応（DMDATA.JP連携に有用）。

---

## 関連リポジトリ・リソース

| リソース | URL | 説明 |
|:---------|:----|:-----|
| kazahana | https://github.com/osprey74/kazahana | リファレンスクライアント（Tauri v2 + React + TS） |
| bsaf-jma-bot | https://github.com/osprey74/bsaf-jma-bot | 日本向けリファレンスBot（予定） |
| 気象庁 XML PULL型 | https://xml.kishou.go.jp/xmlpull.html | 日本のアラートデータソース |
| AT Protocol tags | https://docs.bsky.app/docs/advanced-guides/posts#tags | Bluesky投稿のtagsフィールド仕様 |

---

## 未対応の既知タスク

- [x] `docs/bsaf-jma-bot-spec.md` の旧タグ名を新タグ名に更新する
- [ ] `examples/` ディレクトリ内のサンプルコードを新タグ体系に更新する
- [ ] `presets/jp/` のプリセットJSONを新タグ体系に更新する
- [x] `README.md` をbsaf-spec.mdの内容と整合させる

---

## コーディング規約・注意事項

- 言語: 仕様書は英語がプライマリ、日本語訳を `docs/` に配置
- タグ値は小文字英数字とハイフンのみ使用（例: `weather-warning`, `5+`, `6-`）
  - `target:` は `{country_code}-{region}` 形式（例: `jp-kanto`, `us-california`）
  - 非地理的な `target` はドメインプレフィックス形式（例: `npb-giants`, `epl-arsenal`）
- Bot定義JSON: `filters[].options[].value` は実際の投稿タグ値と **完全一致** しなければならない
- AT Protocol APIを使用する実装では `@atproto/api` パッケージを使用
- 投稿テキストは人間可読を最優先、タグは機械可読のメタデータとして分離

---

## ライセンス

- 仕様書（docs/）: **CC BY 4.0**
- コード（src/, examples/）: **MIT**

---

## Task Management

- **task_file**: none (Issues で管理)
- **done_marker**: N/A

## Documentation

- **primary_docs** (一次資料・正式な規定書類):
  - `docs/bsaf-spec.md` (EN)
  - `docs/bsaf-spec-ja.md` (JA)
- **secondary_docs** (二次資料・抜粋版 — 一次資料に準拠すること):
  - `README.md`
  - `docs/README-ja.md`
- **doc_pairs**:
  - `docs/bsaf-spec.md` ↔ `docs/bsaf-spec-ja.md`
  - `README.md` ↔ `docs/README-ja.md`
- **doc_hierarchy_note**: 不整合が見つかった場合は `docs/bsaf-spec.md` / `docs/bsaf-spec-ja.md` を正とする

## Versioning

- **version_files**: none
- **cargo_lockfile**: false

## CI/CD

- **cicd**: false

## SNS

- **sns_accounts**: none
