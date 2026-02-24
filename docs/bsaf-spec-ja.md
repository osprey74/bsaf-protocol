# Bluesky Structured Alert Feed (BSAF)

Bluesky上の構造化アラート・情報Botのためのオープン規約。クライアント側でのフィルタリングとパーソナライズを実現します。

---

## 課題

災害情報や気象警報、ニュース速報Botは、SNS上で根本的なジレンマに直面しています。**投稿が多すぎればタイムラインが埋まり、少なすぎれば誰かにとって重要な情報を取りこぼす**ということです。

ある国の全地域の注意報を投稿する気象Botは、大多数のユーザーにとってノイズになります。しかし、竜巻の進路上にいるユーザーにとっては、その1件の投稿がフィード内で最も重要な情報です。

現状の解決策はいずれも妥協を強いられます。

- **全国版Bot 1つ** → タイムラインが溢れ、大半の投稿がユーザーにとって無関係
- **地域別Bot** → エコシステムが断片化し、ユーザーは適切なアカウントを探す必要がある
- **プラットフォーム固有の機能** → 特定クライアントへのロックイン、他クライアントで利用不可

---

## 解決策

**BSAF（Bluesky Structured Alert Feed）** は、関心の分離によりこの問題を解決するオープン規約です。

| レイヤー | 責務 | 誰が作るか |
|:---------|:-----|:-----------|
| Bot | アラートを収集し、構造化メタデータ付きで投稿 | Bot開発者（国・情報源ごと） |
| 規約 | 機械可読なフィルタリングのための共通タグ形式 | 本仕様書 |
| クライアント | ユーザー設定に基づいて投稿をフィルタ・表示 | クライアント開発者 |

**Botは広く投稿し、クライアントがローカルでフィルタする。ユーザーは自分に関係のある情報だけを見ることができます。**

```
┌─────────────────────────────┐
│     Bluesky ネットワーク      │
│                             │
┌──────────┐  │  BSAFタグ付き投稿        │  ┌──────────────┐
│ 🇯🇵 気象庁Bot│────▶│                         │◀────│ 任意のBluesky │
│ 🇺🇸 NWS Bot│────▶│  検索可能・フィルタ可能な   │◀────│ クライアント    │
│ 🇪🇺 EU Bot │────▶│  機械可読メタデータ       │◀────│ (kazahana等)  │
│ 🌍 誰でも  │────▶│                         │◀────│              │
└──────────┘  └─────────────────────────────┘  └──────────────┘
                              │
                              ▼
                    ┌──────────────┐
                    │ ユーザーに    │
                    │ 表示されるのは │
                    │ 自分の地域・   │
                    │ 設定に合った   │
                    │ 情報のみ      │
                    └──────────────┘
```

---

## 設計思想

BSAFは**汎用Botパースシステム**であり、防災情報に限定されません。防災Bot、スポーツ速報Bot、ニュースBot、交通情報Botなど、構造化されたフィルタリング可能なメタデータが有益なあらゆるドメインで採用できます。

コアタグは意図的に汎用的な名前（`type`、`value`、`target`）を使用しており、プロトコルの変更なしにあらゆるコンテンツドメインで機能します。

---

## 仕組み

### 1. Botが構造化tags付きで投稿する

AT Protocolの `app.bsky.feed.post` レコードは `tags` フィールドをサポートしています。最大8タグ・合計640バイトの文字列配列で、投稿UIには表示されませんが、機械的に読み取ることができます。

BSAFはこのフィールドを活用して構造化メタデータを埋め込みます。

```json
{
  "$type": "app.bsky.feed.post",
  "text": "🔴 地震情報\n15日11時52分ころ、茨城県南部で地震\nマグニチュード：5.2（深さ約50km）\n最大震度：5強\n津波の心配なし\n（出典：気象庁）",
  "tags": [
    "bsaf:v1",
    "type:earthquake",
    "value:震度5強",
    "time:2026-02-15T02:52:00Z",
    "target:jp-kanto",
    "source:jma"
  ],
  "langs": ["ja"],
  "createdAt": "2026-02-15T02:55:54Z"
}
```

投稿テキストは人間が読める形のまま。タグが機械フィルタリングを可能にします。

### 2. ユーザーがBot定義JSONでBotを登録する

