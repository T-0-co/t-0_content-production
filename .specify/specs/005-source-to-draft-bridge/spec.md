# Spec 005: Source-to-Draft Bridge

## Problem

The content production pipeline has two disconnected halves:

1. **Intake** (WF-2 + WF-3): Sources are ingested, triaged, and clustered into a weekly intelligence digest with topic clusters and content angles.
2. **Production** (WF-1): A Content Creation entry is researched, drafted, evaluated, revised, and published to Ghost.

Nothing connects them. A human must read the digest, manually create Content Creation entries, and fill in the pitch, tags, and source references by hand. This defeats the purpose of having an automated intake pipeline.

## Solution

Extend WF-3 (Intelligence Digest) with a bridge that automatically creates Content Creation entries from high-quality topic clusters. Entries are created with Status = "Idea" so they appear in the Ideas Inbox for human review. Aaron or Julien promotes the best ones by clicking "Run Pipeline", which triggers WF-1.

## Architecture

```
WF-3 (existing)                          Bridge (new)
────────────────                          ────────────
Fetch Notes
  → Prepare Clustering
  → AI Cluster
  → Parse Clusters & Build Report
  → Create Report
  → Add Report Body
  → Append Report Blocks ──────────────→ Filter Publishable Clusters
                                            → Prepare Content Proposals
                                            → Create Content Entries
  → Prepare Note Updates
  → Update Note Status
```

The bridge runs in parallel with the note status updates (both downstream of Append Report Blocks). This way, Content Creation entries and note status updates happen independently.

## Content Entry Creation Rules

### Cluster Selection (Filter Publishable Clusters)
- Only clusters with **3+ items** are eligible (ensures substantive topics)
- Maximum **5 entries per digest** (prevents flooding the Ideas Inbox)
- Clusters are ranked by item count (more items = more signal)

### Entry Fields (Prepare Content Proposals)

| CC Field | Value | Source |
|----------|-------|--------|
| Name | Cluster title | `cluster.title` |
| Status | Idea | Hardcoded — human approval gate |
| Pitch | Content angle from AI clustering | `cluster.content_angle` |
| Category Tags | Union of tags from cluster's Notes | Items' `category_tags` |
| Source Type | "Pipeline Suggested" | Hardcoded |
| Content Type | "Opinion" | Default — human can change |
| Notes | Relation to all Notes in the cluster | `cluster.item_indices` → `notePageIds` |
| Report | Relation to this week's Report | Report page ID from Create Report |
| Description | Cluster summary + relevance | `cluster.summary` + `cluster.relevance` |

### Human Workflow

1. WF-3 runs weekly (Monday 09:00) or on-demand via webhook
2. Ideas appear in "Ideas Inbox" view in Content Creation DB
3. Aaron/Julien review, edit pitch/tags if needed
4. Click "Run Pipeline" on promising entries → WF-1 drafts them
5. Reject others by archiving or leaving as Idea

## Implementation

### Nodes to Add (3 nodes)

**1. Filter Publishable Clusters** (Code node)
- Input: Parse Clusters output (clusters array, items array, notePageIds, reportId)
- Logic: Filter to clusters with 3+ items, sort by item count desc, take top 5
- Output: Array of proposal objects with all CC fields prepared

**2. Prepare Content Proposals** (Code node)
- Input: Filtered clusters
- Logic: Build Notion API request bodies for each CC entry
- Output: One item per proposal with `requestBody` JSON string
- Handles: category tag dedup, note relation mapping, description assembly

**3. Create Content Entries** (HTTP Request node)
- Method: POST
- URL: `https://api.notion.com/v1/pages`
- Auth: `notionApi` credential
- Body: `{{ $json.requestBody }}`
- Runs once per item (n8n loops automatically)

### Connection Changes

Current: `Append Report Blocks` → `Prepare Note Updates`

New:
- `Append Report Blocks` → `Prepare Note Updates` (unchanged)
- `Append Report Blocks` → `Filter Publishable Clusters` → `Prepare Content Proposals` → `Create Content Entries` (new parallel branch)

## Risks

- **Duplicate entries**: If the same topic cluster appears in consecutive weeks, duplicate CC entries could be created. Mitigation: the weekly title includes enough specificity that duplicates are visible in the inbox. Future: add title dedup check.
- **Category tag mismatch**: Notes use `Category Tags` (select, single value) while CC uses `Category Tags` (multi_select). The bridge collects unique tags from all cluster items.
- **Flooding**: Capped at 5 entries per run. With weekly cadence, that's max 5 new Ideas per week — manageable.

## Success Criteria

1. Weekly digest automatically produces 2-5 Content Creation entries
2. Entries have meaningful pitches and correct source references
3. Human can review and trigger WF-1 with one click
4. No manual data entry required to go from source → draft
