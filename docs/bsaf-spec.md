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

## The Solution

**BSAF (Bluesky Structured Alert Feed)** is an open convention that separates concerns:

| Layer | Responsibility | Who builds it |
|:------|:---------------|:--------------|
| **Bots** | Collect and post alerts with structured metadata | Bot developers (per country/source) |
| **Convention** | Shared tag format for machine-readable filtering | This specification |
| **Clients** | Filter and display posts based on user preferences | Client developers |

Bots post broadly. Clients filter locally. **Users see only what matters to them.**

```
                    ┌─────────────────────────────┐
                    │     Bluesky Network          │
                    │                              │
  ┌──────────┐     │   Posts with BSAF tags        │     ┌──────────────┐
  │ 🇯🇵 JMA Bot │────▶│                              │◀────│ Any Bluesky  │
  │ 🇺🇸 NWS Bot │────▶│   Searchable, filterable     │◀────│   Client     │
  │ 🇪🇺 EU  Bot │────▶│   machine-readable metadata  │◀────│ (kazahana,   │
  │ 🌍 Anyone  │────▶│                              │◀────│  etc.)       │
  └──────────┘     └─────────────────────────────┘     └──────────────┘
                                                              │
                                                              ▼
                                                      ┌──────────────┐
                                                      │ User sees:   │
                                                      │ Only relevant│
                                                      │ alerts for   │
                                                      │ their region │
                                                      │ & preferences│
                                                      └──────────────┘
```

## How It Works

### 1. Bots post with structured `tags`

AT Protocol's `app.bsky.feed.post` record supports a [`tags` field](https://docs.bsky.app/docs/advanced-guides/posts#tags) — an array of strings (max 8 tags, max 640 bytes total) that are **not displayed in the post UI** but are machine-readable.

BSAF uses this field to embed structured metadata:

```json
{
  "$type": "app.bsky.feed.post",
  "text": "🔴 Earthquake: M5.2, Ibaraki Prefecture, Japan. Max intensity: Upper 5. Tsunami: None expected.",
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

The post text remains human-readable. The tags enable machine filtering.

### 2. Clients filter based on user preferences

A BSAF-compatible client reads the `tags` field and applies user-defined rules:

```
User's filter configuration:
  - country = JP
  - region = hokkaido
  - Show earthquakes: severity >= 3
  - Always show: tsunami, eruption, severity >= 5

Incoming post tags:
  type:earthquake, severity:2, country:JP, region:kanto

Result: HIDDEN (region mismatch, severity below threshold)
```

```
Incoming post tags:
  type:earthquake, severity:5+, country:JP, region:kanto