Bot開発者は**Bot定義JSON**ファイルを公開します。このファイルにはBotの情報と利用可能なフィルタオプションが定義されています。ユーザーはこのJSONをBSAF対応クライアントにインポートしてBotを登録します。

```
Bot開発者                    ユーザー                    BSAFクライアント
    │                          │                            │
    │ JSONファイルを公開         │                            │
    │（Bluesky/GitHub等で）     │                            │
    │                          │                            │
    │                          │  JSONをインポート            │
    │                          │─────────────────────────▶  │
    │                          │                            │
    │                          │                     JSONをパース
    │                          │                     Botアカウントを自動フォロー
    │                          │                     フィルタ設定UIを動的生成
    │                          │                            │
    │                          │  ◀──── フィルタ設定画面を表示
    │                          │  好みを選択                 │
    │                          │                            │
    │  BSAFタグ付き投稿 ──────────────────────────────────▶  │
    │                          │                     タグベースでフィルタリング
    │                          │  ◀──── 条件に合う投稿のみ表示
```

### 3. クライアントがユーザー設定に基づきフィルタリング

BSAF対応クライアントは `tags` フィールドを読み取り、Bot毎に設定されたフィルタルールを適用します。

```
例：JMA Botに対するユーザーのフィルタ設定
- type: 地震 ✅、津波 ✅、噴火 ✅、降灰 ❌
- value: 震度3 ✅、震度4 ✅、震度5弱 ✅、...（有効）
         震度1 ❌、震度2 ❌（無効）
- target: 関東 ✅、北海道 ✅（有効）
        近畿 ❌、九州 ❌（無効）

受信した投稿のタグ：
  type:earthquake, value:震度2, target:jp-kanto
  結果：非表示（value「震度2」がユーザーの有効値に含まれない）

受信した投稿のタグ：
  type:earthquake, value:震度5強, target:jp-kanto
  結果：表示（type、value、targetすべてがユーザーの有効フィルタに一致）
```

### 4. クライアントが複数Botからの重複投稿を処理する

ユーザーが同一データソースをカバーする複数のBSAF Botを購読している場合、重複投稿が発生します。クライアントは重複を検知し、折りたたんで表示すべき（SHOULD）です。

**重複判定：** 異なるBotアカウントからの投稿で `type` + `value` + `time` + `target` が一致した場合、重複と判定します。

**表示方法：** 最初に受信した投稿を表示し、後続の重複投稿は「他N件のBotも報告」のようなインジケータで折りたたみます。これにより冗長性を維持しつつ（1つのBotがダウンしても他のBotから情報を受信可能）、タイムラインをクリーンに保ちます。

---

## タグ仕様（v1）

### プロトコルタグ

すべてのBSAF対応投稿には以下を**必ず**含めてください。

| タグ | 説明 |
|:----|:-----|
| `bsaf:v1` | この投稿がBSAF対応であることの識別子（バージョン1） |

### コアタグ

すべてのコアタグは、BSAF投稿で**必須**です。

| プレフィックス | 説明 | 値の例 |
|:--------------|:-----|:-------|
| `type:` | 情報の種別 | `earthquake`, `tsunami`, `baseball`, `soccer` |
| `value:` | 規模・程度・重み付け | `震度5強`, `warning`, `final`, `highlight` |
| `time:` | イベント発生時刻（ISO 8601 UTC） | `2026-02-15T02:52:00Z` |
| `target:` | 対象・受け手 | `jp-kanto`, `us-california`, `npb-giants` |
| `source:` | 発表元・データソース | `jma`, `nws`, `espn` |

### タグの使用量制限

AT Protocolでは最大8タグ、合計640バイトまで許可されています。

| スロット | タグ | 必須 |
|:--------|:-----|:-----|
| 1 | `bsaf:v1` | ✅ 必須 |
| 2 | `type:{種別}` | ✅ 必須 |
| 3 | `value:{規模}` | ✅ 必須 |
| 4 | `time:{ISO8601}` | ✅ 必須 |
| 5 | `target:{対象}` | ✅ 必須 |
| 6 | `source:{情報源}` | ✅ 必須 |
| 7 | *（予備）* | 任意 |
| 8 | *（予備）* | 任意 |

必須6枠 + 将来拡張用の予備2枠。

### タグ値のガイドライン

#### `type:` の値

Bot開発者は自身のドメインに適した `type` 値を定義します。

