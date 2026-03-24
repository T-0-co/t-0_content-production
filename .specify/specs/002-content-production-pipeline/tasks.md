# Tasks: T-0 Content Production Pipeline

**Input**: [plan.md](plan.md), [002-content-production-pipeline.md](../002-content-production-pipeline.md)
**Scope**: Phase A (core pipeline) — P0. Phase D (Ghost tag overhaul) — P1, parallel.

**Organization**: Tasks grouped by build phase from plan.md. Phase A is sequential (each step builds on the previous). Phase D runs independently.

## Format: `[ID] [P?] Description`

- **[P]**: Can run in parallel (different systems, no dependencies)
- Exact file paths, API endpoints, and credential names included
- Each task ends with a concrete verification step

---

## Phase A: Prerequisites

**Purpose**: External system setup that must be complete before building WF-1.

### Credentials (on n8n.t-0.co)

- [x] T001 Create OpenRouter credential on n8n.t-0.co
  - Credential: `Aarons OpenRouter account` (id: `zxXiCGlCYD4JGQDr`)

- [x] T002 [P] Create Ghost "Content Pipeline" integration in Ghost Admin
  - Integration created via SQLite on T-0 server
  - Admin key: `T0_GHOST_ADMIN_KEY` in workspace `.env`
  - Content key: `T0_GHOST_CONTENT_KEY` in workspace `.env`

- [x] T003 [P] Verify Notion integration token on n8n.t-0.co
  - Credential: `Aarons Notion account - T-0 Scope` (id: `0xiiQaGDhyAvgSeB`)

- [x] T004 [P] Verify Jina API credential on n8n.t-0.co
  - Credential: `T-0 Jina AI` (id: `F6dbtFahezakDlYP`)

- [x] T005 [P] Verify Perplexity API credential on n8n.t-0.co
  - Credential: `Aarons Perplexity account` (id: `O2sFk40YgiNpnnKu`)

**Checkpoint**: All 5 credentials working on n8n.t-0.co. ✅

---

### Notion DB Setup

- [x] T006 Add 15 API-creatable properties to Content Creation DB
  - All scoring properties, Pipeline Version, Evaluated At, etc. confirmed present

- [x] T007 Verify/add Status values on Content Creation DB (Playwright MCP)
  - All 7 states confirmed: Idea → Backlog → Draft → Pre Review → In Revision → Publishable → Published

- [x] T008 Add "Run Pipeline" button to Content Creation DB (Playwright MCP)
  - Button property added via Playwright

- [x] T009 Set up Notion automation: button → webhook
  - Webhook URL: `https://n8n.t-0.co/webhook/t0-content-pipeline`

**Checkpoint**: Content Creation DB has all new properties, status values, button, and automation. ✅

---

### Prompt Templates & Config (this repo)

- [x] T010 [P] Create `config/style-params.yaml`
- [x] T011 [P] Create `prompts/draft-system-prompt.md`
- [x] T012 [P] Create `prompts/evaluation-prompt.md`
- [x] T013 [P] Create `prompts/research-prompt.md`
- [x] T014 [P] Create `prompts/image-description-prompt.md`
- [x] T015 [P] Create `prompts/triage-prompt.md`
- [x] T016 Commit all prompts and config to repo

**Checkpoint**: All prompt templates and style params committed. Accessible via GitHub raw URLs. ✅

---

## Phase A: WF-1 Build (on n8n.t-0.co)

**Purpose**: Build the core content pipeline workflow, branch by branch.

### WF-1 Skeleton

- [x] T017 Create WF-1 skeleton: Webhook → Parse → Switch → Respond
  - Workflow ID: `dJlw6US8gj6Do4Wf` on n8n.t-0.co
  - Webhook: `POST /webhook/t0-content-pipeline`
  - Response mode: `responseNode` (fire-and-forget)

### WF-1 Branch A: Research → Draft → Evaluate

- [x] T018 Build Branch A parallel context gathering
  - Brand context (3 Notion pages), pipeline config (GitHub raw), merged

- [x] T019 Build Branch A research agent
  - AI Agent with OpenRouter (claude-sonnet-4-6), Perplexity tool

- [x] T020 Build Branch A content type detection + draft prompt assembly

- [x] T021 Build Branch A draft generation agent

