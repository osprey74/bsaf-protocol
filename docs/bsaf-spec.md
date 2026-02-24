# Bluesky Structured Alert Feed (BSAF)

An open convention for structured alert and information bots on Bluesky, enabling client-side filtering and personalization.

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
| Bots | Collect and post alerts with structured metadata | Bot developers (per country/source) |
| Convention | Shared tag format for machine-readable filtering | This specification |
| Clients | Filter and display posts based on user preferences | Client developers |

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

## How It Works

### 1. Bots post with structured tags

AT Protocol's `app.bsky.feed.post` record supports a `tags` field — an array of strings (max 8 tags, max 640 bytes total) that are not displayed in the post UI but are machine-readable.

BSAF uses this field to embed structured metadata:

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

### 2. Users register bots via Bot Definition JSON

Bot developers publish a **Bot Definition JSON** file that describes their bot and its available filter options. Users import this JSON into a BSAF-compatible client to register the bot.

```
Bot Developer                 User                        BSAF Client
    │                          │                            │
    │  Publish JSON file       │                            │
    │  (via Bluesky/GitHub)    │                            │
    │                          │                            │
    │                          │  Import JSON               │
    │                          │─────────────────────────▶  │
    │                          │                            │
    │                          │                     Parse JSON
    │                          │                     Auto-follow bot account
    │                          │                     Generate filter settings UI
    │                          │                            │
    │                          │  ◀──── Filter settings displayed
    │                          │  Select preferences        │
    │                          │                            │
    │  Post with BSAF tags ──────────────────────────────▶  │
    │                          │                     Apply tag-based filtering
    │                          │  ◀──── Show matching posts only
```

### 3. Clients filter based on user preferences

A BSAF-compatible client reads the `tags` field and applies user-defined filter rules configured per bot:

```
Example: User's filter configuration for JMA Bot
- type: earthquake ✅, tsunami ✅, eruption ✅, ashfall ❌
- value: 震度3 ✅, 震度4 ✅, 震度5弱 ✅, ... (enabled)
         震度1 ❌, 震度2 ❌ (disabled)
- target: jp-kanto ✅, jp-hokkaido ✅ (enabled)
        jp-kinki ❌, jp-kyushu ❌ (disabled)

Incoming post tags:
  type:earthquake, value:震度2, target:jp-kanto
  Result: HIDDEN (value "震度2" is not in user's enabled values)

Incoming post tags:
  type:earthquake, value:震度5強, target:jp-kanto
  Result: SHOWN (type, value, and target all match user's enabled filters)
```

### 4. Clients handle duplicate posts from multiple bots

When a user subscribes to multiple BSAF bots that cover the same data source, duplicate posts may occur. Clients SHOULD detect and collapse duplicates:

**Duplicate detection:** Posts are considered duplicates when `type` + `value` + `time` + `target` match across different bot accounts.

**Display behavior:** The first received post is displayed. Subsequent duplicates are collapsed with an indicator (e.g., "Also reported by N other bots"). This preserves redundancy (if one bot goes down, another still delivers) while keeping the timeline clean.

---

## Tag Specification (v1)

### Protocol Tag

Every BSAF-compatible post **MUST** include:

| Tag | Description |
|:----|:------------|
| `bsaf:v1` | Identifies this post as BSAF-compatible (version 1) |

### Core Tags

All core tags are **required** for every BSAF post.

| Prefix | Description | Example Values |
|:-------|:------------|:---------------|
| `type:` | Information category | `earthquake`, `tsunami`, `baseball`, `soccer` |
| `value:` | Scale / magnitude / weight | `震度5強`, `warning`, `final`, `highlight` |
| `time:` | Event timestamp (ISO 8601 UTC) | `2026-02-15T02:52:00Z` |
| `target:` | Target subject / audience | `jp-kanto`, `us-california`, `npb-giants` |
| `source:` | Originating authority / data source | `jma`, `nws`, `espn` |

### Tag Budget

AT Protocol allows max 8 tags and 640 bytes total.