| ドメイン | type値の例 |
|:--------|:-----------|
| 防災（日本） | `earthquake`, `tsunami`, `eruption`, `ashfall`, `weather-warning`, `special-warning`, `landslide-warning`, `tornado-warning`, `heavy-rain`, `nankai-trough` |
| 防災（米国） | `earthquake`, `tsunami`, `tornado`, `hurricane`, `flood`, `wildfire`, `winter-storm` |
| スポーツ | `baseball`, `soccer`, `basketball`, `hockey` |
| ニュース | `breaking`, `politics`, `business`, `technology` |

#### `value:` の値

`value` は情報の**重み付け・規模**を表します。Bot開発者はドメインに適した離散値を定義します。これらはクライアントUIでフィルタオプションとして使用されます。

| ドメイン | value値の例 |
|:--------|:------------|
| 日本の地震 | `震度1`, `震度2`, `震度3`, `震度4`, `震度5弱`, `震度5強`, `震度6弱`, `震度6強`, `震度7` |
| 日本の気象 | `info`, `advisory`, `warning`, `severe-warning`, `special-warning` |
| 米国の気象（NWS） | `advisory`, `watch`, `warning`, `extreme` |
| スポーツ | `pre-game`, `in-progress`, `final`, `highlight`, `breaking` |

#### `time:` の値

ISO 8601形式（UTC）。**元のソースイベントのタイムスタンプ**を使用しなければなりません（Botの投稿時刻ではありません）。

```
time:2026-02-15T02:52:00Z
```

#### `target:` の値

target値は各Botが**Bot定義JSON**で定義する厳格な命名規約に従わなければなりません。地理的な地域には小文字の国コード（ISO 3166-1 alpha-2）をプレフィックスとし、非地理的な対象にはドメイン固有のプレフィックスを使用します。

| ドメイン | target値の例 |
|:--------|:-----------|
| 日本（地方別） | `jp-hokkaido`, `jp-tohoku`, `jp-kanto`, `jp-hokuriku`, `jp-chubu`, `jp-kinki`, `jp-chugoku`, `jp-shikoku`, `jp-kyushu`, `jp-okinawa` |
| 米国（州別） | `us-california`, `us-texas`, `us-new-york` |
| EU（国別） | `eu-germany`, `eu-france`, `eu-spain` |
| スポーツ（チーム別） | `npb-giants`, `npb-tigers`, `epl-arsenal`, `nfl-chiefs` |

**重要：** target値は厳密に正規化されなければなりません。Bot間でtarget値が一貫しなければ、プロトコルとしてのフィルタリングが無意味になります。Bot開発者はBot定義JSONで使用するtarget値の完全なセットを定義し、クライアントはその定義に基づいてフィルタUIを構築します。

### 同一ソースBot間の相互運用性要件

既存のBSAF Botと**同一の情報ソース**を扱う新しいBSAF Botを開発する場合、新しいBotは既存Botが公開しているBot定義JSONの `value` および `target` の値を尊重し、同一の値を使用しなければなりません（**MUST**）。この要件は、クライアント側の重複検知がBot間で正しく機能することを保証するために存在します。

例えば、既存のJMA地震Botが `value:震度5強` および `target:jp-kanto` を使用している場合、同じくJMA地震データを処理する新しいBotは、同一の `value:震度5強` および `target:jp-kanto` を使用しなければなりません。`value:5+`、`value:shindo-5-upper`、`target:jp-kantou`、`target:jp-関東` などの代替値を使用してはいけません。

新しいBotが同一ソースのイベントに対して異なる値を使用した場合、クライアントは重複を検知できず、ユーザーのタイムラインに冗長な投稿が表示され、BSAFプロトコルの核心的な利点が損なわれます。

#### `source:` の値

発表元・データソースの識別子。

| source値 | 発表元 |
|:---------|:-------|
| `jma` | 気象庁（Japan Meteorological Agency） |
| `nws` | National Weather Service（米国） |
| `meteoalarm` | MeteoAlarm（EU） |
| `bom` | Bureau of Meteorology（オーストラリア） |
| `espn` | ESPN |
| `ap` | Associated Press |

---

## Bot定義JSON

### 概要

すべてのBSAF Botは**Bot定義JSONファイル**を公開しなければなりません。このファイルは以下の唯一の情報源（Single Source of Truth）として機能します：

