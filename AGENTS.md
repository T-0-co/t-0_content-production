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

MCP server: `mcp__claude_ai_Notion__*`

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
| Ghost CMS (blog) | `blog.t-0.co` | Ghost Admin API via MCP Hub (`mcp__claude_ai_KSMS_Hub__*`) |
| AI Image Generation | n8n on `n8n.t-0.co` (workflow ID TBD) | Triggered via Notion button or webhook |
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

---
If something in this project surprises you, confuses you, frustrates you,
or you find yourself having repeated problems with it — propose an update
to this file and alert the developer.