| Slot | Tag | Required |
|:-----|:----|:---------|
| 1 | `bsaf:v1` | ✅ Required |
| 2 | `type:{category}` | ✅ Required |
| 3 | `value:{scale}` | ✅ Required |
| 4 | `time:{ISO8601}` | ✅ Required |
| 5 | `target:{target}` | ✅ Required |
| 6 | `source:{authority}` | ✅ Required |
| 7 | *(reserved)* | Optional |
| 8 | *(reserved)* | Optional |

6 required tags + 2 reserved slots for future extensions.

### Tag Value Guidelines

#### `type:` values

Bot developers define `type` values appropriate for their domain. Examples:

| Domain | type values |
|:-------|:------------|
| Disaster (Japan) | `earthquake`, `tsunami`, `eruption`, `ashfall`, `weather-warning`, `special-warning`, `landslide-warning`, `tornado-warning`, `heavy-rain`, `nankai-trough` |
| Disaster (US) | `earthquake`, `tsunami`, `tornado`, `hurricane`, `flood`, `wildfire`, `winter-storm` |
| Sports | `baseball`, `soccer`, `basketball`, `hockey` |
| News | `breaking`, `politics`, `business`, `technology` |

#### `value:` values

`value` represents the **weight or magnitude** of the information. Bot developers define discrete values appropriate for their domain. These are used as filter options in the client UI.

| Domain | value examples |
|:-------|:---------------|
| Japan earthquakes | `震度1`, `震度2`, `震度3`, `震度4`, `震度5弱`, `震度5強`, `震度6弱`, `震度6強`, `震度7` |
| Japan weather | `info`, `advisory`, `warning`, `severe-warning`, `special-warning` |
| US weather (NWS) | `advisory`, `watch`, `warning`, `extreme` |
| Sports | `pre-game`, `in-progress`, `final`, `highlight`, `breaking` |

#### `time:` values

ISO 8601 format in UTC. This **MUST** be the timestamp from the **original source event**, not the bot's posting time.

```
time:2026-02-15T02:52:00Z
```

#### `target:` values

Target values **MUST** follow the strict naming convention defined by each bot in its Bot Definition JSON. Target values are prefixed with a lowercase country code (ISO 3166-1 alpha-2) for geographic areas, or a domain-specific prefix for non-geographic subjects.

| Domain | target examples |
|:-------|:-------------|
| Japan (by region) | `jp-hokkaido`, `jp-tohoku`, `jp-kanto`, `jp-hokuriku`, `jp-chubu`, `jp-kinki`, `jp-chugoku`, `jp-shikoku`, `jp-kyushu`, `jp-okinawa` |
| US (by state) | `us-california`, `us-texas`, `us-new-york` |
| EU (by country) | `eu-germany`, `eu-france`, `eu-spain` |
| Sports (by team) | `npb-giants`, `npb-tigers`, `epl-arsenal`, `nfl-chiefs` |

**Critical:** Target values MUST be strictly normalized. Inconsistent target values across bots render the protocol useless for filtering. Bot developers MUST define the exact set of target values in their Bot Definition JSON, and clients use these definitions to build filter UIs.

### Same-Source Interoperability Requirement

When developing a new BSAF bot that covers the **same data source** as an existing BSAF bot, the new bot **MUST** respect the `value` and `target` values published in the existing bot's Bot Definition JSON and use identical values. This requirement exists to ensure that client-side duplicate detection functions correctly across multiple bots.

For example, if an existing JMA earthquake bot uses `value:震度5強` and `target:jp-kanto`, any new bot that also processes JMA earthquake data MUST use the same `value:震度5強` and `target:jp-kanto` — not alternatives such as `value:5+`, `value:shindo-5-upper`, `target:jp-kantou`, or `target:jp-関東`.

If a new bot uses different values for the same source events, clients will be unable to detect duplicates, resulting in redundant posts appearing in user timelines and undermining the core benefit of the BSAF protocol.

#### `source:` values

Identifier for the originating authority or data source.

| source value | Authority |
|:-------------|:----------|
| `jma` | Japan Meteorological Agency (気象庁) |
| `nws` | National Weather Service (US) |
| `meteoalarm` | MeteoAlarm (EU) |
| `bom` | Bureau of Meteorology (Australia) |
| `espn` | ESPN |
| `ap` | Associated Press |

