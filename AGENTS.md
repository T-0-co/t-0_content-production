---
project: t-0_content-production
business: T-0
repo: https://github.com/T-0-co/t-0_content-production
notion: https://www.notion.so/2b0c0fdf-942d-80ef-bf4c-e515c902c6d7
---

# T-0 Content Production

Autonomous content pipeline: source ingestion (RSS + Slack #tech-shit-talk) → draft generation → automated evaluation → Ghost publication → self-improvement ratchet (autoresearch pattern).

## Documentation

| File | Contents |
|------|----------|
| [docs/notion-architecture.md](docs/notion-architecture.md) | Database hierarchy, IDs, Notes DB properties, Content Sources, Composite Score formula |
| [docs/sources/](docs/sources/) | Design specs: blog writing assistant, AI intelligence pipeline, evaluation guide, content guide |
| [docs/references/](docs/references/) | Existing infrastructure map |

**Session start:** Scan the docs manifest above. Read the files relevant to your current task before proceeding.

## Key Pages

Read these before working on pipeline logic:

| Page | ID | Purpose |
|------|-----|---------|
| Content Evaluation Guide | `2d8c0fdf-942d-81e0-b644-fb36e207f5a7` | Scoring rubric (7 dimensions, 1-5 scale) — the immutable evaluator |
| Blog Content Guide | `2d1c0fdf-942d-81cf-bd59-f54a251a1571` | Structural templates per post type |
| Brand Writing Guide | `2d1c0fdf-942d-810d-8fda-cf181d93bf46` | Voice, tone, AI agent guidance |
| Content Production page | `331c0fdf-942d-8051-aee6-c56e4285ddad` | Canonical home of all content DBs (in Main DB) |
| Content Production Dashboard | `32dc0fdf-942d-8135-bddf-e39bbb62d100` | Operation center with linked DB views |

MCP server: `mcp__notion__*`

## External Services

| Service | URL | Auth |
|---------|-----|------|
| Ghost CMS (blog) | `t-0.co` (API: `t-0.co/ghost/api/admin/`) | 1Password: `Ghost Admin API — T-0 (t-0.co)` in T-0 vault |
| n8n (T-0) | `n8n.t-0.co` | 1Password: `n8n API Key — T-0 Instance` in T-0 vault (field: `credential`) |
| Content Pipeline (WF-1) | n8n workflow `dJlw6US8gj6Do4Wf` | Webhook trigger (POST) |
| Source Ingestion (WF-2) | n8n workflow `U9YvA64bwIMur9Df` | Schedule (Sun 08:00 CET) + webhook |
| Intelligence Digest (WF-3) | n8n workflow `Td8CIlyau6zkWHrI` | Schedule (Mon 09:00 CET) + webhook |
| AI Image Generation | n8n workflow `2b6MMXh1Hz7NmbbR` | Notion automation webhook |
| Slack #tech-shit-talk | T-0 Slack workspace | MCP Hub `slack-rant-augmentation` skill |

## People

| Name | Role |
|------|------|
| Aaron Kokal | Lead — sets editorial direction, reviews drafts |
| Julien | T-0 co-founder — reviews and contributes to #tech-shit-talk |

## Guardrails

- Do NOT publish to Ghost without explicit human approval — drafts only
- Do NOT post to Slack channels without explicit human approval
- Do NOT modify the Content Evaluation Guide rubric autonomously — the rubric is the "immutable evaluator"

## Design Intent

This project follows the **autoresearch pattern** (Karpathy, 2026): a ratchet loop where an automated evaluator scores pipeline outputs, and improvements are kept only when scores increase.

| Autoresearch | This Pipeline |
|---|---|
| `prepare.py` (immutable evaluator) | Content Evaluation Guide (Notion rubric) |
| `train.py` (agent's sandbox) | Pipeline config — source selection, drafting prompts, style parameters |
| `program.md` (human direction) | Editorial strategy — what topics matter, what angles, T-0 voice |

The ratchet can modify pipeline prompts and source weighting. It cannot modify the evaluation rubric or publish without human review.

---
If something in this project surprises you, confuses you, frustrates you,
or you find yourself having repeated problems with it — propose an update
to this file and alert the developer.
