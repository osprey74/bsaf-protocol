# Bluesky Structured Alert Feed (BSAF)

**Bluesky上の構造化アラート・情報Botのためのオープン規約。クライアント側でのフィルタリングとパーソナライズを実現します。**

---

## 課題

災害情報や気象警報、ニュース速報Botは、SNS上で根本的なジレンマに直面しています。**投稿が多すぎればタイムラインが埋まり、少なすぎれば誰かにとって重要な情報を取りこぼす**ということです。

ある国の全地域の注意報を投稿する気象Botは、大多数のユーザーにとってノイズになります。しかし、竜巻の進路上にいるユーザーにとっては、その1件の投稿がフィード内で最も重要な情報です。

現状の解決策はいずれも妥協を強いられます。

- **全国版Bot 1つ** → タイムラインが溢れ、大半の投稿がユーザーにとって無関係
- **地域別Bot** → エコシステムが断片化し、ユーザーは適切なアカウントを探す必要がある
- **プラットフォーム固有の機能** → 特定クライアントへのロックイン、他クライアントで利用不可

## 解決策

**BSAF（Bluesky Structured Alert Feed）** は、関心の分離によりこの問題を解決するオープン規約です。

| レイヤー | 責務 | 誰が作るか |
|:---------|:-----|:-----------|
| **Bot** | アラートを収集し、構造化メタデータ付きで投稿 | Bot開発者（国・情報源ごと） |
| **規約** | 機械可読なフィルタリングのための共通タグ形式 | 本仕様書 |
| **クライアント** | ユーザー設定に基づいて投稿をフィルタ・表示 | クライアント開発者 |

Botは広く投稿し、クライアントがローカルでフィルタする。**ユーザーは自分に関係のある情報だけを見ることができます。**

```
                    ┌─────────────────────────────┐
                    │     Bluesky ネットワーク       │
                    │                              │
  ┌──────────┐     │   BSAFタグ付き投稿             │     ┌──────────────┐
  │ 🇯🇵 気象庁Bot│────▶│                              │◀────│ 任意のBluesky │
  │ 🇺🇸 NWS Bot│────▶│   検索可能・フィルタ可能な     │◀────│   クライアント  │
  │ 🇪🇺 EU Bot │────▶│   機械可読メタデータ          │◀────│ (kazahana等)  │
  │ 🌍 誰でも  │────▶│                              │◀────│              │
  └──────────┘     └─────────────────────────────┘     └──────────────┘
                                                              │
                                                              ▼
                                                      ┌──────────────┐
                                                      │ ユーザーに    │
                                                      │ 表示される    │
                                                      │ のは自分の    │
                                                      │ 地域・設定に  │
                                                      │ 合った情報のみ│
                                                      └──────────────┘
```

## 仕組み

### 1. Botが構造化 `tags` 付きで投稿する

AT Protocolの `app.bsky.feed.post` レコードは [`tags` フィールド](https://docs.bsky.app/docs/advanced-guides/posts#tags) をサポートしています。最大8タグ・合計640バイトの文字列配列で、**投稿UIには表示されません**が、機械的に読み取ることができます。

BSAFはこのフィールドを活用して構造化メタデータを埋め込みます。

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
    "subregion:ibaraki"
  ],
  "createdAt": "2026-02-15T02:55:54Z"
}
```

投稿テキストは人間が読める形のまま。タグが機械フィルタリングを可能にします。

### 2. クライアントがユーザー設定に基づきフィルタリング

BSAF対応クライアントは `tags` フィールドを読み取り、ユーザーが定義したルールを適用します。

```
ユーザーのフィルタ設定：
  - country = JP
  - region = hokkaido
  - 地震の表示条件: severity >= 3
  - 常時表示: 津波、噴火、severity >= 5

受信した投稿のタグ：
  type:earthquake, severity:2, country:JP, region:kanto

結果: 非表示（地域不一致、震度が閾値未満）
```

```
受信した投稿のタグ：
  type:earthquake, severity:5+, country:JP, region:kanto