---

## Bot Definition JSON

### Overview

Every BSAF bot **MUST** publish a Bot Definition JSON file. This file serves as the single source of truth for:

- Bot identity and data source information
- Available filter options for client UI generation
- Self-update URL for version management

Users import this JSON into their BSAF-compatible client to register the bot. The client parses the JSON, auto-follows the bot account, and dynamically generates a filter settings UI.

### Schema

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
        { "value": "tsunami", "label": "Tsunami" },
        { "value": "eruption", "label": "Eruption" },
        { "value": "ashfall", "label": "Ashfall" },
        { "value": "weather-warning", "label": "Weather Warning" },
        { "value": "special-warning", "label": "Special Warning" },
        { "value": "landslide-warning", "label": "Landslide Warning" },
        { "value": "tornado-warning", "label": "Tornado Warning" },
        { "value": "heavy-rain", "label": "Record Heavy Rain" },
        { "value": "nankai-trough", "label": "Nankai Trough Alert" }
      ]
    },
    {
      "tag": "value",
      "label": "Weight",
      "options": [
        { "value": "震度1", "label": "Seismic 1" },
        { "value": "震度2", "label": "Seismic 2" },
        { "value": "震度3", "label": "Seismic 3" },
        { "value": "震度4", "label": "Seismic 4" },
        { "value": "震度5弱", "label": "Seismic 5 Lower" },
        { "value": "震度5強", "label": "Seismic 5 Upper" },
        { "value": "震度6弱", "label": "Seismic 6 Lower" },
        { "value": "震度6強", "label": "Seismic 6 Upper" },
        { "value": "震度7", "label": "Seismic 7" },
        { "value": "info", "label": "Info" },
        { "value": "advisory", "label": "Advisory" },
        { "value": "warning", "label": "Warning" },
        { "value": "severe-warning", "label": "Severe Warning" },
        { "value": "special-warning", "label": "Special Warning" }
      ]
    },
    {
      "tag": "target",
      "label": "Region",
      "options": [
        { "value": "jp-hokkaido", "label": "Hokkaido" },
        { "value": "jp-tohoku", "label": "Tohoku" },
        { "value": "jp-kanto", "label": "Kanto" },
        { "value": "jp-hokuriku", "label": "Hokuriku" },
        { "value": "jp-chubu", "label": "Chubu" },
        { "value": "jp-kinki", "label": "Kinki" },
        { "value": "jp-chugoku", "label": "Chugoku" },
        { "value": "jp-shikoku", "label": "Shikoku" },
        { "value": "jp-kyushu", "label": "Kyushu" },
        { "value": "jp-okinawa", "label": "Okinawa" }
      ]
    }
  ]
}
```

### Field Definitions

| Field | Required | Description |
|:------|:---------|:------------|
| `bsaf_schema` | ✅ | Schema version (currently `"1.0"`) |
| `updated_at` | ✅ | Last update timestamp (ISO 8601) |
| `self_url` | ✅ | URL where this JSON is hosted; clients periodically fetch to check for updates |
| `bot.handle` | ✅ | Bluesky handle of the bot account |
| `bot.did` | ✅ | DID of the bot account |
| `bot.name` | ✅ | Human-readable bot name |
| `bot.description` | ✅ | Brief description of what the bot provides |
| `bot.source` | ✅ | Name of the primary data source |
| `bot.source_url` | Recommended | URL of the primary data source |
| `filters` | ✅ | Array of filter definitions for client UI generation |
| `filters[].tag` | ✅ | Which BSAF tag this filter applies to (`type`, `value`, or `target`) |
| `filters[].label` | ✅ | Display label for the filter group |
| `filters[].options` | ✅ | Array of selectable options |
| `filters[].options[].value` | ✅ | Tag value (must match actual tag values used in posts) |
| `filters[].options[].label` | ✅ | Human-readable label for this option |

### Update Mechanism

Clients **SHOULD** periodically fetch the `self_url` and compare `updated_at` with the stored version. If a newer version is found, the client updates the filter UI and notifies the user of any new options.

Recommended check interval: once per day.

---

## Post Format Requirements

### Source Attribution (Required)

Every BSAF post **MUST** include the primary data source name in the post text body. This ensures source attribution is visible to all users, including those using non-BSAF clients.

Format: `(Source: {source name})` at the end of the post text.

```
🔴 Earthquake Info
Feb 15, 11:52 — Ibaraki, Japan
Magnitude: 5.2 (Depth: ~50km)
Max intensity: Upper 5
Tsunami: None expected
(Source: JMA)
```

This is a **dual guarantee**: the `source:` tag provides machine-readable attribution, while the post text provides human-readable attribution.

### Language

Posts **MUST** set the `langs` field in the AT Protocol record to indicate the language of the post text.

```json
{
  "langs": ["ja"]
}
```

---

## For Bot Developers

### Building a BSAF-Compatible Bot

A BSAF bot has four jobs:

1. **Collect** — Monitor an official data source (e.g., JMA XML feeds, NWS CAP alerts)
2. **Format** — Create a human-readable post text with source attribution + BSAF tags
3. **Post** — Publish to Bluesky via AT Protocol API
4. **Publish** — Maintain a Bot Definition JSON file for client registration

### Minimal Example (TypeScript)

```typescript
import { BskyAgent, RichText } from "@atproto/api";

