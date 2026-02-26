# Bluesky Structured Alert Feed (BSAF)

**An open convention for structured alert and information bots on Bluesky, enabling client-side filtering and personalization.**

---

## The Problem

Disaster information, weather alerts, and breaking news bots face a fundamental tension on social media: **post too much, and you flood timelines; post too little, and you miss critical information for someone, somewhere.**

A weather bot that posts every advisory for an entire country quickly becomes noise. But for the person in the path of a tornado, that single post is the most important thing in their feed.

Current solutions force a choice:

- **One global bot** → Timeline overload, most posts irrelevant to any given user
- **Regional bots** → Fragmented ecosystem, users must find and follow the right accounts
- **Platform-specific features** → Lock-in, not portable across clients

---

## The Solution

**BSAF (Bluesky Structured Alert Feed)** is an open convention that separates concerns:

| Layer | Responsibility | Who builds it |
|:------|:---------------|:--------------|
| **Bots** | Collect and post alerts with structured metadata | Bot developers (per country/source) |
| **Convention** | Shared tag format for machine-readable filtering | This specification |
| **Clients** | Filter and display posts based on user preferences | Client developers |

**Bots post broadly. Clients filter locally. Users see only what matters to them.**

```
┌─────────────────────────────┐
│     Bluesky Network         │
│                             │
┌──────────┐  │  Posts with BSAF tags    │  ┌──────────────┐
│ 🇯🇵 JMA Bot │────▶│                         │◀────│ Any Bluesky  │
│ 🇺🇸 NWS Bot │────▶│  Searchable, filterable  │◀────│ Client       │
│ 🇪🇺 EU Bot  │────▶│  machine-readable        │◀────│ (kazahana,   │
│ 🌍 Anyone  │────▶│  metadata                │◀────│  etc.)       │
└──────────┘  └─────────────────────────────┘  └──────────────┘
                              │
                              ▼
                    ┌──────────────┐
                    │ User sees:   │
                    │ Only relevant│
                    │ alerts for   │
                    │ their target │
                    │ & preferences│
                    └──────────────┘
```

---

## Design Philosophy

BSAF is a **universal bot parsing system**, not limited to disaster alerts. The convention is designed to be fully domain-agnostic: disaster bots, sports score bots, news bots, traffic alert bots — anything that benefits from structured, filterable metadata can adopt BSAF.

The core tags use intentionally generic names (`type`, `value`, `target`) rather than domain-specific terms, ensuring the protocol works across all content domains without modification.

---

## Quick Example

```json
{
  "$type": "app.bsky.feed.post",
  "text": "🔴 Earthquake Info\nFeb 15, 11:52 — Ibaraki, Japan\nMagnitude: 5.2 (Depth: ~50km)\nMax intensity: Upper 5\nTsunami: None expected\n(Source: JMA)",
  "tags": [
    "bsaf:v1",
    "type:earthquake",
    "value:震度5強",
    "time:2026-02-15T02:52:00Z",
    "target:jp-kanto",
    "source:jma"
  ],
  "langs": ["en"],
  "createdAt": "2026-02-15T02:55:54Z"
}
```

The post text remains human-readable. The tags enable machine filtering.

### Tag Overview (v1)

| Slot | Tag | Description | Required |
|:-----|:----|:-----------|:---------|
| 1 | `bsaf:v1` | Protocol identifier | ✅ |
| 2 | `type:{category}` | Information category | ✅ |
| 3 | `value:{scale}` | Scale / magnitude / weight | ✅ |
| 4 | `time:{ISO8601}` | Source event timestamp (UTC) | ✅ |
| 5 | `target:{target}` | Target subject / audience | ✅ |
| 6 | `source:{authority}` | Originating authority / data source | ✅ |
| 7 | *(reserved)* | Optional | |
| 8 | *(reserved)* | Optional | |

6 required tags + 2 reserved slots for future extensions.

---

## Full Specification

📄 **[BSAF Specification v1 (English)](docs/bsaf-spec.md)** — Complete protocol specification

📄 **[BSAF仕様書 v1 (日本語)](docs/bsaf-spec-ja.md)** — 完全なプロトコル仕様書

---

## Key Concepts

### Bot Definition JSON

Every BSAF bot publishes a **Bot Definition JSON** file that describes available filter options. Users import this JSON into a BSAF-compatible client to register the bot and auto-generate filter settings UI.