- Botの識別情報とデータソース情報
- クライアントUI生成のためのフィルタオプション定義
- バージョン管理のためのセルフ更新URL

ユーザーはこのJSONをBSAF対応クライアントにインポートしてBotを登録します。クライアントはJSONをパースし、Botアカウントを自動フォローし、フィルタ設定UIを動的に生成します。

### スキーマ

```json
{
  "bsaf_schema": "1.0",
  "updated_at": "2026-02-24T00:00:00Z",
  "self_url": "https://example.com/bsaf-jma-bot.json",

  "bot": {
    "handle": "jma-alert.bsky.social",
    "did": "did:plc:xxxxx",
    "name": "防災情報Bot（非公式）",
    "description": "気象庁の公開データに基づく防災情報",
    "source": "気象庁 防災情報XML",
    "source_url": "https://www.data.jma.go.jp/developer/xml/feed/"
  },

  "filters": [
    {
      "tag": "type",
      "label": "情報種別",
      "options": [
        { "value": "earthquake", "label": "地震" },
        { "value": "tsunami", "label": "津波" },
        { "value": "eruption", "label": "噴火" },
        { "value": "ashfall", "label": "降灰" },
        { "value": "weather-warning", "label": "気象警報" },
        { "value": "special-warning", "label": "特別警報" },
        { "value": "landslide-warning", "label": "土砂災害" },
        { "value": "tornado-warning", "label": "竜巻注意" },
        { "value": "heavy-rain", "label": "記録的大雨" },
        { "value": "nankai-trough", "label": "南海トラフ臨時" }
      ]
    },
    {
      "tag": "value",
      "label": "重み付け",
      "options": [
        { "value": "震度1", "label": "震度1" },
        { "value": "震度2", "label": "震度2" },
        { "value": "震度3", "label": "震度3" },
        { "value": "震度4", "label": "震度4" },
        { "value": "震度5弱", "label": "震度5弱" },
        { "value": "震度5強", "label": "震度5強" },
        { "value": "震度6弱", "label": "震度6弱" },
        { "value": "震度6強", "label": "震度6強" },
        { "value": "震度7", "label": "震度7" },
        { "value": "info", "label": "情報" },
        { "value": "advisory", "label": "注意報" },
        { "value": "warning", "label": "警報" },
        { "value": "severe-warning", "label": "重大警報" },
        { "value": "special-warning", "label": "特別警報" }
      ]
    },
    {
      "tag": "target",
      "label": "地域",
      "options": [
        { "value": "jp-hokkaido", "label": "北海道" },
        { "value": "jp-tohoku", "label": "東北" },
        { "value": "jp-kanto", "label": "関東" },
        { "value": "jp-hokuriku", "label": "北陸" },
        { "value": "jp-chubu", "label": "中部" },
        { "value": "jp-kinki", "label": "近畿" },
        { "value": "jp-chugoku", "label": "中国" },
        { "value": "jp-shikoku", "label": "四国" },
        { "value": "jp-kyushu", "label": "九州" },
        { "value": "jp-okinawa", "label": "沖縄" }
      ]
    }
  ]
}
```

### フィールド定義

| フィールド | 必須 | 説明 |
|:----------|:-----|:-----|
| `bsaf_schema` | ✅ | スキーマバージョン（現在は `"1.0"`） |
| `updated_at` | ✅ | 最終更新タイムスタンプ（ISO 8601） |
| `self_url` | ✅ | このJSONがホストされているURL。クライアントが定期的に取得して更新を確認 |
| `bot.handle` | ✅ | BotのBlueskyハンドル |
| `bot.did` | ✅ | BotのDID |
| `bot.name` | ✅ | 人間が読めるBot名 |
| `bot.description` | ✅ | Botが提供する情報の簡潔な説明 |
| `bot.source` | ✅ | 主要データソース名 |
| `bot.source_url` | 推奨 | 主要データソースのURL |
| `filters` | ✅ | クライアントUI生成のためのフィルタ定義の配列 |
| `filters[].tag` | ✅ | このフィルタが対応するBSAFタグ（`type`, `value`, `target`） |
| `filters[].label` | ✅ | フィルタグループの表示ラベル |
| `filters[].options` | ✅ | 選択可能なオプションの配列 |
| `filters[].options[].value` | ✅ | タグ値（実際の投稿で使用するタグ値と一致すること） |
| `filters[].options[].label` | ✅ | このオプションの表示ラベル |

