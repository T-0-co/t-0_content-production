# Spec 003: Content Pipeline Dashboard & Observability

**Status**: Complete
**Priority**: P0 (builds on working Branch A pipeline)
**Depends on**: Spec 002 Phase A Branch A (complete)

## Problem

The content pipeline (WF-1 Branch A) is operational — Idea entries get researched, drafted, evaluated, and scored. But the operator experience is poor:

1. No central place to see pipeline state across all content entries
2. Content Creation DB views are generic (Info, In Progress) — none are pipeline-aware
3. Content Production project page is a template shell with no guidance
4. Resources that shape agent behavior (brand guides, rubric, prompts) aren't collected anywhere
5. Pipeline doesn't stamp which config version produced each draft (can't debug or A/B test)
6. No deduplication — triggering the webhook twice runs the full pipeline twice
7. No error handling branch — failures leave entries in limbo
8. Ghost URL property missing from Content Creation DB

## Solution

### A. Content Pipeline Dashboard (Notion)

New page in T-0 Dashboards DB that serves as the control center. Structure:

**Quick Links section (columns)**:
- Brand guides: Writing Guide, Blog Content Guide, Blog Style Analysis
- Evaluation Rubric (Content Evaluation Guide)
- GitHub repo link (prompts + config)
- n8n workflow direct link
- Ghost CMS (blog.t-0.co)

**Pipeline Stage Views (inline Content Creation DB views)**:

| View Name | Filter | Visible Columns |
|-----------|--------|-----------------|
| Ideas Inbox | Status = Idea | Name, Pitch, URL, Source Type, Created |
| In Pipeline | Status = Draft, Pre Review | Name, Composite Score, Quality Tier, Evaluated At |
| Review Queue | Status = Pre Review, In Review | Name, all 7 scores, Quality Tier, Evaluation Notes |
| Needs Revision | Status = In Revision | Name, Evaluation Notes, Voice Score, Composite |
| Ready to Ship | Status = Publishable | Name, Due Date planned, Platform, Ghost ID |
| Recently Published | Status = Published | Name, Publication Date actual, Ghost URL, Composite |

**Agent Behavior Panel**:
- Filtered inline view of T-0 Resources DB
- Filter: Tag contains "Content Pipeline"
- Shows all resources that influence pipeline output

### B. Content Creation DB — New Views

| View | Type | Purpose |
|------|------|---------|
| Evaluation Board | Board by Quality Tier | Visual score distribution |
| Score Breakdown | Table, sorted by Evaluated At desc | All 7 dimension scores visible — ratchet analysis input |
| Review Queue | Table, Status = Pre Review/In Review | Aaron's action items, sorted by composite desc |
| Publication Calendar | Calendar on Due Date planned | Scheduling view |

### C. Content Production Project Page

Transform from template shell to operational guide:
- **Project Mission**: The autoresearch pattern — what the system does and why
- **How to Use**: Step-by-step (create Idea → pipeline runs → review → publish)
- **Architecture**: Text description of the flow
- **Resources**: Populate existing toggle with all pipeline components
- **Dashboard link**: Quick link to the new dashboard

### D. Resources DB — Pipeline Component Tagging

Add "Content Pipeline" tag to Resources DB. Create/tag entries for:
- Brand Writing Guide
- Blog Content Guide
- Blog Style Analysis
- Content Evaluation Guide
- Draft System Prompt (GitHub link)
- Style Parameters (GitHub link)
- n8n WF-1 Workflow
- Ratchet Analysis Prompt

### E. Pipeline Observability (WF-1 modifications)

1. **Pipeline Version stamp**: Write git short hash + timestamp to `Pipeline Version` property on each run
2. **Evaluated At timestamp**: Set `Evaluated At` date property when scores are written
3. **Ghost URL property**: Add to Content Creation DB schema
4. **Deduplication guard**: Code node after Parse Payload — check if Status is already "Draft" or later, skip if re-triggered
5. **Error handling branch**: Error Trigger → write error as Notion comment → don't advance status

## Out of Scope

- Branch B (revision flow) — separate task
- Branch C (Ghost publication) — separate task
- Ratchet automation (needs 20+ drafts first)
- Ghost tag overhaul (Phase D, parallel)

## Acceptance Criteria

1. Dashboard page exists in Dashboards DB and shows all 6 pipeline stage views
2. All 4 new Content Creation DB views created and working
3. Content Production project page has mission, how-to, and architecture sections
4. 8+ resources tagged "Content Pipeline" in Resources DB
5. Pipeline Version populated on new runs
6. Duplicate trigger within 5 minutes is skipped
7. Workflow error writes comment to Notion page