結果: 表示（震度5以上は地域フィルタを無視して常時表示）
```

### 3. ユーザーはプリセットまたはカスタムルールで設定

利便性のため、クライアントはプリセット（事前構成済みフィルタ設定）を提供できます。

```json
{
  "preset_name": "日本 防災情報（北海道）",
  "version": "1.0",
  "accounts": [
    "did:plc:xxxxx"
  ],
  "filters": {
    "country": ["JP"],
    "region": ["hokkaido"]
  },
  "type_settings": {
    "earthquake": { "min_severity": "3" },
    "tsunami": { "always_show": true },
    "weather-warning": { "min_severity": "warning" },
    "eruption": { "always_show": true },
    "special-warning": { "always_show": true }
  },
  "global_overrides": [
    { "field": "severity", "operator": "in", "value": ["critical", "5+", "6+", "6-", "7"], "action": "always_show" }
  ]
}
```

プリセットはエクスポート/インポート可能なJSONファイルです。コミュニティが各国・地域版を公開し、共有できます。

---

## タグ仕様（v1）

### プロトコルタグ

すべてのBSAF対応投稿には以下を**必ず含めてください**。

| タグ | 説明 |
|:-----|:-----|
| `bsaf:v1` | この投稿がBSAF対応であることの識別子（バージョン1） |

### コアタグ

| プレフィックス | 必須 | 説明 | 値の例 |
|:---------------|:-----|:-----|:-------|
| `type:` | **必須** | アラートの種別 | `earthquake`, `tsunami`, `weather-warning`, `weather-watch`, `eruption`, `tornado`, `hurricane`, `flood`, `wildfire`, `special-warning` |
| `severity:` | **必須** | 深刻度 | 下記「深刻度スケール」参照 |
| `country:` | **必須** | ISO 3166-1 alpha-2 国コード | `JP`, `US`, `DE`, `FR`, `TW` |
| `region:` | 推奨 | 国内の地域 | `kanto`, `midwest`, `bavaria` |
| `subregion:` | 任意 | 具体的な地域 | `ibaraki`, `oklahoma`, `munich` |

### 深刻度スケール

深刻度の値は、各国の異なるシステムに対応するため**意図的に柔軟**に設計されています。以下にBSAFの推奨マッピングを示します。

| BSAFレベル | 意味 | 日本（気象庁） | 米国（NWS） | EU（MeteoAlarm） |
|:-----------|:-----|:---------------|:------------|:------------------|
| `info` | 情報提供 | — | Advisory | 緑 |
| `minor` | 軽微 | 震度1-2 / 注意報 | Watch | 黄 |
| `moderate` | 中程度 | 震度3-4 / 警報 | Warning | 橙 |
| `severe` | 重大 | 震度5弱-5強 | Severe | 赤 |
| `critical` | 危機的・生命に関わる | 震度6弱以上 / 特別警報 / 大津波警報 | Extreme | 紫 / 暗赤 |

**地震専用**の深刻度として、数値による震度・メルカリ震度階級も有効です。

```
severity:3        → 気象庁震度3
severity:5+       → 気象庁震度5強
severity:5-       → 気象庁震度5弱
severity:mmi-VII  → 改正メルカリ震度VII
```

### オプションタグ

| プレフィックス | 説明 | 値の例 |
|:---------------|:-----|:-------|
| `source:` | 発表元機関 | `jma`, `nws`, `dwd`, `meteoalarm`, `bom` |
| `scope:` | 地理的範囲 | `local`, `regional`, `national` |
| `expires:` | 有効期限（ISO 8601） | `2026-02-15T12:00:00Z` |

### タグの使用量制限

AT Protocolでは最大**8タグ**、合計**640バイト**まで許可されています。以下の優先順位で割り当ててください。

1. `bsaf:v1`（必須）
2. `type:`（必須）
3. `severity:`（必須）
4. `country:`（必須）
5. `region:`（推奨）
6. `subregion:`（余裕があれば）
7. `source:`（余裕があれば）
8. 追加のコンテキスト（余裕があれば）

---

## Bot開発者向け

### BSAF対応Botの構築

BSAF Botの役割は3つです。

1. **収集** — 公式データソースを監視する（例：気象庁XMLフィード、NWS CAPアラート、MeteoAlarm API）
2. **整形** — 人間が読める投稿テキスト + BSAFタグを作成する
3. **投稿** — AT Protocol API経由でBlueskyに公開する

#### 最小実装例（TypeScript）

```typescript
import { BskyAgent, RichText } from "@atproto/api";

const agent = new BskyAgent({ service: "https://bsky.social" });
await agent.login({ identifier: "your-bot.bsky.social", password: "app-password" });

// 投稿を整形
const text = "🔴 地震情報：茨城県南部 M5.2\n最大震度：5強\n津波の心配なし";
const rt = new RichText({ text });
await rt.detectFacets(agent);

