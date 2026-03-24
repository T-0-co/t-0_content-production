# Spec 002: T-0 Content Production Pipeline — Full System Design

**Created**: 2026-03-24
**Status**: Design Complete — ready for implementation planning
**Type**: System Design — comprehensive specification
**Input**: "Scope out and design the entire system so we can then discuss it before we actually build it."

---

## Vision

An autonomous content pipeline that ingests sources (RSS feeds + Slack #tech-shit-talk), generates blog post drafts that sound like Aaron and Julien actually wrote them, evaluates those drafts against a fixed rubric, and improves its own output over time — while never publishing anything without explicit human approval.

The pipeline follows the **autoresearch pattern** (Karpathy, 2026):

| Autoresearch Role | This Pipeline | Can Pipeline Modify? |
|---|---|---|
| `prepare.py` (immutable evaluator) | Content Evaluation Guide (7 dimensions, 40pt scale) | **No** — hardcoded rubric |
| `train.py` (agent's sandbox) | Pipeline config: prompts, source weights, style parameters | **Yes** — this is where the ratchet acts |
| `program.md` (human direction) | Editorial strategy: topics, angles, T-0 voice | **Human only** — Aaron/Julien set direction |

**Design goal**: Not "produce more content faster." Instead: produce content that scores ≥28 on the evaluation rubric, sounds unmistakably T-0, and improves with each iteration.

**Highest priority**: Process existing Content Creation DB entries to publishable quality. This is why the pipeline is being built now.

---

## Actors & Roles

| Actor | Role in Pipeline | Touchpoints |
|-------|------------------|-------------|
| **Aaron** | Editorial lead. Enters topic ideas, reviews drafts, approves publication, tunes pipeline parameters, monitors ratchet performance. | Notion (ideas, reviews), Ghost Admin (final check), this repo (config) |
| **Julien** | Co-founder. Posts in #tech-shit-talk (raw material for pipeline), occasionally reviews drafts. | Slack (source material), Notion (review) |
| **Pipeline (n8n)** | Orchestrator. Ingests sources, triages, generates drafts, evaluates, generates images. Runs on schedule + on-demand via button. Mostly deterministic with focused AI Agent nodes at specific steps. | n8n workflows on `n8n.t-0.co` |
| **Claude (in conversation)** | Ad-hoc assistant. Helps Aaron refine drafts, debug pipeline issues, analyze ratchet data. | Claude Code / MCP tools |

---

## User Stories

### US-1 — Aaron Presses "Run Pipeline" and Gets a Draft (Priority: P0)

Aaron has a topic in the Content Creation DB. He presses the "Run Pipeline" button. The pipeline inspects the entry's Status and properties, researches the topic, generates a draft following T-0's brand voice, evaluates it, and writes the scored draft back to the Notion page. One button, adaptive behavior.

**Why P0**: This is the core value loop. Everything else builds on it. And the existing entries in the Content Creation DB need processing NOW.

**Trigger**: "Run Pipeline" button on Content Creation DB → Notion automation → webhook to n8n

**Status-dependent behavior**:

| Status | Pipeline Action |
|--------|----------------|
| Idea / Backlog | Research → Draft → Evaluate → Propose Content Type |
| In Revision | Read Aaron's feedback → Re-draft incorporating feedback → Re-evaluate |
| Publishable | Ghost publication (draft or scheduled) → Image generation if needed |

**Acceptance scenarios**:

1. **Given** a Content Creation entry with Name="MCP vs A2A: Was Mittelständler wissen müssen", Pitch="Vergleich der Protokolle...", Status="Backlog", **When** Aaron presses "Run Pipeline", **Then** a draft appears in the page body, all 7 evaluation scores are written to properties, Content Type is auto-detected and proposed, and Status advances to "Pre Review" (if scores meet threshold) or stays at "Draft" (if below).

2. **Given** a Content Creation entry with a source URL pointing to a news article, **When** the pipeline generates a draft, **Then** the draft incorporates specific facts from that source article (fetched via Jina Reader), properly cited as embedded hyperlinks per Brand Writing Guide v2.2.

3. **Given** a Content Creation entry with no pitch (just a name), **When** the pipeline runs, **Then** it still generates a draft by researching the topic via Perplexity, but flags it as "low-context — human input recommended" in a Notion callout block.

4. **Given** a "Publishable" entry with Due Date planned set to next Tuesday, **When** Aaron presses "Run Pipeline", **Then** a Ghost post is created with `status: "scheduled"` and `published_at` set to Due Date planned at 09:00 CET.

5. **Given** a "Publishable" entry with no Due Date planned, **When** Aaron presses "Run Pipeline", **Then** a Ghost draft is created (not scheduled), and Aaron publishes manually in Ghost Admin.

---

### US-2 — Source Ingestion Surfaces New Topics (Priority: P2)

On a schedule, the pipeline fetches RSS feeds from tracked sources and captures meaningful Slack #tech-shit-talk discussions. AI triage selects the most relevant items for T-0's audience. Selected topics appear as new "Idea" entries in the Content Creation DB.

**Why P2**: Without source ingestion, Aaron manually finds every topic. This automates discovery.

**Trigger**: Scheduled (RSS: weekly Sunday 8AM CET; Slack: every 6 hours)

**Acceptance scenarios**:

1. **Given** 5 active RSS sources in the Content Sources DB, **When** the weekly ingestion runs, **Then** new feed items are stored in the Notes DB (deduplicated by URL), AI triage selects top 5-8 topics, and each selected item becomes a Content Creation entry with Status="Idea", an auto-generated Pitch, the source URL, and Category Tags.

2. **Given** a Slack #tech-shit-talk thread where Julien and 2 colleagues discussed a new AI tool (2+ replies from 2+ different people), **When** the Slack ingestion runs, **Then** the discussion is captured in the Notes DB with Origin="slack-auto-capture", summarized, and flagged for triage.

3. **Given** a topic was already ingested last week, **When** a new article about the same topic appears, **Then** the pipeline adds the new URL to the existing Notes DB entry's Related Content rather than creating a duplicate.

---

### US-3 — Automated Evaluation Scores Every Draft (Priority: P0)

Every draft produced by the pipeline gets evaluated against the Content Evaluation Guide rubric. Seven dimension scores plus a weighted composite are written to Notion properties. Drafts meeting all thresholds advance to "Pre Review." Drafts below threshold stay at "Draft" with specific feedback.

**Why P0**: The evaluator IS the autoresearch ratchet's fixed point. Without it, there's no feedback loop.

**Gate rules**:
- Composite ≥ 28 AND all dimensions ≥ 3 AND Voice ≥ 4 → advance to "Pre Review"
- Voice < 4 → **blocked** regardless of composite (constitutional principle)
- Any dimension < 3 → blocked, specific feedback required

**Acceptance scenarios**:

1. **Given** a draft in a Content Creation page body, **When** evaluation runs, **Then** seven dimension scores are written to number properties, plus Composite Score (weighted, max 40), Quality Tier, and Evaluation Notes with per-dimension feedback.

2. **Given** a draft that scores Voice=3 but Composite=30, **When** evaluation completes, **Then** Status remains "Draft" (not advanced) and notes specifically flag: "Voice score below minimum threshold (4)."

3. **Given** a draft scoring Composite=35 with all dimensions ≥4, **When** evaluation completes, **Then** Status advances to "Pre Review" and a Notion comment notifies Aaron.

---

### US-4 — Aaron Reviews and Approves a Draft (Priority: P0)

Aaron opens a "Pre Review" draft in Notion, reads the content and evaluation scores, and decides: approve, request revisions, or reject.

**Why P0**: The human gate is a constitutional principle.

**Aaron's actions**:

| Action | Status Change | Pipeline Response |
|--------|---------------|-------------------|
| Approve | → "Publishable" | Aaron presses "Run Pipeline" → WF-1 Ghost branch |
| Request revision | → "In Revision" + comment | Aaron presses "Run Pipeline" → WF-1 Revision branch |
| Reject | → "Idea" or archive | No pipeline action |
| Edit directly | Stays "Pre Review" | Manual edits; Aaron re-triggers eval if desired |

**Revision handling**: When Aaron presses "Run Pipeline" on an "In Revision" entry:
1. Pipeline reads all recent comments on the page
2. Preserves the current draft in a toggle block: "▸ Previous Version (v{n})"
3. Regenerates using original research bundle + Aaron's feedback
4. Re-evaluates the new draft

---

### US-5 — Publishable Draft Goes to Ghost (Priority: P1)

When Aaron presses "Run Pipeline" on a "Publishable" entry, the pipeline converts it to Ghost-compatible HTML, handles image upload, creates a Ghost post, and conditionally schedules it.

**Why P1**: Publication is the output of the pipeline but depends on P0 stories working first.

**Conditional logic**:
- **Due Date planned is set** → create Ghost post with `status: "scheduled"`, `published_at` = Due Date at 09:00 CET
- **No Due Date planned** → create Ghost draft, Aaron publishes manually

**Newsletter**: Ghost natively ties blog posts to newsletter distribution. Pipeline sets the newsletter association (`default-newsletter`) on all posts. When Aaron publishes (or schedule fires), the newsletter is sent automatically.

**Acceptance scenarios**:

1. **Given** a "Publishable" entry with draft body, featured image, and Category Tags, **When** Aaron presses "Run Pipeline", **Then** a Ghost post is created at `t-0.co` with correct title, HTML body, tags, excerpt, featured image, SEO fields, and newsletter association.

2. **Given** Due Date planned = "2026-04-15", **When** the Ghost branch runs, **Then** the post is created with `status: "scheduled"` and `published_at: "2026-04-15T09:00:00.000+02:00"`.

3. **Given** no Due Date planned, **When** the Ghost branch runs, **Then** the post is created as `status: "draft"`. Aaron reviews in Ghost Admin and manually clicks Publish.

4. **Given** the Ghost post is created, **Then** the Content Creation entry's URL property gets the Ghost post URL, Ghost ID property gets the post ID, and Status advances to "Published".

---

### US-6 — Branded Image Generation (Priority: P2)

Each post gets a branded illustration via the existing Gemini image gen workflow. The workflow is modular — callable from the main pipeline during Ghost publication, or standalone via the existing "Run Image Gen" button.

**Why P2**: Images enhance posts. The workflow already exists and serves other purposes, so keeping it modular is important.

**Acceptance scenarios**:

1. **Given** a Content Creation entry with an Image Gen Description, **When** image generation triggers, **Then** the description is enhanced (incorporating T-0 illustration style), Gemini generates an image, it's uploaded to Google Drive, and the URL is written to the Image property.

2. **Given** no Image Gen Description, **When** image gen triggers, **Then** the pipeline auto-generates a description from the post title and pitch.

3. **Given** image gen runs as part of Ghost publication, **Then** the featured image is also uploaded to Ghost via the image upload API.

---

### US-7 — Self-Improvement Ratchet (Priority: P3 — Dormant Initially)

The ratchet is dormant for the first weeks of pipeline operation. Once enough evaluated content exists (20+ drafts), it activates: analyzes score trends, proposes and A/B tests config changes, applies improvements that demonstrably raise scores.

**Why P3**: Requires US-1 through US-3 generating enough data first. Expected activation: 2-3 months after pipeline launch.

**Scope — full autonomy with versioning**:
The ratchet can modify:
- Prompt templates (wording, instructions, examples)
- Style parameters (opinion strength, word counts, opening/closing pattern weights)
- Source selection and weighting (RSS source priorities, Slack filtering thresholds)

All changes are committed to this repo via PR (not direct to main). Aaron reviews and merges. Every change includes the ratchet run reference, score improvement data, and a rollback path.

**Acceptance scenarios**:

1. **Given** 20+ evaluated drafts, **When** the ratchet runs, **Then** it produces a report showing score trends, proposals tested, and accept/reject decisions with supporting data.

2. **Given** a proposed change passes A/B testing (avg composite improves ≥1 point, no dimension drops >0.5), **Then** a PR is created in this repo with the change, ratchet metadata, and before/after scores.

3. **Given** a ratchet change is merged, **Then** subsequent pipeline runs use the updated config and scores are tracked against the new baseline.

---

### US-8 — Slack Discussion Capture (Priority: P2)

Timer-based ingestion replaces the existing emoji-reaction Slack workflow. Captures meaningful discussions from #tech-shit-talk as raw material for content creation.

**Why P2**: Slack rants are what make T-0 content sound like T-0. This is the authentic voice material.

**Filter**: 2+ replies from 2+ different people (captures real discussions, not just popular one-liners).

**Acceptance scenarios**:

1. **Given** Julien and Aaron discuss a topic in #tech-shit-talk with 3+ replies, **When** the Slack ingestion runs, **Then** the thread is captured in the Notes DB with Origin="slack-auto-capture", full thread text, participants, and a lightweight AI summary.

2. **Given** a captured Slack discussion is selected during triage, **When** it becomes a Content Creation entry, **Then** the rant's strong opinions and specific phrasings are preserved as "voice seeds" in the Notes relation.

3. **Given** the timer-based workflow is deployed, **Then** the existing emoji-reaction Slack-to-Notion workflow becomes superfluous and can be deactivated.

---

### US-9 — Process Existing Content Creation Entries (Priority: P0)

The Content Creation DB has existing entries dating back to 2025. Processing these to publishable quality is the primary reason the pipeline is being built.

**Why P0**: This is the immediate business value. The pipeline's first job is to finish those drafts.

**Acceptance scenarios**:

1. **Given** existing "Idea" and "Backlog" entries, **When** Aaron presses "Run Pipeline" on each, **Then** the pipeline generates drafts, evaluates them, and Aaron reviews/revises until they reach "Publishable".

2. **Given** all existing entries are processed to "Publishable", **Then** Aaron sets Due Date planned on each and uses the pipeline to schedule them for publication, creating a consistent publication cadence.

---

### US-10 — Social Distribution (Priority: Roadmap — Not V1)

Blog posts on Ghost serve as the canonical content source. LinkedIn and other platforms are downstream. The pipeline will generate platform-specific variants from published blog content.

**Why Roadmap**: The blog is the foundation. Distribution is important but depends on the core pipeline working first. Content Creation DB already has a Platform multi-select that includes "LinkedIn."

**Not built in v1. Architectural consideration**: When designing the Ghost publication branch, structure content and metadata in a way that makes downstream platform extraction straightforward.

---

### Edge Cases

- **Empty RSS feed**: Source returns 200 but no items → log, skip, continue
- **Duplicate topic from multiple sources**: Dedup by URL first, then semantic similarity
- **Draft generation fails**: Status stays unchanged, error comment on Notion page
- **Evaluation produces out-of-range scores**: Re-run with explicit range instruction
- **Ghost API error**: Status stays "Publishable", error comment in Notion
- **Paywalled source URL**: Jina returns partial content → fall back to RSS summary + Perplexity research, flag in draft
- **Image generation fails**: Post proceeds without image, Notion comment added
- **Very long drafts**: Notion blocks batched by 100 (Josepha pattern)
- **Conflicting human edits**: Pipeline fetches latest page version before writing
- **Stale Slack discussions**: 3+ weeks old → still capture but lower triage priority

---

## System Architecture

### Component Map

```
┌──────────────────────────────────────────────────────────────┐
│                    THIS REPO (Config Layer)                    │
│  prompts/  ·  config/style-params.yaml  ·  docs/sources/     │
└────────────────────────────┬─────────────────────────────────┘
                             │ consumed by (GitHub raw URL)
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                 n8n.t-0.co (Orchestration Layer)               │
│                                                                │
│  WF-1: Content Pipeline ──→ single-button, branches by Status │
│    ├── Idea/Backlog → Research → Draft → Evaluate              │
│    ├── In Revision  → Feedback → Re-draft → Re-evaluate       │
│    └── Publishable  → Ghost publish + Image gen                │
│                                                                │
│  WF-2: Source Ingestion ──→ scheduled (RSS weekly, Slack 6h)   │
│    └── RSS + Slack → Notes DB → AI Triage → Content Ideas      │
│                                                                │
│  WF-3: Image Generation ──→ modular (from WF-1 or standalone) │
│                                                                │
│  WF-4: Ratchet Analysis ──→ monthly (dormant initially)        │
└──────┬─────────┬──────────┬──────────┬──────────┬────────────┘
       │         │          │          │          │
       ▼         ▼          ▼          ▼          ▼
   ┌───────┐ ┌───────┐ ┌────────┐ ┌──────────┐ ┌────────┐
   │Notion │ │Ghost  │ │OpenRtr │ │Perplexity│ │Gemini  │
   │       │ │CMS    │ │(Claude │ │(AI Agent │ │(imagen)│
   │       │ │t-0.co │ │Sonnet) │ │tool)     │ │        │
   └──┬────┘ └───────┘ └────────┘ └──────────┘ └────────┘
      │
      ├── Content Creation DB (extended)
      ├── Notes DB (extended — replaces Feed Items concept)
      ├── Content Sources DB (new, in T-0 Main DB)
      └── T-0 Wiki pages (Brand Writing Guide, Blog Content Guide, etc.)

External Inputs:
  ├── RSS Feeds (via HTTP GET)
  ├── Slack #tech-shit-talk (via Slack API)
  ├── Jina Reader (article full-text extraction)
  └── Manual entry (Aaron → Notion)
```

### Data Flow (End to End)

```
RSS Feeds ──→ ┐
Slack Rants ──→├──→ [WF-2: Ingest] ──→ Notes DB ──→ [AI Triage]
Manual Ideas ─→┘                           │
                                           ▼
                                 Content Creation DB
                                 (Status: "Idea")
                                           │
                                 Aaron promotes to "Backlog"
                                 (or processes existing entries)
                                           │
                                           ▼
                                 Aaron presses "Run Pipeline"
                                           │
                              ┌────────────┼────────────┐
                              ▼            ▼            ▼
                         Idea/Backlog  In Revision  Publishable
                              │            │            │
                              ▼            ▼            ▼
                         Research +    Feedback +   Ghost Pub +
                         Draft + Eval  Re-draft    Image Gen
                              │            │            │
                              ▼            ▼            ▼
                         "Pre Review"  "Pre Review"  "Published"
                         or "Draft"    or "Draft"    Ghost post +
                                                     Newsletter
                                                         │
                                                    [Future: Social
                                                     Distribution]
```

### Technology Stack Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Orchestration** | n8n at `n8n.t-0.co` | T-0's dedicated instance. Constitution mandates. 29 active workflows. |
| **LLM access** | OpenRouter → Claude Sonnet 4.6 | Single gateway to all models. AI Agent nodes with Chat Model sub-node. |
| **Agent architecture** | Mostly deterministic, agents get 2-3 tools max | Cost-conscious. Only use agent autonomy where essential for quality. |
| **Content DB** | Notion Content Creation DB (extended) | Already exists, team uses it daily, mature status pipeline. |
| **Feed item storage** | Notes DB (existing) | Already has Slack fields, AI enrichment, Content Creation relation, Origin tracking. |
| **Source registry** | Content Sources DB (new, in T-0 Main DB) | Follows Main DB pattern (page → child database). |
| **Publishing** | Ghost CMS at `t-0.co` | Already running, Admin API, handles newsletters natively. |
| **Ghost auth** | Dedicated "Content Pipeline" integration key | Stored in n8n credential management on `n8n.t-0.co`. |
| **Research** | Perplexity as AI Agent tool | Agent decides when to call Perplexity for web-grounded research. |
| **Article extraction** | Jina Reader as AI Agent tool | Agent decides when to deep-fetch article content. |
| **Image generation** | Google Gemini (existing workflow, modular) | Already operational. Keep modular for reuse across projects. |
| **Content format** | HTML via `?source=html` | Ghost converts to Lexical internally. Simpler than building Lexical JSON. |
| **Config source** | Notion wiki pages (brand context) + this repo (prompts, style params) | Brand guides from Notion at runtime; repo files version-controlled for ratchet. |
| **Prompt language** | English system prompts → German output | LLMs follow English instructions more consistently. Output is always German unless explicitly switched to English path. |
| **Monitoring** | n8n execution history | No separate Pipeline Runs DB. n8n's built-in execution logs suffice. |
| **Newsletter** | Ghost native | Blog posts automatically sent to `default-newsletter` subscribers on publish. |

---

## n8n Architecture Patterns

### AI Agent Node Pattern

Every LLM call uses an **n8n AI Agent node** (`@n8n/n8n-nodes-langchain.agent`) with an **OpenRouter Chat Model** sub-node. This provides a consistent interface regardless of whether the step needs tools.

```
AI Agent node (per LLM step)
├── Chat Model: OpenRouter (claude-sonnet-4-6)
├── Tools: [0-3 tools, clearly scoped per step]
├── Output Parser: Structured Output (when JSON output needed)
└── Memory: None (single-shot tasks, not conversational)
```

**Cost-conscious agent design** (from Aaron's direction):
- Large parts of the workflow are deterministic n8n nodes (Code, HTTP, IF, Switch)
- AI Agent nodes are used only where LLM judgment is needed
- Each agent gets **2-3 tools maximum** with a clearly scoped task
- No free planning — the task is well-defined when it reaches the agent
- The agent makes tactical decisions (which tool to use) but not strategic ones (what to do next)

### Agent Nodes in the Pipeline

| Step | Agent? | Tools | Why |
|------|--------|-------|-----|
| Research & enrichment | **Yes** | Jina Reader, Perplexity | Agent decides research strategy based on available sources |
| Draft generation | **Yes** (0 tools) | None — just model + output format | Complex generation task requiring LLM, but no tool decisions |
| Content type detection | No — Code node | — | Simple classification based on keywords/signals |
| Evaluation | **Yes** (0 tools) | Output Parser only | LLM scoring with structured JSON output |
| Triage | **Yes** (0 tools) | Output Parser only | LLM selection from list, structured output |
| Image description | **Yes** (0 tools) | None | Simple generation, no tools needed |
| Ratchet proposals | **Yes** | Perplexity | May need current context for proposals |

### Fire-and-Forget Webhook Pattern

All webhook-triggered workflows:
1. Receive Notion automation payload
2. **Immediately** respond with HTTP 200 via Respond to Webhook node
3. Process asynchronously
4. Write results back to Notion

This prevents Notion automation timeouts (10s limit) and gives instant feedback to the user.

### Notion Automation Payload

Notion automations send: `{ source: {...}, data: { object: "page", id: "...", properties: { Name: { type: "title", ... }, ... } } }`. The first Code node in every webhook-triggered workflow must flatten this to extract `pageId` and properties.

---

## Workflow 1: Content Pipeline (Single-Button Trigger)

**Webhook**: `POST https://n8n.t-0.co/webhook/t0-content-pipeline`
**Trigger**: "Run Pipeline" button on Content Creation DB → Notion automation → webhook
**Response mode**: `responseMode: "responseNode"` (fire-and-forget)

### Node Flow

```
Webhook Trigger (receives Notion automation payload)
  │
  ├──→ Respond to Webhook: { "status": "processing" }
  │
  ├──→ Code: Parse Payload
  │     Extract: pageId, Status, Name, Pitch, URL, Due Date planned,
  │     Content Type, Image, Author, Category Tags
  │
  ├──→ Switch: Route by Status
  │
  ╔══════════════════════════════════════════════════════════════╗
  ║ BRANCH A: Status = "Idea" or "Backlog"                      ║
  ║ Action: Research → Draft → Evaluate                         ║
  ╚══════════════════════════════════════════════════════════════╝
  │
  ├──→ [PARALLEL: Gather Context]
  │     │
  │     ├── Sub-branch 1: Source Material
  │     │   ├── HTTP: GET Notion — read Notes relation (linked notes)
  │     │   ├── Code: Extract URLs, rant text, summaries from notes
  │     │   └── IF: Primary URL set?
  │     │       └── YES: (passed to Research Agent as context)
  │     │
  │     ├── Sub-branch 2: Brand Context
  │     │   ├── HTTP: GET Notion /blocks/{writingGuidePageId}/children
  │     │   ├── HTTP: GET Notion /blocks/{blogContentGuidePageId}/children
  │     │   ├── HTTP: GET Notion /blocks/{blogStyleAnalysisPageId}/children
  │     │   └── Code: Convert blocks to markdown text
  │     │
  │     └── Sub-branch 3: Pipeline Config
  │         ├── HTTP: GET GitHub raw — prompts/draft-system-prompt.md
  │         ├── HTTP: GET GitHub raw — config/style-params.yaml
  │         └── Code: Parse YAML, assemble config object
  │
  ├──→ Code: Merge All Context
  │     Combine: source material + brand context + style params + page properties
  │     Build: research query from Name + Pitch + URL
  │
  ├──→ AI Agent: Research & Enrich
  │     ├── Chat Model: OpenRouter (claude-sonnet-4-6, temp=0.4)
  │     ├── Tool 1: Jina Reader (HTTP Request tool → r.jina.ai/{url})
  │     ├── Tool 2: Perplexity (HTTP Request tool → api.perplexity.ai)
  │     ├── System: "You are a research assistant for T-0. Given a topic,
  │     │   gather relevant information. Use Jina to read specific URLs.
  │     │   Use Perplexity for broader research. Focus on: German market
  │     │   perspective, practical implications, counterarguments, statistics.
  │     │   Return structured research with citations."
  │     └── Output: Research bundle (markdown with citations)
  │
  ├──→ Code: Detect Content Type (if not set)
  │     Rules:
  │     - News event/announcement → Opinion / Analysis
  │     - Specific tool/technique → Resource (Technical)
  │     - Client project reference → Case Study
  │     - Slack rant with strong opinion → Opinion / Speculation
  │     - Comparison/framework → Resource (Explainer)
  │
  ├──→ Code: Assemble Draft Prompt
  │     System prompt: brand context + style params + rules
  │     User prompt: topic + research bundle + voice seeds + content type template
  │
  ├──→ AI Agent: Generate Draft
  │     ├── Chat Model: OpenRouter (claude-sonnet-4-6, temp=0.7)
  │     ├── Tools: None
  │     ├── System: {assembled system prompt with full brand context}
  │     ├── User: {assembled user prompt with topic + research + voice seeds}
  │     └── Output: Full blog post draft in markdown
  │
  ├──→ Code: Parse Draft
  │     Extract: META comment (content_type, word_count, opening/closing patterns)
  │     Clean: markdown formatting
  │
  ├──→ Code: Convert Markdown to Notion Blocks
  │     Josepha converter: h1-h3, paragraphs, bullets, numbered lists,
  │     blockquotes, dividers, bold/italic/code/links, code blocks, callouts
  │     Batch into groups of 100 (Notion API limit)
  │
  ├──→ HTTP: PATCH Notion /pages/{pageId}
  │     Set: Content Type (if auto-detected), Pipeline Version (git hash),
  │     Status → "Draft"
  │
  ├──→ HTTP: PATCH Notion /blocks/{pageId}/children (batched)
  │     Append draft blocks to page body
  │
  ├──→ [EVALUATION — inline, not a separate workflow call]
  │
  ├──→ Code: Extract draft text from generated blocks (pass-through, no re-read)
  │
  ├──→ HTTP: GET GitHub raw — docs/sources/content-evaluation-guide.md
  │
  ├──→ Code: Assemble Evaluation Prompt
  │     Include: full rubric + brand voice reference + draft text
  │
  ├──→ AI Agent: Evaluate Draft
  │     ├── Chat Model: OpenRouter (claude-sonnet-4-6, temp=0.2)
  │     ├── Tools: None
  │     ├── Output Parser: Structured Output (JSON schema)
  │     ├── System: "You are a strict content evaluator for T-0. Score
  │     │   against the Content Evaluation Guide. Be strict — this rubric
  │     │   prevents generic AI consultancy content from being published."
  │     └── Output JSON schema:
  │         {
  │           "clarity": { "score": 1-5, "feedback": "..." },
  │           "structure": { "score": 1-5, "feedback": "..." },
  │           "accuracy": { "score": 1-5, "feedback": "..." },
  │           "credibility": { "score": 1-5, "feedback": "..." },
  │           "engagement": { "score": 1-5, "feedback": "..." },
  │           "voice": { "score": 1-5, "feedback": "..." },
  │           "actionability": { "score": 1-5, "feedback": "..." },
  │           "composite": <weighted total>,
  │           "quality_tier": "Excellent|Good|Acceptable|Below Standard|Poor",
  │           "top_strength": "...",
  │           "primary_weakness": "...",
  │           "voice_assessment": "..."
  │         }
  │
  ├──→ Code: Apply Gate Rules
  │     composite ≥ 28 AND all dimensions ≥ 3 AND voice ≥ 4 → "Pre Review"
  │     voice < 4 → blocked (constitutional)
  │     any dimension < 3 → blocked
  │     else → stays "Draft"
  │
  ├──→ HTTP: PATCH Notion /pages/{pageId}
  │     Write: all 7 scores, Composite Score, Quality Tier, Evaluation Notes,
  │     Evaluated At, Status (conditional advancement)
  │
  ├──→ HTTP: PATCH Notion /blocks/{pageId}/children
  │     Append: Callout block with evaluation summary + per-dimension feedback
  │
  └──→ HTTP: POST Notion /v1/comments
        If passing: "✅ Draft evaluated: {composite}/40 ({tier}). Ready for review."
        If blocked: "⚠️ Below threshold. {reason}. Composite: {score}/40"

  ╔══════════════════════════════════════════════════════════════╗
  ║ BRANCH B: Status = "In Revision"                            ║
  ║ Action: Read feedback → Re-draft → Re-evaluate              ║
  ╚══════════════════════════════════════════════════════════════╝
  │
  ├──→ HTTP: GET Notion /v1/comments (filter: page_id = {pageId})
  │     Extract: Aaron's feedback comments (most recent)
  │
  ├──→ HTTP: GET Notion /blocks/{pageId}/children (paginated)
  │     Read: current draft content
  │
  ├──→ Code: Preserve Previous Version
  │     Wrap current draft blocks in a toggle: "▸ Previous Version (v{n})"
  │
  ├──→ HTTP: PATCH Notion /blocks/{pageId}/children
  │     Replace: draft area with toggle block containing old version
  │
  ├──→ [Same context gathering as Branch A: brand context + pipeline config]
  │
  ├──→ Code: Assemble Revision Prompt
  │     Include: original research + Aaron's feedback + previous draft
  │     Instruction: "Incorporate this feedback while maintaining T-0 voice.
  │     The reviewer specifically said: {feedback_text}"
  │
  ├──→ AI Agent: Re-generate Draft (same config as Branch A)
  │
  ├──→ [Same write-to-Notion + evaluation flow as Branch A]
  │
  └──→ End

  ╔══════════════════════════════════════════════════════════════╗
  ║ BRANCH C: Status = "Publishable"                            ║
  ║ Action: Ghost publication + Image generation                 ║
  ╚══════════════════════════════════════════════════════════════╝
  │
  ├──→ HTTP: GET Notion /pages/{pageId} + /blocks/{pageId}/children
  │     Fetch: full page properties + body blocks (paginated)
  │
  ├──→ [PARALLEL]
  │     ├── Code: Convert Notion blocks to HTML
  │     │   h1→<h1>, h2→<h2>, h3→<h3>, paragraph→<p>,
  │     │   bulleted_list_item→<ul><li>, numbered_list_item→<ol><li>,
  │     │   quote→<blockquote>, code→<pre><code>, divider→<hr>,
  │     │   image→<img>, bold→<strong>, italic→<em>, link→<a>,
  │     │   strikethrough→<s>, inline code→<code>
  │     │
  │     ├── Code: Generate Ghost JWT
  │     │   const [id, secret] = apiKey.split(':');
  │     │   const token = jwt.sign({}, Buffer.from(secret, 'hex'),
  │     │     { keyid: id, algorithm: 'HS256', expiresIn: '5m',
  │     │       audience: '/admin/' });
  │     │
  │     └── IF: Image property set?
  │         ├── YES: HTTP: Download image from Google Drive URL
  │         └── NO: Skip (or trigger WF-3 image gen)
  │
  ├──→ IF: Image downloaded?
  │     └── YES: HTTP: POST https://t-0.co/ghost/api/admin/images/upload/
  │              (multipart/form-data, image binary)
  │
  ├──→ Code: Map Notion Properties to Ghost Fields
  │     title: Title property (fallback: Name)
  │     html: converted HTML
  │     tags: Category Tags → Ghost slugs + "#pipeline-generated" internal tag
  │     authors: Author email → Ghost author
  │     custom_excerpt: Pitch (first 300 chars)
  │     feature_image: uploaded image URL (if available)
  │     meta_title: Title (max 60 chars)
  │     meta_description: Pitch (max 155 chars)
  │     newsletter: { slug: "default-newsletter" }
  │
  ├──→ HTTP: POST https://t-0.co/ghost/api/admin/posts/?source=html
  │     Create Ghost post as draft
  │
  ├──→ IF: Due Date planned is set?
  │     ├── YES: HTTP: PUT https://t-0.co/ghost/api/admin/posts/{id}/
  │     │        Set: status="scheduled", published_at="{dueDate}T09:00:00+02:00"
  │     └── NO: Leave as draft
  │
  ├──→ HTTP: PATCH Notion /pages/{pageId}
  │     URL = Ghost post URL
  │     Ghost ID = Ghost post ID
  │     Publication Date actual = now (or scheduled date)
  │     Status = "Published"
  │
  ├──→ IF: Image property empty?
  │     └── YES: HTTP: POST n8n.t-0.co/webhook/t0-image-gen (trigger WF-3)
  │
  └──→ HTTP: POST Notion /v1/comments
        "Published to Ghost: {url}. Status: {draft|scheduled for date}."
```

### Credentials Required on n8n.t-0.co

| Credential | Type | Purpose |
|------------|------|---------|
| OpenRouter | `openRouterApi` | All LLM calls (Claude Sonnet 4.6 via OpenRouter) |
| Notion | `notionApi` or `httpHeaderAuth` | Content Creation DB, Notes DB, wiki pages |
| Ghost "Content Pipeline" | Custom (stored as generic credential) | Ghost Admin API JWT generation |
| Slack | Bot OAuth token | #tech-shit-talk channel history |
| Jina | `httpHeaderAuth` (Bearer) | Article full-text extraction |
| Perplexity | `httpHeaderAuth` (Bearer) | Web-grounded research |
| Google Drive (T-0) | `googleOAuth2Api` (`aaron@t-0.co`) | Image upload/download |
| Gemini | API key | Image generation |

---

## Workflow 2: Source Ingestion (Scheduled)

**Schedule**: RSS weekly (Sunday 8:00 CET), Slack every 6 hours
**Webhook**: `POST https://n8n.t-0.co/webhook/t0-source-ingestion` (manual trigger)

### Node Flow

```
Schedule Trigger / Webhook Trigger
  │
  ├──→ HTTP: POST Notion /databases/{contentSourcesDbId}/query
  │     Filter: Status = "Active"
  │     Extract: RSS sources (where RSS URL is set) + Slack config
  │
  ╔══════════════════════════════════════════════════════════════╗
  ║ PARALLEL BRANCH A: RSS Feeds                                ║
  ╚══════════════════════════════════════════════════════════════╝
  │
  ├──→ Code: Fan Out RSS Sources (split array)
  │
  ├──→ HTTP: GET {rssUrl} per source
  │     Accept: application/rss+xml, application/atom+xml
  │
  ├──→ Code: Parse RSS/Atom feed
  │     Regex parser (Josepha pattern): handles RSS 2.0 <item> + Atom <entry>
  │     Filter: items from last 7 days (by pubDate/published)
  │     Extract: title, url, published, summary (strip HTML), author
  │
  ├──→ Aggregate: All RSS items into single array
  │
  ╔══════════════════════════════════════════════════════════════╗
  ║ PARALLEL BRANCH B: Slack #tech-shit-talk                    ║
  ╚══════════════════════════════════════════════════════════════╝
  │
  ├──→ HTTP: GET Slack conversations.history
  │     channel={techShitTalkChannelId}, oldest={6h ago or 7d ago}
  │
  ├──→ Code: Filter Meaningful Discussions
  │     Rule: 2+ replies from 2+ different people
  │
  ├──→ HTTP: GET Slack conversations.replies (per qualifying thread)
  │     Fetch full thread text + participants
  │
  ├──→ Code: Extract URLs from messages (regex)
  │
  ├──→ HTTP: GET r.jina.ai/{url} (for each URL found in messages)
  │     Optional deep-fetch of shared article content
  │
  ├──→ Code: Package Slack items into unified format
  │
  ├──→ Aggregate: All Slack items
  │
  ╔══════════════════════════════════════════════════════════════╗
  ║ MERGE + DEDUP + STORE                                       ║
  ╚══════════════════════════════════════════════════════════════╝
  │
  ├──→ Code: Merge RSS + Slack into unified format
  │     Fields: title, url, published, summary, source, origin, full_text
  │
  ├──→ HTTP: POST Notion /databases/{notesDbId}/query
  │     Filter: Origin contains "rss-ingestion" OR "slack-auto-capture",
  │     Time >= 7 days ago
  │     Purpose: get existing items for dedup
  │
  ├──→ Code: Dedup by URL (set-diff against existing Notes DB entries)
  │
  ├──→ Code: Store New Items in Notes DB (loop)
  │     For each new item:
  │     ├── HTTP: POST Notion /pages
  │     │   Properties:
  │     │     Name = article title (or first 100 chars of Slack message)
  │     │     URL = article URL (or Slack thread link)
  │     │     Time = published date (RSS) or thread timestamp (Slack)
  │     │     Origin = "rss-ingestion" or "slack-auto-capture"
  │     │     Origin Description = source feed name
  │     │     Status = "Backlog"
  │     │     Source = relation to Content Sources DB entry
  │     │     Description = RSS summary or Slack thread summary
  │     │     (Slack-specific: thread_ts_slack, channel_id_slack, contributors_slack)
  │     └── HTTP: PATCH Notion /blocks/{pageId}/children
  │         Append: full text as paragraph blocks (if available)
  │
  ├──→ HTTP: PATCH Notion /pages/{sourcePageId}
  │     Update: Last Fetched = now (for each processed source)
  │
  ╔══════════════════════════════════════════════════════════════╗
  ║ AI TRIAGE (only if new items exist)                          ║
  ╚══════════════════════════════════════════════════════════════╝
  │
  ├──→ Code: Build Triage Input
  │     Numbered list with: title, source, summary, source weight, origin type
  │
  ├──→ AI Agent: Triage
  │     ├── Chat Model: OpenRouter (claude-sonnet-4-6, temp=0.2)
  │     ├── Tools: None
  │     ├── Output Parser: Structured Output (JSON array of selected indices)
  │     ├── System: "You are a content strategist for T-0, a German AI
  │     │   transformation agency. T-0's audience: German Mittelstand
  │     │   decision-makers. T-0's stance: pragmatic, show-the-work,
  │     │   strong opinions, anti-hype."
  │     └── User: "Select 5-8 most relevant items. Prioritize:
  │           1. Topics where T-0 has direct experience
  │           2. Higher-weighted sources
  │           3. German/European angles
  │           4. Practical implications over theory
  │           5. Slack discussions (authentic voice material)
  │           Return JSON array of selected item numbers."
  │
  ├──→ Code: Parse selection, update Notes DB status
  │     Selected: Status → "Currently Relevant"
  │     Not selected: Status → "Discarded"
  │
  ├──→ Code: Create Content Creation Entries for Selected Items
  │     For each selected item:
  │     ├── HTTP: POST Notion /pages (in Content Creation DB)
  │     │   Name = article title or discussion topic
  │     │   Status = "Idea"
  │     │   Pitch = AI-generated from summary + triage reasoning
  │     │   URL = source URL
  │     │   Notes = relation to Notes DB entry
  │     │   Source Type = "RSS" or "Slack Rant"
  │     │   Category Tags = AI-suggested
  │     └── HTTP: PATCH Notes DB entry
  │         Content Creation = relation to new entry
  │
  └──→ End (no webhook response needed for scheduled runs)
```

---

## Workflow 3: Image Generation (Modular)

**Webhook**: `POST https://n8n.t-0.co/webhook/t0-image-gen`
**Trigger**: Called from WF-1 (Ghost publication branch), or standalone via "Run Image Gen" button
**Based on**: Existing Josepha image generation workflow (adapted for T-0 style)

```
Webhook Trigger (receives { pageId })
  │
  ├──→ HTTP: GET Notion /pages/{pageId}
  │     Extract: Image Gen Description, Title, Pitch, Name
  │
  ├──→ IF: Image Gen Description empty?
  │     ├── YES: AI Agent — auto-generate description
  │     │   ├── Chat Model: OpenRouter (claude-sonnet-4-6, temp=0.5)
  │     │   └── Prompt: "Summarize this post topic into a 50-word
  │     │       illustration scene description: {title} — {pitch}"
  │     └── NO: Use existing description
  │
  ├──→ HTTP: GET Notion — T-0 illustration style guide page blocks
  │
  ├──→ AI Agent: Enhance Image Prompt
  │     ├── Chat Model: OpenRouter (claude-sonnet-4-6, temp=0.3)
  │     ├── Tools: Perplexity (optional — for visual reference research)
  │     └── Output: { imageDescription: "...", imageTitle: "kebab-case-name" }
  │
  ├──→ Code: Build Gemini request (multimodal payload)
  │
  ├──→ HTTP: POST Gemini API (timeout=60s)
  │     Generate image from enhanced prompt
  │
  ├──→ Code: Parse response, extract base64 → binary PNG
  │
  ├──→ HTTP: POST Google Drive API
  │     Upload to "Image Generation Dump for Blog" folder (13CQ-Y2CRITlUn9fc28LnLLXoVeRtAVLE)
  │
  ├──→ HTTP: POST Google Drive permissions API
  │     Set public sharing
  │
  ├──→ HTTP: PATCH Notion /pages/{pageId}
  │     Image property = Drive URL
  │
  ├──→ HTTP: PATCH Notion /blocks/{pageId}/children
  │     Append image block to page body
  │
  └──→ IF: Ghost post exists (Ghost ID property set)?
        └── YES: HTTP: POST Ghost /images/upload/ + PUT /posts/{id}/
            Upload image to Ghost and set as feature_image
```

---

## Workflow 4: Ratchet Analysis (Dormant → Active)

**Schedule**: Monthly, 1st of month 10:00 CET (dormant until enough data exists)
**Webhook**: `POST https://n8n.t-0.co/webhook/t0-ratchet-analysis` (manual trigger)
**Activation threshold**: 20+ evaluated drafts in Content Creation DB

### Design Principles

- **Full autonomy**: Can modify prompts, style parameters, AND source selection/weighting
- **Versioned changes**: Every modification committed to repo via PR, with before/after scores
- **Rollback-ready**: Each PR includes rollback instructions and the previous config
- **Conservative acceptance**: Changes accepted only if avg composite improves ≥1 AND no dimension drops >0.5

### Flow

```
Schedule Trigger / Webhook Trigger
  │
  ├──→ HTTP: POST Notion Content Creation DB query
  │     Filter: Composite Score ≥ 1, Evaluated At in last 30 days
  │     IF: fewer than 20 results → exit (not enough data)
  │
  ├──→ Code: Compute Statistics
  │     - Average per dimension
  │     - Composite trend vs. previous months
  │     - Per-content-type breakdown
  │     - Per-source-type breakdown (RSS vs Slack vs manual)
  │     - Weakest dimensions
  │
  ├──→ HTTP: GET GitHub raw — current prompts + style params
  │
  ├──→ AI Agent: Generate Improvement Proposals
  │     ├── Chat Model: OpenRouter (claude-sonnet-4-6, temp=0.5)
  │     ├── Tool: Perplexity (for current writing best practices)
  │     └── Output: 1-3 specific, testable changes
  │
  ├──→ A/B Test (loop over proposals × 5 past topics)
  │     ├── AI Agent: Regenerate draft with modified config
  │     ├── AI Agent: Evaluate with same rubric
  │     └── Code: Compare old vs new scores
  │
  ├──→ Code: Accept/Reject decisions
  │     Accept if: avg composite +1 AND no dimension -0.5
  │
  ├──→ IF: Accepted changes exist?
  │     └── YES: HTTP: GitHub API — create PR with changes
  │         Branch: ratchet/{date}
  │         Files: modified prompts/style-params
  │         PR body: score data, proposals, test results
  │
  └──→ HTTP: POST Notion comment on Content Production project page
        Ratchet run summary with trends and decisions
```

---

## Evaluation Rubric (Immutable)

From the Content Evaluation Guide:

| Dimension | Weight | Max Score | What It Measures |
|-----------|--------|-----------|------------------|
| Clarity | 1.0× | 5 | Understandable? No jargon without explanation? |
| Structure | 1.0× | 5 | Progressive disclosure? Scannable? |
| Accuracy | 1.5× | 7.5 | Facts correct? Sources cited? |
| Credibility | 1.5× | 7.5 | Shows the work? Earned confidence? |
| Engagement | 1.0× | 5 | Opening hooks? Holds attention? |
| Voice | 1.0× | 5 | Sounds like T-0? Strong opinions? Du-Form? |
| Actionability | 1.0× | 5 | Reader knows what to do next? |
| **Composite** | — | **40** | Weighted sum |

**Quality tiers**: Excellent (34-40), Good (28-33), Acceptable (22-27), Below Standard (16-21), Poor (<16)

**Constitutional gate**: Voice < 4 blocks advancement regardless of composite score.

**Calibration**: Deferred to implementation phase. After 3-5 posts are manually scored by Aaron as calibration anchors, these examples are stored in `prompts/evaluation-calibration.md` and loaded at evaluation time.

---

## Configuration Layer

### Notion Resources DB (Foundational Context)

The T-0 Resources DB in Notion holds foundational texts that shape pipeline behavior. These are loaded at runtime, ensuring the latest versions are always used (Josepha pattern).

**Loaded at generation time**:
1. **Brand Writing Guide** (page `2d1c0fdf-942d-810d-8fda-cf181d93bf46`) — voice, tone, terminology
2. **Blog Content Guide** (page `2d1c0fdf-942d-81cf-bd59-f54a251a1571`) — structural templates per content type
3. **Blog Style Analysis** (page `2d7c0fdf-942d-813e-a679-e55b56cea58b`) — pattern library from industry leaders

### Repository Structure

```
t-0_content-production/
├── .specify/                          # Spec-kit framework
│   ├── memory/constitution.md
│   ├── specs/
│   └── templates/
├── config/
│   └── style-params.yaml             # Ratchet-tunable style parameters
├── prompts/
│   ├── draft-system-prompt.md         # Master drafting system prompt
│   ├── triage-prompt.md               # Source triage prompt template
│   ├── evaluation-prompt.md           # Evaluation prompt template
│   ├── evaluation-calibration.md      # Calibration examples (deferred)
│   ├── research-prompt.md             # Research agent instructions
│   ├── image-description-prompt.md    # Image auto-description prompt
│   └── ratchet-analysis-prompt.md     # Ratchet proposal prompt
├── docs/
│   ├── sources/                       # Reference documents
│   │   ├── blog-writing-assistant-spec.md
│   │   ├── ai-intelligence-pipeline-spec.md
│   │   ├── content-evaluation-guide.md
│   │   ├── blog-content-guide.md
│   │   ├── blog-style-research.md
│   │   └── josepha-pipeline-reference.md
│   └── references/
│       └── existing-infrastructure.md
├── AGENTS.md
├── CLAUDE.md → AGENTS.md
└── .gitignore
```

### style-params.yaml

```yaml
# T-0 Content Pipeline — Style Parameters
# Modifiable by the ratchet loop. Changes committed via PR with score data.
# Each draft records which version was used (git hash in Pipeline Version property).

voice:
  opinion_strength: 0.8          # 0-1: how strongly to state opinions
  personal_anecdote_required: true
  show_the_work: true
  german_first: true             # German output default
  english_path_available: true   # Switch for English output when needed
  max_anglicisms_per_paragraph: 3

structure:
  progressive_disclosure: true
  technical_sections_marked: true
  target_word_count:
    opinion: 1500
    resource: 1200
    case_study: 1000
    technical: 1800
  max_word_count: 2500

opening:
  preferred_patterns:            # weighted selection (ratchet can adjust)
    concrete_scenario: 0.3
    personal_discovery: 0.3
    provocative_observation: 0.25
    news_hook: 0.15

closing:
  preferred_patterns:
    actionable_next_step: 0.25
    open_questions: 0.25
    honest_limitation: 0.2
    perspective_shift: 0.15
    next_level: 0.15

evidence:
  min_external_sources: 2
  cite_style: embedded_hyperlink
  primary_sources_preferred: true

cta:
  type: soft
  default_label: "Schreib uns"
  opinion_pieces_cta: none
```

### Draft System Prompt Structure

```markdown
# System Prompt: T-0 Blog Draft Generation

Du schreibst Content für T-0, eine KI-Transformations-Agentur.

## Voice

{brand_writing_guide_voice_section}

## Structure

{blog_content_guide_template_for_{content_type}}

## Craft Inspiration

{blog_style_research_summary}

## Style Parameters

- Meinungsstärke: {opinion_strength}
- Persönliche Anekdote: {personal_anecdote_required}
- Ziel-Wortanzahl: {target_word_count}
- Öffnungsmuster: {opening_pattern}
- Schlussmuster: {closing_pattern}

## Rules

1. Schreibe auf Deutsch. Anglizismen nur wenn branchenüblich.
2. Du-Form durchgehend.
3. Keine verbotenen Begriffe: {banned_terms_list}
4. Zeige die Arbeit: konkrete Beispiele, Zahlen, Prozesse.
5. Niemals T-0 explizit bewerben. Die Einsicht ist der Pitch.
6. Alle recherchierten Quellen als eingebettete Hyperlinks.
7. Technische Abschnitte mit ### Technisch: markieren.
8. Keine Em-Dashes. Punkte oder Satzumstellung stattdessen.
```

---

## Database Changes

### Content Creation DB — Extend (database_id: `2afc0fdf-942d-813e-bafa-fe13acf0b35f`)

**Existing properties** (all kept):

| Property | Type |
|----------|------|
| Name | title |
| Status | status |
| Category Tags | multi_select |
| URL | url |
| Run Image Gen | button |
| Due Date planned | date |
| Pitch | rich_text |
| Title | rich_text |
| Related Content | relation |
| Description | rich_text |
| Notes | relation (→ Notes DB) |
| Publication Date actual | date |
| Project | relation |
| Target Audience Tags | multi_select |
| Author | people |
| Image Gen Description | rich_text |
| Campaign | relation |
| Platform | multi_select |
| Image | files |
| Lead | relation |
| Reviewer | people |

**New properties to add**:

| Property | Type | Purpose | API-creatable? |
|----------|------|---------|----------------|
| Run Pipeline | button | Single trigger for content pipeline | **No** — Playwright MCP required |
| Content Type | select | Opinion/Resource/Case Study/Technical/Announcement | Yes |
| Clarity Score | number | Eval dimension (1-5) | Yes |
| Structure Score | number | Eval dimension (1-5) | Yes |
| Accuracy Score | number | Eval dimension (1-5) | Yes |
| Credibility Score | number | Eval dimension (1-5) | Yes |
| Engagement Score | number | Eval dimension (1-5) | Yes |
| Voice Score | number | Eval dimension (1-5) | Yes |
| Actionability Score | number | Eval dimension (1-5) | Yes |
| Composite Score | number | Weighted total (max 40) | Yes |
| Quality Tier | select | Excellent/Good/Acceptable/Below Standard/Poor | Yes |
| Evaluation Notes | rich_text | Top strength + weakness + voice assessment | Yes |
| Pipeline Version | rich_text | Git hash of config used | Yes |
| Evaluated At | date | Last evaluation timestamp | Yes |
| Ghost ID | rich_text | Ghost post ID | Yes |
| Source Type | select | RSS/Slack Rant/Manual/Mixed | Yes |

**Status values** (extend existing):

Current values need verification, but the pipeline expects these states: Idea → Backlog → Draft → Pre Review → In Revision → Publishable → Published. Status property values can only be modified via Notion UI or Playwright MCP.

**Notion automation** (manual setup required):

| Button | Automation | Webhook |
|--------|------------|---------|
| Run Pipeline (click) | Send webhook | `POST n8n.t-0.co/webhook/t0-content-pipeline` with page data |

### Notes DB — Extend (database_id: `2afc0fdf-942d-819a-b859-e69dcc48d67d`)

The Notes DB already has most properties needed. Minimal changes:

**New property**:

| Property | Type | Purpose | API-creatable? |
|----------|------|---------|----------------|
| Source | relation → Content Sources DB | Which RSS/Slack source this came from | Yes |

**New Origin values** (auto-created on first use via API):
- `rss-ingestion` — items ingested from RSS feeds
- `slack-auto-capture` — threads captured by timer-based Slack workflow

**Existing properties used by pipeline**:

| Property | Pipeline Usage |
|----------|---------------|
| Name | Article title / discussion topic |
| URL | Article URL / Slack thread link |
| Time | Published date / thread timestamp |
| Status | Backlog → Currently Relevant → Keep/Discarded |
| Origin | rss-ingestion / slack-auto-capture |
| Origin Description | Feed name or channel name |
| Description | RSS summary / Slack thread summary |
| Summary | AI-generated summary |
| AI-Tags | Auto-assigned topic tags |
| AI-Category | Auto-assigned category |
| thread_ts_slack | Slack thread timestamp (Slack items only) |
| channel_id_slack | Slack channel ID (Slack items only) |
| contributors_slack | Thread participants (Slack items only) |
| Content Creation | Relation to Content Creation entry (if item became content) |
| Source | (NEW) Relation to Content Sources DB |

### Content Sources DB — New (in T-0 Main DB)

Create a page "T-0 Content Sources" in the T-0 Main DB (`2afc0fdf-942d-812e-b5db-f7595c7494e9`), with an inline child database.

| Property | Type | Purpose |
|----------|------|---------|
| Name | title | Source name (e.g., "Simon Willison") |
| URL | url | Homepage URL |
| RSS URL | url | RSS/Atom feed URL |
| Type | select | Blog / Newsletter / News / Research / Podcast |
| Language | select | DE / EN |
| Region | select | DACH / US / EU / Global |
| Status | status | Active / Paused / Archived |
| Weight | number | Priority 1-10 for triage (higher = more likely selected) |
| Last Fetched | date | Timestamp of last successful fetch |
| Notes | rich_text | Why this source matters for T-0 |

**Note**: Status property requires Playwright MCP to create (not API-creatable). Alternative: use a select property instead.

**Initial sources** (from blog-style-research.md + team reading list):

| Source | RSS URL | Type | Weight |
|--------|---------|------|--------|
| Simon Willison | `simonwillison.net/atom/everything/` | Blog | 8 |
| Beyond the Hype | `beyondthehype.dev/feed` | Blog | 7 |
| Anthropic Blog | `anthropic.com/rss.xml` | Research | 9 |
| Pragmatic Engineer | `blog.pragmaticengineer.com/rss/` | Newsletter | 6 |
| t3n | `t3n.de/rss.xml` | News | 5 |
| Import AI | `importai.substack.com/feed` | Newsletter | 7 |
| Forte Labs | `fortelabs.com/feed/` | Blog | 4 |
| Stratechery | `stratechery.com/feed/` | Newsletter | 6 |
| Daring Fireball | `daringfireball.net/feeds/main` | Blog | 3 |
| Intercom Blog | `intercom.com/blog/feed/` | Blog | 5 |

---

## Ghost Tag Overhaul

**Current state**: 62 tags, including duplicates. Full cleanup authorized.

**Target taxonomy**:

### Content Type Tags (internal — prefixed with `#`, not visible to readers)

| Tag | Purpose |
|-----|---------|
| `#opinion` | Opinion/analysis pieces |
| `#resource` | How-to, explainer, guide content |
| `#case-study` | Client stories, project retrospectives |
| `#technical` | Deep technical content |
| `#announcement` | Company news, updates |
| `#pipeline-generated` | Content created by the pipeline (tracking) |

### Topic Tags (public)

| Tag | Slug | Scope |
|-----|------|-------|
| KI-Agenten | `ki-agenten` | Autonomous agents, MCP, tool use |
| KI-Transformation | `ki-transformation` | Organizational AI adoption |
| Automatisierung | `automatisierung` | Workflow automation, n8n, tooling |
| LLM | `llm` | Language models, prompt engineering |
| Daten & Analytics | `daten-analytics` | Data strategy, analytics |
| Mittelstand | `mittelstand` | German SME specific |
| Strategie | `strategie` | Strategic considerations |
| Praxis | `praxis` | Practical guides, implementations |
| MCP | `mcp` | Model Context Protocol specifically |

### Audience Tags (internal)

| Tag | Purpose |
|-----|---------|
| `#entscheider` | For decision makers / C-level |
| `#technisch` | For technical practitioners |

**Migration plan**:
1. Export current tags via Ghost API
2. Map each existing tag to the new taxonomy (merge duplicates)
3. Update all existing posts to use new tags
4. Delete orphaned/duplicate tags
5. Implementation uses Ghost Admin API: GET /tags/, PUT /posts/{id}/ (replace tags array)

---

## API Integration Specifications

### OpenRouter (via n8n AI Agent Chat Model)

**n8n node type**: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`
**Default model**: `anthropic/claude-sonnet-4.6` (via OpenRouter)
**Credential**: `openRouterApi` type on n8n.t-0.co

All LLM calls go through OpenRouter. No direct Anthropic API calls. This gives:
- Single credential for all models
- Model switching without credential changes
- Cost tracking through OpenRouter dashboard

| Pipeline Step | Model | Temperature | Est. Input Tokens | Est. Output Tokens |
|---------------|-------|-------------|--------------------|--------------------|
| Research | claude-sonnet-4.6 | 0.4 | ~3,000 | ~2,000 |
| Draft generation | claude-sonnet-4.6 | 0.7 | ~8,000 | ~4,000 |
| Evaluation | claude-sonnet-4.6 | 0.2 | ~6,000 | ~800 |
| Triage | claude-sonnet-4.6 | 0.2 | ~2,000 | ~200 |
| Image description | claude-sonnet-4.6 | 0.5 | ~500 | ~200 |
| Ratchet proposals | claude-sonnet-4.6 | 0.5 | ~10,000 | ~2,000 |

**Monthly cost estimate** (8 drafts/month, each with eval + 1 revision avg):
- Via OpenRouter: ~$5-8/month total (OpenRouter markup ~10-20% over direct API)

### Notion API

**Version**: `2022-06-28` (forced by n8n `notionApi` credential)
**Auth**: Integration token (Bearer)

| Endpoint | Purpose | Workflows |
|----------|---------|-----------|
| `/databases/{id}/query` | Query Content Sources, Content Creation, Notes | WF-1, WF-2 |
| `/pages` | Create notes, content ideas | WF-2 |
| `/pages/{id}` | Update scores, status, properties | WF-1, WF-3 |
| `/pages/{id}` | Read page properties | WF-1 |
| `/blocks/{id}/children` | Read/append page body, wiki pages | WF-1, WF-3 |
| `/comments` | Create status notifications | WF-1 |

**Known pitfalls**:
- `notionApi` forces `Notion-Version: 2022-06-28`
- Use `database_id` (not `data_source_id`) with `/databases/`
- `status` properties accept `select` syntax with `2022-06-28`
- Block append limit: 100 per PATCH, batch longer content
- Button and status property creation require Playwright MCP

### Ghost Admin API

**Base URL**: `https://t-0.co/ghost/api/admin/`
**Auth**: JWT (HS256, 5min expiry, audience `/admin/`)
**Key**: Dedicated "Content Pipeline" integration (to be created), stored in n8n credential management

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/posts/` | POST | Create draft/scheduled post |
| `/posts/{id}/` | PUT | Update post (schedule, set feature image) |
| `/posts/{id}/` | GET | Verify creation, get updated_at for PUT |
| `/images/upload/` | POST | Upload featured image |
| `/tags/` | GET | List existing tags (for cleanup) |
| `/tags/{id}/` | DELETE | Remove duplicate/orphaned tags |

**Content format**: `?source=html` on POST/PUT — Ghost converts HTML to Lexical internally.

**Newsletter fields on posts**:
```json
{
  "newsletter": { "slug": "default-newsletter" },
  "email_only": false
}
```

**Known pitfalls**:
- `updated_at` required for PUT — always GET first, PUT with fresh timestamp
- Tags and authors REPLACED, not merged — always send complete arrays
- Ghost URL is `t-0.co` (not `blog.t-0.co`)
- Current newsletter sender: `team@chamaeleon.community` (may need updating)

### Perplexity API (as AI Agent Tool)

Configured as an HTTP Request tool within AI Agent nodes. The agent calls it when web-grounded research is needed.

**Base URL**: `https://api.perplexity.ai/chat/completions`
**Auth**: Bearer token
**Model**: `sonar-pro` (used by the agent through the tool)

### Jina Reader API (as AI Agent Tool)

Configured as an HTTP Request tool within AI Agent nodes. The agent calls it to extract article full text from URLs.

**Call pattern**: `GET https://r.jina.ai/{encoded_url}` → returns markdown
**Content cap**: 8,000 characters

### Slack API

**Base URL**: `https://slack.com/api/`
**Auth**: Bot OAuth token (existing T-0 Slack bot)
**Required scopes**: `channels:history`, `channels:read`

| Endpoint | Purpose |
|----------|---------|
| `conversations.history` | Fetch #tech-shit-talk messages |
| `conversations.replies` | Fetch thread replies |

### Google Gemini API

**Usage**: Image generation only (WF-3)
**Model**: Latest image generation model
**Input**: Text prompt + optional reference image (base64)

### Google Drive API

**Credential**: `googleOAuth2Api` for `aaron@t-0.co`
**Folder**: "Image Generation Dump for Blog" (`13CQ-Y2CRITlUn9fc28LnLLXoVeRtAVLE`)
**Usage**: Upload generated images, set sharing permissions

---

## Error Handling & Recovery

| Error | Detection | Response | Recovery |
|-------|-----------|----------|----------|
| RSS feed unreachable | HTTP non-200/timeout | Log, skip source, continue | Next scheduled run retries |
| Slack API rate limited | HTTP 429 | n8n built-in retry with backoff | Auto-retry |
| Triage returns invalid JSON | Parse error | Fall back to top N by source weight | No manual action |
| Draft generation fails | API error / empty response | Status unchanged, error comment in Notion | Re-press "Run Pipeline" |
| Evaluation scores out of range | Validation (score <1 or >5) | Re-run with explicit range instruction | Auto-retry once |
| Notion API 409 conflict | HTTP 409 | Fetch latest version, re-apply | Auto-retry |
| Ghost post creation fails | API error | Status stays "Publishable", error in Notion | Re-press "Run Pipeline" |
| Ghost image upload fails | API error | Post created without image | Trigger WF-3 separately |
| Jina returns empty/paywalled | Content <100 chars | Fall back to Perplexity research | Draft includes disclaimer |
| Gemini image gen fails | API error | Post proceeds without image | Manual "Run Image Gen" |
| OpenRouter rate limit/error | API error | n8n retry with backoff | Auto-retry (3 attempts) |
| n8n workflow timeout | Exceeds default 5min | Execution fails, logged in n8n | Set WF-1 timeout to 10min |
| Ratchet A/B inconclusive | Score diff <1 point | Reject proposal (conservative) | Next monthly run |

### Monitoring

n8n execution history serves as the monitoring dashboard. Each workflow execution is logged with status, duration, and error details.

Optional enhancement: n8n → Telegram notification on WF-1/WF-2 failure (existing pattern in Aaron's stack).

---

## Patterns Borrowed from Josepha

| Pattern | Josepha | T-0 Adaptation |
|---------|---------|----------------|
| Fan-out/aggregate | Split array → process → aggregate | Same — RSS feeds, block batches |
| Fire-and-forget webhook | Acknowledge → process async | Same — all webhook workflows |
| Brand context loaded at runtime | Fetch Notion wiki pages per run | Same — 3 guide pages + repo config |
| JSON repair parser | Regex + trailing comma + newline escape | Same — triage and evaluation JSON |
| Retry with lower temperature | On parse failure, retry at temp-0.1 | Same |
| Markdown-to-Notion-blocks | h1-h3, bullets, quotes, links, etc. | Same converter, extended for callouts |
| Dedup via URL set-diff | Compare against existing entries | Same |
| Status state machine | New → Used | Extended: Idea → ... → Published |

**Key difference**: Josepha terminates at Notion. T-0 adds Ghost publication, evaluation feedback loop, and the ratchet.

---

## Success Criteria

### SC-1: Draft Quality
Pipeline-generated drafts consistently score ≥28 composite (Good tier) within the first month. After 3 months with ratchet active, average reaches ≥32.

### SC-2: Voice Authenticity
Voice dimension averages ≥4.0. Aaron and Julien cannot reliably distinguish pipeline drafts from human-written drafts in a blind test (after review and minor edits).

### SC-3: Existing Entry Processing
All existing Content Creation DB entries processed to publishable quality within the first month. Publication schedule established.

### SC-4: Source Coverage
Pipeline surfaces ≥5 relevant topics per week, of which Aaron promotes ≥2 to drafting.

### SC-5: Time Savings
"Backlog" to "Pre Review" in under 10 minutes of pipeline execution, replacing 2-4 hours of manual work.

### SC-6: Publication Cadence
2-4 blog posts per month (up from current sporadic cadence), all scoring ≥28 composite.

### SC-7: Newsletter
Published posts automatically distributed via Ghost newsletter to subscribers.

### SC-8: Ratchet Improvement
After 3 months, ratchet has proposed and applied ≥2 changes with demonstrable score improvement.

### SC-9: Zero Unauthorized Publications
No content reaches Ghost in published state without Aaron's explicit action. All pipeline-created Ghost posts are `status: "draft"` or `status: "scheduled"` (with human-set date).

---

## Implementation Phases

| Phase | Scope | Priority | Dependencies |
|-------|-------|----------|-------------|
| **Phase A: Core Pipeline** | WF-1 (all three branches). Process existing entries. | P0 — build first | OpenRouter credential on n8n.t-0.co, Ghost "Content Pipeline" key, Notion properties + button |
| **Phase B: Source Ingestion** | WF-2 (RSS + Slack). Content Sources DB. Notes DB extensions. | P2 | Content Sources DB created, Slack bot permissions verified |
| **Phase C: Image Gen Integration** | WF-3 improvements. Integration with WF-1 Ghost branch. | P2 | Existing image gen workflow on n8n.t-0.co |
| **Phase D: Ghost Tag Overhaul** | Clean up 62 tags, establish taxonomy, update posts. | P1 (parallel with A) | Ghost API access |
| **Phase E: Ratchet Activation** | WF-4. Score analysis, A/B testing, PR generation. | P3 — after data accumulates | 20+ evaluated drafts, 2-3 months of operation |
| **Phase F: Newsletter Optimization** | Ghost newsletter config, sender identity, subscriber management. | P2 | Ghost admin access |
| **Future: Social Distribution** | LinkedIn, other platforms. Downstream from Ghost. | Roadmap | Core pipeline stable, platform APIs |

**Phase A is the immediate priority.** It unblocks processing of existing entries, which is the primary business reason for building this pipeline.

### Phase A Prerequisites

1. **Create OpenRouter credential** on n8n.t-0.co (key from workspace `.env`)
2. **Create Ghost "Content Pipeline" integration** in Ghost Admin → store in n8n credentials
3. **Add new properties** to Content Creation DB (number, select, rich_text, date — all API-creatable)
4. **Create "Run Pipeline" button** on Content Creation DB (requires Playwright MCP)
5. **Set up Notion automation**: button click → webhook to n8n
6. **Verify Status values** on Content Creation DB — add missing states if needed (Playwright MCP)
7. **Build WF-1** on n8n.t-0.co

---

## Design Decisions Reference

All 22 questions resolved during design review with Aaron (2026-03-24):

| # | Decision | Resolution |
|---|----------|------------|
| DR-1 | n8n instance | `n8n.t-0.co` confirmed. 29 active workflows. |
| DR-2 | LLM access | OpenRouter + AI Agent nodes. Claude Sonnet 4.6 default for all steps. |
| DR-3 | Ghost URL | `t-0.co` (not `blog.t-0.co`). Dedicated "Content Pipeline" key. |
| DR-4 | Content Creation DB | Extend existing with full freedom. Breaking changes allowed. |
| DR-5 | Feed item storage | Use existing Notes DB (not new Feed Items DB). |
| DR-6 | Content Sources DB | New, in T-0 Main DB pattern (page → child database). |
| DR-7 | Pipeline monitoring | n8n execution history. No separate Pipeline Runs DB. |
| DR-8 | Slack ingestion | Timer-based. 2+ replies from 2+ different people. Replaces emoji workflow. |
| DR-9 | Workflow architecture | Single-button trigger (WF-1) + scheduled ingestion (WF-2) + modular image gen (WF-3). |
| DR-10 | Content type | Auto-detect + propose. Human confirms before next pipeline run. |
| DR-11 | Ghost publish logic | Conditional: Due Date planned set + Publishable → schedule. Otherwise → draft. |
| DR-12 | Image generation | Modular workflow, triggered from pipeline or standalone button. May improve along the way. |
| DR-13 | Prompt language | English system prompts, German output default. Switch for English path when needed. |
| DR-14 | Config loading | Notion Resources DB for brand context (runtime). Repo for prompts/style params (versioned). |
| DR-15 | Evaluation calibration | Deferred. Ratchet dormant for first weeks until enough data exists. |
| DR-16 | Ratchet scope | Full autonomy: prompts, parameters, AND source selection. Versioned via PRs. |
| DR-17 | Newsletter | In scope. Ghost native. Posts sent as newsletter on publish. |
| DR-18 | Social distribution | Roadmap, not v1. Blog is canonical source, platforms downstream. |
| DR-19 | Existing entries | High priority. This is why the pipeline is being built now. |
| DR-20 | Ghost API key | Dedicated "Content Pipeline" integration. Stored in n8n credential management. |
| DR-21 | Agent autonomy | Mostly deterministic. Agents get 2-3 tools max, clearly scoped tasks. No free planning. |
| DR-22 | Notes DB changes | Add `Source` relation + new Origin values. Full trust to design properties. |
| DR-23 | Ghost tag cleanup | Full overhaul authorized. Complete taxonomy redesign. No restrictions on destructive changes. |
