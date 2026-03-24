# Existing Infrastructure Map

> What already exists and where it lives. Reference document — these components are NOT copied into this project.

## Content Quality Framework (Notion — T-0 Resources DB)

All maintained in Notion, referenced by the pipeline at runtime:

| Resource | Notion Page ID | Status |
|----------|---------------|--------|
| Content Evaluation Guide (evaluator rubric) | `2d8c0fdf-942d-81e0-b644-fb36e207f5a7` | In Use v1.0 |
| Blog Content Guide (structural templates) | `2d1c0fdf-942d-81cf-bd59-f54a251a1571` | In Use v1.0 |
| Brand Writing Guide (voice/tone) | `2d1c0fdf-942d-810d-8fda-cf181d93bf46` | In Use |
| Brand Process Documentation (guide order) | `2d1c0fdf-942d-81a1-b123-f44749d581cd` | In Use |
| Blog Style Analysis (industry patterns) | `2d7c0fdf-942d-813e-a679-e55b56cea58b` | Draft v1.0 |
| Content Creation DB Guide (operational manual) | `2dfc0fdf-942d-8149-aaa7-f5bb15012d96` | In Use |

## Content Database (Notion)

| Resource | Database ID | Data Source ID |
|----------|-------------|----------------|
| Content Creation DB | `2afc0fdf-942d-813e-bafa-fe13acf0b35f` | `2afc0fdf-942d-81b6-9bff-000be4682a2d` |

Status workflow: Idea → Backlog → Draft → Pre Review → In Review → In Revision → Publishable → Published / Discarded

## Ghost CMS

| Property | Value |
|----------|-------|
| URL | `blog.t-0.co` (via `t-0.co`) |
| Version | Ghost 6 Alpine |
| Theme repo | `T-0-co/t-0_ghost-theme` (backlog: `t-0_ghost-theme/`) |
| MCP access | `mcp__claude_ai_KSMS_Hub__*` Ghost service |
| Ghost MCP service | `t-0_hub/t0-mcp-ghost/` (Express, port 3012) |
| Ghost admin skill | `t-0_hub/skills/workspace-t-0-ghost-admin/` |
| Server | `t-0_server/services/ghost/` (91.99.125.54) |

## AI Image Generation

| Property | Value |
|----------|-------|
| Workflow | n8n on `n8n.t-0.co` |
| Documented at | Notion page `2d5c0fdf-942d-8124-b835-ec661c19222c` |
| Trigger | Notion Content Creation DB button |
| Flow | Draft → LLM image description → reference image selection → Gemini generation → Google Drive + Notion |
| Drive folder | "Image Generation Dump for Blog" (`13CQ-Y2CRITlUn9fc28LnLLXoVeRtAVLE`) |

## Slack Integration

| Property | Value |
|----------|-------|
| Channel | `#tech-shit-talk` |
| Augmentation skill | `slack-rant-augmentation` (MCP Hub) |
| Captured threads | 6+ in Notion Notes DB (tagged "Shit-Talk") |

## Brand Voice (Wiki — Separate Repo)

`t-0-projects/t-0_wiki/docs/brand/voice/`:
- `content-patterns.md` — Templates for all content types
- `editorial-style.md` — Grammar, formatting, capitalization rules
- `messaging-framework.md` — Message architecture, value pillars

`t-0-projects-backlog/t-0_brand-assets/` — Visual identity, voice, reusable assets

## Related Notion Projects

| Project | Page ID | Status | Relevance |
|---------|---------|--------|-----------|
| Content Production (umbrella) | `2b0c0fdf-942d-80ef-bf4c-e515c902c6d7` | Permanently Active | Parent project |
| Blog Writing Assistant | `2ccc0fdf-942d-81ce-91ca-e2099c8862c3` | Prepared | Core workflow spec → copied to `docs/sources/` |
| AI Intelligence Pipeline | `30dc0fdf-942d-81de-8718-d03ae0bf2259` | Not Started | Upstream feed spec → copied to `docs/sources/` |
| T-0 Automated Research Agent | `317c0fdf-942d-810d-ac8a-efc50f4f2afe` | Not Started | Sibling concept (cron-triggered) |
| Aaron's Content Drafts 2025 | `2d1c0fdf-942d-8142-93d2-ef195d766bc7` | In Progress | 24 content entries, raw material |
| T-0 Priorities/Roadmap | `30dc0fdf-942d-8129-9dad-dbc35bcf09cf` | Not Started | Content = strategic priority |

## Related Notion Tasks

| Task | Page ID | Status | Relevance |
|------|---------|--------|-----------|
| AI news/RSS crawler → Slack | `2aec0fdf-942d-80dc-b85f-dee88f26ffcf` | Backlog (high priority) | Source ingestion idea |
| Research blog best practices | `2d2c0fdf-942d-817c-97c3-f83185343f05` | Not Started | Quality research |
| Initial Outreach zu Bestandskunden | `277c0fdf-942d-8040-a146-c3a0248b2f52` | Blocked | BLOCKED BY publishable content |

## Related Notion Notes

| Note | Page ID | Relevance |
|------|---------|-----------|
| Hybrid RAG Blog Series + Strategy-to-Blog Framework | `2d2c0fdf-942d-8140-86ca-dcf6140ad883` | Agent that derives blog posts from strategy docs |
| AI-Paradigmenwechsel Slack Rant Dekonstruktion | `2c4c0fdf-942d-815f-b99f-f92307e0b4b6` | Showcase of rant→content pipeline + content ideas |
| Analyse unseres eigenen Workspace | `2ccc0fdf-942d-8074-b66b-ce9463c2cfde` | Diagnosed missing blog-content-pipeline |
| 6+ captured #tech-shit-talk threads | Various | Raw material for blog posts |

## Working Analog: Josepha Pipeline

See `docs/sources/josepha-pipeline-reference.md` — fully operational content pipeline on `automation.kokal.cloud` with transferable patterns.