// BSAFタグ付きで投稿
await agent.post({
  text: rt.text,
  facets: rt.facets,
  tags: [
    "bsaf:v1",
    "type:earthquake",
    "severity:5+",
    "country:JP",
    "region:kanto",
    "subregion:ibaraki",
    "source:jma",
  ],
});
```

#### Bot運用者向けガイドライン

- **プロフィール**: Blueskyのプロフィール設定で「Bot」ラベルを設定し、Botであることを明示する
- **自己紹介**: データソースとカバー範囲を記載する（例：「気象庁のデータに基づく日本の地震・津波アラート」）
- **投稿頻度**: 適切な間隔を保つ（投稿間隔10秒以上を推奨）
- **重複排除**: 投稿済みアラートIDを記録し、重複投稿を防止する
- **出典の明記**: 必ず元のデータソースをクレジットする
- **言語**: データソースの主言語で投稿する。重大アラートについては多言語投稿も検討する

### 各国の利用可能なデータソース

コントリビューション歓迎です。公開されているアラートデータソースをご存知の方は、PRをお送りください。

| 国 | ソース | 形式 | アクセス | 備考 |
|:---|:-------|:-----|:---------|:-----|
| 🇯🇵 日本 | [気象庁 XMLフィード](https://xml.kishou.go.jp/xmlpull.html) | Atom/XML | 無料・登録不要 | 毎分更新 |
| 🇺🇸 アメリカ | [NWS CAPアラート](https://alerts.weather.gov/) | CAP/Atom | 無料・登録不要 | リアルタイム |
| 🇪🇺 EU | [MeteoAlarm](https://www.meteoalarm.org/) | CAP/RSS | 無料 | 欧州30カ国以上 |
| 🇦🇺 オーストラリア | [BOM Warnings](http://www.bom.gov.au/catalogue/warnings/) | RSS/XML | 無料 | |
| 🇹🇼 台湾 | [CWA Open Data](https://opendata.cwa.gov.tw/) | JSON/XML | 無料・要登録 | |
| 🇰🇷 韓国 | [KMA Data Portal](https://data.kma.go.kr/) | JSON/XML | 無料・要登録 | |
| 🇨🇦 カナダ | [ECCC Alerts](https://weather.gc.ca/warnings/) | CAP/Atom | 無料 | |
| 🇳🇿 ニュージーランド | [MetService Warnings](https://www.metservice.com/) | RSS | 無料 | |
| 🌍 グローバル | [GDACS](https://www.gdacs.org/) | RSS/CAP | 無料 | 大規模災害のみ |

---

## クライアント開発者向け

### BSAFフィルタリングの実装

あらゆるBlueskyクライアントがBSAFに対応できます。基本的な処理フローは以下の通りです。

```
1. ユーザーがBSAFフィードタブを設定:
   - 監視するBotアカウントを指定
   - フィルタルールを設定（国、地域、深刻度の閾値）
   - またはコミュニティのプリセットをインポート

2. クライアントが指定アカウントの投稿を取得:
   - Blueskyリストフィード経由（推奨）
   - 著者フィードAPI直接呼び出し
   - フォロータイムライン経由（最もシンプル）

3. 各投稿について "bsaf:v1" タグを確認:
   - タグなし → 通常表示（BSAF投稿ではない）
   - タグあり → タグに対してフィルタルールを適用

4. 条件に合致した投稿を専用タブ/ビューに表示
```

#### フィルタリングロジック（疑似コード）

```python
def should_display(post, user_config):
    tags = parse_bsaf_tags(post.tags)
    
    # BSAF投稿でなければ通常表示
    if "bsaf:v1" not in post.tags:
        return True
    
    # グローバルオーバーライドを最初にチェック（重大アラートは常に表示）
    for override in user_config.global_overrides:
        if matches(tags, override):
            return True
    
    # 国フィルタをチェック
    if tags.country not in user_config.countries:
        return False
    
    # 地域フィルタをチェック（設定されている場合）
    if user_config.regions and tags.region not in user_config.regions:
        # 深刻度が地域オーバーライド閾値を超えていれば表示
        if not exceeds_threshold(tags.severity, user_config.regional_override):
            return False
    
    # 種別ごとの設定をチェック
    type_config = user_config.type_settings.get(tags.type)
    if type_config:
        if type_config.always_show:
            return True
        if not meets_severity(tags.severity, type_config.min_severity):
            return False
    
    return True
```

### プリセットの配布

プリセットは、フィルタ設定をまとめたJSONファイルで、簡単に共有できます。

**推奨する配布チャネル：**

- GitHubリポジトリ（本リポジトリの `/presets` ディレクトリ）
- クライアント設定画面でのURL直接インポート
- 他のユーザーからのQRコード / 共有リンク

プリセットファイルは以下の命名規約に従ってください。

```
presets/
  jp-disaster-all.json
  jp-disaster-hokkaido.json
  jp-disaster-kanto.json
  us-weather-national.json
  us-weather-california.json
  eu-meteoalarm-germany.json
  ...