### 更新メカニズム

クライアントは定期的に `self_url` を取得し、保存済みバージョンと `updated_at` を比較すべき（SHOULD）です。新しいバージョンが見つかった場合、フィルタUIを更新し、新しいオプションについてユーザーに通知します。

推奨チェック間隔：1日1回。

---

## 投稿フォーマット要件

### ソース帰属表示（必須）

すべてのBSAF投稿は、投稿テキスト本文に**主要データソース名を含めなければなりません**。これにより、BSAF非対応クライアントのユーザーを含むすべての閲覧者にソース帰属が表示されます。

形式：投稿テキストの末尾に `（出典：{ソース名}）`

```
🔴 地震情報
15日11時52分ころ、茨城県南部で地震
マグニチュード：5.2（深さ約50km）
最大震度：5強
津波の心配なし
（出典：気象庁）
```

これは**二重保証**です。`source:` タグが機械可読な帰属を提供し、投稿テキストが人間可読な帰属を提供します。

### 言語

投稿はAT Protocolレコードの `langs` フィールドで言語を明示しなければなりません。

```json
{
  "langs": ["ja"]
}
```

---

## Bot開発者向け

### BSAF対応Botの構築

BSAF Botの役割は4つです：

1. **収集** — 公式データソースを監視する（例：気象庁XMLフィード、NWS CAPアラート）
2. **整形** — ソース帰属表示付きの人間可読テキスト + BSAFタグを作成する
3. **投稿** — AT Protocol API経由でBlueskyに公開する
4. **公開** — クライアント登録のためのBot定義JSONファイルを維持する

### 最小実装例（TypeScript）

```typescript
import { BskyAgent, RichText } from "@atproto/api";

const agent = new BskyAgent({ service: "https://bsky.social" });
await agent.login({
  identifier: "your-bot.bsky.social",
  password: "app-password",
});

// 投稿を整形（テキスト内のソース帰属表示は必須）
const text =
  "🔴 地震情報\n" +
  "15日11時52分ころ、茨城県南部で地震\n" +
  "マグニチュード：5.2（深さ約50km）\n" +
  "最大震度：5強\n" +
  "津波の心配なし\n" +
  "（出典：気象庁）";

const rt = new RichText({ text });
await rt.detectFacets(agent);

// BSAFタグ付きで投稿（6つのコアタグすべて必須）
await agent.post({
  text: rt.text,
  facets: rt.facets,
  tags: [
    "bsaf:v1",
    "type:earthquake",
    "value:震度5強",
    "time:2026-02-15T02:52:00Z",
    "target:jp-kanto",
    "source:jma",
  ],
  langs: ["ja"],
});
```

### Bot運用者向けガイドライン

- **プロフィール：** Blueskyのプロフィール設定で「Bot」ラベルを設定し、Botであることを明示する
- **自己紹介：** データソースとカバー範囲を記載する
- **投稿頻度：** 適切な間隔を保つ（投稿間隔10秒以上を推奨）
- **重複排除：** 投稿済みアラートIDを記録し、重複投稿を防止する
- **ソース帰属：** タグと投稿テキストの両方にデータソース名を必ず含める
- **言語：** `langs` フィールドを設定する。データソースの主言語で投稿する
- **Bot定義JSON：** 安定したURLでJSONファイルを公開・維持する

### 各国の利用可能なデータソース

コントリビューション歓迎です。公開されているアラートデータソースをご存知の方は、PRをお送りください。

| 国 | ソース | 形式 | アクセス | 備考 |
|:---|:-------|:-----|:---------|:-----|
| 🇯🇵 日本 | 気象庁 XMLフィード | Atom/XML | 無料・登録不要 | 毎分更新 |
| 🇺🇸 アメリカ | NWS CAPアラート | CAP/Atom | 無料・登録不要 | リアルタイム |
| 🇪🇺 EU | MeteoAlarm | CAP/RSS | 無料 | 欧州30カ国以上 |
| 🇦🇺 オーストラリア | BOM Warnings | RSS/XML | 無料 | |
| 🇹🇼 台湾 | CWA Open Data | JSON/XML | 無料・要登録 | |
| 🇰🇷 韓国 | KMA Data Portal | JSON/XML | 無料・要登録 | |
| 🇨🇦 カナダ | ECCC Alerts | CAP/Atom | 無料 | |
| 🇳🇿 ニュージーランド | MetService Warnings | RSS | 無料 | |
| 🌍 グローバル | GDACS | RSS/CAP | 無料 | 大規模災害のみ |

