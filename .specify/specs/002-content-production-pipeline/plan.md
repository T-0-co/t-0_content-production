# Implementation Plan: T-0 Content Production Pipeline

**Date**: 2026-03-24 | **Spec**: [002-content-production-pipeline.md](../002-content-production-pipeline.md)
**Input**: Full system design spec with all 26 design decisions resolved

## Summary

Build an n8n-orchestrated content pipeline on `n8n.t-0.co` that takes Content Creation DB entries through research → draft → evaluate → publish. The pipeline is triggered by a single "Run Pipeline" button on each Notion entry, with behavior determined by the entry's Status property. Implementation follows 6 phases, with Phase A (core pipeline) as the immediate priority to unblock processing of existing entries.

## Technical Context

**Orchestration**: n8n at `n8n.t-0.co` (T-0's dedicated instance, 29 active workflows)
**LLM**: OpenRouter → Claude Sonnet 4.6 via n8n AI Agent nodes
**Primary Dependencies**: Notion API, Ghost Admin API, OpenRouter API, Jina Reader, Perplexity, Gemini, Slack API, Google Drive API
**Storage**: Notion databases (Content Creation, Notes, Content Sources) + Ghost CMS
**Testing**: Manual end-to-end testing with real Content Creation entries. Evaluation rubric IS the test.
**Target Platform**: n8n workflows + Notion automations
**Config**: This repo (prompts/, config/) consumed by n8n via GitHub raw URLs
**Constraints**: n8n AI Agent nodes limit agent tooling to 2-3 tools per step. All LLM calls go through OpenRouter. No direct Anthropic API usage.

## Constitution Check

*GATE: Must pass before proceeding.*

- [x] **Principle I — Human Gate Before Publication**: Ghost posts are created as `status: "draft"` or `status: "scheduled"` (with human-set date). Pipeline never publishes directly. The button-press model ensures human intentionality at every status transition.
- [x] **Principle II — Evaluator Is Immutable**: Evaluation rubric loaded from `docs/sources/content-evaluation-guide.md` and Notion at runtime. Pipeline cannot modify it. Ratchet modifies prompts and style params only.
- [x] **Principle III — T-0 Voice Over Volume**: Voice ≥ 4 gate blocks advancement regardless of composite score. Brand context loaded from Notion at generation time. Style params tuned for opinion strength (0.8), show-the-work, German-first.
- [x] **Principle IV — Build on What Exists**: Uses existing Content Creation DB, Notes DB, Ghost CMS, image gen workflow, Slack bot. New infra only for Content Sources DB (lightweight) and pipeline-specific n8n workflows.

## Project Structure

### Documentation

```text
.specify/specs/
├── 002-content-production-pipeline.md    # Full system spec
└── 002-content-production-pipeline/
    ├── plan.md              # This file
    └── tasks.md             # Implementation tasks (next)
```

### Source Code (this repo)

```text
t-0_content-production/
├── config/
│   └── style-params.yaml             # Ratchet-tunable parameters
├── prompts/
│   ├── draft-system-prompt.md         # Master drafting prompt
│   ├── evaluation-prompt.md           # Evaluation prompt
│   ├── evaluation-calibration.md      # Calibration examples (deferred)
│   ├── triage-prompt.md               # Source triage prompt
│   ├── research-prompt.md             # Research agent instructions
│   └── image-description-prompt.md    # Image description prompt
├── docs/sources/                      # Reference docs (most already exist)
└── docs/references/
```

### n8n Workflows (built on n8n.t-0.co)

```text
WF-1: Content Pipeline         # Single-button, branches by Status
WF-2: Source Ingestion          # Scheduled (RSS weekly, Slack 6h)
WF-3: Image Generation          # Adapt existing workflow
WF-4: Ratchet Analysis          # Future (dormant initially)
```

## Implementation Phases

### Phase A: Core Pipeline (P0 — build first)

The minimum viable pipeline that lets Aaron process existing entries.

**Scope**: WF-1 with all three branches (draft, revision, publish). Notion DB property setup. Ghost "Content Pipeline" integration. Prompt templates. Style params.

**Prerequisites** (must complete before WF-1 build):
1. OpenRouter credential on n8n.t-0.co
2. Ghost "Content Pipeline" integration key → n8n credential
3. Content Creation DB: add 16 new properties (API-creatable)
4. Content Creation DB: add "Run Pipeline" button (Playwright MCP)
5. Content Creation DB: verify/add Status values (Playwright MCP)
6. Notion automation: button → webhook
7. Create prompt templates in repo
8. Create style-params.yaml in repo

**Build order** (sequential — each step builds on the previous):
1. Credential setup (OpenRouter, Ghost, Notion on n8n.t-0.co)
2. Notion DB property additions
3. Prompt templates + style params → commit to repo
4. WF-1 skeleton: webhook → parse → switch on Status → respond
5. WF-1 Branch A: research → draft → evaluate (Idea/Backlog path)
6. WF-1 Branch C: Ghost publication (Publishable path)
7. WF-1 Branch B: revision flow (In Revision path)
8. End-to-end test with a real existing entry
9. Button + Notion automation setup

**Validation**: Process 3 existing Content Creation entries end-to-end. Each should produce a scored draft with composite ≥ 20 on first attempt (we're not expecting ≥28 immediately — prompt tuning comes with usage).

### Phase D: Ghost Tag Overhaul (P1 — parallel with A)

Can be done independently of Phase A since it only touches Ghost, not n8n.

**Scope**: Export current tags, map to new taxonomy, update posts, delete orphans.

### Phase B: Source Ingestion (P2 — after A works)

**Scope**: WF-2. Content Sources DB. Notes DB extensions (Source relation, new Origin values). Google Alerts setup.

### Phase C: Image Gen Integration (P2 — after A works)

**Scope**: Adapt existing image gen workflow to add Ghost upload + Notion write-back. Wire into WF-1 Branch C.

### Phase E: Ratchet Activation (P3 — after data accumulates)

**Scope**: WF-4. Monthly schedule. A/B testing framework. GitHub PR generation. Requires 20+ evaluated drafts.

### Phase F: Newsletter Optimization (P2 — after Ghost publishing works)

**Scope**: Ghost newsletter config, sender identity update, subscriber management.

## Research Notes

### Approach Chosen

**n8n AI Agent nodes with OpenRouter** — deterministic workflow with focused agent nodes at LLM steps. Each agent gets 2-3 tools max. The workflow is a fixed graph of n8n nodes; agents make tactical decisions (which tool to use) within clearly scoped tasks.

**Single-button architecture** — one webhook handles all states via Switch node. This simplifies the UX (Aaron presses one button) and reduces workflow count.

**Notes DB for feed items** — reuse existing database with 28 mature properties instead of creating a new one. Add minimal new properties (Source relation, new Origin values).

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| 6 separate n8n workflows | Clear separation of concerns | Complex to trigger, maintain, debug | Single-button UX is better for Aaron |
| Direct Anthropic API | Slightly cheaper | Different credential, can't switch models easily | OpenRouter gives model flexibility |
| New Feed Items DB | Clean schema from scratch | Duplicates Notes DB functionality, more to maintain | Notes DB already has Slack fields, AI enrichment, Content Creation relation |
| Pipeline Runs DB in Notion | Structured run tracking | Extra overhead, n8n already tracks executions | n8n execution history suffices |
| Fully agentic pipeline | More adaptive | Expensive, unpredictable, hard to debug | Cost-conscious, deterministic is better for reliability |
| Config in Notion only | Easy to edit | No version control, ratchet can't PR | Hybrid: brand context from Notion, prompts from repo |

### Open Questions

None — all 26 design decisions resolved during spec phase.

### Key Risk

**Prompt quality on first run**: The pipeline will likely need 2-3 iterations of prompt tuning before drafts consistently score ≥28. The first batch of existing entries will be the tuning set. This is expected and part of the process.

## Next Steps

1. Create tasks.md with concrete implementation tasks
2. Begin implementation — Phase A first
