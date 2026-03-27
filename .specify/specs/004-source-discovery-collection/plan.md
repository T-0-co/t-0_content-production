# Plan: Source Discovery & Collection

**Input**: [spec.md](spec.md)
**Scope**: All 6 phases. Phases 1-5 complete. Phase 6 (Dashboard) is the next build target.

## Phase Summary

| Phase | Status | What |
|-------|--------|------|
| 1. Database Setup | ✅ Complete | Sources DB created, Notes/Reports/CC modified, all dual relations |
| 2. Source Catalogue | ✅ Complete | 23 sources populated (10 T1, 8 T2, 5 T3 incl. 3 Google Alerts) |
| 3. WF-2 Core | ✅ Complete | RSS ingestion → central processing → Notes DB (20 nodes, workflow `U9YvA64bwIMur9Df`) |
| 4. WF-2 Extensions | ✅ Complete | Slack (10 items), Jina (24 items), Google Alerts (30 items) — all 3 branches live |
| 5. WF-3 Digest | ✅ Complete | Weekly intelligence digest (13 nodes, workflow `Td8CIlyau6zkWHrI`) |
| 6. Dashboard | ✅ Complete | Source Health + Intelligence Digest views on Dashboard + Content Production page |

---

## Phase 3: WF-2 Core — Detailed Plan

### Architecture Decision: Single-Schedule RSS Workflow

WF-2 starts as an RSS-only workflow. Schedule trigger fires weekly (Sunday 08:00 CET). A manual webhook trigger also exists for testing.

**Node flow** (estimated ~25-30 nodes):

```
TRIGGERS
├── Schedule Trigger (Weekly, Sunday 08:00 CET)
└── Webhook Trigger (POST /webhook/t0-source-ingestion)
    Body: { "mode": "all"|"rss", "source_id": "optional" }

PHASE 1: FETCH SOURCES
├── Merge Triggers (combine both trigger paths)
├── Parse Mode (extract mode + optional source_id from webhook)
├── Fetch Active RSS Sources
│   Query Sources DB: Status=Active AND Access Method=RSS
│   (or specific source_id if provided)
└── Loop Setup (iterate over sources)

PHASE 2: RSS EXTRACTION (per source)
├── HTTP GET Feed URL (from source entry)
├── Parse RSS/Atom XML
│   Code node: regex-based parser (Josepha pattern)
│   Extract: title, link, pubDate, description, author
├── Filter Recent Items
│   Only items from last 7 days (configurable)
└── Normalize to Unified Format
    Map RSS fields → { title, url, published, summary, source_name,
    source_page_id, origin: "rss-ingestion", language }

PHASE 3: CENTRAL PROCESSING
├── Collect All Items (from loop iterations)
├── Dedup: URL Check
│   Query Notes DB for each item.url
│   Skip items with existing URL match
├── Dedup: Fuzzy Title Check
│   Normalized title comparison (first 80 chars, lowercase)
│   Skip items > 85% match to recent Notes
├── AI Triage
│   Model: Claude Haiku via OpenRouter (cheapest viable)
│   Input: title + summary + source context
│   Output: relevance_score (1-5), category_tags, ai_tags, summary
│   Gate: only items scoring >= 3 proceed
├── Write to Notes DB (batch)
│   Properties: Name, URL, Time, Summary, Description, Origin,
│   Origin Description, Source (relation), Category Tags, AI-Tags,
│   Status = "To Be Processed"
│   Body: full article text in page content (if available)
└── Update Source: Last Checked = now()

PHASE 4: ERROR HANDLING
├── Error Trigger → Format Error → Post to Source page as comment
└── (Retry logic: source marked Paused after 3 consecutive failures — Phase 4 scope)
```

### Key Design Decisions

1. **RSS parser**: Use a Code node with regex-based XML parsing (no npm dependency). The Josepha pipeline reference (`docs/sources/josepha-pipeline-reference.md`) has a working pattern. Both RSS 2.0 and Atom formats must be handled.

2. **Loop strategy**: Use n8n's SplitInBatches or Code-node-driven loop to process each source independently. This gives per-source error isolation — one feed failing doesn't stop others.

3. **Dedup approach**: Query Notes DB with a filter for exact URL match first (cheap). Fuzzy title match is a Code node comparison (no external service). The 30-day window keeps the comparison set manageable.

4. **AI triage model**: Claude Haiku via OpenRouter. Fast, cheap (~$0.001/item), sufficient for relevance scoring. Uses existing OpenRouter credential on n8n.t-0.co.

5. **Batch vs individual Notes creation**: Create Notes one at a time (not batch) because each needs a unique Source relation and individual page body content. But the Source `Last Checked` update can batch.

6. **Webhook response**: Use `responseMode: responseNode` (same pattern as WF-1) to return immediately and process in background.

### n8n Credentials Required

All exist on n8n.t-0.co (verified in spec 002):

| Credential | ID | Purpose |
|---|---|---|
| Aarons Notion account - T-0 Scope | `0xiiQaGDhyAvgSeB` | Notes DB reads/writes, Sources DB queries |
| Aarons OpenRouter account | `zxXiCGlCYD4JGQDr` | AI triage (Claude Haiku) |

### Prompts

- `prompts/triage-prompt.md` — Already exists, needs update for source ingestion context
- New: triage should receive the source's Category Tags as context hints

### Configuration

- `config/style-params.yaml` — Add `source_ingestion` section:
  - `relevance_threshold: 3` (gate score)
  - `lookback_days: 7` (RSS item age filter)
  - `dedup_title_threshold: 0.85` (fuzzy match cutoff)
  - `dedup_lookback_days: 30` (title match window)

### Testing Strategy

1. Build with webhook trigger first (manual testing)
2. Test with 3 high-quality RSS sources: Simon Willison, OpenAI, HuggingFace
3. Verify: items appear in Notes DB with correct properties and Source relation
4. Verify: dedup works (re-run should create 0 new items)
5. Verify: AI triage filters correctly (check a few discarded items manually)
6. Add schedule trigger last (after E2E validation)
7. Clean up test items in Notes DB (archive test entries)

### Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| RSS XML parsing edge cases | Test with 5+ different feed formats before deployment |
| OpenRouter rate limits | Process items sequentially, not in parallel |
| Large feeds (ArXiv, HN) | `lookback_days` filter + item count cap (max 20 per source per run) |
| Notes DB pollution during testing | Use a test tag to identify and clean up test items |

---

## Phase 4-6: Brief Plans (to be detailed when Phase 3 completes)

### Phase 4: WF-2 Extensions
- Branch B (Slack): Fetch from Slack API → filter engaged threads → normalize → central processing
- Branch C (Jina): For Anthropic, Meta AI, Mistral blogs → content hash comparison → normalize
- Branch D (Google Alerts): Same as RSS branch but with Google Alerts feed URLs
- Each branch reuses the central processing pipeline from Phase 3

### Phase 5: WF-3 Intelligence Digest
- Weekly schedule (Monday 09:00 CET)
- Query Notes DB (Status = "To Be Processed")
- AI clustering with Sonnet (heavier model needed for synthesis)
- Create Report entry with structured body
- Update constituent Notes status

### Phase 6: Dashboard Integration
- Add "Source Health" view to Content Pipeline Dashboard
- Add "Recent Digests" view
- Link from Dashboard to Sources DB and Reports DB