---

## クライアント開発者向け

### BSAFサポートの実装

あらゆるBlueskyクライアントがBSAFに対応できます。基本的な処理フローは以下の通りです。

#### 1. Bot登録

ユーザーがBot定義JSONファイルをインポートします。クライアントは：
- JSONをパースしスキーマを検証する
- Botの Blueskyアカウント（`bot.handle`）を自動フォローする
- `filters` 定義からフィルタ設定UIを動的に生成する
- 定期的な更新チェック用に `self_url` を保存する

#### 2. フィルタ設定UI

クライアントは `filters` 配列に基づいて、登録された各Botの設定画面を動的に生成します。すべてのフィルタは**マルチセレクトオプショングループ**として描画されます：

```
┌─ 防災情報Bot（非公式） ─────────────────────────┐
│                                                  │
│ 📌 情報種別（type）                              │
│ [✅ 地震] [✅ 津波] [✅ 噴火] [□ 降灰]         │
│ [✅ 特別警報] [□ 気象警報] [□ 竜巻注意]        │
│                                                  │
│ 📌 重み付け（value）                             │
│ [□ 震度1] [□ 震度2] [✅ 震度3] [✅ 震度4]      │
│ [✅ 震度5弱] [✅ 震度5強] [✅ 震度6弱]          │
│ [✅ 震度6強] [✅ 震度7]                          │
│ [□ 情報] [□ 注意報] [✅ 警報]                  │
│ [✅ 重大警報] [✅ 特別警報]                      │
│                                                  │
│ 📌 地域（target）                                  │
│ [✅ 北海道] [□ 東北] [✅ 関東] [□ 北陸]        │
│ [□ 中部] [□ 近畿] [□ 中国] [□ 四国]          │
│ [□ 九州] [□ 沖縄]                              │
└──────────────────────────────────────────────────┘
```

すべてのフィルタが同じ入力タイプ（マルチセレクト）を使用し、クライアントの実装をシンプルかつ一貫性のあるものに保ちます。

#### 3. フィルタリングロジック

登録されたBotからの受信投稿ごとに：

```python
def should_display(post, bot_config):
    # BSAF投稿でなければ通常表示
    if "bsaf:v1" not in post.tags:
        return True

    tags = parse_bsaf_tags(post.tags)

    # 各フィルタをチェック：投稿のタグ値がユーザーの有効オプションに含まれること
    for filter in bot_config.filters:
        tag_value = tags.get(filter.tag)
        if tag_value and tag_value not in filter.enabled_options:
            return False

    return True
```

#### 4. 重複検知と折りたたみ

複数の登録Botが同一イベントを投稿した場合：

```python
def is_duplicate(post_a, post_b):
    tags_a = parse_bsaf_tags(post_a.tags)
    tags_b = parse_bsaf_tags(post_b.tags)

    return (
        tags_a.type == tags_b.type and
        tags_a.value == tags_b.value and
        tags_a.time == tags_b.time and
        tags_a.target == tags_b.target and
        post_a.author != post_b.author  # 異なるBot
    )
```

**表示方法：**
- 最初に受信した投稿を表示
- 重複はインジケータで折りたたみ：「他N件のBotも報告」
- ユーザーは展開してすべてのバージョンを確認可能

これにより**冗長性が機能として維持**されます — 1つのBotがオフラインになっても、同じソースをカバーする他のBotからアラートを受信できます。

#### 5. JSON更新チェック

クライアントは各Botの `self_url` を定期的に取得し、`updated_at` を比較します：

```python
def check_for_updates(bot_registration):
    response = fetch(bot_registration.self_url)
    remote_json = parse(response)

    if remote_json.updated_at > bot_registration.updated_at:
        # 保存済みJSONを更新
        # 新しい/削除されたオプションでフィルタUIを更新
        # ユーザーに変更を通知
```

---

## FAQ

### Bluesky組み込みのFeed Generatorを使わないのはなぜ？

