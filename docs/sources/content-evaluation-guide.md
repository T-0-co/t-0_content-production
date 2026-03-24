# T-0 Content Evaluation Guide — Evaluator Reference

> **Source:** Notion T-0 Resources DB → "T-0 Content Evaluation Guide"
> **Page ID:** `2d8c0fdf-942d-81e0-b644-fb36e207f5a7`
> **Status:** In Use (v1.0, 2025-12-29)
> **Role in pipeline:** This is the **evaluator** — the autoresearch `prepare.py` equivalent.

## Purpose

Systematic evaluation rubrics for scoring T-0 content quality. Covers both external content (blog posts, social media, newsletters) and internal documentation.

## Evaluation Tracks

| Track | Applies To | Dimensions |
|-------|-----------|------------|
| **Universal** | All content | Clarity, Structure, Accuracy |
| **External** | Blog, social, newsletter, website | Universal + Credibility, Engagement, Voice, Actionability |
| **Internal** | Team docs, system explanations, Slack writeups | Universal + Context, Usability, Efficiency, Currency |

## Scoring Scale (1-5)

| Score | Label | Meaning |
|-------|-------|---------|
| 1 | Poor | Fails to meet basic standards; needs complete rework |
| 2 | Below Standard | Significant issues; needs substantial revision |
| 3 | Acceptable | Meets minimum requirements; functional but not impressive |
| 4 | Good | Above average; minor improvements possible |
| 5 | Excellent | Exemplary; could serve as a reference for others |

## External Content Dimensions (Blog Pipeline)

### 1. Clarity
Can the intended audience understand this without re-reading?

### 2. Structure
Is the content organized in a way that serves the reader's needs?

### 3. Accuracy (Weight: 1.5x)
Is the information correct and verifiable?

### 4. Credibility (Weight: 1.5x)
Does this content demonstrate genuine expertise through showing the work? (Based on Google E-E-A-T)

### 5. Engagement
Does the content earn and hold attention?

### 6. Voice
Does this sound like T-0 — or like generic consultancy content?

### 7. Actionability
Does the reader know what to do next?

## Composite Scoring (External Content)

| Dimension | Weight | Max Weighted |
|-----------|--------|-------------|
| Clarity | 1.0 | 5 |
| Structure | 1.0 | 5 |
| Accuracy | 1.5 | 7.5 |
| Credibility | 1.5 | 7.5 |
| Engagement | 1.0 | 5 |
| Voice | 1.0 | 5 |
| Actionability | 1.0 | 5 |
| **Total** | **8.0** | **40** |

## Quality Tiers

| Score | Tier | Action |
|-------|------|--------|
| 34-40 | Excellent | Publish with confidence |
| 28-33 | Good | Minor polish recommended |
| 22-27 | Acceptable | Review specific weak dimensions |
| 16-21 | Below Standard | Significant revision needed |
| <16 | Poor | Requires rework |

## Readability Baseline (Flesch-Kincaid Grade Level)

| Content Type | Target Grade |
|-------------|-------------|
| Blog (general audience) | 8-10 |
| Blog (technical) | 10-12 |
| Social media | 6-8 |

## Red Flags (Auto-Cap at Score 3)

- **Clarity:** Jargon without explanation, passive voice dominates, main point not in first paragraph
- **Accuracy:** Claims without evidence, outdated stats, "Experten sagen" without naming experts
- **Credibility:** No specific examples/numbers, pronouncement confidence, could be written by anyone
- **Voice:** Banned terms (disruptiv, revolutionär, game-changer), Sie-Form, generic consultancy language

## JSON Schema (For Programmatic Evaluation)

```json
{
  "evaluation": {
    "content_id": "string",
    "content_type": "external | internal",
    "evaluated_at": "ISO-8601",
    "evaluator": "string",
    "readability": {
      "flesch_kincaid_grade": "number",
      "target_min": "number",
      "target_max": "number",
      "within_range": "boolean"
    },
    "dimensions": [
      {
        "name": "string",
        "score": "1-5",
        "weight": "number",
        "weighted_score": "number",
        "notes": "string"
      }
    ],
    "composite_score": "number",
    "max_possible": 40,
    "quality_tier": "excellent | good | acceptable | below_standard | poor",
    "strengths": ["string"],
    "improvements": [
      {
        "dimension": "string",
        "issue": "string",
        "suggestion": "string"
      }
    ],
    "recommendation": "publish | revise | rework"
  }
}
```

> **Note:** The full rubric with detailed per-score descriptions for each dimension lives in Notion (see page ID above). This file captures the scoring structure for programmatic use. Always reference the Notion page for the authoritative evaluation criteria.
