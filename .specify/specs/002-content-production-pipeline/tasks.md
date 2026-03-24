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

- [ ] T001 Create OpenRouter credential on n8n.t-0.co
  - Type: `openRouterApi`
  - Key from workspace `.env` (`OPENROUTER_API_KEY`)
  - Verify: test API call to `anthropic/claude-sonnet-4-6` returns a response

- [ ] T002 [P] Create Ghost "Content Pipeline" integration in Ghost Admin
  - Ghost Admin → Settings → Integrations → Add custom integration
  - Name: "Content Pipeline"
  - Store API key in n8n.t-0.co credential management (generic credential)
  - Verify: JWT generated from key can call `GET /ghost/api/admin/posts/?limit=1`

- [ ] T003 [P] Verify Notion integration token on n8n.t-0.co
  - Credential type: `notionApi` or `httpHeaderAuth`
  - Must have access to Content Creation DB (`2afc0fdf-942d-813e-bafa-fe13acf0b35f`)
  - Verify: `POST /databases/{id}/query` returns results

- [ ] T004 [P] Verify Jina API credential on n8n.t-0.co
  - Type: `httpHeaderAuth` (Bearer)
  - Key from workspace `.env` (`JINA_API_KEY`)
  - Verify: `GET https://r.jina.ai/https://t-0.co` returns content

- [ ] T005 [P] Verify Perplexity API credential on n8n.t-0.co
  - Type: `httpHeaderAuth` (Bearer)
  - Key from workspace `.env` (`PERPLEXITY_API_KEY`)
  - Verify: test chat completion call to `sonar-pro` returns response

**Checkpoint**: All 5 credentials working on n8n.t-0.co.

---

### Notion DB Setup

- [ ] T006 Add 15 API-creatable properties to Content Creation DB
  - Database: `2afc0fdf-942d-813e-bafa-fe13acf0b35f`
  - Properties (all via Notion API):
    - `Content Type` (select: Opinion, Resource, Case Study, Technical, Announcement)
    - `Clarity Score` (number)
    - `Structure Score` (number)
    - `Accuracy Score` (number)
    - `Credibility Score` (number)
    - `Engagement Score` (number)
    - `Voice Score` (number)
    - `Actionability Score` (number)
    - `Composite Score` (number)
    - `Quality Tier` (select: Excellent, Good, Acceptable, Below Standard, Poor)
    - `Evaluation Notes` (rich_text)
    - `Pipeline Version` (rich_text)
    - `Evaluated At` (date)
    - `Ghost ID` (rich_text)
    - `Source Type` (select: RSS, Slack Rant, Manual, Mixed)
  - Verify: query DB, confirm all 15 properties appear in schema

- [ ] T007 Verify/add Status values on Content Creation DB (Playwright MCP)
  - Required states: Idea → Backlog → Draft → Pre Review → In Revision → Publishable → Published
  - Check existing values via Playwright (browse to DB in Notion)
  - Add any missing states
  - Verify: all 7 states visible in Status property config

- [ ] T008 Add "Run Pipeline" button to Content Creation DB (Playwright MCP)
  - Button property on Content Creation DB
  - Verify: button visible on each entry in Notion

- [ ] T009 Set up Notion automation: button → webhook
  - "Run Pipeline" button click → Send webhook
  - Webhook URL: `https://n8n.t-0.co/webhook/t0-content-pipeline`
  - Payload: page data (Notion automation default payload format)
  - Verify: clicking button on a test entry sends POST to webhook URL

**Checkpoint**: Content Creation DB has all new properties, status values, button, and automation.

---

### Prompt Templates & Config (this repo)

- [ ] T010 [P] Create `config/style-params.yaml`
  - Content as specified in spec § "style-params.yaml"
  - Voice, structure, opening/closing patterns, evidence, CTA settings
  - Verify: valid YAML, parseable

- [ ] T011 [P] Create `prompts/draft-system-prompt.md`
  - Template with placeholder variables for brand context injection
  - Structure: Voice → Structure → Craft Inspiration → Style Parameters → Rules
  - Follow spec § "Draft System Prompt Structure"
  - Verify: all placeholder variables documented

- [ ] T012 [P] Create `prompts/evaluation-prompt.md`
  - System prompt for evaluation agent
  - Include: rubric (7 dimensions), scoring instructions, JSON output schema
  - Gate rules: composite ≥28 AND all dims ≥3 AND voice ≥4
  - Verify: JSON schema matches spec § evaluation output