const agent = new BskyAgent({ service: "https://bsky.social" });
await agent.login({
  identifier: "your-bot.bsky.social",
  password: "app-password",
});

// Format the post (source attribution required in text)
const text =
  "🔴 Earthquake Info\n" +
  "Feb 15, 11:52 — Ibaraki, Japan\n" +
  "Magnitude: 5.2 (Depth: ~50km)\n" +
  "Max intensity: Upper 5\n" +
  "Tsunami: None expected\n" +
  "(Source: JMA)";

const rt = new RichText({ text });
await rt.detectFacets(agent);

// Post with BSAF tags (all 6 core tags required)
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

### Guidelines for Bot Operators

- **Profile:** Clearly identify as a bot (use "Bot" label in Bluesky profile settings)
- **Bio:** State the data source and coverage area
- **Posting frequency:** Keep reasonable intervals (10+ seconds between posts)
- **Deduplication:** Track posted alert IDs to avoid duplicate posts
- **Source attribution:** Always include the data source name in both tags AND post text
- **Language:** Set the `langs` field; post in the primary language of the data source
- **Bot Definition JSON:** Publish and maintain a JSON file at a stable URL

### Available Data Sources by Country

We welcome contributions! If you know of a public alert data source, please open a PR.

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

## For Client Developers

### Implementing BSAF Support

Any Bluesky client can support BSAF. Here's the basic flow:

#### 1. Bot Registration

User imports a Bot Definition JSON file. The client:
- Parses the JSON and validates the schema
- Auto-follows the bot's Bluesky account (`bot.handle`)
- Generates a filter settings UI from `filters` definitions
- Stores the `self_url` for periodic update checks

#### 2. Filter Settings UI

The client dynamically generates a settings screen for each registered bot based on the `filters` array. Every filter is rendered as a **multi-select option group**:

```
┌─ Japan Disaster Alerts (Unofficial) ────────────────┐
│                                                      │
│ 📌 Alert Type (type)                                │
│ [✅ Earthquake] [✅ Tsunami] [✅ Eruption] [□ Ashfall]│
│ [✅ Special Warning] [□ Weather Warning]             │
│                                                      │
│ 📌 Weight (value)                                   │
│ [□ Seismic 1] [□ Seismic 2] [✅ Seismic 3] ...     │
│ [□ Info] [□ Advisory] [✅ Warning] ...              │
│                                                      │
│ 📌 Region (target)                                    │
│ [✅ Hokkaido] [□ Tohoku] [✅ Kanto] [□ Hokuriku]    │
│                                                      │
└──────────────────────────────────────────────────────┘
```

All filters use the same input type (multi-select), keeping client implementation simple and consistent.

#### 3. Filtering Logic

For each incoming post from a registered bot:

