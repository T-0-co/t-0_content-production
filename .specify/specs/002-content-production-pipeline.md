# Spec 002: T-0 Content Production Pipeline — Full System Design

**Created**: 2026-03-24
**Status**: Draft (Design Review)
**Type**: System Design — comprehensive specification for discussion before implementation
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

---

## Actors & Roles

| Actor | Role in Pipeline | Touchpoints |
|-------|------------------|-------------|
| **Aaron** | Editorial lead. Enters topic ideas, reviews drafts, approves publication, tunes pipeline parameters, monitors ratchet performance. | Notion (ideas, reviews), Ghost Admin (final check), this repo (config) |
| **Julien** | Co-founder. Posts in #tech-shit-talk (raw material for pipeline), occasionally reviews drafts. | Slack (source material), Notion (review) |
| **Pipeline (n8n)** | Autonomous agent. Ingests sources, triages, generates drafts, evaluates, generates images. Runs on schedule + on-demand. | n8n workflows on `n8n.t-0.co` |
| **Claude (in conversation)** | Ad-hoc assistant. Helps Aaron refine drafts, debug pipeline issues, analyze ratchet data. | Claude Code / MCP tools |

---

## User Stories

### US-1 — Aaron Enters a Topic Idea and Gets a Draft (Priority: P1)

Aaron has a topic in mind — maybe from a client conversation, a news article, or a Slack rant. He creates an entry in the Content Creation DB with a name, pitch, and optional source URL. The pipeline picks it up, researches the topic, generates a draft following T-0's brand voice, evaluates it, and presents the scored draft in Notion for review.

**Why P1**: This is the core value loop. Everything else builds on this.

**Independent test**: Create a Content Creation DB entry with status "Backlog" → pipeline generates a scored draft within the same Notion page → Aaron reads it and can provide feedback.

**Acceptance scenarios**:

1. **Given** a Content Creation entry with Name="MCP vs A2A: Was Mittelständler wissen müssen", Pitch="Vergleich der Protokolle...", Status="Backlog", **When** the pipeline runs, **Then** a draft appears in the page body, all 7 evaluation scores are written to properties, and Status advances to "Pre Review" (if scores meet threshold) or stays at "Draft" (if below).

2. **Given** a Content Creation entry with a source URL pointing to a news article, **When** the pipeline generates a draft, **Then** the draft incorporates specific facts and quotes from that source article (fetched via Jina Reader), properly cited as embedded hyperlinks per Brand Writing Guide v2.2.

3. **Given** a Content Creation entry with no pitch (just a name), **When** the pipeline runs, **Then** it still generates a draft by researching the topic via Perplexity, but flags it as "low-context — human input recommended" in a Notion callout block.

---

### US-2 — Weekly Source Ingestion Surfaces New Topics (Priority: P2)

Every week, the pipeline automatically fetches RSS feeds from tracked sources, reads recent Slack #tech-shit-talk discussions, deduplicates against previously seen items, and uses AI triage to select the most relevant topics for T-0's audience. Selected topics appear as new "Idea" entries in the Content Creation DB, ready for Aaron to promote to "Backlog" (triggering US-1).

**Why P2**: Without source ingestion, Aaron manually finds and enters every topic. This automates the discovery half of content creation.

**Independent test**: Configure 5 RSS sources → run ingestion → see new Idea entries appear in Content Creation DB with source links and AI-generated pitches.

**Acceptance scenarios**:

1. **Given** 5 active RSS sources in the Sources DB and the pipeline runs on Sunday 8AM, **When** ingestion completes, **Then** new feed items are stored (deduplicated against previous weeks), AI triage selects top 5-8 topics, and each becomes a Content Creation entry with Status="Idea", a generated Pitch, the source URL, and Category Tags.

2. **Given** a Slack #tech-shit-talk thread where Julien ranted about a new AI tool, **When** the pipeline runs, **Then** the rant is captured, summarized, and appears as an Idea entry with the original Slack thread linked and the rant's key arguments preserved in the Description field.

3. **Given** that a topic was already ingested last week and still exists in the Content Creation DB, **When** a new article about the same topic appears in RSS this week, **Then** the pipeline adds the new URL to the existing entry's Related Content rather than creating a duplicate.

---

### US-3 — Automated Evaluation Scores Every Draft (Priority: P1)

Every draft produced by the pipeline (or written manually and placed in the page body) gets automatically evaluated against the Content Evaluation Guide rubric. Seven dimension scores plus a weighted composite are written to Notion properties. Drafts scoring ≥28 composite with no dimension <3 and Voice ≥4 advance to "Pre Review." Drafts below threshold stay at "Draft" with specific feedback.

**Why P1**: The evaluator IS the autoresearch ratchet's fixed point. Without automated evaluation, there's no feedback loop.

**Independent test**: Manually paste any draft text into a Content Creation page body → trigger evaluation → scores appear in properties within 60 seconds.

**Acceptance scenarios**:

1. **Given** a draft in a Content Creation page body, **When** evaluation runs, **Then** seven dimension scores (Clarity, Structure, Accuracy, Credibility, Engagement, Voice, Actionability) are written to number properties, plus a Composite Score (weighted, max 40), plus a Quality Tier text (Excellent/Good/Acceptable/Below Standard/Poor), plus dimension-specific feedback in a "Evaluation Notes" block at the bottom of the page.

2. **Given** a draft that scores Voice=3 but Composite=30, **When** evaluation completes, **Then** Status remains "Draft" (not advanced to "Pre Review") and the evaluation notes specifically call out: "Voice score below minimum threshold (4). Draft does not advance regardless of composite score."

3. **Given** a draft that scores Composite=35 with all dimensions ≥4, **When** evaluation completes, **Then** Status advances to "Pre Review" and a Notion comment is created: "Draft ready for human review. Composite: 35/40 (Excellent)."

---

### US-4 — Aaron Reviews and Approves a Draft (Priority: P1)

Aaron opens a "Pre Review" draft in Notion, reads the content and evaluation scores, and either approves it (moves to "Publishable"), requests revisions (moves to "In Revision" with comments), or rejects it (moves back to "Idea" or archives). When moved to "In Revision," the pipeline can optionally re-draft based on Aaron's feedback and re-evaluate.

**Why P1**: The human gate is a constitutional principle. The pipeline must support this workflow cleanly.

**Independent test**: Move a scored draft to "In Revision" with a comment "Zu generisch, mehr eigene Erfahrung einbauen" → pipeline regenerates with that feedback incorporated → new scores appear.

**Acceptance scenarios**:

1. **Given** a "Pre Review" draft, **When** Aaron moves Status to "Publishable", **Then** no automated action occurs — the draft waits for explicit publication trigger (US-5).

2. **Given** a "Pre Review" draft, **When** Aaron moves Status to "In Revision" and adds a Notion comment with feedback, **Then** the pipeline reads the comment, regenerates the draft incorporating the feedback while preserving the original draft (appended as a collapsed "Previous Version" toggle block), and re-evaluates. Status returns to "Draft" → "Pre Review" if scores pass.

3. **Given** a "Pre Review" draft, **When** Aaron moves Status to "Idea" (rejection), **Then** no further automated action. The entry returns to the idea pool.

---

### US-5 — Publishable Draft Goes to Ghost (Priority: P2)

When Aaron moves a draft to "Publishable," the pipeline converts it to Ghost-compatible HTML, uploads the featured image, creates a Ghost draft post with proper tags/SEO/excerpt, and notifies Aaron. Aaron does a final visual check in Ghost Admin and hits publish (or schedules).

**Why P2**: Publication is the output of the pipeline but depends on P1 stories working first.

**Independent test**: Move a "Publishable" entry to trigger Ghost draft creation → verify the post appears in Ghost Admin at blog.t-0.co with correct formatting, tags, and image.

**Acceptance scenarios**:

1. **Given** a "Publishable" Content Creation entry with a draft body, featured image, and Category Tags, **When** the publication workflow triggers, **Then** a Ghost draft post is created at `blog.t-0.co` with: title from the entry's Title property (or Name if Title is empty), HTML body converted from the Notion blocks, `custom_excerpt` from the Pitch property, tags mapped from Category Tags, author set to the entry's Author, `feature_image` uploaded via Ghost image API, and `meta_title`/`meta_description` auto-generated from the content.

2. **Given** the Ghost draft is created, **When** the workflow completes, **Then** the Content Creation entry's Status advances to "Published", the Ghost post URL is written to the URL property, and the Publication Date actual is set to now.

3. **Given** that Aaron wants to schedule a post for next Tuesday, **When** he sets the Due Date planned property before moving to "Publishable", **Then** the Ghost post is created with `status: "scheduled"` and `published_at` set to Due Date planned at 09:00 CET.

---

### US-6 — Branded Image Generation for Each Post (Priority: P3)

Each publishable post gets a branded illustration generated using the existing Gemini image generation workflow. The image follows T-0's illustration style, incorporates the post's topic, and is uploaded to Google Drive and embedded in both Notion and Ghost.

**Why P3**: Images enhance posts but aren't blocking for the core pipeline. The image gen workflow already exists (Josepha + n8n.t-0.co).

**Independent test**: Click "Run Image Gen" button on a Content Creation entry → image appears in the Image property and as an embedded block in the page body.

**Acceptance scenarios**:

1. **Given** a Content Creation entry with an Image Gen Description, **When** the image generation workflow triggers, **Then** the description is enhanced via Perplexity (incorporating T-0 illustration style guide), Gemini generates an image, the image is uploaded to Google Drive, the Drive URL is written to the Image property, and an image block is appended to the Notion page.

2. **Given** a Content Creation entry with NO Image Gen Description, **When** image generation triggers, **Then** the pipeline auto-generates an image description from the post title and pitch, proceeding as in scenario 1.

---

### US-7 — Self-Improvement Ratchet (Priority: P3)

Monthly, the pipeline analyzes evaluation scores across all drafts produced in the period, identifies which prompt/parameter changes correlated with score improvements, and proposes configuration updates. Changes are applied only if they demonstrably improve scores on a held-out set of past topics.

**Why P3**: The ratchet is the long-term differentiator but requires US-1 through US-3 generating enough data first. Likely relevant after 2-3 months of pipeline operation.

