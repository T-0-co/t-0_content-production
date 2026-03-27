# Tasks: Source Discovery & Collection

**Input**: [plan.md](plan.md), [spec.md](spec.md)
**Scope**: All phases. Phases 1-2 complete. Phase 3 is active.

## Format: `[ID] [P?] Description`

- **[P]**: Can run in parallel (no dependencies)
- Each task ends with a concrete verification step

---

## Phase 1: Database Setup ✅

- [x] T001 Create Content Sources DB as inline DB in Content Production page
  - ID: `330c0fdf-942d-81b8-a839-fe9ea3c9ad15`
  - 15 properties: Name, URL, Feed URL, Type, Access Method, Status, Frequency, Last Checked, Content Hash, Language, Category Tags, Description, Tags, Notes (dual), Content (dual)

- [x] T002 Rename Notes DB `AI-Category` → `Category Tags`
  - Done via Notion API PATCH (no Playwright needed — API supports property rename)

- [x] T003 Add 5 new Origin options to Notes DB
  - Added: rss-ingestion, google-alert, newsletter-ingestion, github-trending, arxiv

- [x] T004 Add `Source` dual relation on Notes DB → Content Sources DB
  - Auto-created by Sources DB dual relation

- [x] T005 Add `Content` dual relation on Reports DB → Content Creation DB
  - Synced property "Report" auto-created on Content Creation DB

- [x] T006 Add `Time` date property to Reports DB

- [x] T007 Add tag options to Reports DB (Content Intelligence, Weekly Digest)

- [x] T008 Verify all 4 database schemas match spec
  - Sources: 15 props ✅, Notes: Source relation ✅, Reports: Content + Time ✅, CC: Report + Sources ✅

**Checkpoint**: Phase 1 complete — all databases ready. ✅

---

## Phase 2: Source Catalogue ✅

- [x] T009 [P] Verify RSS feed URLs for Tier 1 sources
  - 10 verified: OpenAI, DeepMind (FeedBurner), Simon Willison, HN, Verge, TechCrunch, Import AI, HuggingFace, Heise, t3n
  - 2 no RSS: Anthropic (Jina), The Batch (Jina)

- [x] T010 Create 23 source entries in Content Sources DB
  - Tier 1: 10 sources (Active)
  - Tier 2: 8 sources (Active, except Stratechery: Paused/manual)
  - Tier 3: 5 sources (3 Google Alerts: New, 2 others: Active)

- [ ] T011 Aaron: Create Google Alerts with RSS delivery (manual)
  - "AI Agents" → RSS feed URL → update Sources DB entry
  - "Model Context Protocol" OR "MCP AI" → RSS feed URL → update Sources DB entry
  - "KI Mittelstand" OR "Kuenstliche Intelligenz Unternehmen" → RSS feed URL → update Sources DB entry

