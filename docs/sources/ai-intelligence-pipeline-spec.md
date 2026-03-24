# AI Intelligence Pipeline — Original Spec

> **Source:** Notion T-0 Projects DB → "AI Intelligence Pipeline"
> **Page ID:** `30dc0fdf-942d-81de-8718-d03ae0bf2259`
> **Status (in Notion):** Not Started
> **Owner:** Aaron Kokal
> **Last edited:** 2026-02-20

## Vision

Build a passive AI industry intelligence system that continuously collects signals from a wide catalogue of sources, triages them automatically using a lightweight AI model, and delivers consolidated, relevant insights to the team via Notion and Slack.

The goal is not active scraping — it's connecting to as many existing data streams as possible (RSS, alerts, newsletters, API feeds) and letting AI sort through the noise.

## Architecture Overview

### 1. Source Catalogue & Collection Layer
- **Google Alerts** — AI industry news, new AI consultancies in Germany, competitor movements, key technology announcements
- **RSS feeds** — Major model providers (OpenAI, Anthropic, Google DeepMind, Meta AI, Mistral), AI news outlets (The Verge AI, TechCrunch AI, Ars Technica, Import AI, The Batch), Hacker News AI-tagged posts
- **Newsletter ingestion** — Parse incoming AI newsletters from Gmail (if feasible)
- **GitHub trending** — New AI/ML repos gaining traction
- **ArXiv feeds** — Key ML/AI categories for breakthrough papers

Explore existing open-source frameworks: Miniflux, FreshRSS, Huginn, changedetection.io, or custom collectors.

### 2. Raw Data Store (Server-Side)
- Database on kokal.cloud (Hetzner server) — stores all incoming items with metadata (source, timestamp, raw content, URL)
- PostgreSQL or SQLite — depending on volume expectations
- Retention policy — keep raw data for N months, then archive or purge

### 3. AI Triage Layer
- Lightweight model (Haiku-class or smaller local model) — must be cheap enough for high-volume processing
- **Deduplication** — fuzzy matching across sources (same story from 5 outlets → 1 entry)
- **Relevance scoring** — configurable criteria: relevance to T-0's consulting focus, German market, AI tooling, enterprise adoption
- **Categorization** — tag items by topic (new models, funding, regulation, tools, research, competitors)
- Only items above a relevance threshold get promoted to Notion

### 4. Notion Output
- Write curated items to T-0 Notes DB — with source URL, summary, tags, relevance score
- Status: "To Be Processed" — so they enter the normal processing workflow
- Link back to this project via relation

### 5. Slack Briefings
- Dedicated #ai-briefing channel (or similar)
- Daily or weekly digest (TBD) — formatted summary of top items
- Brief format: headline, 1-2 sentence summary, source link, relevance tag
- Possibly also a weekly "deep dive" on the most significant development

## Open Questions
1. Which RSS aggregator/framework to use? Self-hosted (Miniflux, FreshRSS) vs. custom collector vs. n8n RSS nodes
2. Which AI model for triage? Haiku, Gemini Flash, local LLM (Phi, Qwen), or embedding-based similarity?
3. Daily vs. weekly briefing cadence — or both (daily Slack ping + weekly deep digest)?
4. Which Google Alerts specifically? Need to define the full keyword/topic catalogue
5. Database choice for raw store — Postgres (already on server) vs. SQLite vs. dedicated search DB (Meilisearch)?
6. Should triaged items also be searchable via the T-0 Hub, or just Notion + Slack?

## Implementation Phases

### Phase 1: Research & Source Catalogue
- Evaluate open-source aggregation frameworks
- Define the full list of Google Alert keywords and RSS feed URLs
- Set up Google Alerts on the T-0 Workspace account

### Phase 2: Collection Infrastructure
- Deploy chosen aggregation tool on kokal.cloud
- Set up raw data database
- Connect all sources and validate data flow

### Phase 3: AI Triage
- Build triage pipeline (dedup, relevance scoring, categorization)
- Define and tune relevance criteria for T-0's interests
- Connect promoted items → T-0 Notes DB in Notion

### Phase 4: Briefing System
- Create Slack channel and briefing format
- Build digest generation (daily/weekly TBD)
- Possibly integrate with T-0 Hub as a queryable skill

## Tech Stack Candidates
- **RSS aggregation:** Miniflux (Go, lightweight, API-first) or FreshRSS (PHP, more features) or Huginn (Ruby, agent-based, very flexible)
- **Orchestration:** n8n workflows (already deployed) for gluing pieces together
- **Triage model:** Claude Haiku, Gemini Flash, or a local model like Phi-3/Qwen2 for cost efficiency
- **Deduplication:** Embedding-based similarity (sentence-transformers) + fuzzy title matching
- **Database:** PostgreSQL (already on server) or Meilisearch for full-text search
- **Deployment:** Docker on kokal.cloud alongside existing services