→ See [full specification](docs/bsaf-spec.md#bot-definition-json) for schema details.

### Duplicate Detection

When multiple bots cover the same data source, clients detect duplicates by comparing `type` + `value` + `time` + `target` across posts from different bot accounts, and collapse them with an indicator.

→ See [full specification](docs/bsaf-spec.md#4-clients-handle-duplicate-posts-from-multiple-bots) for details.

### Same-Source Interoperability

When developing a new BSAF bot covering the same data source as an existing bot, the new bot **MUST** use identical `value` and `target` values from the existing bot's Bot Definition JSON.

→ See [full specification](docs/bsaf-spec.md#same-source-interoperability-requirement) for details.

---

## For Bot Developers

A BSAF bot has four jobs:

1. **Collect** — Monitor an official data source (e.g., JMA XML feeds, NWS CAP alerts)
2. **Format** — Create a human-readable post text with source attribution + BSAF tags
3. **Post** — Publish to Bluesky via AT Protocol API
4. **Publish** — Maintain a Bot Definition JSON file for client registration

→ See [full specification](docs/bsaf-spec.md#for-bot-developers) for implementation guide, code examples, and available data sources.

## For Client Developers

Any Bluesky client can support BSAF:

1. **Bot Registration** — User imports Bot Definition JSON → auto-follow + generate filter UI
2. **Filter Settings** — Dynamic multi-select UI generated from the JSON's `filters` array
3. **Filtering Logic** — Check each post's `tags` against user's enabled options
4. **Duplicate Collapsing** — Detect and collapse duplicate posts from multiple bots

→ See [full specification](docs/bsaf-spec.md#for-client-developers) for implementation guide and pseudocode.

---

## Available Data Sources by Country

| Country | Source | Format | Access | Notes |
|:--------|:-------|:-------|:-------|:------|
| 🇯🇵 Japan | JMA XML Feed | Atom/XML | Free, no registration | Updated every minute |
| 🇺🇸 USA | NWS CAP Alerts | CAP/Atom | Free, no registration | Real-time |
| 🇪🇺 EU | MeteoAlarm | CAP/RSS | Free | 30+ European countries |
| 🇦🇺 Australia | BOM Warnings | RSS/XML | Free | |
| 🇹🇼 Taiwan | CWA Open Data | JSON/XML | Free, registration required | |
| 🇰🇷 Korea | KMA Data Portal | JSON/XML | Free, registration required | |
| 🇨🇦 Canada | ECCC Alerts | CAP/Atom | Free | |
| 🇳🇿 New Zealand | MetService Warnings | RSS | Free | |
| 🌍 Global | GDACS | RSS/CAP | Free | Major disasters only |

---

## Roadmap

### Phase 1: Foundation (Current)

- [x] Publish BSAF specification v1
- [ ] Launch reference bot: Japan earthquake & tsunami alerts
- [ ] Implement BSAF filtering in kazahana as reference client
- [ ] Publish Bot Definition JSON for reference bot

### Phase 2: Ecosystem Growth

- [ ] Community-contributed bots for US (NWS), EU (MeteoAlarm), and others
- [ ] Bot Definition JSON directory / registry
- [ ] BSAF tag validation tool for bot developers
- [ ] Adoption by additional Bluesky clients

### Phase 3: Evolution

- [ ] BSAF v2 specification (based on community feedback)
- [ ] Support for media attachments (alert maps, radar images)
- [ ] Rich embed cards with structured alert data
- [ ] Integration with AT Protocol's labeling system

---

## FAQ

### Why not use Bluesky's built-in feed generators?

Feed generators run server-side and require hosting infrastructure for each personalized feed. BSAF filtering runs entirely client-side with zero server cost per user. Bot developers only need to host the bot itself.

### Why not use hashtags instead of the `tags` field?

Hashtags clutter the post text and are prone to typos and inconsistencies. The `tags` field is invisible to users but machine-readable, keeping posts clean while enabling precise filtering.

### Can I use BSAF for non-disaster content?

Absolutely. BSAF is a **universal bot parsing system** designed to be fully domain-agnostic. News bots, sports score bots, traffic alert bots — anything that benefits from structured, filterable metadata can adopt BSAF. Simply define appropriate `type:` and `value:` options for your domain in your Bot Definition JSON.

### How does duplicate detection work across bots?

Clients detect duplicates by comparing `type` + `value` + `time` + `target` across posts from different bot accounts. The `time` tag uses the **original source event timestamp** (not the bot's posting time), ensuring identical events produce identical `time` values regardless of which bot processes them.

### How is this different from CAP (Common Alerting Protocol)?

CAP is a comprehensive XML standard for emergency alerts used by national agencies. BSAF is a lightweight convention for social media posts on Bluesky. A typical BSAF bot *consumes* CAP data from national sources and *produces* Bluesky posts with BSAF tags. They're complementary, not competing.

---

## Contributing

We welcome contributions of all kinds:

- 🤖 **Build a bot** for your country's disaster alerts or any information domain
- 📋 **Publish a Bot Definition JSON** for your bot
- 🐛 **Report issues** with the specification
- 📝 **Improve documentation** and translations
- 💡 **Propose features** for BSAF v2

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## Reference Implementation

- **Bot:** [bsaf-jma-bot](https://github.com/osprey74/bsaf-jma-bot) — Japan earthquake/tsunami/weather alerts *(coming soon)*
- **Client:** [kazahana](https://github.com/osprey74/kazahana) — Bluesky desktop client with BSAF support *(coming soon)*

---

## License

This specification is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You are free to adopt, modify, and redistribute it with attribution.

Reference implementations are licensed under [MIT License](LICENSE).

---

**BSAF is an open community effort. Built for safety. Designed for everyone.**

[Feedback](https://github.com/osprey74/bsaf-protocol/issues) · [Discussions](https://github.com/osprey74/bsaf-protocol/discussions) · [Contribute](CONTRIBUTING.md)
