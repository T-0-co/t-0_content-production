# T-0 Content Production System

Autonomous content pipeline for T-0's blog at [t-0.co](https://t-0.co). Discovers topics, produces drafts in T-0's voice, evaluates them against a fixed rubric, and improves its own output over time. Nothing publishes without explicit human approval.

## How it works

The system follows the **autoresearch pattern**: a ratchet loop where an immutable evaluator scores pipeline outputs, and the pipeline can only change its own behavior — never the scoring rules. Over time, drafts get better because the system learns what scores well.

```
                                    ┌─────────────────────────┐
                                    │   Human Direction        │
                                    │   (Aaron & Julien)       │
                                    │   Sets topics, angles,   │
                                    │   reviews, approves      │
                                    └────────┬────────────────┘
                                             │
    ┌────────────────┐              ┌────────▼────────────────┐
    │ Source Discovery │──── Ideas ──▶│   Content Processing    │
    │ (WF-2)          │              │   (WF-1)                │
    │                 │              │   Research → Draft →     │
    │ RSS, Slack,     │              │   Evaluate → Revise →   │
    │ Alerts, etc.    │              │   Publish               │
    └────────────────┘              └────────┬───────┬────────┘
                                             │       │
                                    ┌────────▼──┐ ┌──▼──────────┐
                                    │ Ghost CMS  │ │ Self-       │
                                    │ Publication│ │ Improvement │
                                    │ (blog)     │ │ Ratchet     │
                                    └───────────┘ └─────────────┘
```

## System Components

### 1. Source Discovery

Automated topic discovery from external signals. Feeds new Ideas into the Content Creation database without human effort.

**Input modes**: RSS feeds, Slack #tech-shit-talk threads, Google Alerts, newsletters, GitHub trending, ArXiv feeds — each source has its own extraction method, all converging into a central processing pipeline.

**Data model** (4 databases, all relations dual):

- **Content Sources DB** — Registry of all monitored sources (blogs, podcasts, channels), their extraction mode, schedule, and health status
- **T-0 Notes DB** — Items extracted from sources land here alongside manual notes and Slack captures. The convergence point for all raw material regardless of origin.
- **T-0 Reports DB** — Periodic aggregation of notes into topic clusters and editorial recommendations (weekly intelligence digests)
- **T-0 Content Creation DB** — Where actual blog posts are drafted, evaluated, and published. The final pipeline stage.

**Orchestration**: Two n8n workflows on `n8n.t-0.co`:

- WF-2 (Source Ingestion) — extracts items on schedule, runs AI triage, writes to Notes DB
- WF-3 (Intelligence Digest) — weekly aggregation of notes into Reports DB with topic clustering

### 2. Content Processing

The core engine. Takes a topic from Idea to Published through research, drafting, evaluation, and revision cycles.

**Three branches, one workflow** (WF-1, 62 nodes on `n8n.t-0.co`):

| Branch | Trigger condition | What happens |
|--------|-------------------|--------------|
| **A: Draft** | Status = Idea or Backlog | Gathers brand context + pipeline config → researches topic (Perplexity) → generates draft (Claude via OpenRouter) → evaluates against rubric → writes scored draft to Notion |
| **B: Revise** | Status = In Revision | Reads human feedback from comments → archives old content → re-drafts incorporating feedback → re-evaluates |
| **C: Publish** | Status = Publishable | Converts to Ghost HTML → creates Ghost post (always as draft) → writes Ghost URL back to Notion |

**Trigger**: "Run Pipeline" button on any Content Creation DB entry → Notion automation → webhook.

**Error handling**: Failures post a diagnostic comment to the Notion page with execution details. No entry is left in limbo.

### 3. Quality Evaluation

Fixed scoring rubric that judges every draft. This is the immutable evaluator — the pipeline cannot modify it.

**7 dimensions**, weighted to a **40-point composite**:

| Dimension | What it measures |
|-----------|-----------------|
| Relevance & Timeliness | Does the topic matter to T-0's audience right now? |
| Depth & Insight | Does it go beyond surface-level? |
| Voice & Authenticity | Does it sound like Aaron and Julien wrote it? |
| Structure & Readability | Is it well-organized and scannable? |
| Accuracy & Credibility | Are claims sourced and correct? |
| Actionability | Can the reader do something with this? |
| Originality | Does it add something new to the conversation? |

**Gate rules**:
- Composite >= 28 AND all dimensions >= 3 AND Voice >= 4 → advances to "Pre Review"
- Voice < 4 → **blocked regardless of other scores** (constitutional principle)
- Any dimension < 3 → blocked with specific feedback

**Source of truth**: [Content Evaluation Guide](https://www.notion.so/2d8c0fdf-942d-81e0-b644-fb36e207f5a7) in Notion. Changes require deliberate human decision.

### 4. Publication

Ghost CMS at [t-0.co](https://t-0.co) is the publishing endpoint.

- Posts are **always created as drafts** — never auto-published
- Ghost tags mapped from Notion Category Tags (German-first taxonomy, 29 canonical tags)
- Newsletter association set automatically (Ghost handles distribution on publish)
- SEO fields (meta title, meta description) populated from draft content
- If Due Date is set → post is scheduled; otherwise → manual publish from Ghost Admin

### 5. Image Generation

AI-generated featured images for blog posts.

- Generates image descriptions from post content
- Uses Gemini API for generation, stores in Google Drive
- Attaches to both Notion entry and Ghost post
- Exists as a separate n8n workflow, triggerable from WF-1 or standalone

### 6. Brand Voice Framework

The reference material that defines how T-0 sounds. Consumed by the pipeline at runtime (not hardcoded).

| Guide | Purpose |
|-------|---------|
| Brand Writing Guide | Voice, tone, AI agent guidance |
| Blog Content Guide | Structural templates per content type (opinion, resource, case study, technical) |
| Blog Style Analysis | Patterns extracted from industry-leading blogs |
| Brand Process | The order in which guides should be applied |

All maintained as Notion pages in the T-0 Resources DB.

### 7. Pipeline Config (this repo)

The tunable knobs — prompt templates, style parameters, source weights. This is what the ratchet modifies.

```
config/
  style-params.yaml     — Voice strength, word counts, opening/closing pattern weights

prompts/
  draft-system-prompt.md       — Core drafting instructions
  research-prompt.md           — How to research a topic
  evaluation-prompt.md         — How to score against the rubric
  evaluation-calibration.md    — Score calibration examples
  triage-prompt.md             — Source relevance filtering
  image-description-prompt.md  — Featured image generation
  ratchet-analysis-prompt.md   — Self-improvement analysis
```

Every draft records which config version produced it (git short hash in Notion's `Pipeline Version` property). This makes A/B comparison possible.

### 8. Self-Improvement Ratchet

Monthly analysis of evaluation scores across all drafts. Proposes changes to prompts and style parameters. Changes are kept only if they improve scores.

**The contract**:
- The ratchet **can** modify: prompt templates, style parameters, source weights
- The ratchet **cannot** modify: the evaluation rubric, editorial strategy, or publish without review
- All changes committed via PR with score improvement data and rollback path

**Prerequisite**: 20+ scored drafts to have meaningful data. Not yet active.

### 9. Operator Dashboard

Notion-based control center showing pipeline state at a glance.

- 6 pipeline stage views (Ideas Inbox → Recently Published)
- Score Breakdown and Evaluation Board views
- Review Queue (Aaron's action items)
- Publication Calendar
- Quick links to all brand guides, rubric, GitHub repo, n8n workflow, Ghost CMS
- Agent Behavior Panel showing all resources that influence pipeline output

### 10. Claude Code Skill

The same pipeline logic packaged as a conversational flow in Claude Code. Same prompts, same brand context, same evaluation — but executed interactively using Claude directly instead of OpenRouter through n8n. For when Aaron wants more control or wants to iterate on a draft conversationally.

**Skill**: `.claude/skills/content-pipeline/SKILL.md`
**Trigger**: `/content [topic or page ID]`
**Phases**: Context Assembly → Research → Draft → Evaluate → Notion Update

## The autoresearch contract

| Role | Maps to | Who can change it? |
|------|---------|-------------------|
| `prepare.py` (immutable evaluator) | Quality Evaluation (#3) | Humans only, deliberately |
| `train.py` (tunable sandbox) | Pipeline Config (#7) | Ratchet proposes, humans approve |
| `program.md` (human direction) | Editorial strategy | Aaron and Julien |

## Technology stack

| Layer | Technology | Location |
|-------|-----------|----------|
| Orchestration | n8n | `n8n.t-0.co` |
| Content database | Notion (Content Creation DB) | T-0 Notion workspace |
| Publishing | Ghost CMS | `t-0.co` |
| AI models | Claude (via OpenRouter), Perplexity, Gemini | n8n credentials |
| Research tools | Perplexity (deep search), Jina (article extraction) | n8n credentials |
| Image generation | Gemini API → Google Drive | n8n workflow |
| Pipeline config | This repo (prompts, style params) | GitHub `T-0-co/t-0_content-production` |
| Source ingestion | n8n WF-2 (`U9YvA64bwIMur9Df`) | `n8n.t-0.co` |
| Intelligence digest | n8n WF-3 (`Td8CIlyau6zkWHrI`) | `n8n.t-0.co` |

## Content lifecycle

```
[Source Discovery]     [Manual Entry]
       │                     │
       ▼                     ▼
   ┌───────────────────────────┐
   │  Content Creation DB       │
   │  Status: Idea              │
   └───────────┬───────────────┘
               │  "Run Pipeline" button
               ▼
   ┌───────────────────────────┐
   │  Research + Draft          │
   │  Status: Draft             │
   └───────────┬───────────────┘
               │  Auto-evaluation
               ▼
   ┌───────────────────────────┐
   │  Scored Draft              │
   │  Status: Pre Review        │◄──── Score >= 28, Voice >= 4
   │  (or stays Draft if below) │
   └───────────┬───────────────┘
               │  Human review
               ▼
        ┌──────┴──────┐
        │             │
    Approve      Request revision
        │             │
        ▼             ▼
   Publishable    In Revision ──── "Run Pipeline" ──→ Re-draft + Re-evaluate
        │
        │  "Run Pipeline" button
        ▼
   ┌───────────────────────────┐
   │  Ghost Post (draft)        │
   │  Status: Published         │
   └───────────────────────────┘
        │
        │  Human publishes in Ghost Admin
        ▼
   Live on t-0.co + Newsletter sent
```

## Guardrails

- **No auto-publishing.** Ghost posts are always created as drafts. A human clicks Publish.
- **No Slack posting.** The pipeline does not post to any Slack channel without explicit approval.
- **Rubric is immutable.** The evaluation rubric cannot be modified by the pipeline or the ratchet.
- **Voice is sacred.** A draft scoring below 4 on Voice is blocked regardless of its composite score.

## Repository structure

```
t-0_content-production/
├── AGENTS.md                    # Project context (Claude Code / AI agents)
├── CLAUDE.md → AGENTS.md        # Symlink
├── README.md                    # This file — system overview
├── config/
│   └── style-params.yaml        # Tunable style parameters
├── prompts/                     # Prompt templates (consumed by n8n at runtime)
│   ├── draft-system-prompt.md
│   ├── research-prompt.md
│   ├── evaluation-prompt.md
│   ├── evaluation-calibration.md
│   ├── triage-prompt.md
│   ├── image-description-prompt.md
│   └── ratchet-analysis-prompt.md
├── docs/
│   ├── sources/                 # Original design specs from Notion
│   ├── references/              # Infrastructure maps, existing components
│   └── wf1-snapshot-*.json      # Workflow snapshots (secrets redacted)
└── .specify/                    # Spec-kit artifacts
    ├── memory/constitution.md   # Project principles
    ├── specs/                   # Feature specifications
    └── templates/               # Spec-kit templates
```
