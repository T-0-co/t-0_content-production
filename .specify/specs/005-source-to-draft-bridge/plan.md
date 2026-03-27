# Spec 005 Plan: Source-to-Draft Bridge

## Status: Complete

## Approach

Extend WF-3 (Intelligence Digest) rather than create a new workflow. The cluster data is already in memory — adding a parallel branch after report creation is simpler than a separate workflow that would need to re-fetch and re-parse.

## Implementation Summary

Two nodes added to WF-3 as a parallel branch from "Append Report Blocks":

1. **Prepare Content Proposals** (Code node) — reads clusters from AI Cluster response, filters to clusters with 3+ items, sorts by item count, takes top 5, builds Notion page creation payloads
2. **Create Content Entries** (HTTP Request) — POSTs each proposal to Notion Content Creation DB

Both run in parallel with the existing note status update branch (Prepare Note Updates → Update Note Status).

## Human Workflow

1. WF-3 runs weekly (Monday 09:00) or on-demand via webhook
2. 3-5 entries appear in Ideas Inbox (Status = "Idea", Source Type = "Pipeline Suggested")
3. Aaron/Julien review, refine pitches, adjust tags
4. Click "Run Pipeline" on best entries → WF-1 handles research, drafting, evaluation, Ghost publishing
5. Reject others by archiving or leaving as Idea

## Data Flow

```
WF-2 (weekly)           WF-3 (weekly)              WF-1 (on-demand)
Sources → Notes    →    Notes → Clusters    →    CC Entry → Draft → Ghost
  23 sources               3-8 clusters              Research → Evaluate
  ~200 items/run           Top 5 → CC entries         → Revise → Publish
```
