# T-0 Content Production — Constitution

## Core Principles

### I. Human Gate Before Publication

No content reaches Ghost, Slack, or any public channel without explicit human approval. The pipeline produces drafts and recommendations — humans decide what ships. This is not a temporary safety measure to be relaxed later. It is a permanent architectural constraint.

**Enforced**: Ghost publishing requires explicit approval. Pipeline outputs land in the Content Creation DB as drafts (status: "Draft" or "Pre Review"), never as published content.

### II. The Evaluator Is Immutable

The Content Evaluation Guide (7 dimensions, 40-point weighted composite) is the fixed scoring rubric — the `prepare.py` equivalent. The pipeline can optimize everything else: source selection, drafting prompts, angle selection, style parameters. It cannot modify the rubric or the scoring weights.

**Enforced**: Evaluation rubric lives in Notion (not in this repo). Changes to the rubric require a deliberate human decision and are versioned in the guide's changelog.

### III. T-0 Voice Over Volume

Producing more content faster is not the goal. Producing content that is unmistakably T-0 — pragmatic, credibility-through-demonstration, showing the work — is the goal. A pipeline that produces 10 generic posts per week is a failure. A pipeline that produces 2 posts per month that sound like Aaron and Julien actually wrote them is a success.

**Enforced**: The Voice dimension in the evaluation rubric. Content scoring below 4 on Voice does not advance to "Publishable" status regardless of other dimension scores.

### IV. Build on What Exists

The quality framework, Notion databases, Ghost CMS, image generation workflow, and slack-rant-augmentation skill already work. This project connects existing pieces — it does not rebuild them. New infrastructure only where no existing component can serve.

**Enforced**: `docs/references/existing-infrastructure.md` maps all existing components. Check before building.

## Technology Stack

| Layer | Technology | Status |
|-------|-----------|--------|
| **Orchestration** | n8n (T-0 instance at `n8n.t-0.co`) | Running |
| **Content DB** | Notion Content Creation DB | Running |
| **Publishing** | Ghost CMS at `t-0.co` via MCP Hub | Running |
| **Image generation** | n8n → Gemini API → Google Drive | Running |
| **Source ingestion (Slack)** | MCP Hub `slack-rant-augmentation` | Running |
| **Source ingestion (RSS/Jina/Slack)** | n8n WF-2 on `n8n.t-0.co` | Running |
| **Research/enrichment** | Perplexity, Jina, Claude | Available via MCP |
| **Pipeline config** | This repo (prompts, source weights, parameters) | This project |
| **Evaluation** | Content Evaluation Guide rubric (Notion) | In use, v1.0 |

### Key Decisions

- **n8n over custom code** for workflow orchestration — already deployed, already integrated with Notion/Ghost/Slack, visual debugging, the Josepha pipeline proves it works for content
- **Notion as content DB** (not a separate CMS or database) — the Content Creation DB workflow is mature, the team already uses it, the status pipeline is production-ready
- **Ghost as publishing endpoint** (not WordPress, not static site) — already running, has API, has MCP integration, handles newsletters
- **This repo holds pipeline config, not workflow logic** — n8n workflows are the execution engine; this repo stores the prompts, evaluation scripts, source lists, and style parameters that the workflows consume

## Development Workflow

1. Feature idea or improvement identified
2. `/speckit.specify` — define what the feature does and how it's evaluated
3. `/speckit.plan` — technical plan referencing existing infrastructure
4. `/speckit.tasks` — concrete task breakdown
5. `/speckit.implement` — build it
6. Test against the evaluation rubric with sample content
7. PR reviewed and merged

### Quality Gate

The pipeline's output quality is measured by the Content Evaluation Guide composite score. A feature is successful if it produces drafts that score higher (or at least not lower) than before. This is the autoresearch ratchet principle applied to features, not just content.

## Governance

This constitution supersedes conflicting practices. Amendments require:
1. Discussion between Aaron and Julien
2. PR with updated constitution
3. Corresponding update to the Content Evaluation Guide if scoring criteria change

**Version**: 1.0.0 | **Created**: 2026-03-24