- [ ] T013 [P] Create `prompts/research-prompt.md`
  - Instructions for research agent
  - Focus areas: German market perspective, practical implications, counterarguments, statistics
  - Tool usage: when to use Jina vs Perplexity
  - Verify: covers all research agent responsibilities from spec

- [ ] T014 [P] Create `prompts/image-description-prompt.md`
  - Auto-generate image description from title + pitch
  - T-0 illustration style reference
  - Verify: output format matches WF-3 expectations

- [ ] T015 [P] Create `prompts/triage-prompt.md`
  - Source triage for WF-2 (Phase B, but create template now)
  - Selection criteria: T-0 experience, source weight, German/EU angles, practical focus
  - JSON array output format
  - Verify: matches spec § WF-2 triage agent

- [ ] T016 Commit all prompts and config to repo
  - `git add config/ prompts/`
  - Verify: files accessible via GitHub raw URL pattern

**Checkpoint**: All prompt templates and style params committed. Accessible via GitHub raw URLs.

---

## Phase A: WF-1 Build (on n8n.t-0.co)

**Purpose**: Build the core content pipeline workflow, branch by branch.

### WF-1 Skeleton

- [ ] T017 Create WF-1 skeleton: Webhook → Parse → Switch → Respond
  - Webhook node: `POST /webhook/t0-content-pipeline`
  - Response mode: `responseMode: "responseNode"` (fire-and-forget)
  - Respond to Webhook node: `{ "status": "processing" }` (immediate)
  - Code node "Parse Payload": extract `pageId`, `Status`, `Name`, `Pitch`, `URL`, `Due Date planned`, `Content Type`, `Image`, `Author`, `Category Tags` from Notion automation payload format (`data.id`, `data.properties.*`)
  - Switch node: route on Status property
    - Case "Idea" / "Backlog" → Branch A output
    - Case "In Revision" → Branch B output
    - Case "Publishable" → Branch C output
    - Default → error comment on Notion page
  - Verify: send test webhook, confirm correct routing by Status value

### WF-1 Branch A: Research → Draft → Evaluate

- [ ] T018 Build Branch A parallel context gathering
  - **Sub-branch 1: Source Material**
    - HTTP node: `GET /pages/{pageId}` — read full page properties
    - HTTP node: query Notes relation pages (if Notes relation has entries)
    - Code node: extract URLs, rant text, summaries from linked notes
  - **Sub-branch 2: Brand Context**
    - HTTP node: `GET /blocks/{writingGuidePageId}/children` (`2d1c0fdf-942d-810d-8fda-cf181d93bf46`)
    - HTTP node: `GET /blocks/{blogContentGuidePageId}/children` (`2d1c0fdf-942d-81cf-bd59-f54a251a1571`)
    - HTTP node: `GET /blocks/{blogStyleAnalysisPageId}/children` (`2d7c0fdf-942d-813e-a679-e55b56cea58b`)
    - Code node: convert Notion blocks to markdown text
  - **Sub-branch 3: Pipeline Config**
    - HTTP node: `GET` GitHub raw — `prompts/draft-system-prompt.md`
    - HTTP node: `GET` GitHub raw — `config/style-params.yaml`
    - Code node: parse YAML, assemble config object
  - Code node "Merge All Context": combine all three sub-branches
  - Verify: all context gathered successfully for a test entry

- [ ] T019 Build Branch A research agent
  - AI Agent node: "Research & Enrich"
  - Chat Model sub-node: OpenRouter (`anthropic/claude-sonnet-4-6`, temp=0.4)
  - Tool 1: HTTP Request tool configured for Jina Reader (`GET r.jina.ai/{url}`)
  - Tool 2: HTTP Request tool configured for Perplexity API (`POST api.perplexity.ai/chat/completions`)
  - System prompt loaded from `prompts/research-prompt.md` content
  - Output: structured research bundle (markdown with citations)
  - Verify: agent produces research for a test topic

- [ ] T020 Build Branch A content type detection + draft prompt assembly
  - Code node "Detect Content Type": rule-based classification (if Content Type not already set)
  - Code node "Assemble Draft Prompt": merge system prompt template + brand context + style params + research bundle + topic details
  - Verify: assembled prompt includes all expected sections

- [ ] T021 Build Branch A draft generation agent
  - AI Agent node: "Generate Draft"
  - Chat Model: OpenRouter (`anthropic/claude-sonnet-4-6`, temp=0.7)
  - Tools: None
  - System: assembled system prompt with full brand context
  - User: assembled user prompt with topic + research + voice seeds
  - Output: full blog post draft in markdown
  - Verify: agent generates a coherent German draft for a test topic