- [x] T022 Build Branch A draft parsing + Notion write-back
  - Markdown → Notion blocks converter, batched append

- [x] T023 Build Branch A inline evaluation
  - 7-dimension scoring, gate rules, score write-back, evaluation comment

### WF-1 Branch C: Ghost Publication

- [x] T024 Build Branch C Ghost publication
  - 6 nodes: Fetch Full Page → Fetch Blocks → Build Ghost HTML → Create Ghost Post → Update Notion → Post Comment
  - Ghost JWT via `require('crypto')` (added `crypto` to `NODE_FUNCTION_ALLOW_BUILTIN` on T-0 n8n)
  - HTTP via `require('https')` (n8n sandbox blocks `fetch`)
  - Creates as draft (never auto-publishes), writes Ghost URL + ID back to Notion

### WF-1 Branch B: Revision

- [x] T025 Build Branch B revision flow
  - 9 nodes: Fetch Feedback → Fetch Draft → Fetch Writing Guide → Assemble Prompt → Archive & Redraft → Delete Old Blocks → Generate Revised Draft (AI Agent) → Parse & Write Revision
  - Sequential fetch chain (parallel was unreliable in n8n)
  - Archives old content in toggle, re-drafts with AI, sets Status → Draft

### WF-1 Error Handling

- [x] T026 Add error handling to WF-1
  - Error Trigger → Format Error Comment → Post Error Comment (Notion)
  - Added in spec 003 (D19)

**Checkpoint**: WF-1 all branches complete (50 nodes total, 52 after Phase E hardening). ✅

---

## Phase A: End-to-End Validation

- [x] T027 End-to-end test: Idea/Backlog entry through Branch A
  - Execution 11104 (2026-03-24): "MCP Hub – Artikel (Kurzfassung)" → 32 nodes, 4.5 min
  - Research (Perplexity) + Draft + Evaluation (29.5/40, blocked on Accuracy/Credibility)
  - Status: Idea → Draft, all scores written, evaluation comment posted

- [x] T028 End-to-end test: In Revision entry through Branch B
  - Execution 11127 (2026-03-24): Same article with human feedback → 14 nodes, 17s
  - Feedback incorporated, old content archived in toggle, revised draft written
  - Status: In Revision → Draft

- [x] T029 End-to-end test: Publishable entry through Branch C
  - Execution 11116 (2026-03-24): Same article → 11 nodes, 13s
  - Ghost post created as draft (slug: `mcp-hub-artikel-kurzfassung-2`)
  - Ghost URL + ID written to Notion, publication comment posted
  - Status: Publishable → Published

**Checkpoint**: Phase A complete — all 3 E2E tests pass. ✅

---

## Phase E: Production Hardening (post-E2E audit)

**Purpose**: Fix all issues discovered during deep audit of deployed pipeline. 52 nodes after hardening.

### Bug Fixes

- [x] T034 Fix error handler credential (wrong Notion credential ID)
  - Changed `Post Error Comment` from `t0NotionCred` to `0xiiQaGDhyAvgSeB`

- [x] T035 Fix Branch B dead end (revision had no re-evaluation)
  - Added `B: Build Re-Eval Prompt` node between `B: Parse & Write Revision` → `Evaluate Draft`
  - Branch B revisions now flow through the same evaluation chain as Branch A

- [x] T036 Fix error handler pageId extraction
  - Error Trigger doesn't include `runData` from failed execution
  - `require('https')` fails silently inside Code node sandbox
  - Solution: Added `Fetch Failed Execution` HTTP Request node to call n8n API `GET /executions/{id}?includeData=true`
  - `Format Error Comment` now parses upstream HTTP response (no outbound calls from Code node)
  - Verified: exec 11154 correctly extracted pageId and built error comment

- [x] T037 Fix Notion 2000-char content limit in Branch B
  - `B: Parse & Write Revision` now splits any `rich_text[].text.content` > 2000 chars into multiple segments
  - Preserves links and annotations across chunks

- [x] T038 Fix Branch B markdown parser (inline formatting)
  - Added `parseInlineFormatting()` for **bold**, *italic*, `code`, [links](url), code fences
  - Previous parser only handled block-level structure

### Enhancements