**Independent test**: After 20+ evaluated drafts exist, run the ratchet analysis → receive a report of which dimensions improved/degraded and proposed prompt adjustments.

**Acceptance scenarios**:

1. **Given** 20+ evaluated drafts in the Content Creation DB, **When** the ratchet analysis runs, **Then** it produces a report (Notion page) showing: score trends per dimension, average composite over time, best/worst performing content types, and specific prompt parameter change proposals.

2. **Given** a proposed prompt change (e.g., "add more concrete examples in opening"), **When** the ratchet tests it against 5 previously-generated topics (regenerating drafts with the new prompt and re-evaluating), **Then** it accepts the change only if the average composite improves by ≥1 point AND no dimension score decreases by >0.5 on average.

3. **Given** that a ratchet-proposed change is accepted, **When** it is applied, **Then** the change is committed to this repo's prompt templates with a commit message referencing the ratchet run, the score improvement, and the specific change made.

---

### US-8 — Slack Rant Augmentation (Priority: P2)

Discussions in #tech-shit-talk often contain raw, opinionated takes that embody T-0's voice. The pipeline captures these, extracts the core arguments and strong opinions, and injects them into draft generation as "voice seeds" — ensuring drafts have the authentic edge that comes from real conversations.

**Why P2**: This is what makes T-0 content sound like T-0, not like generic AI consultancy content. The rants are the raw voice material.

**Independent test**: Post a rant in #tech-shit-talk about a topic → see it appear as source material linked to a Content Creation entry → verify the generated draft incorporates the rant's key arguments and tone.

**Acceptance scenarios**:

1. **Given** Julien posts in #tech-shit-talk: "ChatGPT ist Spielzeug. Wer damit ernsthaft arbeiten will, braucht Agents mit echtem Tool-Zugang", **When** the pipeline ingests this, **Then** it creates a feed item capturing: the original text, the author, the timestamp, extracted arguments ("ChatGPT = toy for serious work", "agents need real tool access"), and a suggested topic angle.

2. **Given** a Content Creation entry about AI agents with a linked Slack rant, **When** draft generation runs, **Then** the draft incorporates the rant's strong opinion ("Spielzeug für den normalen Nutzer") as a voice seed — not as a quote, but as a tonal anchor that influences the draft's stance and language.

---

### Edge Cases

- **Empty RSS feed**: Source returns 200 but no items → log, skip, don't create empty feed items
- **Duplicate topic from multiple sources**: Same news covered by 3 RSS feeds → dedup by URL first, then by semantic similarity (Perplexity) for different URLs about the same event
- **Draft generation fails**: API error mid-generation → status stays at "Backlog", error logged to pipeline run entry, Notion comment added: "Draft generation failed: [error]. Will retry on next run."
- **Evaluation produces inconsistent scores**: E.g., Clarity=5 but Structure=1 → flag as anomaly, request re-evaluation with explicit reasoning
- **Ghost API quota/rate limits**: Ghost Admin API has no published rate limits, but batch operations should use 1-second delays between posts
- **Notion API pagination**: Content Creation DB queries may exceed 100 items → all queries must handle pagination (start_cursor + has_more)
- **Very long drafts**: Notion blocks API accepts max 100 blocks per PATCH → batch block appends (Josepha pattern)
- **Stale Slack rants**: Rant is 3+ weeks old → still capture but tag as "stale" for lower priority in triage
- **Conflicting human edits**: Aaron edits a draft in Notion while pipeline is also updating → pipeline should always fetch latest version before writing, use Notion's last_edited_time as collision guard
- **Image generation fails**: Gemini API error → post proceeds without image, Image property stays empty, Notion comment: "Image generation failed. Generate manually via Run Image Gen button."
- **Source URL is paywalled**: Jina Reader returns partial/no content → fall back to RSS summary + Perplexity research, flag in draft: "Source article was paywalled; content based on available summary."

---

## System Architecture

### Component Map

```
┌─────────────────────────────────────────────────────────────┐
│                    THIS REPO (Config Layer)                   │
│  prompts/  ·  sources.yaml  ·  style-params.yaml  ·  docs/  │
└──────────────────────────┬──────────────────────────────────┘
                           │ consumed by
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                n8n.t-0.co (Orchestration Layer)               │
│                                                               │
│  WF-1: Source Ingestion ──→ WF-2: Draft Generation           │
│           │                        │                          │
│           ▼                        ▼                          │
│  WF-3: Evaluation ←──────── WF-5: Image Gen                 │
│           │                                                   │
│           ▼                                                   │
│  WF-4: Ghost Publication                                     │
│           │                                                   │
│  WF-6: Ratchet Analysis (monthly)                            │
└──────┬─────────┬──────────┬──────────┬──────────┬───────────┘
       │         │          │          │          │
       ▼         ▼          ▼          ▼          ▼
   ┌───────┐ ┌───────┐ ┌────────┐ ┌────────┐ ┌────────┐
   │Notion │ │Ghost  │ │Perplx. │ │Claude  │ │Gemini  │
   │Content│ │CMS    │ │sonar   │ │Opus/   │ │imagen  │
   │DB     │ │blog.  │ │-pro    │ │Sonnet  │ │        │
   │       │ │t-0.co │ │        │ │        │ │        │
   └───┬───┘ └───────┘ └────────┘ └────────┘ └────────┘
       │
       ▼
   ┌───────┐  ┌───────┐  ┌───────┐
   │Sources│  │Feed   │  │Pipeline│
   │DB     │  │Items  │  │Runs DB │
   │(new)  │  │DB(new)│  │(new)   │
   └───────┘  └───────┘  └───────┘

External Inputs:
  ├── RSS Feeds (via HTTP GET)
  ├── Slack #tech-shit-talk (via Slack API / MCP Hub)
  ├── Jina Reader (article full-text extraction)
  └── Manual entry (Aaron → Notion)
```

### Data Flow (End to End)

```
RSS Feeds ──→ ┐
Slack Rants ──→├──→ [WF-1: Ingest] ──→ Feed Items DB ──→ [AI Triage]
Manual Ideas ─→┘                              │
                                              ▼
                                    Content Creation DB
                                    (Status: "Idea")
                                              │
                                    Aaron promotes to "Backlog"
                                              │
                                              ▼
                            [WF-2: Draft Generation]
                            ├── Fetch source material (Jina, Perplexity)
                            ├── Load brand context (Writing Guide, Style Research)
                            ├── Claude generates draft
                            └── Write draft to Notion page body
                                              │
                                              ▼
                            [WF-3: Evaluation]
                            ├── Read draft from Notion
                            ├── Claude scores against rubric
                            ├── Write 7 scores + composite to properties
                            └── Advance status if passing
                                              │
                                    ┌─────────┴─────────┐
                                    ▼                   ▼
                            Status: "Pre Review"  Status: "Draft"
                            (scores pass)         (scores fail)
                                    │                   │
                            Aaron reviews         Feedback loop
                                    │             (US-4 revision)
                                    ▼
                            Status: "Publishable"
                                    │
                        ┌───────────┤
                        ▼           ▼
                [WF-5: Image] [WF-4: Ghost Publish]
                        │           │
                        ▼           ▼
                Google Drive    Ghost CMS draft
                + Notion img    at blog.t-0.co
                                    │
                            Aaron publishes in Ghost
                                    │
                                    ▼
                            Status: "Published"
                            (URL + date written back)
```

### Technology Stack Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Orchestration** | n8n at `n8n.t-0.co` | T-0's dedicated instance. Constitution mandates. Image gen workflow already here. |
| **Content DB** | Notion Content Creation DB | Already exists, team uses it daily, mature status pipeline. |
| **Publishing** | Ghost CMS at `blog.t-0.co` | Already running, has Admin API, handles newsletters. |
| **Drafting model** | Claude (Anthropic) | Voice-sensitive work requires Claude's nuance with complex style guides. Perplexity lacks the fine-grained instruction following needed for T-0 voice. |
| **Research model** | Perplexity `sonar-pro` | Web access, citations, proven in Josepha pipeline. |
| **Triage model** | Perplexity `sonar-pro` | Same as Josepha — select relevant items from a list. |
| **Evaluation model** | Claude (Anthropic) | Structured JSON output against a 7-dimension rubric requires strong instruction following. |
| **Image generation** | Google Gemini | Already operational in Josepha + n8n.t-0.co. Style guide and reference image workflow proven. |
| **Ghost content format** | HTML via `?source=html` | Simpler than building Lexical JSON. Sufficient for standard blog content. Upgrade to Lexical later if card features needed. |
| **Slack ingestion** | n8n Slack node (conversations.history) | Direct API access gives full control. MCP Hub `slack-rant-augmentation` skill is an alternative but adds a dependency. |
| **Article extraction** | Jina Reader API (`r.jina.ai`) | Proven in Josepha. Returns clean markdown from URLs. |
| **Config storage** | This repo (`prompts/`, `config/`) | Git-versioned, diffable, the ratchet commits changes here. |

### n8n Instance Access

The T-0 n8n instance at `n8n.t-0.co` is separate from kokal-central's `automation.kokal.cloud`.

**Current state**: Claude's `mcp__claude_ai_n8n__*` MCP tools connect to `automation.kokal.cloud`, NOT to `n8n.t-0.co`. The Josepha workflows run on `automation.kokal.cloud`.

**Implication for this pipeline**: All workflows will be built on `n8n.t-0.co`. For Claude Code to interact with them (list, inspect, trigger), one of these is needed:
1. Add a second n8n MCP server in `.mcp.json` pointing to `n8n.t-0.co` (preferred)
2. Use MCP Hub `n8n` module reconfigured for `n8n.t-0.co`
3. Use direct `curl` API calls with a `n8n.t-0.co` API key

**Action needed before implementation**: Verify `n8n.t-0.co` is reachable, has an Admin API key, and has the required credentials (Notion, Perplexity, Anthropic, Gemini, Ghost, Slack, Jina, Google Drive).

---

## Pipeline Phases — Detailed Design

### Phase 1: Source Ingestion