- [ ] T022 Build Branch A draft parsing + Notion write-back
  - Code node "Parse Draft": extract META comment (content_type, word_count, patterns), clean markdown
  - Code node "Markdown to Notion Blocks": Josepha converter (h1-h3, paragraphs, bullets, numbered lists, blockquotes, dividers, bold/italic/code/links, code blocks, callouts). Batch into groups of 100.
  - HTTP node: `PATCH /pages/{pageId}` — set Content Type (if detected), Pipeline Version (git hash), Status → "Draft"
  - HTTP node: `PATCH /blocks/{pageId}/children` — append draft blocks (batched)
  - Verify: draft appears in Notion page body with correct formatting

- [ ] T023 Build Branch A inline evaluation
  - Code node: extract draft text from generated blocks (pass-through)
  - HTTP node: `GET` GitHub raw — `docs/sources/content-evaluation-guide.md`
  - Code node "Assemble Evaluation Prompt": rubric + brand voice ref + draft text
  - AI Agent node: "Evaluate Draft"
    - Chat Model: OpenRouter (`anthropic/claude-sonnet-4-6`, temp=0.2)
    - Output Parser: Structured Output (JSON schema per spec)
  - Code node "Apply Gate Rules":
    - composite ≥28 AND all dims ≥3 AND voice ≥4 → "Pre Review"
    - voice <4 → blocked (constitutional)
    - any dim <3 → blocked
    - else → stays "Draft"
  - HTTP node: `PATCH /pages/{pageId}` — write 7 scores, Composite, Quality Tier, Evaluation Notes, Evaluated At, Status (conditional)
  - HTTP node: `PATCH /blocks/{pageId}/children` — append callout with eval summary
  - HTTP node: `POST /v1/comments` — status notification comment
  - Verify: evaluation scores appear in Notion properties, status advances correctly

### WF-1 Branch C: Ghost Publication

- [ ] T024 Build Branch C Ghost publication
  - HTTP node: `GET /pages/{pageId}` + `GET /blocks/{pageId}/children` (paginated)
  - Code node "Notion Blocks to HTML": convert blocks to Ghost-compatible HTML
  - Code node "Generate Ghost JWT": from Content Pipeline integration key
  - IF node: Image property set?
    - YES: HTTP node: download image from Google Drive URL
    - NO: skip
  - IF node: image downloaded?
    - YES: HTTP node: `POST /ghost/api/admin/images/upload/` (multipart)
  - Code node "Map Properties to Ghost Fields": title, html, tags, authors, excerpt, feature_image, meta_title, meta_description, newsletter slug
  - HTTP node: `POST /ghost/api/admin/posts/?source=html` — create Ghost post
  - IF node: Due Date planned set?
    - YES: HTTP node: `PUT /ghost/api/admin/posts/{id}/` — set `status: "scheduled"`, `published_at`
    - NO: leave as draft
  - HTTP node: `PATCH /pages/{pageId}` — write Ghost URL, Ghost ID, Publication Date actual, Status → "Published"
  - HTTP node: `POST /v1/comments` — "Published to Ghost: {url}"
  - Verify: Ghost draft/scheduled post created, Notion entry updated with Ghost URL and ID

### WF-1 Branch B: Revision

- [ ] T025 Build Branch B revision flow
  - HTTP node: `GET /v1/comments` — filter by page_id, extract Aaron's feedback
  - HTTP node: `GET /blocks/{pageId}/children` (paginated) — current draft
  - Code node "Preserve Previous Version": wrap current blocks in toggle "▸ Previous Version (v{n})"
  - HTTP node: `PATCH /blocks/{pageId}/children` — replace draft area with toggle
  - Reuse Branch A context gathering (brand context + pipeline config)
  - Code node "Assemble Revision Prompt": original research + feedback + previous draft
  - AI Agent: re-generate draft (same config as Branch A)
  - Reuse Branch A write-to-Notion + evaluation flow
  - Verify: old draft preserved in toggle, new draft written, re-evaluated

### WF-1 Error Handling

- [ ] T026 Add error handling to WF-1
  - n8n Error Trigger node on WF-1
  - On error: write error comment to Notion page (`POST /v1/comments`)
  - Set workflow timeout to 10 minutes (Settings → Execution Timeout)
  - Add n8n retry settings: 3 attempts, exponential backoff on HTTP nodes
  - Verify: intentional error (e.g., invalid page ID) produces Notion error comment

**Checkpoint**: WF-1 complete with all 3 branches and error handling.

---

## Phase A: End-to-End Validation