Result: SHOWN (severity >= 5 overrides region filter)
```

### 3. Users configure via presets or custom rules

For ease of use, clients can offer presets (pre-built filter configurations):

```json
{
  "preset_name": "Japan Disaster Alerts (Hokkaido)",
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

Presets are shareable JSON files that users can import/export, and communities can publish for their country or region.

---

## Tag Specification (v1)

### Protocol Tag

Every BSAF-compatible post **MUST** include:

| Tag | Description |
|:----|:------------|
| `bsaf:v1` | Identifies this post as BSAF-compatible (version 1) |

### Core Tags

| Prefix | Required | Description | Example Values |
|:-------|:---------|:------------|:---------------|
| `type:` | **Yes** | Alert category | `earthquake`, `tsunami`, `weather-warning`, `weather-watch`, `eruption`, `tornado`, `hurricane`, `flood`, `wildfire`, `special-warning` |
| `severity:` | **Yes** | Severity level | See Severity Scale below |
| `country:` | **Yes** | ISO 3166-1 alpha-2 | `JP`, `US`, `DE`, `FR`, `TW` |
| `region:` | Recommended | Sub-national region | `kanto`, `midwest`, `bavaria` |
| `subregion:` | Optional | Specific area | `ibaraki`, `oklahoma`, `munich` |

### Severity Scale

Severity values are **intentionally flexible** to accommodate different national systems. However, BSAF defines a recommended mapping:

| BSAF Level | Meaning | Japan (JMA) | US (NWS) | EU (MeteoAlarm) |
|:-----------|:--------|:------------|:---------|:-----------------|
| `info` | Informational | — | Advisory | Green |
| `minor` | Minor | 震度1-2 / 注意報 | Watch | Yellow |
| `moderate` | Moderate | 震度3-4 / 警報 | Warning | Orange |
| `severe` | Severe | 震度5弱-5強 | Severe | Red |
| `critical` | Critical / Life-threatening | 震度6弱+ / 特別警報 / 大津波警報 | Extreme | Purple / Dark Red |

For **earthquake-specific** severity, numeric Shindo / Modified Mercalli values are also valid:

```
severity:3        → JMA Shindo 3
severity:5+       → JMA Shindo 5-upper (5強)
severity:5-       → JMA Shindo 5-lower (5弱)
severity:mmi-VII  → Modified Mercalli VII
```

### Optional Tags

| Prefix | Description | Example Values |
|:-------|:------------|:---------------|
| `source:` | Originating authority | `jma`, `nws`, `dwd`, `meteoalarm`, `bom` |
| `scope:` | Geographic scope | `local`, `regional`, `national` |
| `expires:` | Expiration (ISO 8601) | `2026-02-15T12:00:00Z` |

### Tag Budget

AT Protocol allows max **8 tags** and **640 bytes total**. Prioritize in this order:

1. `bsaf:v1` (required)
2. `type:` (required)
3. `severity:` (required)
4. `country:` (required)
5. `region:` (recommended)
6. `subregion:` (if space allows)
7. `source:` (if space allows)
8. Additional context (if space allows)

---

## For Bot Developers

### Building a BSAF-Compatible Bot

A BSAF bot has three jobs:

1. **Collect** — Monitor an official data source (e.g., JMA XML feeds, NWS CAP alerts, MeteoAlarm API)
2. **Format** — Create a human-readable post text + BSAF tags
3. **Post** — Publish to Bluesky via AT Protocol API

#### Minimal Example (TypeScript)

```typescript
import { BskyAgent, RichText } from "@atproto/api";

const agent = new BskyAgent({ service: "https://bsky.social" });
await agent.login({ identifier: "your-bot.bsky.social", password: "app-password" });

// Format the post
const text = "🔴 Earthquake: M5.2, Ibaraki, Japan\nMax intensity: Upper 5\nTsunami: None expected";
const rt = new RichText({ text });
await rt.detectFacets(agent);

// Post with BSAF tags
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

#### Guidelines for Bot Operators

- **Profile**: Clearly identify as a bot (use "Bot" label in Bluesky profile settings)
- **Bio**: State the data source and coverage area (e.g., "Japan earthquake & tsunami alerts from JMA")
- **Posting frequency**: Keep reasonable intervals (10+ seconds between posts)
- **Deduplication**: Track posted alert IDs to avoid duplicate posts
- **Attribution**: Always credit the original data source
- **Language**: Post in the primary language of the data source; consider multilingual posts for critical alerts

### Available Data Sources by Country

We welcome contributions! If you know of a public alert data source, please open a PR.

| Country | Source | Format | Access | Notes |
|:--------|:-------|:-------|:-------|:------|
| 🇯🇵 Japan | [JMA XML Feed](https://xml.kishou.go.jp/xmlpull.html) | Atom/XML | Free, no registration | Updated every minute |
| 🇺🇸 USA | [NWS CAP Alerts](https://alerts.weather.gov/) | CAP/Atom | Free, no registration | Real-time |
| 🇪🇺 EU | [MeteoAlarm](https://www.meteoalarm.org/) | CAP/RSS | Free | 30+ European countries |
| 🇦🇺 Australia | [BOM Warnings](http://www.bom.gov.au/catalogue/warnings/) | RSS/XML | Free | |
| 🇹🇼 Taiwan | [CWA Open Data](https://opendata.cwa.gov.tw/) | JSON/XML | Free, registration required | |
| 🇰🇷 Korea | [KMA Data Portal](https://data.kma.go.kr/) | JSON/XML | Free, registration required | |
| 🇨🇦 Canada | [ECCC Alerts](https://weather.gc.ca/warnings/) | CAP/Atom | Free | |
| 🇳🇿 New Zealand | [MetService Warnings](https://www.metservice.com/) | RSS | Free | |
| 🌍 Global | [GDACS](https://www.gdacs.org/) | RSS/CAP | Free | Major disasters only |

---

## For Client Developers

### Implementing BSAF Filtering

Any Bluesky client can support BSAF. Here's the basic flow:

```
1. User configures BSAF feed tab:
   - Specifies bot account(s) to monitor
   - Sets filter rules (country, region, severity thresholds)
   - Or imports a community preset

2. Client fetches posts from specified account(s):
   - Via Bluesky List feed (recommended)
   - Via direct author feed API
   - Via following timeline (simplest)

3. For each post, client checks for "bsaf:v1" tag:
   - If absent → display normally (not a BSAF post)
   - If present → apply filter rules against tags

4. Display matching posts in dedicated tab/view
```

#### Filtering Logic (Pseudocode)

```python
def should_display(post, user_config):
    tags = parse_bsaf_tags(post.tags)
    
    # Not a BSAF post — show normally
    if "bsaf:v1" not in post.tags:
        return True
    
    # Check global overrides first (always show critical alerts)
    for override in user_config.global_overrides:
        if matches(tags, override):
            return True
    
    # Check country filter
    if tags.country not in user_config.countries:
        return False
    
    # Check region filter (if configured)
    if user_config.regions and tags.region not in user_config.regions:
        # Still show if severity exceeds regional override threshold
        if not exceeds_threshold(tags.severity, user_config.regional_override):
            return False
    
    # Check type-specific settings
    type_config = user_config.type_settings.get(tags.type)
    if type_config:
        if type_config.always_show:
            return True
        if not meets_severity(tags.severity, type_config.min_severity):
            return False
    
    return True
```

### Preset Distribution

Presets are JSON files that bundle filter configurations for easy sharing.

**Recommended distribution channels:**

- GitHub repository (this repo's `/presets` directory)
- Direct URL import in client settings
- QR code / share link from another user

Preset files should follow the naming convention:

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

## Roadmap

### Phase 1: Foundation (Current)
- [x] Publish BSAF specification v1
- [ ] Launch reference bot: Japan earthquake & tsunami alerts
- [ ] Implement BSAF filtering in [kazahana](https://github.com/osprey74/kazahana) as reference client
- [ ] Publish Japan preset files

### Phase 2: Ecosystem Growth
- [ ] Community-contributed bots for US (NWS), EU (MeteoAlarm), and others
- [ ] Preset sharing platform / directory
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

Absolutely. The convention is designed to be content-agnostic. News bots, sports score bots, traffic alert bots — anything that benefits from structured, filterable metadata can adopt BSAF. Simply define appropriate `type:` values for your domain.

### What if I need more than 8 tags?

Prioritize the core tags (`bsaf:v1`, `type`, `severity`, `country`, `region`). If you consistently need more metadata, consider encoding compound values (e.g., `geo:JP-kanto-ibaraki` as a single tag) or proposing an extension for BSAF v2.

### How is this different from CAP (Common Alerting Protocol)?

[CAP](https://en.wikipedia.org/wiki/Common_Alerting_Protocol) is a comprehensive XML standard for emergency alerts used by national agencies. BSAF is a lightweight convention for social media posts on Bluesky. A typical BSAF bot **consumes** CAP data from national sources and **produces** Bluesky posts with BSAF tags. They're complementary, not competing.

---

## Contributing

We welcome contributions of all kinds:

- 🤖 **Build a bot** for your country's disaster alerts
- 📋 **Create presets** for your region
- 🐛 **Report issues** with the specification
- 📝 **Improve documentation** and translations
- 💡 **Propose features** for BSAF v2

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## Reference Implementation

- **Bot**: [bsaf-jma-bot](https://github.com/osprey74/bsaf-jma-bot) — Japan earthquake/tsunami/weather alerts (coming soon)
- **Client**: [kazahana](https://github.com/osprey74/kazahana) — Bluesky desktop client with BSAF support (coming soon)

---

## License

This specification is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You are free to adopt, modify, and redistribute it with attribution.

Reference implementations are licensed under [MIT License](LICENSE).

---

<p align="center">
  <strong>BSAF is an open community effort.</strong><br>
  Built for safety. Designed for everyone.<br><br>
  <a href="https://github.com/osprey74/bsaf/issues">Feedback</a> ·
  <a href="https://github.com/osprey74/bsaf/discussions">Discussions</a> ·
  <a href="CONTRIBUTING.md">Contribute</a>
</p>