- [x] T039 [P] Add Notion block pagination (>100 blocks)
  - `C: Fetch Page Blocks` and `B: Fetch Current Draft` converted from HTTP Request to paginated Code nodes
  - Loop with `has_more`/`next_cursor`, aggregates all blocks

- [x] T040 [P] Add Category Tags → Ghost tag mapping
  - `C: Build Ghost HTML` maps Notion `categoryTags` multi-select to Ghost tags

- [x] T041 [P] Add Ghost newsletter, SEO, and author fields
  - `C: Create Ghost Post` now includes `newsletter: {slug: 'default-newsletter'}`, `meta_title`, `meta_description`

- [x] T042 [P] Fix CET/CEST timezone for Ghost scheduling
  - Dynamic offset: March-October → CEST (+02:00), otherwise CET (+01:00)

- [x] T043 [P] Add duplicate Ghost post guard
  - `C: Create Ghost Post` checks for existing `ghostId` in Notion properties before creating
  - Prevents duplicate posts on re-runs

- [x] T044 Redact secrets from workflow snapshot in git
  - Notion token, Ghost key ID, Ghost secret → `REDACTED_*` in `docs/wf1-snapshot-2026-03-24.json`

### Validation (post-hardening)

- [x] T045 Re-test Branch A (Backlog → Draft)
  - Execution 11135 (2026-03-24): 33 nodes, 233s, score 33/40 Good ✅

- [x] T046 Re-test Branch B (In Revision → Draft + re-evaluation)
  - Execution 11138 (2026-03-24): 22 nodes, 31s, full re-evaluation chain ✅
  - Note: scored 8/40 because test page had no real content to revise (data cascade from prior failure)

- [x] T047 Re-test error handler
  - Execution 11154 (2026-03-24): Fetch Failed Execution → Format Error Comment → Post Error Comment
  - pageId correctly extracted from n8n API response ✅
  - Comment build verified (POST failed only because test used fake page ID)

- [x] T048 Verify Notion button automation
  - Playwright: "Run Pipeline" button property confirmed with "Send webhook" action
  - URL: `https://n8n.t-0.co/webhook/t0-content-pipeline`, all properties included in payload

- [x] T049 Clean up test artifacts
  - Ghost: no test posts or duplicate tags found (clean)
  - Notion: test page `32dc0fdf-942d-81be-95bd-d496be95d9db` archived (trashed)

**Checkpoint**: Phase E complete — 52 nodes, all branches and error handling verified. ✅

---

## Phase D: Ghost Tag Overhaul (parallel with Phase A)

- [x] T030 [P] Export current Ghost tags and map to new taxonomy
  - 64 tags exported, taxonomy mapping designed (consolidate duplicates, normalize case, German-first)

- [x] T031 [P] Create new Ghost tags per taxonomy
  - Canonical tags: Agenten, MCP, MCP Hub, Automation, Strategie, Digitaler Wandel, Opinion, Dev, How-To, etc.
  - Internal: #pipeline-generated, #case-study, #opinion, #resource, #technical, #announcement

- [x] T032 Update existing Ghost posts with new tags
  - 3 posts remapped: Strategy→Strategie, KI-Agenten→Agenten, Agents→Agenten, How-to→How-To

- [x] T033 Delete orphaned/duplicate Ghost tags
  - 64 → 29 tags (35 deleted): all case duplicates, -2 slug variants, import artifacts, unused tech tags

**Checkpoint**: Ghost tags cleaned up. New taxonomy in place. ✅

---

## Dependencies & Execution Order

### Phase A (sequential)

```text
T001-T005 (credentials, parallel among themselves) ✅
  → T006-T009 (Notion DB setup, mostly sequential) ✅
  → T010-T016 (prompts & config, parallel, then commit) ✅
  → T017 (WF-1 skeleton) ✅
  → T018-T023 (Branch A, sequential) ✅
  → T024 (Branch C) ✅
  → T025 (Branch B) ✅
  → T026 (error handling) ✅
  → T027-T029 (E2E validation) ✅
  → T034-T049 (production hardening) ✅
```

### Phase D (independent)

```text
T030-T033 ✅ — completed in parallel with Phase A E2E testing.
```

### Phase E (sequential, after Phase A)

```text
T034-T044 (bug fixes + enhancements) ✅
  → T045-T049 (re-validation) ✅
```