Feed Generatorはサーバーサイドで動作し、パーソナライズされたフィードごとにホスティングインフラが必要です。BSAFのフィルタリングは完全にクライアントサイドで実行されるため、ユーザーあたりのサーバーコストはゼロです。Bot開発者はBot自体のホスティングだけで済みます。

### `tags` フィールドではなくハッシュタグを使わないのはなぜ？

ハッシュタグは投稿テキストを煩雑にし、表記揺れが発生しやすくなります。`tags` フィールドはユーザーには見えませんが機械可読であり、投稿をクリーンに保ちながら精密なフィルタリングを実現します。

### 防災以外のコンテンツにもBSAFを使えますか？

もちろんです。BSAFは**汎用Botパースシステム**であり、完全にドメイン非依存で設計されています。ニュースBot、スポーツ速報Bot、交通情報Botなど、構造化されたフィルタリング可能なメタデータが有益なあらゆるものがBSAFを採用できます。Bot定義JSONで適切な `type:` と `value:` のオプションを定義するだけです。

### 8タグでは足りない場合は？

現行仕様は利用可能な8スロット中6スロットを使用し、2スロットを将来の拡張用に予約しています。追加のメタデータが必要な場合は、BSAF v2 への拡張を提案してください。

### 複数Bot間での重複検知はどう動作しますか？

クライアントは異なるBotアカウントからの投稿について `type` + `value` + `time` + `target` を比較して重複を検知します。`time` タグは**元のソースイベントのタイムスタンプ**（Botの投稿時刻ではなく）を使用するため、どのBotが処理しても同一イベントは同一の `time` 値を生成します。

### CAP（Common Alerting Protocol）との違いは？

CAPは各国機関が使用する緊急アラートの包括的なXML標準です。BSAFはBluesky上のSNS投稿のための軽量な規約です。典型的なBSAF Botは各国ソースから CAPデータを*取り込み*、BSAFタグ付きのBluesky投稿を*出力*します。両者は競合するものではなく、補完的な関係です。

---

## ロードマップ

### Phase 1：基盤構築（現在）

- [ ] BSAF仕様 v1 の公開
- [ ] リファレンスBot公開：日本の地震・津波アラート
- [ ] kazahana でのBSAFフィルタリング実装（リファレンスクライアント）
- [ ] リファレンスBotのBot定義JSONを公開

### Phase 2：エコシステムの成長

- [ ] コミュニティによるBot開発（米国NWS、EU MeteoAlarm等）
- [ ] Bot定義JSONディレクトリ / レジストリ
- [ ] Bot開発者向けBSAFタグバリデーションツール
- [ ] 他のBlueskyクライアントでの採用

### Phase 3：発展

- [ ] BSAF v2 仕様策定（コミュニティのフィードバックに基づく）
- [ ] メディア添付への対応（アラートマップ、レーダー画像）
- [ ] 構造化アラートデータによるリッチ埋め込みカード
- [ ] AT Protocolのラベリングシステムとの統合

---

## コントリビューション

あらゆる種類の貢献を歓迎します。

- 🤖 **Botを作る** — あなたの国の災害アラートやあらゆる情報ドメインで
- 📋 **Bot定義JSONを公開する** — あなたのBot用
- 🐛 **問題を報告する** — 仕様に関する課題
- 📝 **ドキュメントを改善する** — 翻訳を含む
- 💡 **機能を提案する** — BSAF v2 に向けて

[CONTRIBUTING.md](../CONTRIBUTING.md) のガイドラインをご参照ください。

---

## リファレンス実装

- **Bot:** [bsaf-jma-bot](https://github.com/osprey74/bsaf-jma-bot) — 日本の地震・津波・気象アラート（近日公開）
- **クライアント:** [kazahana](https://github.com/osprey74/kazahana) — BSAF対応 Blueskyデスクトップクライアント（近日公開）

---

## ライセンス

本仕様書は [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) の下で公開されています。帰属表示を行えば、自由に採用・改変・再配布が可能です。

リファレンス実装は [MITライセンス](../LICENSE) の下で公開されています。

---

**BSAFはオープンなコミュニティの取り組みです。安全のために。すべての人のために。**

[フィードバック](https://github.com/osprey74/bsaf-protocol/issues) · [ディスカッション](https://github.com/osprey74/bsaf-protocol/discussions) · [コントリビュート](../CONTRIBUTING.md)
