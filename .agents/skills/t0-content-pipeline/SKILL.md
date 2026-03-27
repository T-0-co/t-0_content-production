---
name: t0-content-pipeline
description: "T-0 blog content production pipeline (interactive). Same prompts, brand context, and evaluation rubric as the n8n pipeline — but executed conversationally using Claude. ONLY for T-0 (t-0.co) content. Triggers: /t0-content, T-0 blog post, T-0 draft, T-0 content"
argument-hint: "[topic or Notion page ID] [--type opinion|resource|case_study|technical] [--eval-only]"
---

# T-0 Content Pipeline (Claude Code)

> **Scope: T-0 only.** This skill produces content for `t-0.co` using T-0's brand voice, evaluation rubric, and Notion workspace. Do NOT use for other brands, businesses, or projects.

Same pipeline as WF-1 on n8n.t-0.co, but executed interactively. Uses the same prompts, brand voice framework, evaluation rubric, and Notion integration. The difference: Claude (via MCP tools) instead of OpenRouter, and Aaron is in the loop at every stage.

## When to Use

- Drafting a new blog post from an Idea in the Content Creation DB
- Iterating on a draft that needs more human guidance than the automated pipeline provides
- Evaluating an existing draft against the rubric (--eval-only)
- Experimenting with style parameter changes before committing them

## Invocation

`/t0-content [topic or page ID] [options]`

- `/t0-content` — prompts for topic
- `/t0-content 330c0fdf-...` — loads an existing Content Creation DB entry
- `/t0-content "Why MCP changes everything"` — starts from scratch with a topic
- `/t0-content 330c0fdf-... --eval-only` — evaluates the current draft, no rewrite

## Pipeline Phases

### Phase 0: Context Assembly

Before anything else, load all brand context. This is what makes the output T-0 and not generic.

1. **Style parameters** — read `config/style-params.yaml` from this repo
2. **Brand Writing Guide** — fetch Notion page `2d1c0fdf-942d-810d-8fda-cf181d93bf46` (voice, tone, AI agent guidance)
3. **Blog Content Guide** — fetch Notion page `2d1c0fdf-942d-81cf-bd59-f54a251a1571` (structural templates per content type)
4. **Blog Style Analysis** — fetch Notion page `2d7c0fdf-942d-813e-a679-e55b56cea58b` (industry patterns to emulate)
5. **Brand Process** — fetch Notion page `2d1c0fdf-942d-81a1-b123-f44749d581cd` (order of guide usage)

Use `mcp__notion__API-get-block-children` to fetch page content. These are Notion pages (not DB entries), so fetch their block children to get the actual content.

If a Notion page ID was provided, also fetch the Content Creation DB entry:
- Use `mcp__notion__API-retrieve-a-page` with the page ID
- Extract: Title, Content Type, Category Tags, Description, any existing Draft content, Voice Seeds, Source URLs

### Phase 1: Research

Use the research approach from `prompts/research-prompt.md`:

1. If source URLs are available (from the Notion entry or user input), read them with Jina (`mcp__jina__read_url`)
2. Use Perplexity (`mcp__perplexity__perplexity_ask`) for broader context, German market perspective, counterarguments
3. Structure research output per the prompt template: Kernfakten, DACH-Perspektive, Gegenargumente, Statistiken, Quellen

**Present research to Aaron** before proceeding. Ask if anything is missing or if the angle should shift.

### Phase 2: Draft

Construct the full system prompt by resolving all placeholders in `prompts/draft-system-prompt.md`:

| Placeholder | Source |
|-------------|--------|
| `{brand_writing_guide_voice_section}` | Phase 0: Brand Writing Guide content |
| `{blog_content_guide_template_for_{content_type}}` | Phase 0: Blog Content Guide, section matching content_type |
| `{blog_style_research_summary}` | Phase 0: Blog Style Analysis content |
| `{opinion_strength}` | `config/style-params.yaml` → `voice.opinion_strength` |
| `{personal_anecdote_required}` | `config/style-params.yaml` ��� `voice.personal_anecdote_required` |
| `{target_word_count}` | `config/style-params.yaml` → `structure.target_word_count.{content_type}` |
| `{opening_pattern}` | Weighted random from `config/style-params.yaml` → `opening.preferred_patterns` |
| `{closing_pattern}` | Weighted random from `config/style-params.yaml` → `closing.preferred_patterns` |
| `{banned_terms_list}` | "Synergien, ganzheitlich, Paradigmenwechsel, Disruption, Best-in-Class, state-of-the-art, leverage, game-changer, cutting-edge, revolutionär" |

Generate the draft using the resolved prompt + research context + any Voice Seeds from the Notion entry.

**Present the full draft to Aaron.** Ask for feedback before evaluating.

