---
project: t-0_content-production
business: T-0
repo: https://github.com/T-0-co/t-0_content-production
notion: https://www.notion.so/2b0c0fdf-942d-80ef-bf4c-e515c902c6d7
---

# T-0 Content Production

Autonomous content pipeline: source ingestion (RSS + Slack #tech-shit-talk) → draft generation → automated evaluation → Ghost publication → self-improvement ratchet (autoresearch pattern).

## Notion Architecture

### Where Databases Live

All content production databases live under the **T-0 Content Production page** in the T-0 Main DB. This follows the workspace rule: important databases must originate in the Main DB or one level deeper.

```text
T-0 Main db
└── T-0 Content Production page (331c0fdf-942d-8051-aee6-c56e4285ddad)
    ├── Content Sources db (inline)
    ├── T-0 Content Creation page
    │   └── T-0 Content Creation db (inline)
    └── T-0 Public Notion Content page
        └── Public Notion Content db (inline)
```

The **Content Production Dashboard** (`32dc0fdf-942d-8135-bddf-e39bbb62d100`, in Dashboards DB) links to these databases via linked database views with tab switching — it does NOT own any databases.

The **Content Production Pipeline project** (`2b0c0fdf-942d-80ef-bf4c-e515c902c6d7`, in Projects DB) tracks progress only — tasks, milestones, notes, status. It contains NO databases. The project is about the process of building the system, not the end result data.

### Databases

| Resource | Database ID | Data Source ID | Purpose |
|----------|-------------|----------------|---------|
| Content Creation | `2afc0fdf-942d-813e-bafa-fe13acf0b35f` | `2afc0fdf-942d-81b6-9bff-000be4682a2d` | Content tracking (Idea → Published) |
| Content Sources | `330c0fdf-942d-81b8-a839-fe9ea3c9ad15` | `330c0fdf-942d-81e4-a26d-000b8527f0f5` | Source registry (blogs, feeds, alerts) |
| Notes | `2afc0fdf-942d-819a-b859-e69dcc48d67d` | `2afc0fdf-942d-812c-975d-000b9108667f` | Raw items + manual notes + Slack captures |
| Reports | `2e8c0fdf-942d-8059-8aaa-d2fb0f6f3a4b` | `2e8c0fdf-942d-803b-b059-000bc702d4c3` | Periodic intelligence digests |

MCP server: `mcp__notion__*`

### Notes DB: Filtering Pipeline vs Human Notes

The `Source` relation (links to Content Sources DB) is the primary filter: **Source is not empty = pipeline-ingested note, Source is empty = human note**. All pipeline-ingested notes (RSS, Jina, Slack) have a Source relation set by WF-2.

Notes DB has been cleaned to 25 properties. When writing notes, use:

- `Name` (title) — the note title
- `Tags` (multi_select) — general organizational tags for internal use ("Meeting Notes", "Idea", etc.). Differentiates types of notes across the company.
- `Category Tags` (multi_select) — content production taxonomy ("AI Models", "MCP", etc.). Maps to Ghost blog tags. Only relevant for content-related notes.
- `Source` (relation) — link to Content Sources DB entry
- `Summary`, `Message`, `URL`, `Status`, `Time` — standard fields
- Slack-specific: `thread_ts_slack`, `channel_id_slack`, `contributors_slack`

**Tags vs Category Tags**: Tags is general-purpose (the Notes DB serves the whole company — meetings, ideas, content). Category Tags is specifically for the content production taxonomy and carries over to Ghost's tagging system. Keep them separate.

Deleted properties (do NOT write to these): `Title`, `AI Title`, `Origin`, `Origin Description`, `Description`, `AI-Tags`, `Brand Product System Tags`, `trigger reaction`, `Checked`, `Reprocessing Notes`.

### Content Sources DB

14 properties. The `Frequency` property was deleted — it was misleading (all sources are checked at the same weekly cadence by WF-2, regardless of how often they publish). Use `Access Method` (RSS/Jina/Slack) and `Last Checked` for operational info.

### Composite Score (Formula)

The Content Creation DB's `Composite Score` property is a **formula** that auto-calculates a weighted average of the 7 dimension scores, normalized to the same 1-5 scale:

```text
round((Voice*w + Clarity*w + Structure*w + Accuracy*w + Credibility*w + Engagement*w + Actionability*w) / sum(weights) * 10) / 10
```

All weights are currently set to 1 (equal). To change weights, edit the formula in the Notion DB property settings — each `* 1` multiplier can be adjusted (e.g., `* 1.5` for Accuracy/Credibility per the evaluation guide).

**Do NOT write to `Composite Score`** — it is read-only (formula). Only write the 7 individual dimension scores. The `Quality Tier` property has been removed; use Composite Score directly for quality assessment.

### Key Pages

Read these before working on pipeline logic:

| Page | ID | Purpose |
|------|-----|---------|
| Content Evaluation Guide | `2d8c0fdf-942d-81e0-b644-fb36e207f5a7` | Scoring rubric (7 dimensions, 1-5 scale) — this is the evaluator |
| Blog Content Guide | `2d1c0fdf-942d-81cf-bd59-f54a251a1571` | Structural templates per post type |
| Brand Writing Guide | `2d1c0fdf-942d-810d-8fda-cf181d93bf46` | Voice, tone, AI agent guidance |
| Brand Process | `2d1c0fdf-942d-81a1-b123-f44749d581cd` | Order of guide usage |
| Blog Style Analysis | `2d7c0fdf-942d-813e-a679-e55b56cea58b` | Reference patterns from industry leaders |
| Content Production page | `331c0fdf-942d-8051-aee6-c56e4285ddad` | Canonical home of all content DBs (in Main DB) |
| Content Production Dashboard | `32dc0fdf-942d-8135-bddf-e39bbb62d100` | Operation center with linked DB views |
| Content Production project | `2b0c0fdf-942d-80ef-bf4c-e515c902c6d7` | Progress tracking (tasks, milestones) |

### External Services

| Service | URL | Auth |
|---------|-----|------|
| Ghost CMS (blog) | `t-0.co` (API: `t-0.co/ghost/api/admin/`) | 1Password: `Ghost Admin API — T-0 (t-0.co)` in T-0 vault |
| n8n (T-0) | `n8n.t-0.co` | 1Password: `n8n API Key — T-0 Instance` in T-0 vault (field: `credential`) |
| Content Pipeline WF (WF-1) | n8n workflow `dJlw6US8gj6Do4Wf` | Webhook trigger (POST) |
| Source Ingestion (WF-2) | n8n workflow `U9YvA64bwIMur9Df` on `n8n.t-0.co` | Schedule (Sun 08:00 CET) + webhook |
| Intelligence Digest (WF-3) | n8n workflow `Td8CIlyau6zkWHrI` on `n8n.t-0.co` | Schedule (Mon 09:00 CET) + webhook |
| Content Pipeline Dashboard | Notion page `32dc0fdf-942d-8135-bddf-e39bbb62d100` | — |
| AI Image Generation | n8n workflow `2b6MMXh1Hz7NmbbR` on `n8n.t-0.co` | Notion automation webhook (fire-and-forget) |
| Gemini Image Gen API | n8n workflow `tJkGnbUzEWPjN64s` on `n8n.t-0.co` | Webhook (called by Image Gen WF) |
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
