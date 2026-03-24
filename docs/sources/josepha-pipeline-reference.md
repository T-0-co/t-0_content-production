# Josepha Content Pipeline — Architecture Reference

> **Source:** `/home/aaron/VSCodeProjects/personal-projects/Josepha/`
> **Status:** Fully operational (different business, same pattern)
> **Role:** Working template/analog for the T-0 pipeline

## Why This Matters

The Josepha project implements a fully automated content pipeline for a different business (postpartum/mommy content). The architecture is directly transferable to T-0's content production pipeline.

## Architecture: Three n8n Workflows

### 1. Mommy Scene Weekly Report
- **Workflow ID:** `FL4UWECfEsF31K9S` (automation.kokal.cloud)
- **Trigger:** Cron, Sundays 8AM
- **Flow:** RSS fetch from 23 sources → Jina Reader for homepage fallback → Perplexity AI triage (top 10-12 articles) → deep article fetch → structured report generation → Notion output with markdown-to-blocks converter
- **Key detail:** Tags feed items as "Used in Report" to avoid re-processing

### 2. Generate Posts From Report
- **Workflow ID:** `gU9V7ovOD3LhW2gw`
- **Trigger:** Webhook (Notion button on Reports)
- **Flow:** Reads report body + all 3 CI wiki pages → Perplexity generates 3-5 post ideas with captions, hashtags, image prompts, aspect ratios

### 3. Image Generation
- **Workflow ID:** `HWQHisDe1yiAi83Z`
- **Trigger:** Two entry points (direct webhook + Notion button)
- **Flow:** Fetches illustration style guide from Notion → enhances prompt with Perplexity → selects reference image via AI Agent (OpenRouter) → generates with Gemini → uploads to Google Drive → writes back to Notion
- **Key detail:** Optional photo-to-illustration transformation mode

## APIs Used
- Notion (content tracking, CI docs)
- Perplexity (sonar-pro/sonar — triage, content generation)
- Google Gemini (image generation)
- Jina Reader (article fetching)
- Google Drive (image storage)
- RSS feeds (source ingestion)

## Key Patterns Transferable to T-0

1. **RSS → AI triage → structured output** — Same pattern needed for AI Intelligence Pipeline
2. **Notion button triggers webhook** — Same UX for T-0 content creation
3. **CI docs loaded at generation time** — Same approach for Brand Writing Guide / Blog Content Guide
4. **Reference image selection** — Same pattern already used in T-0's existing image gen workflow
5. **Markdown-to-Notion-blocks converter** — Reusable utility

## Documentation
Full node-by-node workflow docs at `personal-projects/Josepha/docs/n8n-workflows.md`
