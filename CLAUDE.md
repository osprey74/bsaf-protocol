# CLAUDE.md — BSAF プロジェクト概要

## プロジェクト概要

**BSAF（Bluesky Structured Alert Feed）** は、Bluesky上の災害情報・アラートBotのためのオープン規約（コンベンション）です。  
Bot側が構造化メタデータ（AT Protocolの `tags` フィールド）付きで投稿し、クライアント側でユーザー設定に基づきフィルタリングする、というハイブリッドアーキテクチャを採用しています。

### 背景

- **kazahana**（https://github.com/osprey74/kazahana）は11言語対応のBlueskyデスクトップクライアント（Tauri v2 + React + TypeScript）
- 当初は日本向け防災機能として構想されたが、国際化の課題（各国のデータソースが異なる）を踏まえ、**汎用的な構造化フィードフィルタ規約**に昇華した
- kazahanaはBSAFのリファレンスクライアントとして、タグベースのフィルタリング機能を実装予定

### 核となる設計思想

```
関心の分離:
  Bot     = データ収集・投稿（国・情報源ごとに独立）
  規約    = 共通タグ形式（本リポジトリ）
  クライアント = ユーザー設定に基づくフィルタリング（kazahana等）
```

BSAFは災害情報に限定しない。ニュース、スポーツ、交通情報など、構造化フィルタリングが有益なあらゆるBotに適用可能な**コンテンツ非依存**の設計。

---

## リポジトリ構成

```
bsaf/
├── CLAUDE.md              ← 本ファイル（Claude Code向けプロジェクト概要）
├── README.md              ← BSAF仕様書（英語版、リポジトリのメインドキュメント）
├── LICENSE                ← CC BY 4.0（仕様書）+ MIT（コード）のデュアルライセンス
├── docs/
│   └── README-ja.md       ← BSAF仕様書（日本語版）
├── presets/
│   └── jp/                ← 日本向けフィルタプリセット（JSON）
├── examples/              ← Bot・クライアント実装のサンプルコード
├── src/                   ← バリデーションツール等の共通ユーティリティ
└── .github/               ← Issue/PRテンプレート等
```

---

## BSAF タグ仕様 v1（要約）

### 必須タグ

| タグ | 説明 | 例 |
|:-----|:-----|:---|
| `bsaf:v1` | プロトコル識別子（全投稿に必須） | — |
| `type:` | アラート種別 | `earthquake`, `tsunami`, `tornado`, `weather-warning` |
| `severity:` | 深刻度 | `info`, `minor`, `moderate`, `severe`, `critical`, 数値震度(`3`, `5+`) |
| `country:` | ISO 3166-1 alpha-2 国コード | `JP`, `US`, `DE` |

### 推奨タグ

| タグ | 説明 | 例 |
|:-----|:-----|:---|
| `region:` | 国内地域 | `kanto`, `midwest`, `bavaria` |
| `subregion:` | 具体的地域 | `ibaraki`, `oklahoma` |

### オプションタグ

| タグ | 説明 | 例 |
|:-----|:-----|:---|
| `source:` | 発表元機関 | `jma`, `nws`, `meteoalarm` |
| `scope:` | 地理的範囲 | `local`, `regional`, `national` |

### 制約

- AT Protocolの制限: **最大8タグ、合計640バイト**
- タグは `app.bsky.feed.post` の `tags` フィールドに格納（UIに表示されない機械可読メタデータ）

### 投稿例

```json
{
  "$type": "app.bsky.feed.post",
  "text": "🔴 地震情報：茨城県南部 M5.2\n最大震度：5強\n津波の心配なし",
  "tags": [
    "bsaf:v1",
    "type:earthquake",
    "severity:5+",
    "country:JP",
    "region:kanto",
    "subregion:ibaraki",
    "source:jma"
  ],
  "createdAt": "2026-02-15T02:55:54Z"
}
```

---

## 深刻度マッピング

| BSAFレベル | 日本（気象庁） | 米国（NWS） | EU（MeteoAlarm） |
|:-----------|:---------------|:------------|:------------------|
| `info` | — | Advisory | 緑 |
| `minor` | 震度1-2 / 注意報 | Watch | 黄 |
| `moderate` | 震度3-4 / 警報 | Warning | 橙 |
| `severe` | 震度5弱-5強 | Severe | 赤 |
| `critical` | 震度6弱以上 / 特別警報 / 大津波警報 | Extreme | 紫 |

地震専用: `severity:3`, `severity:5+`, `severity:5-`, `severity:6+`, `severity:7` など数値表現も有効。

---

## 関連リポジトリ・リソース

| リソース | URL | 説明 |
|:---------|:----|:-----|
| kazahana | https://github.com/osprey74/kazahana | リファレンスクライアント（Tauri v2 + React + TS） |
| bsaf-jma-bot | https://github.com/osprey74/bsaf-jma-bot | 日本向けリファレンスBot（予定） |
| 気象庁 XML PULL型 | https://xml.kishou.go.jp/xmlpull.html | 日本のアラートデータソース |
| AT Protocol tags | https://docs.bsky.app/docs/advanced-guides/posts#tags | Bluesky投稿のtagsフィールド仕様 |

---

## JMA（気象庁）データソース情報

BSAFリファレンスBot（bsaf-jma-bot）のデータソースとなる気象庁フィード情報。

### フィード構成

| フィード | 内容 | 更新間隔 |
|:---------|:-----|:---------|
| `eqvol.xml` | 地震・火山 | 毎分（高頻度）|
| `extra.xml` | 気象警報・随時情報 | 毎分（高頻度）|
| `regular.xml` | 定時予報 | 毎時（長期）|
| `other.xml` | その他（海上警報等）| 毎時（長期）|

### 技術的制約

- 推奨ポーリング間隔: **30〜60秒**
- CDNキャッシュ: 新データ反映まで毎分約20秒のラグ
- ダウンロード上限: **10GB/日**（超過でIPブロック）
- HTTPS必須（2022年よりHSTS）
- 形式: RFC 4287 Atom XML（エントリ内のlink hrefに詳細XMLのURL）

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

## 開発フェーズ

### Phase 1: 基盤構築（現在）
- [x] BSAF仕様 v1 公開
- [ ] リファレンスBot: bsaf-jma-bot（日本の地震・津波・気象）
- [ ] kazahanaでのBSAFフィルタリング実装
- [ ] 日本向けプリセット公開

### Phase 2: エコシステム成長
- [ ] コミュニティBot（米国NWS、EU MeteoAlarm等）
- [ ] プリセット共有ディレクトリ
- [ ] タグバリデーションツール

### Phase 3: 発展
- [ ] BSAF v2 仕様
- [ ] メディア添付対応（アラートマップ、レーダー画像）
- [ ] AT Protocolラベリングシステム統合

---

## ライセンス

- **仕様書**（README.md, docs/）: CC BY 4.0
- **コード**（src/, examples/）: MIT

---

## コーディング規約・注意事項

- 言語: 仕様書は英語がプライマリ、日本語訳を docs/ に配置
- タグ値は**小文字英数字とハイフン**のみ使用（例: `weather-warning`）。ただし `country:` の値はISO 3166-1 alpha-2のため大文字（例: `JP`, `US`）
- プリセットJSON: 命名規約 `{country}-{category}-{region}.json`（例: `jp-disaster-hokkaido.json`）
- 投稿テキストは人間可読を最優先、タグは機械可読のメタデータとして分離
- AT Protocol APIを使用する実装では `@atproto/api` パッケージを使用
