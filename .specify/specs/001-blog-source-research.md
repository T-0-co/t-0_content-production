# Spec 001: Blog Source Research — Finding the Right Voices to Learn From

**Created**: 2026-03-24
**Status**: Complete
**Type**: Research (not a software feature)

## Context

T-0 has a Blog Style Analysis that covers 3 sources (Sequoia, LangChain, Fortune). The Brand Writing Guide v2.0 changelog mentions Simon Willison, TDS, and t3n as additional references but their patterns were never documented. The team shares sources in #tech-shit-talk (Beyond the Hype, FAZ, CIO, ai-2027, Forte Labs) that haven't been analyzed for writing style.

Before building the content production pipeline's drafting engine, we need a proper understanding of what good looks like — not from AI consultancies (we don't want to sound like everyone else), but from diverse voices that do excellent work in their respective fields.

## Goal

Analyze 10-15 blogs/publications across diverse fields for structural patterns, voice techniques, and content strategies that could inform T-0's content pipeline. Produce a research document that extends the existing Blog Style Analysis.

## Selection Criteria

Sources should be selected for **craft quality**, not topical similarity. We're looking for:
- Distinctive voice (you'd recognize the source without seeing the byline)
- "Show the work" credibility (earned confidence, not pronouncement)
- Structural innovation (not just listicles or generic how-tos)
- Diversity across: solo practitioners, company blogs, newsletters, German/European voices

## Research Targets

### Tier 1 — Already read by the team (analyze writing style, not just content)

| Source | Type | Why |
|--------|------|-----|
| **Simon Willison** (simonwillison.net) | Solo developer blog | Mentioned in Brand Writing Guide v2.0 but never analyzed. Master of "show the work." |
| **Beyond the Hype** (beyondthehype.dev) | Technical analysis | Team already reads this. Deep technical reverse-engineering. |
| **Forte Labs** (fortelabs.com) | Knowledge management | Aaron subscribes. Tiago Forte's ability to make abstract frameworks tangible. |
| **t3n** (t3n.de) | German tech media | German market reference. How does good German tech writing work? |

### Tier 2 — Exemplary craft from adjacent/different fields

| Source | Type | Why |
|--------|------|-----|
| **Julia Evans** (jvns.ca) | Developer education | Makes complex topics accessible through illustrations and zines. Distinctive style. |
| **The Pragmatic Engineer** (blog.pragmaticengineer.com) | Engineering leadership newsletter | Gergely Orosz. European voice. Data-heavy, opinionated, show-the-work. |
| **Paul Graham** (paulgraham.com) | Essays | The archetype of opinionated tech writing. Contrarian, clear, influential. |
| **Stratechery** (stratechery.com) | Tech strategy analysis | Ben Thompson. Framework-heavy analysis. How to build an analytical voice. |
| **Anthropic's blog** (anthropic.com/research) | AI research comms | How a frontier lab communicates technical work accessibly. |
| **Lenny's Newsletter** (lennyrachitsky.com) | Product management | Clear frameworks, data-driven, massive audience. |
| **Intercom blog** (intercom.com/blog) | Product/customer comms | European, practical, opinionated about building products. |

### Tier 3 — Wild cards (style inspiration from unexpected places)

| Source | Type | Why |
|--------|------|-----|
| **Wait But Why** (waitbutwhy.com) | Long-form explainers | Tim Urban. Makes enormously complex topics accessible through humor and illustration. |
| **Daring Fireball** (daringfireball.net) | Tech opinion | John Gruber. 20 years of distinctive voice. How to maintain opinion without becoming shrill. |
| **Import AI** (importai.substack.com) | AI newsletter | Jack Clark. Concise analytical newsletter format. |

## Deliverable

A research document at `docs/sources/blog-style-research.md` containing:

1. **Per-source analysis** (for each target):
   - Opening patterns (how do they hook readers?)
   - Structure (how do they organize content?)
   - Voice (what makes them distinctive?)
   - Evidence patterns (how do they build credibility?)
   - Frequency and length
   - One "best example" post with URL
   - Key takeaway for T-0

2. **Cross-source patterns** — what do the best ones have in common?

3. **Recommendations for T-0** — which patterns to adopt, adapt, or avoid, mapped to T-0's existing Brand Writing Guide voice principles

## Success Criteria

- At least 10 sources analyzed with real examples (not surface-level summaries)
- Patterns documented are actionable (a writer could apply them, not just admire them)
- Research explicitly connects findings to T-0's existing voice principles
- Document is useful as a reference when drafting blog posts

## What This Is NOT

- Not a competitive analysis of AI consultancies
- Not a "best practices" listicle
- Not about finding sources to copy — it's about understanding craft
