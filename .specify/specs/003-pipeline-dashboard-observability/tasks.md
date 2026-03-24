# Tasks: Content Pipeline Dashboard & Observability

**Input**: [spec.md](spec.md)
**Approach**: Notion work first (no code risk), then workflow modifications.

---

## Phase 1: Notion Schema & Tagging (API)

- [x] D01 Add "Ghost URL" property (url type) to Content Creation DB
- [x] D02 Add "Content Pipeline" tag option to T-0 Resources DB Tags property
- [x] D03 Create/tag 8 Resource entries for pipeline components:
  - Brand Writing Guide, Blog Content Guide, Blog Style Analysis
  - Content Evaluation Guide
  - Draft System Prompt (GitHub raw URL)
  - Style Parameters (GitHub raw URL)
  - n8n WF-1 Workflow (n8n URL)
  - Ratchet Analysis Prompt (repo link)

## Phase 2: Content Creation DB Views (Notion API)

- [x] D04 Create "Evaluation Board" view — Board grouped by Quality Tier
- [x] D05 Create "Score Breakdown" view — Table with all 7 scores + composite, sorted by Evaluated At desc
- [x] D06 Create "Review Queue" view — Table filtered to Pre Review + In Review, sorted by composite desc
- [x] D07 Create "Publication Calendar" view — Calendar on Due Date planned

## Phase 3: Content Pipeline Dashboard (Notion API + Playwright)

- [x] D08 Create dashboard page in T-0 Dashboards DB
- [x] D09 Add Quick Links section (brand guides, rubric, GitHub, n8n, Ghost)
- [x] D10 Add 6 inline Content Creation DB views (Ideas Inbox, In Pipeline, Review Queue, Needs Revision, Ready to Ship, Recently Published)
- [x] D11 Add Agent Behavior Panel (filtered Resources DB view, tag = Content Pipeline)

## Phase 4: Content Production Project Page (Notion API)

- [x] D12 Write Project Mission section (autoresearch pattern explanation)
- [x] D13 Write How to Use section (step-by-step guide)
- [x] D14 Write Architecture section (pipeline flow description)
- [x] D15 Add dashboard quick-link

## Phase 5: Pipeline Observability (n8n workflow)

- [x] D16 Add Pipeline Version stamp to Write Scores node (git hash + ISO timestamp)
- [x] D17 Add Evaluated At timestamp to Write Scores node (already existed in build script)
- [x] D18 Add deduplication guard after Parse Payload (skip if Status >= Draft)
- [x] D19 Add error handling branch (Error Trigger → Notion comment)

---

## Execution Order

```
D01-D03 (schema + tagging, parallel)
  → D04-D07 (views, parallel)
  → D08-D11 (dashboard, sequential)
  → D12-D15 (project page, sequential)
  → D16-D19 (workflow mods, sequential)
```