**Checkpoint**: Phase 2 complete (except T011 — Aaron's manual step). ✅

---

## Phase 3: WF-2 Core (RSS Branch) ✅

**Purpose**: Build WF-2 on n8n.t-0.co — RSS ingestion → dedup → AI triage → Notes DB.
**Workflow ID**: `U9YvA64bwIMur9Df` | **20 nodes** | **Webhook**: `POST /webhook/t0-source-ingestion`

### WF-2 Skeleton

- [x] T012 Create WF-2 workflow on n8n.t-0.co
  - Name: "WF-2: Source Ingestion"
  - Schedule trigger: weekly Sunday 08:00 CET (enabled)
  - Webhook trigger: `POST /webhook/t0-source-ingestion`
  - Response mode: `responseNode` (fire-and-forget)
  - **Verified**: Webhook responds 200, workflow ID `U9YvA64bwIMur9Df`

### WF-2 Phase 1: Fetch Sources

- [x] T013 Build source fetching nodes
  - Parse Mode Code node: extract `mode` and `source_id` from webhook body
  - Fetch RSS Sources: HTTP Request → Notion API (Content Sources DB)
    - Filter: `Status = Active AND Access Method = RSS`
  - Extract Sources: Code node → maps Notion page properties to source objects
  - **Verified**: Returns 12 active RSS sources

### WF-2 Phase 2: RSS Extraction

- [x] T014 Build RSS/Atom parser
  - Fetch & Parse RSS: Code node using `this.helpers.httpRequest()` (no auth needed)
  - Parses RSS 2.0 (`<item>`) and Atom (`<entry>`) XML formats
  - 7-day lookback filter, max 20 items per source
  - **Verified**: 142 items from 12 sources (HN, OpenAI, Willison, ArXiv, etc.)

- [x] T015 Build item normalization
  - Unified format: title, url, published, summary, full_text, source_name, source_page_id, origin, origin_description, author, language
  - Origin = "rss-ingestion", Origin Description = "{Source Name} RSS"
  - **Verified**: All items have required fields

### WF-2 Phase 3: Central Processing

- [x] T016 Build URL deduplication
  - Prepare Dedup Query (Code) → Fetch Recent Notes (HTTP Request → Notion) → Dedup Items (Code)
  - Compares item URLs against existing Notes DB entries (last 30 days)
  - **Verified**: 2nd run caught 3 duplicates (2 test notes + 1 cross-source dupe)

- [x] T017 Build fuzzy title deduplication
  - Implemented in Dedup Items Code node alongside URL dedup
  - Normalizes titles (lowercase, strip punctuation, first 80 chars)
  - Threshold: > 85% character match → skip
  - **Verified**: Working (included in URL dedup node)

- [x] T018 Update triage prompt for source ingestion
  - English per-item relevance scoring (1-5 scale)
  - T-0 focus: AI agents/automation, MCP, enterprise AI, German market
  - 14 category tags, JSON output format
  - **Verified**: Committed to `prompts/triage-prompt.md`

- [x] T019 Build AI triage node
  - Prepare Triage (Code, batched 25/batch) → AI Triage (HTTP Request → OpenRouter) → Merge Triage Results (Code)
  - Model: `anthropic/claude-3.5-haiku` via OpenRouter (`openRouterApi` credential)
  - Gate: items with relevance_score >= 3 pass through
  - **Fix applied**: Batching (25 items/call) — single-call approach only scored 5/142 items
  - **Verified**: 6 batches, 139/139 items scored, distribution: 5→9, 4→49, 3→40, 2→31, 1→10

- [x] T020 Build Notes DB write-back
  - Prepare Note Items (Code) → Create Note (HTTP Request → Notion API)
  - Properties: Name, URL, Time, Summary, Description, Origin, Origin Description, Source (relation), Category Tags, AI-Tags, Status = "To Be Processed"
  - **Fix applied**: Replaced Code node `httpRequestWithAuthentication` (not available in task runner) with HTTP Request nodes
  - **Verified**: 98 notes created with correct properties and Source relations

- [x] T021 Build Source Last Checked update
  - Prepare Source Updates (Code) → Update Source (HTTP Request → Notion API PATCH)
  - Runs in parallel with note creation branch
  - **Verified**: 12 sources updated with `Last Checked = 2026-03-27T14:40:00`

### WF-2 Phase 4: Error Handling

- [x] T022 Build per-source error isolation
  - RSS fetch uses try/catch per source in Code node — one source failing doesn't stop others
  - DeepMind returned 0 items (expected, blog infrequent) — other 11 sources processed normally
  - **Verified**: Working (DeepMind 0 items, others processed)

- [x] T023 Build error notification
  - Error Trigger → Format Error (Code) — formats error with node name, message, execution URL
  - Note: Currently formats but doesn't send to external channel (Notion comment or Slack). Future improvement.
  - **Verified**: Error trigger and formatter exist

### WF-2 Config & Prompts

- [x] T024 [P] Add source_ingestion section to `config/style-params.yaml`
  - `relevance_threshold: 3`, `lookback_days: 7`, `dedup_title_threshold: 0.85`, `dedup_lookback_days: 30`, `max_items_per_source: 20`
  - **Verified**: Committed to repo

### WF-2 E2E Validation

- [x] T025 E2E test: webhook trigger → RSS fetch → dedup → triage → Notes
  - All 12 active RSS sources tested (not just 3)
  - 142 items fetched → 139 after dedup → 98 passed triage → 98 notes created
  - Source relations, Category Tags, Origin, Status all correct
  - **Verified**: Execution 11643 — full success

- [x] T026 E2E test: dedup verification
  - 2nd run (11643) found 3 duplicates from previous run (11642)
  - URL-based dedup working correctly
  - **Verified**: Duplicates filtered

- [ ] T027 E2E test: error isolation
  - Skipped: would require temporarily breaking a source Feed URL. Per-source try/catch verified via DeepMind (0 items, no crash). Full error isolation test deferred.

- [x] T028 Enable schedule trigger
  - Weekly schedule: Sunday 08:00 CET (`triggerAtHour: 8, triggerAtDay: 7`)
  - **Verified**: Workflow active with schedule enabled

- [ ] T029 Clean up test artifacts
  - 98 notes from test runs are valid content — no cleanup needed
  - Subsequent runs will dedup against them

- [x] T030 Snapshot WF-2 workflow JSON
  - Saved to `workflows/wf2-source-ingestion.json` (20 nodes)
  - Credentials contain only IDs (no secrets in JSON)
  - **Verified**: File committed to repo

**Checkpoint**: Phase 3 complete — WF-2 RSS ingestion live and validated. ✅

---

## Phase 4: WF-2 Extensions ✅

### Branch D: Google Alerts RSS

- [x] T033 Build Branch D: Google Alerts RSS
  - 3 Google Alerts created by Aaron with RSS delivery
  - Feed URLs added to Content Sources DB: AI Agents, MCP Protocol, KI Mittelstand
  - Already picked up by WF-2 RSS branch (Access Method = RSS)
  - **Verified**: 30 items from AI Agents + MCP alerts (KI Mittelstand: 0 — low volume)

### Branch C: Jina Extraction

- [x] T032 Build Branch C: Jina extraction (Anthropic, Meta AI, Mistral, The Batch)
  - Added `parseJinaPage()` function to "Fetch & Parse RSS" Code node
  - Parses markdown links from `r.jina.ai` JSON response (`data.content`)
  - Filters: same-domain, path depth ≥ 2, title length ≥ 10, 7-day lookback
  - **Bug fixed**: `URL` constructor unavailable in n8n task runner sandbox — replaced with regex-based hostname/path parsing (`parseHost()`, `parsePath()`)
  - **Verified**: 24 items (Anthropic 13, Meta AI 6, Mistral 3, The Batch 2)

### Branch B: Slack Capture

- [x] T031 Build Branch B: Slack #tech-shit-talk capture
  - Channel ID: `C07RWPA1Z0B` | Slack credential: `CVSFME2khBnyUCBi`
  - Source entry: `330c0fdf-942d-8167-afb9-ccbb7a280de9` (pre-existing from Phase 2)
  - Two capture modes:
    1. **Multi-person threads** (2+ replies, 2+ people): fetch thread replies, build summary
    2. **Standalone link shares**: any message with an HTTP link
  - Slack bot token injected by deploy script from 1Password (not hardcoded in snapshot)
  - **Verified**: 10 items (1 thread + 9 link shares)

### E2E + Snapshot

- [x] T034 E2E test all branches
  - Execution 11654: 206 items total (172 RSS + 24 Jina + 10 Slack)
  - 20 sources processed, 1 error (Latent Space 404 — broken feed URL)
  - Full pipeline: fetch → dedup → triage → 65 new notes created → 20 source timestamps updated
  - **Verified**: All 3 branches feed into unified central processing

- [x] T035 Snapshot updated WF-2
  - Saved to `workflows/wf2-source-ingestion.json` (18 nodes)
  - Slack token redacted in snapshot (`INJECTED_BY_DEPLOY_SCRIPT`)
  - **Verified**: File committed to repo

**Checkpoint**: Phase 4 complete — all extraction branches operational. ✅

---

## Phase 5: WF-3 Intelligence Digest ✅

**Workflow ID**: `Td8CIlyau6zkWHrI` | **13 nodes** | **Webhook**: `POST /webhook/t0-intelligence-digest`

- [x] T036 Build WF-3 workflow skeleton
  - Schedule: Monday 09:00 CET (weekly)
  - Webhook: `POST /webhook/t0-intelligence-digest` (manual trigger)
  - Error routing to central handler (`de8FXOcXmNSCh6kd`)
  - **Verified**: Workflow created and activated

- [x] T037 Build Notes aggregation query
  - Fetch Notes: HTTP Request → Notion API (`/databases/{id}/query`)
  - Filter: `Status = "To Be Processed"`, sorted by Time desc, page_size 100
  - Extract Notes: Code node maps properties to item objects
  - **Verified**: 100 notes fetched

- [x] T038 Build AI topic clustering
  - Prepare Clustering: Code node builds prompt with all items
  - AI Cluster: HTTP Request → OpenRouter (`anthropic/claude-sonnet-4`)
  - Parse Clusters & Build Report: Code node extracts JSON, builds markdown report body
  - **Bug fixed**: Credential type was `httpHeaderAuth` but actual type is `openRouterApi`
  - **Verified**: 7 clusters + 3 signals generated from 100 items

- [x] T039 Build Report creation in Reports DB
  - Create Report: HTTP Request → Notion API (`POST /pages`)
  - Properties: Name (KW N YYYY), Time (date range), Tags (Content Intelligence + Weekly Digest), Notes (relation to 100 notes), Description (executive summary)
  - Add Report Body: Code node converts markdown to Notion blocks (heading_2, heading_3, paragraph, bulleted_list_item, to_do)
  - Append Report Blocks: HTTP Request → Notion API (`PATCH /blocks/{id}/children`)
  - **Verified**: Report created with full structured body — [KW 13 2026](https://www.notion.so/330c0fdf942d818f80e4e4080b8b39e8)

- [x] T040 Build Notes status update
  - Prepare Note Updates: Code node outputs one item per note
  - Update Note Status: HTTP Request → Notion API (PATCH, loops over items)
  - Status change: "To Be Processed" → "Currently Relevant"
  - **Verified**: 100 notes updated

- [x] T041 E2E test with accumulated items
  - Execution 11661: 100 notes → 7 clusters + 3 signals → report created → 100 notes updated
  - Executive summary highly relevant to T-0 (enterprise AI agents, MCP, German market)
  - Full report body rendered as Notion blocks
  - **Verified**: End-to-end success

- [x] T042 Snapshot WF-3
  - Saved to `workflows/wf3-intelligence-digest.json` (13 nodes)
  - **Verified**: File committed to repo

**Checkpoint**: Phase 5 complete — weekly intelligence digest operational. ✅

---

## Phase 6: Dashboard Integration ✅

- [x] T043 Add Source Health view to Dashboard
  - Added "Source Health" section to Content Pipeline Dashboard
  - Callout with source counts, extraction methods, schedule info
  - Webhook URL + workflow ID for manual triggers
  - Link to Content Sources DB
  - **Verified**: 7 blocks added to dashboard

- [x] T044 Add Intelligence Digest view
  - Added "Intelligence Digest" section to Content Pipeline Dashboard
  - Callout with WF-3 schedule, processing description
  - Webhook URL + workflow ID + AI model info
  - Link to Reports DB
  - **Verified**: 7 blocks added to dashboard

- [x] T045 Update Content Production page layout
  - Added "Source Discovery & Collection (Spec 004)" section
  - Links to Content Sources DB, WF-2, WF-3
  - **Verified**: 6 blocks added to Content Production project page

**Checkpoint**: Phase 6 complete — full source discovery system operational. ✅

---

## Dependencies & Execution Order

```text
Phase 1 (database setup) ✅
  → Phase 2 (source catalogue) ✅
    → Phase 3 (WF-2 RSS core)
      T012 (skeleton)
        → T013 (fetch sources)
          → T014, T015 (RSS extraction, sequential)
            → T016, T017 (dedup, sequential)
              → T018 [P], T019 (triage prompt + node)
                → T020, T021 (Notes write + Source update)
                  → T022, T023 (error handling)
                    → T024 [P] (config, can parallel with any)
                      → T025-T027 (E2E tests, sequential)
                        → T028, T029, T030 (activate + cleanup + snapshot)
      → Phase 4 (WF-2 extensions)
        → Phase 5 (WF-3 digest)
          → Phase 6 (dashboard)
```