- [ ] T027 End-to-end test: Idea/Backlog entry through Branch A
  - Pick an existing Content Creation entry with Status="Idea" or "Backlog"
  - Press "Run Pipeline" button
  - Verify: draft appears, evaluation scores written, status advances correctly
  - Check: brand voice present, German output, proper formatting

- [ ] T028 End-to-end test: In Revision entry through Branch B
  - Take a "Pre Review" entry, add revision feedback comment, set Status="In Revision"
  - Press "Run Pipeline"
  - Verify: old draft preserved in toggle, new draft incorporates feedback, re-evaluated

- [ ] T029 End-to-end test: Publishable entry through Branch C
  - Take a high-scoring entry, set Status="Publishable"
  - Test 1: without Due Date planned → verify Ghost draft created
  - Test 2: with Due Date planned → verify Ghost scheduled post at 09:00 CET
  - Verify: Ghost post has correct HTML, tags, excerpt, newsletter association

**Checkpoint**: Phase A complete. 3 existing entries processed end-to-end successfully.

---

## Phase D: Ghost Tag Overhaul (parallel with Phase A)

- [ ] T030 [P] Export current Ghost tags and map to new taxonomy
  - `GET /ghost/api/admin/tags/?limit=all` — export all 62 tags
  - Create mapping: old tag → new tag (from spec § "Ghost Tag Overhaul")
  - Identify: duplicates, orphans, tags to merge
  - Document mapping in a temporary file or Notion comment

- [ ] T031 [P] Create new Ghost tags per taxonomy
  - Content type tags (internal): `#opinion`, `#resource`, `#case-study`, `#technical`, `#announcement`, `#pipeline-generated`
  - Topic tags (public): `ki-agenten`, `ki-transformation`, `automatisierung`, `llm`, `daten-analytics`, `mittelstand`, `strategie`, `praxis`, `mcp`
  - Audience tags (internal): `#entscheider`, `#technisch`
  - `POST /ghost/api/admin/tags/` for each
  - Verify: all new tags created

- [ ] T032 Update existing Ghost posts with new tags
  - For each post: GET current tags → map to new taxonomy → PUT with new tag array
  - Remember: tags array is REPLACED, not merged. Always GET first.
  - `updated_at` required for PUT — always use fresh timestamp
  - Verify: spot-check 5 posts have correct new tags

- [ ] T033 Delete orphaned/duplicate Ghost tags
  - After all posts updated, any tag with 0 posts → delete
  - `DELETE /ghost/api/admin/tags/{id}/`
  - Verify: `GET /tags/?limit=all` returns only the new taxonomy tags

**Checkpoint**: Ghost tags cleaned up. New taxonomy in place.

---

## Dependencies & Execution Order

### Phase A (sequential)

```
T001-T005 (credentials, parallel among themselves)
  → T006-T009 (Notion DB setup, mostly sequential)
  → T010-T016 (prompts & config, parallel, then commit)
  → T017 (WF-1 skeleton)
  → T018-T023 (Branch A, sequential)
  → T024 (Branch C)
  → T025 (Branch B)
  → T026 (error handling)
  → T027-T029 (E2E validation)
```

### Phase D (independent)

```
T030-T033 can run at any time, in parallel with Phase A.
Requires Ghost API access only (T002).
```

### Within WF-1 Branches

- Branch A first (core flow, most complex)
- Branch C second (depends on Ghost credential from T002, reuses some Branch A patterns)
- Branch B third (reuses Branch A context gathering + evaluation, adds revision logic)

---

## Implementation Strategy

### Phase A First

1. Credentials (T001-T005)
2. Notion DB setup (T006-T009)
3. Prompts + config (T010-T016)
4. WF-1 skeleton (T017)
5. Branch A (T018-T023) — **STOP and test with one real entry**
6. Branch C (T024) — test with a manual "Publishable" entry
7. Branch B (T025) — test revision flow
8. Error handling (T026)
9. Full E2E validation (T027-T029)

### Phase D Parallel

Start T030-T033 any time after Ghost credential (T002) is ready.

---

## Notes

- All n8n workflow building happens on `n8n.t-0.co` (T-0's dedicated instance)
- Notion API version: `2022-06-28` (forced by n8n `notionApi` credential)
- Ghost API base: `https://t-0.co/ghost/api/admin/`
- GitHub raw URL pattern: `https://raw.githubusercontent.com/T-0-co/t-0_content-production/main/{path}`
- Button + status property creation require Playwright MCP (not Notion API)
- Ghost `updated_at` required for every PUT — always GET first