```

---

## ロードマップ

### Phase 1：基盤構築（現在）
- [x] BSAF仕様 v1 の公開
- [ ] リファレンスBot公開：日本の地震・津波アラート
- [ ] [kazahana](https://github.com/osprey74/kazahana) でのBSAFフィルタリング実装（リファレンスクライアント）
- [ ] 日本向けプリセットファイルの公開

### Phase 2：エコシステムの成長
- [ ] コミュニティによるBot開発（米国NWS、EU MeteoAlarm等）
- [ ] プリセット共有プラットフォーム / ディレクトリ
- [ ] Bot開発者向けBSAFタグバリデーションツール
- [ ] 他のBlueskyクライアントでの採用

### Phase 3：発展
- [ ] BSAF v2 仕様策定（コミュニティのフィードバックに基づく）
- [ ] メディア添付への対応（アラートマップ、レーダー画像）
- [ ] 構造化アラートデータによるリッチ埋め込みカード
- [ ] AT Protocolのラベリングシステムとの統合

---

## FAQ

### Bluesky組み込みのFeed Generatorを使わないのはなぜ？

Feed Generatorはサーバーサイドで動作し、パーソナライズされたフィードごとにホスティングインフラが必要です。BSAFのフィルタリングは完全にクライアントサイドで実行されるため、ユーザーあたりのサーバーコストはゼロです。Bot開発者はBot自体のホスティングだけで済みます。

### `tags` フィールドではなくハッシュタグを使わないのはなぜ？

ハッシュタグは投稿テキストを煩雑にし、表記揺れが発生しやすくなります。`tags` フィールドはユーザーには見えませんが機械可読であり、投稿をクリーンに保ちながら精密なフィルタリングを実現します。

### 防災以外のコンテンツにもBSAFを使えますか？

もちろんです。BSAFはコンテンツに依存しない汎用的な設計です。ニュースBot、スポーツ速報Bot、交通情報Botなど、構造化されたフィルタリング可能なメタデータが有益なあらゆるものがBSAFを採用できます。対象ドメインに適した `type:` 値を定義するだけです。

### 8タグでは足りない場合は？

コアタグ（`bsaf:v1`、`type`、`severity`、`country`、`region`）を優先してください。常により多くのメタデータが必要な場合は、複合値のエンコーディング（例：`geo:JP-kanto-ibaraki` を1つのタグとする）を検討するか、BSAF v2 への拡張を提案してください。

### CAP（Common Alerting Protocol）との違いは？

[CAP](https://ja.wikipedia.org/wiki/Common_Alerting_Protocol) は各国機関が使用する緊急アラートの包括的なXML標準です。BSAFはBluesky上のSNS投稿のための軽量な規約です。典型的なBSAF Botは各国ソースからCAPデータを**取り込み**、BSAFタグ付きのBluesky投稿を**出力**します。両者は競合するものではなく、補完的な関係です。

---

## コントリビューション

あらゆる種類の貢献を歓迎します。

- 🤖 **Botを作る** — あなたの国の災害アラート用
- 📋 **プリセットを作る** — あなたの地域用
- 🐛 **問題を報告する** — 仕様に関する課題
- 📝 **ドキュメントを改善する** — 翻訳を含む
- 💡 **機能を提案する** — BSAF v2 に向けて

[CONTRIBUTING.md](CONTRIBUTING.md) のガイドラインをご参照ください。

---

## リファレンス実装

- **Bot**: [bsaf-jma-bot](https://github.com/osprey74/bsaf-jma-bot) — 日本の地震・津波・気象アラート（近日公開）
- **クライアント**: [kazahana](https://github.com/osprey74/kazahana) — BSAF対応 Blueskyデスクトップクライアント（近日公開）

---

## ライセンス

本仕様書は [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.ja) の下で公開されています。帰属表示を行えば、自由に採用・改変・再配布が可能です。

リファレンス実装は [MITライセンス](LICENSE) の下で公開されています。

---

<p align="center">
  <strong>BSAFはオープンなコミュニティの取り組みです。</strong><br>
  安全のために。すべての人のために。<br><br>
  <a href="https://github.com/osprey74/bsaf-protocol/issues">フィードバック</a> ·
  <a href="https://github.com/osprey74/bsaf-protocol/discussions">ディスカッション</a> ·
  <a href="CONTRIBUTING.md">コントリビュート</a>
</p>
