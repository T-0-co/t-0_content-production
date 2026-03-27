---
project: t-0_content-production
business: T-0
repo: https://github.com/T-0-co/t-0_content-production
notion: https://www.notion.so/2b0c0fdf-942d-80ef-bf4c-e515c902c6d7
---

# T-0 Content Production

Autonomous content pipeline: source ingestion (RSS + Slack #tech-shit-talk) → draft generation → automated evaluation → Ghost publication → self-improvement ratchet (autoresearch pattern).

## Undiscoverable Context

### Notion Resources

| Resource | Database ID | Data Source ID | Purpose |
|----------|-------------|----------------|---------|
| Content Creation | `2afc0fdf-942d-813e-bafa-fe13acf0b35f` | `2afc0fdf-942d-81b6-9bff-000be4682a2d` | Content tracking (Idea → Published) |
| Content Sources | `330c0fdf-942d-81b8-a839-fe9ea3c9ad15` | — | Source registry (blogs, feeds, alerts) |
| Notes | `2afc0fdf-942d-819a-b859-e69dcc48d67d` | `2afc0fdf-942d-812c-975d-000b9108667f` | Raw items + manual notes + Slack captures |
| Reports | `2e8c0fdf-942d-8059-8aaa-d2fb0f6f3a4b` | `2e8c0fdf-942d-803b-b059-000bc702d4c3` | Periodic intelligence digests |

MCP server: `mcp__notion__*`

Key Notion pages (read these before working on pipeline logic):

| Page | ID | Purpose |
|------|-----|---------|
| Content Evaluation Guide | `2d8c0fdf-942d-81e0-b644-fb36e207f5a7` | Scoring rubric (7 dimensions, 40pt scale) — this is the evaluator |
| Blog Content Guide | `2d1c0fdf-942d-81cf-bd59-f54a251a1571` | Structural templates per post type |
| Brand Writing Guide | `2d1c0fdf-942d-810d-8fda-cf181d93bf46` | Voice, tone, AI agent guidance |
| Brand Process | `2d1c0fdf-942d-81a1-b123-f44749d581cd` | Order of guide usage |
| Blog Style Analysis | `2d7c0fdf-942d-813e-a679-e55b56cea58b` | Reference patterns from industry leaders |
| Content Production project | `2b0c0fdf-942d-80ef-bf4c-e515c902c6d7` | Umbrella project in Notion |

### External Services

| Service | URL | Auth |
|---------|-----|------|
| Ghost CMS (blog) | `t-0.co` (API: `t-0.co/ghost/api/admin/`) | Ghost Admin API key: `T0_GHOST_ADMIN_KEY` in workspace `.env` |
| n8n (T-0) | `n8n.t-0.co` | API key: `T0_N8N_API_KEY` in workspace `.env` |
| Content Pipeline WF | n8n workflow `dJlw6US8gj6Do4Wf` | Webhook trigger (POST) |
| Content Pipeline Dashboard | Notion page `32dc0fdf-942d-8135-bddf-e39bbb62d100` | — |
| AI Image Generation | n8n workflow `2b6MMXh1Hz7NmbbR` on `n8n.t-0.co` | Notion automation webhook (fire-and-forget) |
| Slack #tech-shit-talk | T-0 Slack workspace | MCP Hub `slack-rant-augmentation` skill |

### People

| Name | Role |
|------|------|
| Aaron Kokal | Lead — sets editorial direction, reviews drafts |
| Julien | T-0 co-founder — reviews and contributes to #tech-shit-talk |

## Guardrails

- Do NOT publish to Ghost without explicit human approval — drafts only
- Do NOT post to Slack channels without explicit human approval
- Do NOT modify the Content Evaluation Guide rubric autonomously — the rubric is the "immutable evaluator" (autoresearch `prepare.py` equivalent)

## Design Intent

This project follows the **autoresearch pattern** (Karpathy, 2026): a ratchet loop where an automated evaluator scores pipeline outputs, and improvements are kept only when scores increase. The three-file contract maps as:

| Autoresearch | This Pipeline |
|---|---|
| `prepare.py` (immutable evaluator) | Content Evaluation Guide (Notion rubric) |
| `train.py` (agent's sandbox) | Pipeline config — source selection, drafting prompts, style parameters |
| `program.md` (human direction) | Editorial strategy — what topics matter, what angles, T-0 voice |

The ratchet can modify pipeline prompts and source weighting. It cannot modify the evaluation rubric or publish without human review.

## Spec-Driven Development

This project uses [spec-kit](https://github.com/github/spec-kit) for structured development.

### Quick Reference

| Command | Description |
|---------|-------------|
| `/speckit.constitution` | Define project principles |
| `/speckit.specify` | Create feature specification |
| `/speckit.plan` | Create technical plan |
| `/speckit.tasks` | Generate task breakdown |
| `/speckit.implement` | Execute tasks |

### Artifacts

- **Constitution**: [.specify/memory/constitution.md](.specify/memory/constitution.md)
- **Specs**: [.specify/specs/](.specify/specs/)
- **Templates**: [.specify/templates/](.specify/templates/)

### Source Documents

Consolidated design specs and references from Notion and other projects:

| Document | What It Contains |
|----------|-----------------|
| `docs/sources/blog-writing-assistant-spec.md` | 8-phase workflow spec (from Notion project) |
| `docs/sources/ai-intelligence-pipeline-spec.md` | 5-layer source ingestion architecture (from Notion project) |
| `docs/sources/content-evaluation-guide.md` | Scoring rubric with JSON schema for programmatic evaluation |
| `docs/sources/blog-content-guide.md` | Structural templates per content type |
| `docs/sources/josepha-pipeline-reference.md` | Working analog — fully operational content pipeline (different business) |
| `docs/references/existing-infrastructure.md` | Map of all existing components, Notion IDs, and related projects |

---
If something in this project surprises you, confuses you, frustrates you,
or you find yourself having repeated problems with it — propose an update
to this file and alert the developer.