```
def should_display(post, bot_config):
    # Not a BSAF post — display normally
    if "bsaf:v1" not in post.tags:
        return True

    tags = parse_bsaf_tags(post.tags)

    # Check each filter: post's tag value must be in user's enabled options
    for filter in bot_config.filters:
        tag_value = tags.get(filter.tag)
        if tag_value and tag_value not in filter.enabled_options:
            return False

    return True
```

#### 4. Duplicate Detection and Collapsing

When multiple registered bots post about the same event:

```
def is_duplicate(post_a, post_b):
    tags_a = parse_bsaf_tags(post_a.tags)
    tags_b = parse_bsaf_tags(post_b.tags)

    return (
        tags_a.type == tags_b.type and
        tags_a.value == tags_b.value and
        tags_a.time == tags_b.time and
        tags_a.target == tags_b.target and
        post_a.author != post_b.author  # Different bots
    )
```

**Display behavior:**
- Show the first received post
- Collapse duplicates with indicator: "Also reported by N other bot(s)"
- Users can expand to see all versions

This preserves **redundancy as a feature** — if one bot goes offline, users still receive alerts from other bots covering the same source.

#### 5. JSON Update Check

Clients periodically fetch each bot's `self_url` and compare `updated_at`:

```
def check_for_updates(bot_registration):
    response = fetch(bot_registration.self_url)
    remote_json = parse(response)

    if remote_json.updated_at > bot_registration.updated_at:
        # Update stored JSON
        # Refresh filter UI with new/removed options
        # Notify user of changes
```

---

## FAQ

### Why not use Bluesky's built-in feed generators?

Feed generators run server-side and require hosting infrastructure for each personalized feed. BSAF filtering runs entirely client-side with zero server cost per user. Bot developers only need to host the bot itself.

### Why not use hashtags instead of the `tags` field?

Hashtags clutter the post text and are prone to typos and inconsistencies. The `tags` field is invisible to users but machine-readable, keeping posts clean while enabling precise filtering.

### Can I use BSAF for non-disaster content?

Absolutely. BSAF is a **universal bot parsing system** designed to be fully domain-agnostic. News bots, sports score bots, traffic alert bots — anything that benefits from structured, filterable metadata can adopt BSAF. Simply define appropriate `type:` and `value:` options for your domain in your Bot Definition JSON.

### What if I need more than 8 tags?

The current specification uses 6 of 8 available slots, with 2 reserved for future extensions. If you need additional metadata, propose an extension for BSAF v2.

### How does duplicate detection work across bots?

Clients detect duplicates by comparing `type` + `value` + `time` + `target` across posts from different bot accounts. The `time` tag uses the **original source event timestamp** (not the bot's posting time), ensuring identical events produce identical `time` values regardless of which bot processes them.

### How is this different from CAP (Common Alerting Protocol)?

CAP is a comprehensive XML standard for emergency alerts used by national agencies. BSAF is a lightweight convention for social media posts on Bluesky. A typical BSAF bot *consumes* CAP data from national sources and *produces* Bluesky posts with BSAF tags. They're complementary, not competing.

---

## Roadmap

### Phase 1: Foundation (Current)

- [ ] Publish BSAF specification v1
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

## Contributing

We welcome contributions of all kinds:

- 🤖 **Build a bot** for your country's disaster alerts or any information domain
- 📋 **Publish a Bot Definition JSON** for your bot
- 🐛 **Report issues** with the specification
- 📝 **Improve documentation** and translations
- 💡 **Propose features** for BSAF v2

See [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

---

## Reference Implementation

- **Bot:** [bsaf-jma-bot](https://github.com/osprey74/bsaf-jma-bot) — Japan earthquake/tsunami/weather alerts *(coming soon)*
- **Client:** [kazahana](https://github.com/osprey74/kazahana) — Bluesky desktop client with BSAF support *(coming soon)*

---

## License

This specification is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You are free to adopt, modify, and redistribute it with attribution.

Reference implementations are licensed under [MIT License](../LICENSE).

---

**BSAF is an open community effort. Built for safety. Designed for everyone.**

[Feedback](https://github.com/osprey74/bsaf-protocol/issues) · [Discussions](https://github.com/osprey74/bsaf-protocol/discussions) · [Contribute](../CONTRIBUTING.md)
