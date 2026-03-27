# Spec 004: Source Discovery & Collection

**Status**: Design
**Priority**: P0 (blocks content velocity — pipeline can only process what's manually entered)
**Depends on**: Spec 002 (WF-1 deployed), Spec 003 (dashboard/observability complete)

## Problem

The content pipeline (WF-1) can research, draft, evaluate, and publish — but it has no automated input. Every topic is manually discovered by Aaron and entered into the Content Creation DB. This limits content velocity to however many topics Aaron happens to find and enter, and misses signals from the broader AI industry that T-0 should be reacting to.

The AI Intelligence Pipeline spec (Notion, Feb 2026) envisioned a passive intelligence system that continuously collects signals from RSS, Slack, Google Alerts, newsletters, and more. This spec designs and builds that system, integrated with the existing content pipeline.

## Solution Overview

A new n8n workflow (WF-2) that monitors external sources on schedule, extracts items, runs AI triage for relevance, stores qualified items in the T-0 Notes DB, and periodically aggregates them into intelligence digests in the T-0 Reports DB. From there, topics flow into the Content Creation DB for drafting via WF-1.

### Data Model: 4 Databases, 1 Pipeline

```
Content Sources DB (NEW)              T-0 Notes DB (EXISTS)
  "Where do we look?"                   "What did we find?"
  Registry of all monitored      ←→     Raw items + manual notes
  sources with extraction mode          + Slack captures
           │                                    │
           │  WF-2 extracts on schedule         │  Periodic aggregation
           └──────────────→─────────────────────┘
                                                │
                                                ▼
                                        T-0 Reports DB (EXISTS)
                                          "What matters this period?"
                                          Topic clusters + recommendations
                                                │
                                                │  Human picks topics
                                                ▼
                                        T-0 Content Creation DB (EXISTS)
                                          "What are we writing?"
                                          Drafts → evaluation → Ghost
```

All relations are **dual** (bidirectional).

---

## Part A: Database Schema Changes

### A1. Content Sources DB (NEW)

Registry of all monitored sources — blogs, podcasts, Slack channels, RSS feeds, etc.

| Property | Type | Options / Notes |
|----------|------|-----------------|
| `Name` | title | Human-readable source name (e.g., "Anthropic Blog", "Hacker News AI") |
| `URL` | url | Base URL of the source |
| `Feed URL` | url | RSS/Atom feed URL if applicable |
| `Type` | select | Blog, Podcast, Newsletter, YouTube, Slack Channel, GitHub, ArXiv, Google Alert, Forum, News Outlet |
| `Access Method` | select | RSS, API, Jina, Slack API, Email Parse, Manual |
| `Status` | status | New → Active → Paused → Archived |
| `Frequency` | select | Realtime, Hourly, Daily, Weekly, Monthly |
| `Last Checked` | date | When WF-2 last polled this source |
| `Content Hash` | rich_text | SHA-256 for change detection on non-feed sources |
| `Language` | select | DE, EN, Mixed |
| `Category Tags` | multi_select | Cross-cutting taxonomy (shared with Notes + Content Creation) |
| `Description` | rich_text | What this source covers, any access notes |
| `Tags` | multi_select | Cross-cutting labels: Competitor, Model Provider, German Market, etc. |
| `Notes` | relation (dual) | → T-0 Notes DB (items produced by this source) |
| `Content` | relation (dual) | → T-0 Content Creation DB (content that used this source) |

**Location**: Inline database inside the T-0 Content Production page (alongside Content Creation DB).

### A2. T-0 Notes DB — Modifications

| Change | Details |
|--------|---------|
| **RENAME** `AI-Category` → `Category Tags` | Make it a standard property (not AI-generated). Same cross-cutting taxonomy. **Requires Playwright** — API cannot rename properties. |
| **ADD** `Source` relation (dual) | → Content Sources DB. Traces each item to its originating source. |
| **ADD** Origin options | New values: `rss-ingestion`, `google-alert`, `newsletter-ingestion`, `github-trending`, `arxiv`. Existing options unchanged. |

**Properties that already work for source ingestion (no changes needed):**
- `Origin` + `Origin Description` — how the item arrived
- `URL` — article/item link
- `Time` — published date
- `Summary` — AI-generated triage summary
- `Description` — RSS excerpt or longer description
- `AI-Tags` — machine-assigned topic tags (keep AI- prefix, these are triage-assigned)
- `Tags` — human curation layer
- `Status` — Backlog → To Be Processed → Currently Relevant → Keep → Archived → Discarded
- `Content Creation` relation — already links to Content Creation DB
- `Created time` — serves as "fetched date" (when we ingested it)

### A3. T-0 Reports DB — Modifications

| Change | Details |
|--------|---------|
| **ADD** `Content` relation (dual) | → T-0 Content Creation DB. Tracks which posts were created from this report. |
| **ADD** `Time` property | Date with start+end period. Covers the time window the report aggregates. |
| **ADD** Tag options | New values: `Content Intelligence`, `Weekly Digest`. Existing options unchanged. |

### A4. T-0 Content Creation DB — Modifications

| Change | Details |
|--------|---------|
| **ADD** `Report` relation (dual) | → T-0 Reports DB. Links content to the digest that recommended it. |
| **ADD** `Sources` relation (dual) | → Content Sources DB. Direct link to sources the content builds upon. Replaces the need for Source Type select expansion. |

**Existing `Source Type` select** (RSS, Slack Rant, Manual, Mixed): Leave as-is. Still useful for quick filtering. No new options needed — the `Sources` relation provides full traceability.

### A5. Cross-Cutting Category Taxonomy

The `Category Tags` multi_select shares a common vocabulary across three databases:

**Topic categories** (what the content is about):
- AI Models, Agents & Automation, Context Engineering, Dev Tools, Infrastructure, Security
- Industry Moves, Regulation, Funding, Competitors
- Strategy, Work & Society, German Market
- MCP, Ghost, Notion (T-0 stack-specific)

**Format indicators** (only on Content Creation DB, via `Content Type` select):
- Opinion, Resource, Case Study, Technical, Announcement

The exact option list will be finalized during implementation by merging the current Notes `AI-Category` options, Content Creation `Category Tags` options, and any gaps. The goal is one shared vocabulary.

---

## Part B: Source Catalogue

### B1. Initial Source List

Based on the AI Intelligence Pipeline spec, spec 001 blog source research, and T-0's focus areas:

**Tier 1 — Core (weekly or more frequent)**

| Source | Type | Access Method | Feed URL / Notes |
|--------|------|--------------|------------------|
| Anthropic Blog | Blog | RSS | `anthropic.com/research/rss.xml` (verify) |
| OpenAI Blog | Blog | RSS | `openai.com/blog/rss.xml` (verify) |
| Google DeepMind Blog | Blog | RSS/Jina | May not have RSS — Jina fallback |
| Simon Willison's Weblog | Blog | RSS | `simonwillison.net/atom/everything/` |
| Hacker News (AI tagged) | Forum | RSS/API | `hnrss.org/newest?q=AI` or Algolia API |
| The Verge AI | News Outlet | RSS | `/ai/rss/index.xml` (verify) |
| TechCrunch AI | News Outlet | RSS | Category feed |
| Import AI Newsletter | Newsletter | RSS/Email | Jack Clark's weekly newsletter |
| The Batch (Andrew Ng) | Newsletter | RSS/Email | DeepLearning.AI weekly |
| Slack #tech-shit-talk | Slack Channel | Slack API | Already captured via MCP Hub — needs WF-2 integration |

**Tier 2 — Extended (weekly)**

| Source | Type | Access Method |
|--------|------|--------------|
| Stratechery (Ben Thompson) | Blog | RSS |
| Latent Space Podcast | Podcast | RSS |
| Lenny's Newsletter (AI sections) | Newsletter | RSS |
| ArXiv cs.AI / cs.CL | ArXiv | API/RSS |
| GitHub Trending (ML/AI) | GitHub | API |
| Mistral Blog | Blog | RSS |
| Meta AI Blog | Blog | RSS/Jina |
| Hugging Face Blog | Blog | RSS |

**Tier 3 — German Market (weekly/monthly)**

| Source | Type | Access Method |
|--------|------|--------------|
| Heise (KI section) | News Outlet | RSS |
| t3n (KI section) | News Outlet | RSS |
| KI Bundesverband | Institutional | Jina |
| German AI startups (curated) | Mixed | Google Alerts |

**Google Alerts** (to be configured on T-0 Workspace Google account):
- "KI Beratung Deutschland"
- "AI consulting Germany"
- "MCP protocol"
- "AI agents enterprise"
- T-0 brand mentions

### B2. Source Onboarding Process

For each source:
1. Verify access method works (fetch feed, confirm structure)
2. Create entry in Content Sources DB with all properties
3. Set Status = Active, Frequency, Language, Category Tags
4. Test extraction in WF-2 (one-off manual trigger)
5. Monitor first automated run for quality

---

## Part C: WF-2 — Source Ingestion Workflow

### C1. Architecture

A single n8n workflow with multiple input branches converging into a central processing pipeline.

```
┌─────────────────────────────────────────────────────────┐
│  WF-2: Source Ingestion                                  │
│                                                          │
│  TRIGGERS                                                │
│  ├── Schedule: RSS sources (weekly, Sunday 8:00 CET)     │
│  ├── Schedule: Slack capture (every 6 hours)             │
│  ├── Schedule: Google Alerts (daily)                     │
│  ├── Webhook: Manual trigger (for testing / on-demand)   │
│  └── (Future: Newsletter email parse, GitHub trending)   │
│                                                          │
│  PHASE 1: FETCH SOURCES                                  │
│  ├── Query Content Sources DB (Status = Active)          │
│  ├── Group by Access Method                              │
│  └── Fan out to extraction branches                      │
│                                                          │
│  PHASE 2: EXTRACTION BRANCHES                            │
│  ├── Branch A: RSS/Atom feeds                            │
│  │   ├── HTTP GET each Feed URL                          │
│  │   ├── Parse RSS/Atom (regex parser, Josepha pattern)  │
│  │   ├── Filter: items from last N days                  │
│  │   └── Normalize to unified item format                │
│  │                                                       │
│  ├── Branch B: Slack #tech-shit-talk                     │
│  │   ├── Fetch threads (Slack API via MCP Hub)           │
│  │   ├── Filter: 2+ replies from 2+ people              │
│  │   ├── Summarize thread                                │
│  │   └── Normalize to unified item format                │
│  │                                                       │
│  ├── Branch C: Jina (non-RSS web sources)                │
│  │   ├── Read URL via Jina                               │
│  │   ├── Compare Content Hash for changes                │
│  │   └── Normalize to unified item format                │
│  │                                                       │
│  └── Branch D: Google Alerts (RSS feed of alerts)        │
│      ├── HTTP GET alert feed                             │
│      ├── Parse alert items                               │
│      └── Normalize to unified item format                │
│                                                          │
│  PHASE 3: CENTRAL PROCESSING                             │
│  ├── Merge all branches into unified stream              │
│  ├── Deduplication (URL-based + fuzzy title matching)    │
│  │   Query Notes DB for existing URLs                    │
│  ├── AI Triage (relevance scoring)                       │
│  │   ├── Score each item for T-0 relevance               │
│  │   ├── Assign Category Tags and AI-Tags                │
│  │   ├── Generate Summary                                │
│  │   └── Gate: only items above threshold proceed        │
│  ├── Write to Notes DB                                   │
│  │   ├── Set Origin, Origin Description, Source relation  │
│  │   ├── Set Time, URL, Summary, Category Tags, AI-Tags  │
│  │   ├── Set Status = "To Be Processed"                  │
│  │   └── Full article text in page body (if available)   │
│  └── Update Source: Last Checked = now()                 │
│                                                          │
│  PHASE 4: ERROR HANDLING                                 │
│  ├── Per-source error isolation (one failing source      │
│  │   doesn't block others)                               │
│  ├── Error Trigger → Notion comment on Source page        │
│  └── Source Status → "Paused" after 3 consecutive fails  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### C2. Unified Item Format

All extraction branches normalize items to this shape before central processing:

```json
{
  "title": "Article or thread title",
  "url": "https://...",
  "published": "2026-03-25T10:00:00Z",
  "summary": "First 500 chars or RSS description",
  "full_text": "Full article text (if available via Jina/RSS)",
  "source_name": "Anthropic Blog",
  "source_page_id": "notion-page-id-of-source",
  "origin": "rss-ingestion",
  "origin_description": "Anthropic Blog RSS",
  "author": "Author name (if available)",
  "language": "EN"
}
```

### C3. AI Triage

The triage step uses a lightweight model (Claude Haiku or Gemini Flash via OpenRouter) to:

1. **Score relevance** (1-5) against T-0's focus areas:
   - AI agents and automation (core business)
   - MCP and tool-use protocols
   - Enterprise AI adoption
   - German/DACH market AI developments
   - AI consulting and services
   - Infrastructure and deployment patterns

2. **Assign Category Tags** from the shared taxonomy

3. **Assign AI-Tags** for specific topics

4. **Generate a 2-3 sentence Summary**

5. **Gate**: Items scoring >= 3 get written to Notes DB. Items scoring 1-2 are discarded (not stored anywhere — they never existed as far as the system is concerned).

**Triage prompt** will be stored in this repo at `prompts/triage-prompt.md` (already exists from spec 002, needs update for source ingestion context).

### C4. Deduplication

Before writing to Notes DB:

1. **Exact URL match**: Query Notes DB for `URL = item.url`. Skip if found.
2. **Fuzzy title match**: If no URL match, check for similar titles among items from the last 30 days. Use normalized string comparison (lowercase, strip punctuation, compare first 80 chars). If > 85% match, skip and optionally add as "Related Content" on existing Note.
3. **Cross-source dedup**: Same story from 5 different outlets → 1 Note entry with multiple source URLs mentioned in body.

### C5. Schedules

| Trigger | Frequency | Sources | Rationale |
|---------|-----------|---------|-----------|
| RSS batch | Weekly, Sunday 08:00 CET | All RSS/Atom sources | Weekly is sufficient — AI news has a ~1 week relevance window |
| Slack capture | Every 6 hours | #tech-shit-talk | Conversations happen throughout the day |
| Google Alerts | Daily, 07:00 CET | All alert feeds | Alerts aggregate daily |
| Manual webhook | On-demand | Any/all | For testing and ad-hoc runs |

**Webhook**: `POST https://n8n.t-0.co/webhook/t0-source-ingestion`
- Body: `{ "mode": "all" | "rss" | "slack" | "alerts", "source_id": "optional-specific-source" }`

---

## Part D: WF-3 — Intelligence Digest Workflow

### D1. Purpose

Periodically aggregate Notes DB items into a structured intelligence report in the Reports DB. This is the step between "we collected 30 items this week" and "here are the 3-5 topics worth writing about."

### D2. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  WF-3: Intelligence Digest                               │
│                                                          │
│  TRIGGER: Weekly, Monday 09:00 CET                       │
│  (+ manual webhook for testing)                          │
│                                                          │
│  1. Query Notes DB                                       │
│     Filter: Status = "To Be Processed"                   │
│             OR (Status = "Backlog" AND Origin in          │
│             [rss-ingestion, google-alert, ...])           │
│     Sort: Time desc                                      │
│                                                          │
│  2. Cluster items by topic                               │
│     AI groups related items into 3-8 topic clusters      │
│     Each cluster: title, summary, constituent items,     │
│     relevance assessment, content angle suggestion        │
│                                                          │
│  3. Create Report in Reports DB                          │
│     Name: "Content Intelligence — KW {week} {year}"      │
│     Time: period covering the items                      │
│     Tags: "Content Intelligence", "Weekly Digest"        │
│     Notes: link all constituent Notes                    │
│     Body: structured digest with clusters                │
│                                                          │
│  4. Update Notes                                         │
│     Status: "To Be Processed" → "Currently Relevant"     │
│     (for items included in report)                       │
│                                                          │
│  5. Notification                                         │
│     Post summary to Slack or Notion comment              │
│     "Weekly digest ready: {N} items, {M} topic clusters" │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### D3. Report Body Structure

Each weekly report page contains:

```markdown
## Summary
{1-2 paragraph executive summary of the week's signals}

## Topic Clusters

### 1. {Cluster Title} (N items)
**Relevance**: {Why this matters for T-0}
**Content Angle**: {Suggested blog post angle}
**Items**:
- {Item title} — {Source name} — {Date}
- {Item title} — {Source name} — {Date}

### 2. {Cluster Title} (N items)
...

## Signals to Watch
{Items that don't cluster but are individually noteworthy}

## Recommended Actions
- [ ] Write post about {cluster 1} — angle: {suggestion}
- [ ] Write post about {cluster 3} — angle: {suggestion}
- [ ] Monitor {signal} for next week
```

### D4. From Report to Content

When Aaron reads the digest and decides to act on a recommendation:

1. Create a new Content Creation entry (or promote an existing Idea)
2. Link it to the Report via the `Report` relation
3. Link relevant Notes via the `Notes` relation
4. Link relevant Sources via the `Sources` relation
5. Press "Run Pipeline" → WF-1 handles the rest

This step remains manual and intentional — the pipeline suggests, the human decides.

---

## Part E: Implementation Phases

### Phase 1: Database Setup
- Create Content Sources DB
- Modify Notes DB (rename AI-Category, add Source relation, add Origin options)
- Modify Reports DB (add Content relation, add Time property, add tag options)
- Modify Content Creation DB (add Report relation, add Sources relation)
- Partially requires Playwright for property rename

### Phase 2: Source Catalogue Population
- Verify RSS feed URLs for all Tier 1 sources
- Create entries in Content Sources DB
- Configure Google Alerts on T-0 Workspace account
- Test each source's extraction method

### Phase 3: WF-2 Core (RSS Branch)
- Build WF-2 skeleton on n8n.t-0.co
- Implement Phase 1 (fetch active sources)
- Implement Branch A (RSS/Atom extraction)
- Implement Phase 3 (dedup + triage + Notes write)
- Implement Phase 4 (error handling)
- E2E test with 3-5 RSS sources

### Phase 4: WF-2 Extensions
- Implement Branch B (Slack capture)
- Implement Branch C (Jina for non-RSS sources)
- Implement Branch D (Google Alerts)
- E2E test all branches

### Phase 5: WF-3 Intelligence Digest
- Build WF-3 on n8n.t-0.co
- Implement topic clustering
- Implement report generation
- E2E test with real accumulated items

### Phase 6: Dashboard Integration
- Add Source Health view to Content Pipeline Dashboard
- Add Intelligence Digest view
- Update Content Production project page

---

## Out of Scope

- **Ratchet automation** (spec TBD — needs 20+ drafts)
- **Newsletter email parsing** (Phase 2 extension — requires Gmail API integration)
- **GitHub trending** (Phase 2 extension — needs GitHub API)
- **ArXiv ingestion** (Phase 2 extension — needs ArXiv API)
- **Automatic Content Creation entry generation** from reports — the human decides what to write
- **Slack posting** of digests (future — currently Notion-only)

## Acceptance Criteria

1. Content Sources DB exists with at least 10 active sources
2. WF-2 runs on schedule and writes items to Notes DB with correct Origin, Source relation, Category Tags
3. Deduplication prevents duplicate Notes entries for the same URL
4. AI triage filters out irrelevant items (only >= 3 relevance score stored)
5. Source errors are isolated (one failing source doesn't block others)
6. WF-3 produces a weekly intelligence digest in Reports DB linking constituent Notes
7. Report body contains topic clusters with content angle suggestions
8. All new relations are dual (bidirectional)
9. Category Tags taxonomy is consistent across Sources, Notes, and Content Creation DBs
10. Dashboard shows source health and recent digests