**Trigger**: Weekly schedule (Sunday 8:00 CET) + manual webhook
**n8n Workflow**: WF-1
**Inputs**: Sources DB, Slack #tech-shit-talk
**Outputs**: Feed Items DB entries, Content Creation DB "Idea" entries

#### 1a. RSS Feed Ingestion

**Sources DB** (new Notion database — child of Content Production project page):

| Property | Type | Purpose |
|----------|------|---------|
| Name | title | Source name (e.g., "Simon Willison") |
| URL | url | Homepage URL |
| RSS URL | url | RSS/Atom feed URL |
| Type | select | `Blog`, `Newsletter`, `News`, `Research`, `Podcast` |
| Language | select | `DE`, `EN` |
| Region | select | `DACH`, `US`, `EU`, `Global` |
| Status | status | `Active`, `Paused`, `Archived` |
| Weight | number | 1-10 priority score for triage (higher = more likely selected) |
| Last Fetched | date | Timestamp of last successful fetch |
| Notes | rich_text | Why this source matters for T-0 |

**Initial sources** (from blog-style-research.md + team reading list):

| Source | RSS URL | Type | Weight |
|--------|---------|------|--------|
| Simon Willison | `https://simonwillison.net/atom/everything/` | Blog | 8 |
| Beyond the Hype | `https://beyondthehype.dev/feed` | Blog | 7 |
| Anthropic Blog | `https://www.anthropic.com/rss.xml` | Research | 9 |
| Pragmatic Engineer | `https://blog.pragmaticengineer.com/rss/` | Newsletter | 6 |
| t3n | `https://t3n.de/rss.xml` | News | 5 |
| Import AI | `https://importai.substack.com/feed` | Newsletter | 7 |
| Forte Labs | `https://fortelabs.com/feed/` | Blog | 4 |
| Stratechery | `https://stratechery.com/feed/` | Newsletter | 6 |
| Daring Fireball | `https://daringfireball.net/feeds/main` | Blog | 3 |
| Intercom Blog | `https://www.intercom.com/blog/feed/` | Blog | 5 |

**RSS fetch flow** (per source):

1. HTTP GET `{rss_url}` with `Accept: application/rss+xml, application/atom+xml`
2. Parse response (regex parser, Josepha pattern — handles RSS 2.0 `<item>` + Atom `<entry>`)
3. Filter items to last 7 days (by `pubDate` or `published`)
4. Extract per item: `title`, `url`, `published`, `summary` (strip HTML), `author`
5. Dedup against Feed Items DB by URL

**Feed Items DB** (new Notion database):

| Property | Type | Purpose |
|----------|------|---------|
| Name | title | Article title |
| URL | url | Article URL |
| Source | relation → Sources DB | Which source this came from |
| Published | date | Original publication date |
| Fetched | date | When pipeline fetched it |
| Summary | rich_text | RSS summary or AI-generated summary |
| Status | status | `New`, `Triaged`, `Selected`, `Used`, `Skipped` |
| Tags | multi_select | AI-assigned topic tags |
| Content | relation → Content Creation DB | If this feed item became a content idea |
| Full Text | rich_text | Jina-extracted full text (truncated to 6000 chars) |

#### 1b. Slack #tech-shit-talk Ingestion

**Mechanism**: n8n Slack node using `conversations.history` API

**Flow**:
1. Fetch messages from #tech-shit-talk for last 7 days
2. Filter: only messages with >2 replies OR >3 reactions OR length >200 chars (skip "lol" and link-only posts)
3. For threaded discussions: fetch thread replies via `conversations.replies`
4. Extract: author, timestamp, full text (message + thread), any URLs shared
5. Store as Feed Items DB entry with Type="Slack Rant", Source relation to a special "Slack #tech-shit-talk" source entry

**Slack credentials needed on n8n.t-0.co**:
- OAuth token with scopes: `channels:history`, `channels:read`, `groups:history` (if private channel)
- Channel ID for #tech-shit-talk

**Content extraction from shared URLs**: If a Slack message contains a URL, fetch the linked article via Jina Reader and store the full text alongside the rant. This gives the triage step both the rant's take AND the source material.

#### 1c. AI Triage

After ingestion, the triage step selects which feed items are most relevant for T-0 content.

**Input**: All `New` feed items from the current week
**Model**: Perplexity `sonar-pro`, temperature 0.2
**Output**: 5-8 selected items advanced to `Selected` status

**Triage prompt structure**:
```
You are a content strategist for T-0, a German AI transformation agency.
T-0's audience: German Mittelstand decision-makers exploring AI adoption.
T-0's stance: Pragmatic, show-the-work, strong opinions, anti-hype.

Here are this week's {n} articles/rants:

{numbered list with: title, source, summary, source weight, tags}

Select the 5-8 most relevant items for T-0 blog content. Prioritize:
1. Topics where T-0 has direct experience or a strong opinion
2. Items from higher-weighted sources
3. German/European angles over US-only stories
4. Practical implications over theoretical advances
5. Slack rants (they contain T-0's authentic voice)

Return a JSON array of selected item numbers: [1, 4, 7, ...]
```

**Post-triage actions**:
- Selected items: Status → `Selected`, deep-fetch full text via Jina Reader
- For each selected item: Create a Content Creation DB entry with Status="Idea", link to feed item, auto-generated Pitch from the article summary + triage reasoning
- Non-selected items: Status → `Skipped`

---

### Phase 2: Research & Enrichment

**When**: Part of WF-2 (Draft Generation), triggered when a Content Creation entry moves to "Backlog"
**Purpose**: Gather all source material needed for a high-quality draft

**Steps**:

1. **Fetch primary source** (if URL is set):
   - Jina Reader: `GET https://r.jina.ai/{url}` → returns markdown, cap at 8000 chars
   - Extract: key facts, quotes, statistics, author claims

2. **Fetch related sources** (from Related Content relations and Feed Items):
   - For each related feed item: read Full Text from Notion property
   - For each related content entry: read pitch and description

3. **Supplementary research** (via Perplexity):
   - Prompt Perplexity `sonar-pro` with the topic + context gathered so far
   - Ask for: recent developments, German market perspective, counterarguments, statistics
   - Temperature: 0.3, max_tokens: 3000

4. **Load Slack rant context** (if feed item is a rant or has linked rants):
   - Include the original rant text as "voice seed"
   - Extract: strong opinions, specific phrasings, dismissals, comparisons

5. **Package research bundle**:
   - Primary source text
   - Supplementary research with citations
   - Slack rant voice seeds (if any)
   - Topic metadata (title, pitch, category, target audience)

This research bundle becomes the input to Phase 3 (Draft Generation).

---

### Phase 3: Draft Generation

**When**: Part of WF-2, after research is complete
**Model**: Claude (Anthropic) — `claude-sonnet-4-6` for standard drafts, `claude-opus-4-6` for high-priority or complex topics
**Output**: Full blog post draft written to Notion page body

#### Brand Context Loading

Before generating any draft, the pipeline loads the complete brand context. This is NOT hardcoded in prompts — it's fetched from Notion at generation time (Josepha pattern), ensuring the latest guide versions are always used.

**Documents loaded** (from Notion wiki pages):
1. Brand Writing Guide (page `2d1c0fdf-942d-810d-8fda-cf181d93bf46`) — voice, tone, terminology
2. Blog Content Guide (page `2d1c0fdf-942d-81cf-bd59-f54a251a1571`) — structural templates
3. Blog Style Analysis (page `2d7c0fdf-942d-813e-a679-e55b56cea58b`) — pattern library

**Documents loaded** (from this repo, committed files):
4. `docs/sources/blog-style-research.md` — expanded 14-source craft analysis
5. `prompts/draft-system-prompt.md` — the master drafting prompt (ratchet-tunable)
6. `config/style-params.yaml` — style parameters (ratchet-tunable)

#### Draft Generation Prompt Architecture

The draft prompt is assembled from multiple components:

```
┌─────────────────────────────────────────────┐
│ SYSTEM PROMPT (from prompts/draft-system-prompt.md) │
│                                             │
│ Role: Du schreibst Content für T-0...       │
│ Voice rules: (from Brand Writing Guide)     │
│ Structure rules: (from Blog Content Guide)  │
│ Style inspiration: (from Blog Style Research)│
│ Craft patterns: (from Blog Style Analysis)  │
│                                             │
│ STYLE PARAMETERS (from config/style-params.yaml) │
│ opinion_strength: 0.8  (0-1 scale)         │
│ technical_depth: 0.6   (0-1 scale)         │
│ personal_anecdote: true                      │
│ progressive_disclosure: true                 │
│ show_the_work: true                          │
│ german_first: true                           │
│ max_anglicisms: 5                            │
│ target_word_count: 1200                      │
│ content_type: auto-detect                    │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ USER PROMPT (assembled per-draft)           │
│                                             │
│ Topic: {name}                                │
│ Pitch: {pitch}                               │
│ Content Type: {auto-detected or manual}      │
│ Target Audience: {from Target Audience Tags}│
│                                             │
│ SOURCE MATERIAL:                             │
│ {primary source text}                        │
│ {supplementary research with citations}      │
│                                             │
│ VOICE SEEDS (if available):                  │
│ {Slack rant text}                            │
│ Key arguments: {extracted arguments}         │
│ Strong phrasings to incorporate: {...}       │
│                                             │
│ PREVIOUS FEEDBACK (if revision):             │
│ {Aaron's comments from Notion}               │
│                                             │
│ Generate a complete blog post draft.         │
│ Follow the structural template for           │
│ {content_type} from the Blog Content Guide.  │
│ Open with pattern {1-4} from the Writing     │
│ Guide. Close with pattern {1-5}.             │
│ All sources used in research must appear as  │
│ embedded hyperlinks in the text.             │
└─────────────────────────────────────────────┘
```

#### Content Type Auto-Detection

If the Content Creation entry doesn't specify a content type, the pipeline infers it:

| Signal | Inferred Type |
|--------|---------------|
| Topic is a news event/announcement | Opinion / Analysis |
| Topic has a specific tool/technique | Resource (Technical) |
| Topic references a client project | Case Study |
| Topic is a Slack rant with strong opinion | Opinion / Speculation |
| Topic is a comparison/framework | Resource (Explainer) |