### Phase 3: Evaluate

Apply the evaluation rubric from `prompts/evaluation-prompt.md`:

1. Score all 7 dimensions (Clarity, Structure, Accuracy, Credibility, Engagement, Voice, Actionability)
2. Calculate weighted composite (max 40)
3. Apply gate rules:
   - Voice < 4 → BLOCKED
   - Any dimension < 3 → BLOCKED
   - Composite >= 28 AND all gates pass → PASS

**Present scores to Aaron** with the evaluation JSON and top strength / primary weakness.

If BLOCKED: explain which dimensions failed and why, suggest specific improvements.
If PASS: recommend next status ("Pre Review").

### Phase 4: Notion Update

If Aaron approves, update the Content Creation DB entry:

1. **If new topic (no existing page):** Create a new page in Content Creation DB (`2afc0fdf-942d-813e-bafa-fe13acf0b35f`)
   - Properties: Name (title), Content Type, Status ("Draft" or "Pre Review"), Category Tags
   - Add draft as page content (Notion blocks)

2. **If existing page:** Update the page
   - Clear existing content blocks and replace with new draft
   - Update Status based on evaluation result
   - Set Pipeline Version to current git short hash of this repo

3. **Always set these properties:**
   - `Composite Score` — the weighted total
   - `Quality Tier` — Excellent/Good/Acceptable/Below Standard/Poor
   - `Pipeline Version` — git short hash from this repo
   - `Evaluated At` — today's date
   - `Evaluation Notes` — top strength, primary weakness, word count delta
   - `Description` — short summary of topic, sources, score
   - `Platform` — `multi_select` with `[{"name": "Blog"}]`
   - All 7 dimension scores: `Clarity Score`, `Structure Score`, `Accuracy Score`, `Credibility Score`, `Engagement Score`, `Voice Score`, `Actionability Score`

Use `mcp__notion__API-patch-page` for properties and `mcp__notion__API-post-page` for page creation. **Important:** The `Platform` property is `multi_select` (not `select`).

**Content blocks:** The MCP `patch-block-children` tool only supports `paragraph` and `bulleted_list_item` block types. Blog drafts require `heading_2`, `heading_3`, and `heading_4` blocks. Use direct `curl` to the Notion API instead:

```bash
bash -c 'source /home/aaron/VSCodeProjects/.env && curl -s -X PATCH \
  "https://api.notion.com/v1/blocks/{page_id}/children" \
  -H "Authorization: Bearer $NOTION_INTEGRATION_SECRET" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d @/tmp/notion-blocks.json'
```

Write blocks to a temp JSON file first, then POST. Supported block types: `heading_2`, `heading_3`, `paragraph`, `bulleted_list_item`. Map draft H2→heading_2, H3→heading_3, H4→heading_3 (bold first line), bullets→bulleted_list_item, paragraphs→paragraph.

### Phase 5: Revision (if needed)

If Aaron requests changes after evaluation:

1. Read Aaron's feedback
2. Re-draft incorporating feedback (same prompt assembly, but include feedback as context)
3. Re-evaluate
4. Present new scores alongside previous scores (delta view)
5. Update Notion if approved

This mirrors WF-1 Branch B (revision loop).

## Eval-Only Mode

When invoked with `--eval-only`:

1. Fetch the existing draft content from the Notion page
2. Skip Phases 1-2 (research and draft)
3. Run Phase 3 (evaluate) on the existing content
4. Present scores and recommendation
5. Optionally update Notion properties (Composite Score, Quality Tier)

## Rules

- **Never auto-publish to Ghost.** This skill produces drafts in Notion only. Publication is a separate step (WF-1 Branch C or manual).
- **Never modify the evaluation rubric.** The rubric in `prompts/evaluation-prompt.md` is the immutable evaluator.
- **Always present intermediate results.** Research before drafting, draft before evaluating. Aaron is in the loop.
- **Record pipeline version.** Every draft must record which config version produced it.
- **German by default.** All drafts in German unless Aaron explicitly requests English.

## Notion IDs Quick Reference

| Resource | ID |
|----------|-----|
| Content Creation DB | `2afc0fdf-942d-813e-bafa-fe13acf0b35f` |
| Brand Writing Guide | `2d1c0fdf-942d-810d-8fda-cf181d93bf46` |
| Blog Content Guide | `2d1c0fdf-942d-81cf-bd59-f54a251a1571` |
| Blog Style Analysis | `2d7c0fdf-942d-813e-a679-e55b56cea58b` |
| Brand Process | `2d1c0fdf-942d-81a1-b123-f44749d581cd` |
| Content Evaluation Guide | `2d8c0fdf-942d-81e0-b644-fb36e207f5a7` |
