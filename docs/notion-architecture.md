# Notion Architecture — Content Production

All content production databases live under the **T-0 Content Production page** in the T-0 Main DB.

```text
T-0 Main db
└── T-0 Content Production page (331c0fdf-942d-8051-aee6-c56e4285ddad)
    ├── Content Sources db (inline)
    ├── T-0 Content Creation page
    │   └── T-0 Content Creation db (inline)
    └── T-0 Public Notion Content page
        └── Public Notion Content db (inline)
```

The **Content Production Dashboard** (`32dc0fdf-942d-8135-bddf-e39bbb62d100`, in Dashboards DB) links via linked database views — it does NOT own any databases.

The **Content Production Pipeline project** (`2b0c0fdf-942d-80ef-bf4c-e515c902c6d7`, in Projects DB) tracks progress only — NO databases.

## Databases

| Resource | Database ID | Data Source ID | Purpose |
|----------|-------------|----------------|---------|
| Content Creation | `2afc0fdf-942d-813e-bafa-fe13acf0b35f` | `2afc0fdf-942d-81b6-9bff-000be4682a2d` | Content tracking (Idea → Published) |
| Content Sources | `330c0fdf-942d-81b8-a839-fe9ea3c9ad15` | `330c0fdf-942d-81e4-a26d-000b8527f0f5` | Source registry (blogs, feeds, alerts) |
| Notes | `2afc0fdf-942d-819a-b859-e69dcc48d67d` | `2afc0fdf-942d-812c-975d-000b9108667f` | Raw items + manual notes + Slack captures |
| Reports | `2e8c0fdf-942d-8059-8aaa-d2fb0f6f3a4b` | `2e8c0fdf-942d-803b-b059-000bc702d4c3` | Periodic intelligence digests |

## Notes DB: Filtering Pipeline vs Human Notes

The `Source` relation (links to Content Sources DB) is the primary filter: **Source is not empty = pipeline-ingested note, Source is empty = human note**.

Notes DB has been cleaned to 25 properties. When writing notes, use:

- `Name` (title) — the note title
- `Tags` (multi_select) — general organizational tags ("Meeting Notes", "Idea", etc.)
- `Category Tags` (multi_select) — content production taxonomy ("AI Models", "MCP", etc.). Maps to Ghost blog tags.
- `Source` (relation) — link to Content Sources DB entry
- `Summary`, `Message`, `URL`, `Status`, `Time` — standard fields
- Slack-specific: `thread_ts_slack`, `channel_id_slack`, `contributors_slack`

**Tags vs Category Tags**: Tags is general-purpose (Notes DB serves the whole company). Category Tags is specifically for the content production taxonomy and carries over to Ghost's tagging system. Keep them separate.

Deleted properties (do NOT write to these): `Title`, `AI Title`, `Origin`, `Origin Description`, `Description`, `AI-Tags`, `Brand Product System Tags`, `trigger reaction`, `Checked`, `Reprocessing Notes`.

## Content Sources DB

14 properties. The `Frequency` property was deleted — all sources are checked at the same weekly cadence by WF-2. Use `Access Method` (RSS/Jina/Slack) and `Last Checked` for operational info.

## Composite Score (Formula)

The Content Creation DB's `Composite Score` is a formula: weighted average of 7 dimension scores (Voice, Clarity, Structure, Accuracy, Credibility, Engagement, Actionability), normalized to 1-5 scale.

All weights currently set to 1 (equal). Edit the formula in Notion to adjust (e.g., `* 1.5` for Accuracy).

**Do NOT write to `Composite Score`** — it is read-only (formula). Only write the 7 individual dimension scores.