The inferred type determines which structural template from the Blog Content Guide is used.

#### Draft Output Format

Claude's response is structured markdown:

```markdown
<!-- META
content_type: Opinion
target_word_count: 1400
opening_pattern: News Hook
closing_pattern: Open Questions
-->

# {Title}

{Opening paragraph — pattern-specific}

{Promise/thesis — within first 150 words}

## {Section 1}

{Body content with progressive disclosure}

### Technisch: {optional deep-dive section}

{Technical detail, clearly marked}

## {Section 2}

{More body content}

> {Blockquote if citing specific source}

## {Closing section}

{Closing paragraph — pattern-specific}

{Soft CTA if appropriate}

<!-- SOURCES
- https://source1.com (referenced in paragraph 3)
- https://source2.com (referenced in section 2)
-->
```

#### Writing Draft to Notion

The markdown output is converted to Notion blocks using a markdown-to-blocks converter (Josepha pattern, handles: h1-h3, paragraphs, bullets, numbered lists, blockquotes, dividers, bold/italic/code/links, code blocks).

Blocks are appended to the Content Creation page body in batches of 100 (Notion API limit).

The META comment is parsed and used to set properties:
- Content type → Category Tags
- Word count → stored for evaluation

---

### Phase 4: Automated Evaluation

**When**: Auto-triggered after draft generation (WF-3, chained from WF-2) or manually triggered via webhook
**Model**: Claude `claude-sonnet-4-6` (strong instruction following for structured JSON)
**Output**: 7 dimension scores + composite + feedback written to Notion properties

#### Evaluation Rubric (Immutable)

From the Content Evaluation Guide (`docs/sources/content-evaluation-guide.md`):

| Dimension | Weight | Max Score | What It Measures |
|-----------|--------|-----------|------------------|
| Clarity | 1.0× | 5 | Is it understandable? No jargon without explanation? |
| Structure | 1.0× | 5 | Does it follow progressive disclosure? Scannable? |
| Accuracy | 1.5× | 7.5 | Are facts correct? Sources cited? |
| Credibility | 1.5× | 7.5 | Does it show the work? Earned confidence? |
| Engagement | 1.0× | 5 | Does the opening hook? Does it hold attention? |
| Voice | 1.0× | 5 | Does it sound like T-0? Strong opinions? Du-Form? |
| Actionability | 1.0× | 5 | Does the reader know what to do next? |
| **Composite** | — | **40** | Weighted sum |

**Quality tiers**: Excellent (34-40), Good (28-33), Acceptable (22-27), Below Standard (16-21), Poor (<16)

**Gate rules**:
- Composite ≥ 28 AND all dimensions ≥ 3 AND Voice ≥ 4 → advance to "Pre Review"
- Voice < 4 → **blocked** regardless of composite (constitutional principle)
- Any dimension < 3 → blocked, specific feedback required
- Composite < 28 → stay at "Draft"

#### Evaluation Prompt

```
You are a content evaluator for T-0. Score the following draft against
the Content Evaluation Guide rubric. Be strict — this rubric exists to
prevent generic AI consultancy content from being published.

RUBRIC:
{full Content Evaluation Guide text}

BRAND VOICE REFERENCE:
{Brand Writing Guide voice principles section}

DRAFT TO EVALUATE:
{full draft text}

Return a JSON object exactly matching this schema:
{
  "clarity": { "score": <1-5>, "feedback": "<specific feedback>" },
  "structure": { "score": <1-5>, "feedback": "<specific feedback>" },
  "accuracy": { "score": <1-5>, "feedback": "<specific feedback>" },
  "credibility": { "score": <1-5>, "feedback": "<specific feedback>" },
  "engagement": { "score": <1-5>, "feedback": "<specific feedback>" },
  "voice": { "score": <1-5>, "feedback": "<specific feedback>" },
  "actionability": { "score": <1-5>, "feedback": "<specific feedback>" },
  "composite": <weighted total>,
  "quality_tier": "<Excellent|Good|Acceptable|Below Standard|Poor>",
  "top_strength": "<what this draft does best>",
  "primary_weakness": "<the single most impactful improvement>",
  "voice_assessment": "<specific assessment of T-0 voice alignment>"
}
```

#### Writing Scores to Notion

The Content Creation DB needs these **new properties** (to be added):

| Property | Type | Purpose |
|----------|------|---------|
| Clarity Score | number | Dimension score (1-5) |
| Structure Score | number | Dimension score (1-5) |
| Accuracy Score | number | Dimension score (1-5) |
| Credibility Score | number | Dimension score (1-5) |
| Engagement Score | number | Dimension score (1-5) |
| Voice Score | number | Dimension score (1-5) |
| Actionability Score | number | Dimension score (1-5) |
| Composite Score | number | Weighted total (max 40) |
| Quality Tier | select | Excellent, Good, Acceptable, Below Standard, Poor |
| Evaluation Notes | rich_text | Top strength + primary weakness + voice assessment |
| Pipeline Version | rich_text | Git hash of the prompt/config used (for ratchet tracking) |
| Evaluated At | date | Timestamp of last evaluation |

After writing scores, the workflow checks gate rules and advances Status accordingly.

If advancing to "Pre Review," a Notion comment is created:
```
✅ Draft evaluated: {composite}/40 ({quality_tier})
Strongest: {top_strength}
To improve: {primary_weakness}
Voice: {voice_assessment}
```

If blocked, a Notion comment is created:
```
⚠️ Draft below threshold. {specific reason}
Composite: {composite}/40 | Voice: {voice_score}/5
Key feedback: {primary_weakness}
```

---

### Phase 5: Human Review

**When**: Aaron opens a "Pre Review" entry in Notion
**Not automated** — this is the human gate (constitutional principle)

**What Aaron sees in Notion**:
- The draft text in the page body
- All 7 dimension scores as number properties (visible in DB view)
- Composite score and quality tier
- Evaluation notes with specific feedback
- Source material links (Feed Items, Related Content)
- Pipeline version (which prompts were used)

**Aaron's actions**:

| Action | Status Change | Pipeline Response |
|--------|---------------|-------------------|
| Approve | → "Publishable" | Triggers WF-4 (Ghost) + WF-5 (Image) |
| Request revision | → "In Revision" + comment | WF-2 re-runs with Aaron's feedback as input |
| Reject | → "Idea" or archive | No pipeline action |
| Edit directly | Stays "Pre Review" | Aaron can edit the draft text manually |

**Revision handling**: When Aaron moves status to "In Revision" and adds a comment, the pipeline:
1. Reads all recent comments on the page
2. Preserves the current draft in a toggle block: "▸ Previous Version (v{n})"
3. Regenerates using original research bundle + Aaron's feedback
4. Re-evaluates the new draft
5. If passing, advances to "Pre Review" again

---

### Phase 6: Publication to Ghost

**When**: Status changes to "Publishable"
**n8n Workflow**: WF-4
**Ghost instance**: `blog.t-0.co`

#### Content Conversion

Notion blocks → HTML conversion:

| Notion Block | HTML Output |
|-------------|-------------|
| heading_1 | `<h1>` |
| heading_2 | `<h2>` |
| heading_3 | `<h3>` |
| paragraph | `<p>` |
| bulleted_list_item | `<ul><li>` |
| numbered_list_item | `<ol><li>` |
| quote | `<blockquote>` |
| code | `<pre><code>` |
| divider | `<hr>` |
| image | `<img src="..." alt="...">` |
| toggle (heading) | `<details><summary>` (or skip — Ghost doesn't support natively) |
| callout | `<div class="kg-callout-card">` (or HTML card in Lexical) |

Rich text annotations:
| Annotation | HTML |
|-----------|------|
| bold | `<strong>` |
| italic | `<em>` |
| code | `<code>` |
| link | `<a href="...">` |
| strikethrough | `<s>` |

#### Ghost API Call Sequence

**Step 1: Upload featured image** (if Image property is set)

```
POST https://blog.t-0.co/ghost/api/admin/images/upload/
Content-Type: multipart/form-data

file: {image binary downloaded from Google Drive URL}
purpose: image
ref: {content-creation-entry-id}
```

Response: `{ "images": [{ "url": "https://blog.t-0.co/content/images/..." }] }`

**Step 2: Create Ghost draft post**

```
POST https://blog.t-0.co/ghost/api/admin/posts/?source=html
Authorization: Ghost {JWT}
Content-Type: application/json

{
  "posts": [{
    "title": "{Title property, or Name if Title empty}",
    "html": "{converted HTML from Notion blocks}",
    "status": "draft",
    "tags": ["{mapped from Category Tags}"],
    "authors": ["{email of Author}"],
    "feature_image": "{uploaded image URL}",
    "feature_image_alt": "{auto-generated alt text}",
    "custom_excerpt": "{Pitch property, first 300 chars}",
    "meta_title": "{Title, max 60 chars}",
    "meta_description": "{Pitch, max 155 chars}",
    "og_title": "{Title}",
    "og_description": "{Pitch, max 200 chars}"
  }]
}
```

**Step 3: Handle scheduling** (if Due Date planned is set)

```
PUT https://blog.t-0.co/ghost/api/admin/posts/{id}/?source=html
Authorization: Ghost {JWT}

{
  "posts": [{
    "status": "scheduled",
    "published_at": "{Due Date planned}T09:00:00.000+01:00",
    "updated_at": "{current timestamp}"
  }]
}
```

**Step 4: Update Notion**

- Write Ghost post URL to URL property
- Write Ghost post ID to a new `Ghost ID` property (rich_text)
- Set Publication Date actual to now (or scheduled date)
- Advance Status to "Published"

#### Ghost Tag Mapping

| Content Creation Category Tag | Ghost Tag |
|-------------------------------|-----------|
| AI Agents | `ai-agents` |
| MCP | `mcp` |
| Automation | `automation` |
| Opinion | `opinion` |
| Technical | `technical` |
| Case Study | `case-study` |
| Mittelstand | `mittelstand` |
| (any new tag) | auto-created in Ghost |

Internal tags (not public): `#pipeline-generated`, `#v{pipeline-version}`

#### Ghost Authentication

Ghost Admin API key (format: `{id}:{secret}`) stored as n8n credential on `n8n.t-0.co`.

JWT generation (per Ghost docs):
```javascript
const jwt = require('jsonwebtoken');
const [id, secret] = apiKey.split(':');
const token = jwt.sign({}, Buffer.from(secret, 'hex'), {
  keyid: id,
  algorithm: 'HS256',
  expiresIn: '5m',
  audience: '/admin/'
});
```

n8n handles this via a Code node that generates the JWT before each Ghost API call.

---

### Phase 7: Image Generation

**When**: Triggered alongside Ghost publication (WF-5), or manually via "Run Image Gen" button
**Based on**: Josepha image generation workflow (adapted for T-0 style)

#### Flow

1. **Input**: Content Creation entry with draft text + optional Image Gen Description
2. **Auto-generate description** (if Image Gen Description is empty):
   - Claude summarizes the post topic into a 50-word image scene description
3. **Fetch T-0 illustration style guide** from Notion wiki page
4. **Enhance prompt** via Perplexity `sonar` (temp 0.3):
   - Input: image description + style guide
   - Output: detailed 100-300 word image prompt + kebab-case filename
5. **Reference image selection** (optional):
   - List images in T-0 reference folder on Google Drive
   - AI selects best-matching reference
   - Download as binary
6. **Generate image** via Gemini API:
   - Model: `gemini-2.0-flash-exp` (or latest imagen model)
   - Input: enhanced prompt + style guide text + optional reference image
   - Aspect ratio: 16:9 for blog featured images (default), configurable
7. **Upload to Google Drive** → T-0 Generated Images folder
8. **Set public sharing** permission
9. **Write back to Notion**: Image property + embedded image block
10. **Upload to Ghost** (if Ghost post exists): via Ghost image upload API

---

### Phase 8: Self-Improvement Ratchet

**When**: Monthly schedule (1st of month) + manual trigger
**n8n Workflow**: WF-6
**Purpose**: Analyze what's working, propose and test prompt/parameter changes

#### Ratchet Analysis Flow

1. **Gather data**:
   - Query Content Creation DB for all entries with Composite Score ≥ 1 from last 30 days
   - Extract: scores per dimension, content type, pipeline version, source material type

2. **Compute trends**:
   - Average composite score this month vs. previous months
   - Per-dimension averages and trends
   - Best/worst performing content types
   - Correlation between source type (RSS vs. Slack rant) and scores

3. **Identify improvement targets**:
   - Dimensions consistently below 4 → prompt adjustment needed
   - Voice scores trending down → style parameter review
   - Specific content types underperforming → template adjustment

4. **Generate improvement proposals** (Claude):
   - Input: trend data + current prompts + current style params
   - Output: 1-3 specific, testable changes (e.g., "Add a requirement to include at least one personal experience in opinion pieces")

5. **A/B test proposals** (offline):
   - Select 3-5 past topics (diverse content types)
   - Regenerate drafts with proposed changes
   - Evaluate with same rubric
   - Compare scores: proposed vs. original

6. **Accept or reject**:
   - Accept if: average composite improves ≥1 point AND no dimension drops >0.5
   - Reject otherwise
   - Accepted changes: committed to this repo with ratchet metadata

7. **Report**:
   - Create Notion page in Pipeline Runs DB (new) with:
     - Date range analyzed
     - Score trends (table)
     - Proposals tested
     - Accept/reject decisions with scores
     - Applied changes (with git commit hash)

#### Pipeline Runs DB (new Notion database)

| Property | Type | Purpose |
|----------|------|---------|
| Name | title | "Ratchet Run 2026-04-01" or "Pipeline Run WF-2 #{id}" |
| Type | select | `Ratchet Analysis`, `Draft Generation`, `Evaluation`, `Publication`, `Ingestion` |
| Date | date | Run timestamp |
| Status | status | `Running`, `Success`, `Failed`, `Partial` |
| Entries Processed | number | How many content entries were processed |
| Avg Composite | number | Average composite score (for ratchet runs) |
| Changes Applied | rich_text | Description of any config changes |
| Git Commit | url | Link to the commit with changes (for ratchet runs) |
| Error Log | rich_text | Error details (for failed runs) |
| Duration | number | Execution time in seconds |

---

## n8n Workflow Specifications

### WF-1: Source Ingestion

**Webhook**: `POST https://n8n.t-0.co/webhook/t0-source-ingestion`
**Schedule**: Weekly, Sunday 8:00 CET
**Estimated nodes**: 35-45

```
Schedule Trigger / Webhook Trigger
  │
  ├──→ Query Sources DB (HTTP: POST Notion /databases/{sourcesDbId}/query, filter Status=Active)
  │      ├── Extract RSS sources (Code: filter where RSS URL is set)
  │      └── Extract Slack config (Code: get #tech-shit-talk channel ID)
  │
  ├──→ [PARALLEL BRANCH A: RSS Feeds]
  │      ├── Fan Out RSS Sources (Code: split array)
  │      ├── Fetch RSS Feed (HTTP: GET {rssUrl})
  │      ├── Parse RSS Feed (Code: regex parser for RSS 2.0 + Atom)
  │      ├── Filter to Last 7 Days (Code: date comparison)
  │      ├── Aggregate All Items (Aggregate)
  │      └── Package RSS Results (Code)
  │
  ├──→ [PARALLEL BRANCH B: Slack]
  │      ├── Fetch Channel History (HTTP: GET Slack conversations.history, oldest=7d ago)
  │      ├── Filter Meaningful Messages (Code: >2 replies OR >3 reactions OR >200 chars)
  │      ├── Fetch Threads (HTTP: GET conversations.replies, for each qualifying message)
  │      ├── Extract URLs from Messages (Code: regex URL extraction)
  │      ├── Fetch URL Content (HTTP: GET r.jina.ai/{url}, for each URL found)
  │      ├── Package Slack Results (Code)
  │      └── Aggregate Slack Items (Aggregate)
  │
  ├──→ Merge All Items (Code: combine RSS + Slack into unified format)
  │
  ├──→ Dedup Against Existing (HTTP: POST Notion query Feed Items DB, filter Fetched >= weekStart)
  │      └── Filter New Only (Code: set-diff by URL)
  │
  ├──→ Store New Feed Items (loop)
  │      ├── Create Feed Item Page (HTTP: POST Notion /pages)
  │      └── Append Summary Blocks (HTTP: PATCH Notion /blocks/{pageId}/children)
  │
  ├──→ AI Triage (conditional: only if new items exist)
  │      ├── Build Triage Prompt (Code: numbered list with source weights)
  │      ├── Call Perplexity (HTTP: POST /chat/completions, sonar-pro, temp=0.2)
  │      ├── Parse Selection (Code: extract JSON array, fallback to top items by weight)
  │      └── Update Feed Item Status (HTTP: PATCH Notion, Selected items)
  │
  ├──→ Deep Fetch Selected Items
  │      ├── Fan Out Selected (Code: split)
  │      ├── Jina Deep Fetch (HTTP: GET r.jina.ai/{url}, 8000 char limit)
  │      ├── Update Feed Item Full Text (HTTP: PATCH Notion)
  │      └── Aggregate Deep Results
  │
  ├──→ Create Content Ideas
  │      ├── Build Idea Prompt (Code: for each selected item, generate pitch + angle)
  │      ├── Call Perplexity (HTTP: POST, generate pitches for selected items)
  │      ├── Parse Ideas (Code)
  │      └── Create Content Creation Entries (HTTP: POST Notion /pages, Status=Idea)
  │
  └──→ Build Summary + Respond to Webhook
```

**Credentials required on n8n.t-0.co**:
- `notionApi` or `httpHeaderAuth` (Notion integration token)
- `perplexityApi` (Perplexity API key)
- Slack OAuth token (Bot token with channel read permissions)
- Jina API key (optional — free tier may suffice for read operations)

---

### WF-2: Draft Generation Pipeline

**Webhook**: `POST https://n8n.t-0.co/webhook/t0-draft-generation`
**Trigger**: Notion automation on Status change to "Backlog" (fires webhook with page data)
**Also triggerable**: Manually via webhook with `{ "pageId": "..." }`
**Estimated nodes**: 25-35

```
Webhook Trigger
  │
  ├──→ Extract Page Data (Code: parse Notion automation payload, extract pageId, name, pitch, url, etc.)
  │
  ├──→ Acknowledge Webhook (Respond to Webhook: {"status":"accepted"})
  │
  ├──→ [PARALLEL: Gather Context]
  │      ├── Fetch Source Material
  │      │     ├── Read Feed Items relation (HTTP: GET Notion page properties)
  │      │     ├── Jina Fetch Primary URL (HTTP: GET r.jina.ai/{url}, if URL set)
  │      │     └── Read Related Content (HTTP: GET Notion, for each relation)
  │      │
  │      ├── Fetch Brand Context
  │      │     ├── Fetch Writing Guide (HTTP: GET Notion /blocks/{writingGuidePageId}/children)
  │      │     ├── Fetch Blog Content Guide (HTTP: GET Notion /blocks/{blogContentGuidePageId}/children)
  │      │     └── Fetch Blog Style Analysis (HTTP: GET Notion /blocks/{blogStyleAnalysisPageId}/children)
  │      │
  │      └── Fetch Pipeline Config
  │            ├── Read draft-system-prompt.md (from repo — loaded as n8n static data or fetched from GitHub raw)
  │            ├── Read style-params.yaml (same)
  │            └── Read blog-style-research.md (same)
  │
  ├──→ Merge All Context (Code: combine into structured bundle)
  │
  ├──→ Detect Content Type (Code: if not set, infer from topic signals)
  │
  ├──→ Assemble Draft Prompt (Code: system prompt + user prompt with all context)
  │
  ├──→ Generate Draft (HTTP: POST Anthropic /v1/messages)
  │      ├── Model: claude-sonnet-4-6
  │      ├── Max tokens: 8000
  │      ├── Temperature: 0.7
  │      ├── System prompt: assembled brand context
  │      └── User prompt: topic + research + voice seeds
  │
  ├──→ Parse Draft Response (Code: extract markdown, parse META comment)
  │
  ├──→ Convert Markdown to Notion Blocks (Code: Josepha converter, batched)
  │
  ├──→ Write Draft to Notion
  │      ├── Clear existing page body (if revision: preserve in toggle first)
  │      ├── Append draft blocks (HTTP: PATCH Notion /blocks/{pageId}/children, batched by 100)
  │      └── Update properties (HTTP: PATCH Notion /pages/{pageId})
  │            ├── Set Pipeline Version = current git hash
  │            ├── Set Content Type if auto-detected
  │            └── Status → "Draft"
  │
  └──→ Trigger Evaluation (HTTP: POST n8n.t-0.co/webhook/t0-draft-evaluation, { pageId })
```

**Anthropic API call details**:

```json
POST https://api.anthropic.com/v1/messages
x-api-key: {anthropic_api_key}
anthropic-version: 2023-06-01
Content-Type: application/json

{
  "model": "claude-sonnet-4-6",
  "max_tokens": 8000,
  "temperature": 0.7,
  "system": "{assembled system prompt — ~6000 tokens of brand context}",
  "messages": [
    {
      "role": "user",
      "content": "{assembled user prompt — topic + research + voice seeds}"
    }
  ]
}
```

**Estimated token usage per draft**: ~8,000 input tokens (system + user) + ~4,000 output tokens = ~12,000 total

**Config loading strategy**: Pipeline config files (`prompts/`, `config/`) are loaded from this GitHub repo at generation time. Options:
1. **GitHub raw URL**: `GET https://raw.githubusercontent.com/T-0-co/t-0_content-production/main/prompts/draft-system-prompt.md` — simple, always latest
2. **n8n static data**: Copy config to n8n workflow static data on deploy — faster but requires sync step
3. **Git clone in Code node**: `git archive` the latest config — most robust but heavier

Recommendation: GitHub raw URL for simplicity. The ratchet commits changes to the repo; the pipeline always reads the latest committed version.

---

### WF-3: Evaluation & Scoring

**Webhook**: `POST https://n8n.t-0.co/webhook/t0-draft-evaluation`
**Trigger**: Chained from WF-2, or manual
**Estimated nodes**: 15-20

```
Webhook Trigger (receives { pageId })
  │
  ├──→ Fetch Draft Content (HTTP: GET Notion /blocks/{pageId}/children, paginated)
  │      └── Convert Blocks to Text (Code: blocks → plain text markdown)
  │
  ├──→ Fetch Evaluation Rubric
  │      └── Read content-evaluation-guide.md (HTTP: GET GitHub raw URL)
  │
  ├──→ Fetch Brand Voice Reference
  │      └── Read Writing Guide voice section (HTTP: GET Notion, or from cache)
  │
  ├──→ Assemble Evaluation Prompt (Code)
  │
  ├──→ Run Evaluation (HTTP: POST Anthropic /v1/messages)
  │      ├── Model: claude-sonnet-4-6
  │      ├── Max tokens: 2000
  │      ├── Temperature: 0.2 (low — we want consistent scoring)
  │      └── System: "You are a strict content evaluator..."
  │
  ├──→ Parse Scores (Code: extract JSON, validate schema, compute composite)
  │
  ├──→ Apply Gate Rules (Code)
  │      ├── Check: composite ≥ 28?
  │      ├── Check: all dimensions ≥ 3?
  │      ├── Check: voice ≥ 4?
  │      └── Determine: advance to "Pre Review" or stay at "Draft"
  │
  ├──→ Write Scores to Notion (HTTP: PATCH Notion /pages/{pageId})
  │      ├── Set all 7 dimension score properties
  │      ├── Set Composite Score
  │      ├── Set Quality Tier
  │      ├── Set Evaluation Notes
  │      ├── Set Evaluated At
  │      └── Set Status (conditional)
  │
  ├──→ Append Evaluation Feedback Block (HTTP: PATCH Notion /blocks/{pageId}/children)
  │      └── Callout block with scores summary + per-dimension feedback
  │
  └──→ Create Notion Comment (HTTP: POST Notion /v1/comments)
         └── Status update notification for Aaron
```

**Evaluation consistency**: To prevent score drift, the evaluation prompt includes 3 calibration examples (one "Excellent", one "Good", one "Below Standard" draft with expected scores). These examples are stored in `prompts/evaluation-calibration.md` and loaded at evaluation time.

---

### WF-4: Ghost Publication

**Webhook**: `POST https://n8n.t-0.co/webhook/t0-ghost-publish`
**Trigger**: Notion automation on Status change to "Publishable"
**Estimated nodes**: 20-25

```
Webhook Trigger (receives { pageId } from Notion automation)
  │
  ├──→ Acknowledge Webhook
  │
  ├──→ Fetch Full Page Data (HTTP: GET Notion /pages/{pageId} + /blocks/{pageId}/children)
  │
  ├──→ [PARALLEL]
  │      ├── Convert Notion Blocks to HTML (Code: block-to-HTML converter)
  │      ├── Generate Ghost JWT (Code: from Admin API key)
  │      └── Download Featured Image (HTTP: GET Google Drive URL, if Image property set)
  │
  ├──→ Upload Image to Ghost (HTTP: POST Ghost /images/upload/, if image exists)
  │
  ├──→ Map Properties to Ghost Fields (Code)
  │      ├── Title: Title property (fallback: Name)
  │      ├── Tags: Category Tags → Ghost tag names + #pipeline-generated internal tag
  │      ├── Author: Author email → Ghost author
  │      ├── Excerpt: Pitch, first 300 chars
  │      ├── SEO: auto-generate meta_title (60 chars), meta_description (155 chars)
  │      └── Schedule: Due Date planned → published_at (if set)
  │
  ├──→ Create Ghost Post (HTTP: POST Ghost /posts/?source=html)
  │
  ├──→ IF: Scheduled?
  │      ├── YES → Update Post Status to Scheduled (HTTP: PUT Ghost /posts/{id}/)
  │      └── NO → Leave as draft (Aaron publishes manually in Ghost Admin)
  │
  └──→ Update Notion (HTTP: PATCH Notion /pages/{pageId})
         ├── URL = Ghost post URL
         ├── Ghost ID = Ghost post ID
         ├── Publication Date actual = now or scheduled date
         └── Status = "Published"
```

---

### WF-5: Image Generation

**Webhook**: `POST https://n8n.t-0.co/webhook/t0-image-gen`
**Trigger**: Notion "Run Image Gen" button, or chained from WF-4
**Based on**: Josepha Image Generation workflow (adapted)
**Estimated nodes**: 25-30

```
Webhook Trigger (receives { pageId } or button automation payload)
  │
  ├──→ Extract Page Data (Code: get Image Gen Description, Title, Pitch)
  │
  ├──→ IF: Image Gen Description empty?
  │      ├── YES → Auto-Generate Description (HTTP: POST Anthropic, short summary prompt)
  │      └── NO → Use existing description
  │
  ├──→ Fetch T-0 Illustration Style Guide (HTTP: GET Notion wiki page blocks)
  │
  ├──→ Enhance Prompt (HTTP: POST Perplexity /chat/completions, sonar, temp=0.3)
  │      └── Input: description + style guide → Output: {imageDescription, imageTitle}
  │
  ├──→ [OPTIONAL: Reference Image]
  │      ├── List Reference Images in Drive Folder (HTTP: GET Google Drive API)
  │      ├── AI Select Best Match (HTTP: POST Anthropic, structured output)
  │      └── Download Reference (HTTP: GET Google Drive)
  │
  ├──→ Build Gemini Request (Code: multimodal payload with text + optional reference)
  │
  ├──→ Generate Image (HTTP: POST Gemini API, timeout=60s)
  │
  ├──→ Parse & Convert (Code: extract base64 → binary PNG)
  │
  ├──→ Upload to Google Drive (HTTP: POST Drive API, to T-0 Generated Images folder)
  │
  ├──→ Set Public Sharing (HTTP: POST Drive permissions API)
  │
  └──→ Update Notion
         ├── Image property = Drive URL
         └── Append image block to page body
```

---

### WF-6: Ratchet Analysis

**Schedule**: Monthly, 1st of month 10:00 CET
**Webhook**: `POST https://n8n.t-0.co/webhook/t0-ratchet-analysis`
**Estimated nodes**: 20-30

```
Schedule Trigger / Webhook Trigger
  │
  ├──→ Query Evaluated Entries (HTTP: POST Notion query Content Creation DB)
  │      └── Filter: Composite Score ≥ 1, Evaluated At in last 30 days
  │
  ├──→ Compute Statistics (Code)
  │      ├── Average per dimension
  │      ├── Composite trend vs. previous months
  │      ├── Per-content-type breakdown
  │      ├── Per-source-type breakdown (RSS vs Slack rant vs manual)
  │      └── Identify weakest dimensions
  │
  ├──→ Fetch Current Config (HTTP: GET GitHub raw URLs for prompts + style-params)
  │
  ├──→ Generate Improvement Proposals (HTTP: POST Anthropic /v1/messages)
  │      └── Claude analyzes trends + current config → proposes 1-3 specific changes
  │
  ├──→ A/B Test Proposals
  │      ├── Select 5 Past Topics (Code: diverse sample from Content Creation DB)
  │      ├── For Each Proposal × Each Topic:
  │      │     ├── Regenerate Draft with Modified Config (HTTP: POST Anthropic)
  │      │     └── Evaluate Draft (HTTP: POST Anthropic, same rubric)
  │      └── Aggregate Test Results (Code: compare old vs. new scores)
  │
  ├──→ Accept/Reject Decisions (Code: apply thresholds)
  │
  ├──→ Apply Accepted Changes
  │      ├── Update Config Files in GitHub (HTTP: PUT GitHub API /repos/.../contents/...)
  │      └── Commit with ratchet metadata
  │
  └──→ Create Ratchet Report in Notion (HTTP: POST Notion, Pipeline Runs DB)
         └── Full analysis with trends, proposals, test results, decisions
```

---

## API Integration Specifications

### Notion API

**Version**: `2022-06-28` (via n8n `notionApi` credential — injects this version automatically)
**Base URL**: `https://api.notion.com/v1/`
**Auth**: Integration token (Bearer)

**Endpoints used**:

| Endpoint | Method | Purpose | Workflow |
|----------|--------|---------|----------|
| `/databases/{id}/query` | POST | Query Sources DB, Feed Items DB, Content Creation DB | WF-1, WF-3, WF-6 |
| `/pages` | POST | Create feed items, content ideas, pipeline run entries | WF-1, WF-6 |
| `/pages/{id}` | PATCH | Update scores, status, properties | WF-2, WF-3, WF-4, WF-5 |
| `/pages/{id}` | GET | Read page properties | WF-2, WF-4 |
| `/blocks/{id}/children` | GET | Read page body, wiki pages | WF-2, WF-3, WF-5 |
| `/blocks/{id}/children` | PATCH | Append draft blocks, eval feedback, image blocks | WF-2, WF-3, WF-5 |
| `/comments` | POST | Create status notification comments | WF-3 |

**Pagination**: All list/query endpoints paginated at 100 items. All Code nodes must handle `has_more` + `start_cursor`.

**Block append limit**: Max 100 blocks per PATCH. Longer drafts must be batched (Josepha pattern).

**Known pitfalls**:
- `notionApi` credential forces `Notion-Version: 2022-06-28` — cannot override
- Use `database_id` (not `data_source_id`) with `/databases/` endpoints
- `status` properties accept `select` syntax with `2022-06-28`
- Page creation with children is unreliable via MCP — use two-step: create page, then PATCH children

### Ghost Admin API

**Base URL**: `https://blog.t-0.co/ghost/api/admin/`
**Auth**: JWT (HS256, 5 min expiry, audience `/admin/`)
**Key location**: n8n credential on `n8n.t-0.co` (stored), also in T-0 server SOPS

**Endpoints used**:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/posts/` | POST | Create draft post |
| `/posts/{id}/` | PUT | Update post (schedule, publish) |
| `/posts/{id}/` | GET | Read post (verify creation) |
| `/images/upload/` | POST | Upload featured image |
| `/tags/` | GET | List existing tags |

**Content format**: `?source=html` query param on POST/PUT → Ghost converts HTML to Lexical internally.

**Tag handling**: Short-form `"tags": ["tag-name"]` auto-creates missing tags. Include `"#pipeline-generated"` as internal tag on all pipeline-created posts.

**Image format**: WEBP or PNG, multipart/form-data upload. Max size: Ghost default is 5MB.

**Rate limiting**: No documented rate limits. Use 1s delay between successive calls as courtesy.

**Known pitfalls**:
- `mobiledoc` field is null — do not use (Ghost 6 = Lexical)
- Settings API returns 403 with integration tokens — not relevant for this pipeline
- `updated_at` required for PUT operations — always GET first, then PUT with fresh timestamp
- Tags and authors are REPLACED, not merged — always send complete arrays

### Anthropic Claude API

**Base URL**: `https://api.anthropic.com/v1/`
**Auth**: `x-api-key` header
**Version**: `anthropic-version: 2023-06-01`

**Usage in pipeline**:

| Purpose | Model | Temperature | Max Tokens | Est. Input | Est. Output |
|---------|-------|-------------|------------|------------|-------------|
| Draft generation | claude-sonnet-4-6 | 0.7 | 8,000 | ~8,000 | ~4,000 |
| Evaluation | claude-sonnet-4-6 | 0.2 | 2,000 | ~6,000 | ~800 |
| Ratchet proposals | claude-opus-4-6 | 0.5 | 4,000 | ~10,000 | ~2,000 |
| Image description | claude-sonnet-4-6 | 0.5 | 500 | ~2,000 | ~200 |
| Revision generation | claude-sonnet-4-6 | 0.7 | 8,000 | ~10,000 | ~4,000 |

**Monthly cost estimate** (assuming 8 drafts/month, each with eval + 1 revision):
- Drafting: 8 × 12K tokens × $0.003/1K (sonnet in) + 8 × 4K × $0.015/1K (sonnet out) = ~$0.77
- Evaluation: 16 × 6.8K tokens = ~$0.33 + ~$0.19 = ~$0.52
- Ratchet: 1 run × ~50K total tokens ≈ ~$1.00
- **Total: ~$3-5/month** (very low)

### Perplexity API

**Base URL**: `https://api.perplexity.ai/`
**Auth**: Bearer token

**Usage in pipeline**:

| Purpose | Model | Temperature | Max Tokens |
|---------|-------|-------------|------------|
| Source triage | sonar-pro | 0.2 | 1,000 |
| Supplementary research | sonar-pro | 0.3 | 3,000 |
| Pitch generation | sonar-pro | 0.4 | 1,000 |
| Image prompt enhancement | sonar | 0.3 | 500 |

**Response format**: Chat completion compatible. Citations in `citations` array.

### Google Gemini API

**Base URL**: `https://generativelanguage.googleapis.com/v1beta/`
**Auth**: API key as query parameter
**Model**: `gemini-2.0-flash-exp` (or latest image generation model)

**Usage**: Image generation only. Multimodal input (text + optional reference image as base64 inlineData).

### Jina Reader API

**Base URL**: `https://r.jina.ai/`
**Auth**: Bearer token (optional — free tier available)
**Usage**: Article full-text extraction from URLs

**Call pattern**: `GET https://r.jina.ai/{encoded_url}` → returns markdown
**Content cap**: 8,000 characters for full fetch, 4,000 for homepage fallback

### Slack API

**Base URL**: `https://slack.com/api/`
**Auth**: Bot OAuth token

**Endpoints used**:

| Endpoint | Purpose |
|----------|---------|
| `conversations.history` | Fetch recent messages from #tech-shit-talk |
| `conversations.replies` | Fetch thread replies |
| `conversations.info` | Get channel details |

**Required scopes**: `channels:history`, `channels:read`
**Rate limit**: Tier 3 (50+ requests/minute) — not a concern for weekly ingestion

---

## Configuration Layer (This Repo)

### Repository Structure

```
t-0_content-production/
├── .specify/                          # Spec-kit framework
│   ├── memory/constitution.md
│   ├── specs/
│   └── templates/
├── config/
│   ├── sources.yaml                   # Initial source registry (seeded, pipeline manages via Notion)
│   └── style-params.yaml              # Ratchet-tunable style parameters
├── prompts/
│   ├── draft-system-prompt.md         # Master drafting system prompt
│   ├── triage-prompt.md               # Source triage prompt template
│   ├── evaluation-prompt.md           # Evaluation prompt template
│   ├── evaluation-calibration.md      # Calibration examples for consistent scoring
│   ├── research-prompt.md             # Supplementary research prompt
│   ├── image-description-prompt.md    # Image auto-description prompt
│   └── ratchet-analysis-prompt.md     # Ratchet improvement proposal prompt
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
# Modifiable by the ratchet loop. Changes require git commit with ratchet metadata.
# Version tracked via git hash. Each draft records which version was used.

voice:
  opinion_strength: 0.8          # 0-1: how strongly to state opinions (0.8 = "Spielzeug" level directness)
  personal_anecdote_required: true  # require at least one first-person experience
  show_the_work: true             # require showing process/commands/prompts
  german_first: true              # German as primary language
  max_anglicisms_per_paragraph: 3  # limit English terms in German text

structure:
  progressive_disclosure: true    # enforce Surface → Middle → Depth structure
  technical_sections_marked: true # require ### Technisch: headers
  target_word_count:
    opinion: 1500
    resource: 1200
    case_study: 1000
    technical: 1800
  max_word_count: 2500

opening:
  preferred_patterns:             # weighted selection (ratchet can adjust weights)
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
  min_external_sources: 2         # minimum cited sources per post
  cite_style: embedded_hyperlink  # per Brand Writing Guide v2.1
  primary_sources_preferred: true # original > aggregator

cta:
  type: soft                      # soft | none | primary
  default_label: "Schreib uns"
  opinion_pieces_cta: none        # opinion pieces don't need CTA
```

### Prompt Templates

Prompt templates in `prompts/` use `{variable}` placeholders that are filled at runtime by n8n Code nodes. Example structure for `draft-system-prompt.md`:

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
6. Alle recherchierten Quellen müssen als eingebettete Hyperlinks erscheinen.
7. Technische Abschnitte mit ### Technisch: markieren.
8. Keine Em-Dashes. Punkte oder Satzumstellung stattdessen.
```

---

## User Touchpoints & Interactions

### Aaron's Daily/Weekly Workflow

| When | Action | Where | Pipeline Response |
|------|--------|-------|-------------------|
| Anytime | Enter a topic idea | Notion: create Content Creation entry with Name + Pitch | Entry appears as "Idea" |
| Anytime | Promote idea to pipeline | Notion: move Status → "Backlog" | WF-2 triggers: research → draft → evaluate |
| After eval notification | Review draft | Notion: read page body + scores | — (human reads) |
| After review | Approve | Notion: move Status → "Publishable" | WF-4 + WF-5 trigger: Ghost draft + image |
| After review | Request revision | Notion: move Status → "In Revision" + add comment | WF-2 re-runs with feedback |
| After Ghost draft | Final check | Ghost Admin: review post | — (human reviews in Ghost) |
| After Ghost check | Publish | Ghost Admin: click "Publish" (or let schedule fire) | Notion status auto-updates |
| Weekly | Review new ideas | Notion: browse "Idea" entries from WF-1 | — (human curates) |
| Monthly | Review pipeline health | Notion: read ratchet report in Pipeline Runs DB | — (human reviews trends) |

### Notion Automations Required

These automations fire webhooks to n8n when Notion page status changes:

| Trigger | Webhook Target | Payload |
|---------|---------------|---------|
| Content Creation Status → "Backlog" | `n8n.t-0.co/webhook/t0-draft-generation` | `{ "pageId": "..." }` |
| Content Creation Status → "In Revision" | `n8n.t-0.co/webhook/t0-draft-generation` | `{ "pageId": "...", "mode": "revision" }` |
| Content Creation Status → "Publishable" | `n8n.t-0.co/webhook/t0-ghost-publish` | `{ "pageId": "..." }` |

**Notion automation format** (as documented in n8n pitfalls): Notion automations send `{ source: {...}, data: { object: "page", id: "...", properties: {...} } }`. The webhook-receiving Code node must flatten this.

### Ghost Admin Interaction

After WF-4 creates a Ghost draft, Aaron interacts with it in Ghost Admin:

1. Open `blog.t-0.co/ghost/` → Posts → filter by "Draft"
2. Review: formatting, images, tag correctness, SEO preview
3. Make minor edits in Ghost if needed (formatting tweaks, not content changes)
4. Click "Publish" (or verify schedule is correct)

**Note**: Content edits should happen in Notion (where the pipeline can track them). Ghost is for final visual verification only.

---

## Error Handling & Recovery

| Error | Detection | Response | Recovery |
|-------|-----------|----------|----------|
| RSS feed unreachable | HTTP non-200 or timeout | Log error, skip source, continue with others | Next weekly run retries |
| Slack API rate limited | HTTP 429 | Exponential backoff (n8n retry built-in) | Auto-retry |
| Perplexity triage returns invalid JSON | Parse error | Fall back to top N items by source weight | No manual action needed |
| Claude draft generation fails | API error or empty response | Status stays "Backlog", error comment on Notion page | Manual retry or re-trigger webhook |
| Claude evaluation returns scores outside range | Validation (score <1 or >5) | Re-run evaluation with explicit range instruction | Auto-retry once, then flag |
| Notion API 409 (conflict) | HTTP 409 | Fetch latest page version, re-apply changes | Auto-retry with fresh data |
| Ghost post creation fails | API error | Status stays "Publishable", error comment on Notion page | Manual retry or debug |
| Ghost image upload fails | API error | Create post without image, flag in Notion comment | Generate image separately |
| Jina Reader returns empty/paywalled | Content <100 chars | Fall back to RSS summary + Perplexity research | Draft includes "limited source" disclaimer |
| Gemini image gen fails | API error or no inlineData | Post proceeds without image, Notion comment added | Manual retry via "Run Image Gen" button |
| Ratchet A/B test inconclusive | Score difference <1 point | Reject proposal (conservative — don't change what works) | Try different proposal next month |
| n8n workflow execution timeout | Exceeds n8n timeout (default: 5min for webhooks) | Workflow fails, execution logged | Increase timeout for long workflows (WF-2 should be 10min) |

### Monitoring & Alerting

**Pipeline Runs DB** serves as the monitoring dashboard. Each workflow execution creates an entry with:
- Run type (ingestion, draft, evaluation, publication, image, ratchet)
- Status (success/failed/partial)
- Duration
- Error log (if failed)
- Entries processed

**Optional**: n8n → Telegram notification on failure (existing pattern in Aaron's stack).

---

## Patterns Borrowed from Josepha

| Pattern | Josepha Implementation | T-0 Adaptation |
|---------|----------------------|----------------|
| Fan-out/aggregate | Split array → process individually → aggregate | Same — for RSS feeds, feed items, block batches |
| Fire-and-forget webhook | Acknowledge immediately, process async | Same — for draft gen, publication, image gen |
| CI docs loaded at generation time | Fetch 3 Notion wiki pages per run | Fetch 3 Notion guide pages + repo config files |
| JSON repair parser | Regex extract + trailing comma + newline escape | Same — for triage and evaluation JSON responses |
| Retry with lower temperature | On parse failure, retry at temp-0.1 | Same — for triage and evaluation |
| Markdown-to-Notion-blocks converter | h1-h3, bullets, numbered, quotes, dividers, bold/italic/code/links | Same converter, extended for callouts and toggles |
| Dedup via URL set-diff | Compare feed URLs against existing Notion entries | Same |
| Notion status state machine | New → Used in Report | Extended: Idea → Backlog → Draft → Pre Review → ... → Published |

**Key difference from Josepha**: Josepha terminates at Notion (Instagram posts as Notion pages). T-0 adds the Ghost publication layer and the evaluation/ratchet feedback loop.

---

## Notion Database Setup Checklist

### Existing (no changes needed)
- [x] Content Creation DB (`2afc0fdf-942d-813e-bafa-fe13acf0b35f`)

### Properties to ADD to Content Creation DB

| Property | Type | Default |
|----------|------|---------|
| Clarity Score | number | — |
| Structure Score | number | — |
| Accuracy Score | number | — |
| Credibility Score | number | — |
| Engagement Score | number | — |
| Voice Score | number | — |
| Actionability Score | number | — |
| Composite Score | number | — |
| Quality Tier | select (Excellent, Good, Acceptable, Below Standard, Poor) | — |
| Evaluation Notes | rich_text | — |
| Pipeline Version | rich_text | — |
| Evaluated At | date | — |
| Ghost ID | rich_text | — |
| Content Type | select (Opinion, Resource, Case Study, Technical, Announcement) | — |
| Source Type | select (RSS, Slack Rant, Manual, Mixed) | — |
| Feed Items | relation → Feed Items DB | — |

### New Databases to Create

- [ ] **Sources DB** (child of Content Production project page)
- [ ] **Feed Items DB** (child of Content Production project page)
- [ ] **Pipeline Runs DB** (child of Content Production project page)

See Phase 1 and Phase 8 sections for full property schemas.

---

## Open Questions

These require discussion before implementation:

### OQ-1: n8n.t-0.co Readiness

What credentials are already configured on `n8n.t-0.co`? What needs to be added? Is there an API key for remote access?

**Impact**: Blocks all workflow development. If `n8n.t-0.co` isn't ready, we could temporarily build on `automation.kokal.cloud` and migrate later.

### OQ-2: Slack Bot Token

Does T-0's Slack workspace already have a bot with `channels:history` permission? If not, we need to create a Slack app + install it to the workspace.

**Impact**: Blocks US-2 (Slack rant ingestion) and US-8 (rant augmentation).

### OQ-3: Ghost Admin API Key

Does `blog.t-0.co` already have a Custom Integration configured in Ghost Admin? We need the `{id}:{secret}` Admin API key.

**Impact**: Blocks US-5 (Ghost publication).

### OQ-4: Anthropic API Access

Which Anthropic API key should the pipeline use? Aaron's personal key, or a T-0 organizational key? Need to ensure the key has access to `claude-sonnet-4-6` / `claude-opus-4-6`.

**Impact**: Blocks US-1, US-3 (drafting and evaluation).

### OQ-5: Google Drive Folder Structure for T-0

Where should T-0 generated images be stored? In Aaron's personal Drive, a shared T-0 Drive, or a specific folder? Josepha uses `aaron@kokal-eventsupport.de` Drive; T-0 should probably use `aaron@t-0.co`.

**Impact**: Affects WF-5 (image generation) Google OAuth credentials.

### OQ-6: Notion Automation Webhooks

Notion automations can fire webhooks on property changes, but this requires the Notion "Automations" feature (available on Plus plan or higher). Does the T-0 Notion workspace have this?

**Alternative**: If Notion automations aren't available, use n8n polling (check Content Creation DB every 5 minutes for status changes). Less elegant but functional.

### OQ-7: Ratchet Commit Permissions

Should the ratchet loop be able to commit directly to this repo's `main` branch? Or should it create a PR for human review?

**Recommendation**: Create a PR. Aaron reviews the proposed changes, merges if approved. This maintains human oversight over the "training" side of the autoresearch pattern.

### OQ-8: Content Creation DB — Preserve or Rebuild?

The existing Content Creation DB has entries dating back to 2025. Should we:
a) Add new properties to the existing DB and work with it as-is
b) Create a fresh "V2" DB with all properties from the start and migrate select entries

**Recommendation**: (a) — extend the existing DB. The existing entries are valuable as backlog ideas.

### OQ-9: Pipeline Prompt Language

The Brand Writing Guide says "Deutsch zuerst" for content. Should the pipeline prompts themselves (system prompt, evaluation prompt) also be in German? Or English prompts producing German output?

**Consideration**: Claude follows German instructions well, but English system prompts tend to produce more consistent instruction following. The Brand Writing Guide's AI Agent prompt is written in mixed (German terms, English structure).

**Recommendation**: English system prompts with German-specific rules embedded. Output is always German.

---

## Success Criteria

### SC-1: Draft Quality

Pipeline-generated drafts consistently score ≥28 composite (Good tier) on the Content Evaluation Guide within the first month of operation. After 3 months with the ratchet active, average composite score reaches ≥32 (approaching Excellent).

### SC-2: Voice Authenticity

Voice dimension scores average ≥4.0 across all pipeline-generated drafts. Aaron and Julien cannot reliably distinguish pipeline drafts from human-written drafts in a blind test (after human review and minor edits).

### SC-3: Source Coverage

Pipeline surfaces ≥5 relevant topics per week from tracked sources, of which Aaron promotes ≥2 to "Backlog" for drafting.

### SC-4: Time Savings

End-to-end time from "Backlog" to "Pre Review" (draft + evaluation) is under 10 minutes of pipeline execution time, replacing what would be 2-4 hours of manual drafting and self-review.

### SC-5: Publication Cadence

T-0 publishes 2-4 blog posts per month (up from the current sporadic cadence), all scoring ≥28 composite.

### SC-6: Ratchet Improvement

After 3 months, the ratchet has successfully proposed and applied ≥2 prompt/parameter changes that demonstrably improved average composite scores.

### SC-7: Zero Unauthorized Publications

No content reaches Ghost in published state without Aaron's explicit action. Verify: all Ghost posts created by pipeline have `status: "draft"` until human publishes.

---

## Implementation Phases (Suggested)

| Phase | Scope | User Stories | Est. Effort |
|-------|-------|-------------|-------------|
| **Phase A** | Manual draft generation + evaluation | US-1, US-3, US-4 | Core pipeline: WF-2 + WF-3 |
| **Phase B** | Source ingestion | US-2, US-8 | WF-1 with RSS + Slack |
| **Phase C** | Ghost publication + images | US-5, US-6 | WF-4 + WF-5 |
| **Phase D** | Self-improvement ratchet | US-7 | WF-6 (after enough data) |

Phase A is the MVP — everything else builds on it. Start there.
