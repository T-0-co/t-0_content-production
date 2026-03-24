# Blog Style Research: 14 Sources Analyzed for T-0 Content Strategy

> **Spec:** 001-blog-source-research
> **Date:** 2026-03-24
> **Status:** Complete
> **Extends:** Notion "Blog Style Analysis" (Sequoia, LangChain, Fortune — Dec 2025)

## Selection Philosophy

These sources were chosen for **craft quality**, not topical similarity. T-0 deliberately does not want to sound like other AI consultancies. The selection spans solo practitioners, company blogs, newsletters, German media, and wild cards from unrelated fields. The goal: learn from diverse excellence, then synthesize a voice that is unmistakably T-0.

---

## Tier 1: Sources the Team Already Reads

### 1. Simon Willison (simonwillison.net)

**Voice:** Enthusiastic, precise, conversational, self-deprecating, transparent.

**Opening pattern:** "Here's what happened → Here's why it matters" — drops you into concrete action immediately, never abstracts first.

> *"I vibe coded my dream macOS presentation app. I gave a talk this weekend at Social Science FOO Camp..."*

**Structure:** Narrative-chronological with embedded digressions. Dense hypertext web of links to his own prior work. Uses `####` subheadings liberally as section markers.

**Evidence:** Radical transparency of process — exact prompts quoted verbatim, source code on GitHub, specific numbers ("355KB, or 76KB compressed", "$23 of Codex tokens"). Almost entirely first-person empirical: "I tried this, here's what happened."

**Signature move:** Honest admissions of contradiction: *"I mostly run Claude with dangerously skip permissions even though I'm the world's foremost expert on why you shouldn't do that."*

**Frequency:** Essentially daily. Mix of full articles (1,500-4,000 words) and link posts (200-500 words).

**Best example:** [Pragmatic Summit fireside chat](https://simonwillison.net/2026/Mar/14/pragmatic-summit)

**Key takeaway for T-0:** Show your exact process, including the prompts. Never just say "we helped Company X implement AI" — show the prompt, the iteration, the failure, the code.

---

### 2. Beyond the Hype (beyondthehype.dev) — Yifan Zhao

**Voice:** Investigative, data-driven, slightly conspiratorial. Like a security researcher who blogs.

**Opening pattern:** "I investigated X. Here are the numbers that surprised me." — leads with proprietary data from personal tracking.

> *"Over the past 4 months, I tracked 103 feature implementation runs across Claude Code and Cursor. Here's what I discovered: 78% of implementations required at least one debugging session..."*

**Structure:** Investigation-report skeleton. Hook with data → methodology → findings → "Stuff that didn't make the video" (newsletter exclusive) → actionable takeaways split by audience.

**Evidence:** Self-conducted benchmarks and reverse engineering. Head-to-head comparisons with specific task counts. Technical artifacts: decompiled code, intercepted API traffic.

**Frequency:** Irregular (~monthly). Posts are 1,500-2,000 words, designed as newsletter companions to YouTube videos.

**Best example:** [Inside Claude Code](https://beyondthehype.dev/p/inside-claude-code-prompt-engineering-masterpiece)

**Key takeaway for T-0:** Lead with proprietary data from your own practice. Track metrics from real client engagements (anonymized) — success rates, iteration counts, time savings — and lead posts with those numbers. Original data is the strongest differentiator.

---

### 3. Forte Labs (fortelabs.com/blog) — Tiago Forte

**Voice:** Earnest, philosophical, confessional, slightly grandiose. Warm but ambitious.

**Opening pattern:** Personal narrative that creates emotional stakes before revealing the framework.

> *"In early 2023, after teaching 6,000 students over six years, I abruptly decided to stop teaching our Building a Second Brain cohorts. Even at that early stage... it felt like so much of what I'd built had suddenly become obsolete."*

**Structure:** For essays: Personal story → universal problem → named framework → contrarian positioning → CTA. For annual reviews: radical business transparency (exact revenue, subscriber counts, conversion rates).

**Evidence:** Business metrics ($2.15M revenue, 373k subscribers), personal experience narratives, occasional named statistics. Does NOT show technical process.

**Signature move:** Names everything. CODE, PARA, ARC Method, "the Perspective Era." A named framework is memorable, shareable, and positions the author as originator.

**Frequency:** Roughly monthly. 2,500-4,500 words. Long-form, not quick reads.

**Best example:** [The Book I've Been Waiting to Write for 15 Years](https://fortelabs.com/blog/the-book-ive-been-waiting-to-write-for-15-years)

**Key takeaway for T-0:** Name your frameworks. T-0 should name its consulting approaches and AI implementation patterns. Even a simple process becomes a shareable asset when it has a name.

---

### 4. t3n (t3n.de)

**Voice:** Authoritative, explanatory, balanced, German-formal-but-modern. Institutional (not personal).

**Opening pattern:** Provocative framing statement → "Aber/Doch" pivot → the actual story.

> *"KI-Tools gelten in vielen Unternehmen als Schlüssel zu mehr Effizienz. Nvidia-Chef Jensen Huang soll sogar erklärt haben... Aber während viele noch damit kämpfen..."*

**Structure:** Classic German tech journalism. Teaser → opening context with authority hook → 3-4 H2 sections → embedded editorial recommendations → closing implications.

**Evidence:** Sourced external data and expert quotes with name/title/company. International comparisons with specific numbers. Synthesis journalism — aggregating and contextualizing.

**Frequency:** Multiple articles per day (multi-author). 600-1,000 words. 3-5 minute reads.

**Best example:** [KI-Nutzung in Deutschland](https://t3n.de/news/ki-nutzung-in-deutschland-verdoppelt-aber-warum-bleiben-die-usa-stehen-1734485)

**Key takeaway for T-0:** Use the German market contrast angle. Bridge the English-language AI discourse with German market specifics. The "Germany is different, here's how and why" angle is underserved.

---

## Tier 2: Exemplary Craft From Adjacent Fields

### 5. Julia Evans (jvns.ca)

**Voice:** Curious, humble, precise, enthusiastic, informal. Sounds like a very smart person who is genuinely delighted to be confused.

**Opening pattern:** Casual, conversational lead that drops you into what she's been doing. No thesis, no grand claim.

> *"Hello! My big takeaway from last month's musings about man pages was that examples in man pages are really great..."*

**Signature:** The word "Hello!" opens many posts. Frank admissions of ignorance without performing them.

**Structure:** "Here's what I tried / here's what I learned." Brief context → the thing she made → several short sections exploring facets → honest reflection on what she still doesn't understand → no conclusion (just stops).

**Evidence:** Process, not authority. Links to specific commits, tools, people by name. Credits reviewers. Never cites statistics.

**Frequency:** Every 3-6 weeks. 800-2,000 words. Short by newsletter standards.

**Best example:** [Examples for the tcpdump and dig man pages](https://jvns.ca/blog/2026/03/10/examples-for-the-tcpdump-and-dig-man-pages)

**Key takeaway for T-0:** The "I tried X and learned Y" format. Honest process documentation is more compelling than polished thought leadership. Specificity is the trick.

---

### 6. The Pragmatic Engineer (blog.pragmaticengineer.com) — Gergely Orosz

**Voice:** Authoritative, European, skeptical-then-fair, experience-weighted. The "however" pivot is his signature rhetorical move.

**Opening pattern:** Contrarian setup — states the prevailing narrative, then signals he's about to complicate it.

> *"I have been sceptical of the manifold claims that SaaS will be killed by LLMs..."*

**Structure:** Two formats. "The Pulse" (news analysis): numbered sections each tackling one angle. Deep dives: thesis + alternating narrative and evidence. Heavy bold for key claims.

**Evidence:** Screenshots, tweets, revenue figures, survey data, named sources. Embeds actual tweets, shows actual UIs, quotes actual conversations.

**Frequency:** ~2x/week. 2,000-6,000+ words.

**Best example:** [The Pulse: Cloudflare rewrites Next.js](https://blog.pragmaticengineer.com/the-pulse-cloudflare-rewrites-next-js-as-ai-rewrites-commercial-open-source)

**Key takeaway for T-0:** The "numbered breakdown of a current event" format. When something happens in AI that affects Mittelstand, publish a rapid analysis with 4-5 numbered sections. Timeliness + structure + opinion = authority.

---

### 7. Paul Graham (paulgraham.com)

**Voice:** Oracular, plain-spoken, contrarian, confident. Short sentences. Simple vocabulary. "I" frequently making universal claims.

**Opening pattern:** Deceptively simple question or observation that sounds naive, then signals depth.

> *"There are two senses in which writing can be good: it can sound good, and the ideas can be right."*

**Structure:** A single unbroken argument with footnotes. No headings, no bullet points, no images. Just paragraphs. The structure IS the argument.

**Signature move:** The analogy that reframes — writing is like shaking a bin of objects. Swiss watches are a case study in brand. One great analogy beats ten data points.

**Evidence:** Almost none externally. His own reasoning, experience, and historical narratives. Builds credibility through argument quality alone.

**Frequency:** 2-4 essays per year. 1,500-8,000+ words.

**Best example:** [Good Writing](https://paulgraham.com/goodwriting.html)

**Key takeaway for T-0:** The single powerful analogy from another industry. "What Swiss watchmaking teaches us about AI adoption" is the kind of post T-0 should write. Find concrete analogies from manufacturing, craftsmanship, or German business culture.

---

### 8. Stratechery (stratechery.com) — Ben Thompson

**Voice:** Analytical, framework-obsessed, professorial, confident, patient. Trusts readers to follow complex arguments.

**Opening pattern:** Framing from outside tech (geopolitics, economics, philosophy) that pivots to the tech argument. Longer openings than anyone else — building elaborate frames.

**Structure:** Highly formal with markdown headings. Extended frame → 3-5 named sections each building on previous → synthesis. Self-references to prior posts ("I've talked about three LLM inflection points").

**Signature move:** The framework callback — creates conceptual frameworks (Aggregation Theory, the Smiling Curve) and applies them to new situations for years.

**Evidence:** Blockquotes from primary sources, framework callbacks, logical deduction. Rarely uses charts. Narrative and logical evidence.

**Frequency:** 3-4 posts/week. 3,000-6,000 words. Remarkably consistent.

**Best example:** [Agents Over Bubbles](https://stratechery.com/2026/agents-over-bubbles)

**Key takeaway for T-0:** Develop 2-3 proprietary frameworks and consistently apply them to new events. Every time something happens in AI, T-0 should be the one applying their framework to explain what it means for German mid-market companies.

---

### 9. Anthropic Blog (anthropic.com/research)

**Voice:** Measured, precise, intellectually generous, institutional-but-warm. Expresses genuine surprise when findings are unexpected.

**Opening pattern:** Calm, authoritative context-setting that grounds before making claims.

> *"Over the past year, we've worked with dozens of teams building LLM agents. Consistently, the most successful implementations weren't using complex frameworks."*

**Structure:** Most explicitly structured format analyzed. Executive summary → definitional section → taxonomy of patterns → each pattern follows identical sub-structure (description, diagram, "When to use:", "Examples:") → practical guidance → appendix.

**Signature move:** Admits limitations explicitly: *"Even on short, simple prompts, our method only captures a fraction of the total computation."*

**Evidence:** Diagrams, structured taxonomies, case studies from customers, references to published papers. Experimental results with interventions.

**Frequency:** 2-4 posts/month. 3,000-8,000 words.

**Best example:** [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)

**Key takeaway for T-0:** The practical taxonomy that becomes reference vocabulary. Create definitive categorizations: "The 5 Patterns of AI Integration for Manufacturing." Each pattern: description, when to use, when NOT to use, real example. If you name and categorize well enough, your taxonomy becomes the shared language.

---

### 10. Lenny's Newsletter (lennysnewsletter.com) — Lenny Rachitsky

**Voice:** Warm, structured, practical, generous, community-oriented. Positions himself as curator and amplifier rather than sole expert.

**Opening pattern:** Warm expert intro ("April Dunford is the world's leading expert...") → "Let's get into it." (every post) → substantive hook.

**Structure:** Problem → named framework → framework explained with visual metaphor → each layer explored → diagnostic checklist. Aggressive bold text for scannability.

**Signature move:** Superlative endorsement of guest experts. Named frameworks with visual metaphors ("The Waterline Model" with a boat metaphor).

**Evidence:** Named companies (Stripe, Anthropic), named leaders, personal anecdotes from guests. No data charts. Experiential and name-weighted.

**Frequency:** Weekly. 3,000-5,000 words.

**Best example:** [How to Debug a Team That Isn't Working](https://www.lennysnewsletter.com/p/how-to-debug-a-team-that-isnt-working)

**Key takeaway for T-0:** Named framework + visual metaphor. Not "how to evaluate AI readiness" but "The AI Readiness Waterline." The Waterline Model will be referenced for years because it has a name and an image.

---

### 11. Intercom Blog (intercom.com/blog)

**Voice:** Product-confident, future-oriented, opinionated-but-measured, institutional. Strong opinions about where customer service is heading.

**Opening pattern:** Bold thesis sentence → immediate contextualization. Declarative, forward-looking. No personal anecdotes.

**Structure:** Series-based approach (5-part deep dives). Individual posts: bold thesis → reference to underlying report → 3-4 H2 sections → customer case study → forward-looking conclusion.

**Signature move:** Creates a proprietary report, then generates 4-6 blog posts from it. The report is the authority; the posts are distribution.

**Evidence:** Own survey data, customer case studies with named people and specific metrics (84% resolution rate, 130% sales increase), quotes from internal leaders.

**Frequency:** ~2 posts/week. 1,500-3,000 words.

**Best example:** [Preparing for the Customer Agent Future](https://www.intercom.com/blog/preparing-for-the-customer-agent-future)

**Key takeaway for T-0:** The proprietary report as content engine. Create a periodic "State of AI in German Mittelstand" report. It fuels 4-6 blog posts, each covering one dimension. Owning the data means owning the narrative.

---

## Tier 3: Wild Cards

### 12. Wait But Why (waitbutwhy.com) — Tim Urban

**Voice:** Conversational, irreverent, earnest, nerdy, anxiously enthusiastic. Writes like a smart friend who went down a rabbit hole for three weeks.

**Opening pattern:** Disarmingly simple observation or participatory question.

> *"What does it feel like to stand here?"* (five words, attached to a graph of exponential progress)

**Structure:** Escalating zoom pattern. Familiar ground → reframing → escalation → the gut punch (abstract becomes personal) → takeaways. Each section ratchets up stakes.

**Signature moves:** Made-up taxonomies that stick (Die Progress Unit, Instant Gratification Monkey). Casual profanity deployed at precise moments. Dot grids and visual math as emotional arguments.

**Evidence:** Visual math — dot grids representing every day of a life, time-scaled bars, boxes representing "remaining visits." Not academic citations but conceptual frameworks that make data feel inevitable.

**Frequency:** Extremely infrequent (essentially dormant). 3,000-15,000+ words.

**Best example:** [The Tail End](https://waitbutwhy.com/2015/12/the-tail-end.html) — visual math to deliver an emotional gut punch about mortality.

**Key takeaway for T-0:** Visual reframing of familiar concepts. Show a client "you have 14 quarters left before your competitors' AI advantage becomes irreversible" using a dot grid, and they'll feel it differently than reading the same sentence.

---

### 13. Daring Fireball (daringfireball.net) — John Gruber

**Voice:** Precise, opinionated, wry, controlled, literate, occasionally scorching. 20+ years of earned trust.

**Opening pattern:** Devastatingly direct declarative statement, or quotes someone and reacts.

> *"My mom died at the end of June this year."* (then immediately acknowledges the reader's discomfort)

**Structure:** Two modes. Linked List (daily): quoted passage + 1-3 sentences of commentary. Long-form (monthly): historical context → implicit thesis → detailed analysis → digressions that reinforce → quiet landing.

**Evidence:** Accumulated specificity. Tables of photos shot with each lens (338/3,076/476). Exact pricing matrices. Physical measurements. When speculating, always flags it ("my gut feeling").

**Frequency:** Linked List items: multiple per day, for 20+ years straight (50-200 words). Long-form: 1-3/month (1,000-8,000 words).

**Best example:** [How It Went](https://daringfireball.net/2024/11/how_it_went) — personal narrative braided with media analysis.

**Key takeaway for T-0:** The link-commentary rhythm as trust architecture. Frequent brief posts curating AI developments with a consistent voice. Over months, this builds a reputation as a reliable lens. The long-form pieces then land harder because readers already trust the voice.

---

### 14. Import AI (jack-clark.net) — Jack Clark

**Voice:** Sober, informed, quietly alarmed, analytical. Former OpenAI policy director.

**Opening pattern:** Standard tagline → straight into bolded research summary with italicized editorial sub-headline (where his voice lives).

**Structure:** Rigid, repeatable skeleton. Intro → 3-5 research items (each with headline, analysis, bulleted findings, bold "Why this matters" section, read-more link) → "Tech Tales" (short fiction inspired by the issue's themes).

**Signature move:** The "Why this matters" escalation — consistently elevates technical findings to civilizational stakes. Plus the Tech Tales fiction pieces, which process the week's research emotionally.

**Evidence:** Direct quotes from research papers, benchmark scores, arXiv citations, self-referential issue numbering (proves continuity). Credibility through demonstrated reading volume.

**Frequency:** Weekly, without fail, 440+ issues since ~2017. 2,000-4,000 words.

**Best example:** [Import AI #404](https://jack-clark.net/2025/03/17/import-ai-404-scaling-laws-for-distributed-training-misalignment-predictions-made-real-and-alibabas-good-translation-model)

**Key takeaway for T-0:** The "Why this matters" escalation pattern. Every T-0 post about an AI development should end with explicit connection to business or civilizational stakes. Do the interpretive labor that busy readers won't do themselves.

---

## Cross-Source Synthesis

### The 8 Techniques T-0 Should Combine

| # | Technique | Source | Application |
|---|-----------|--------|-------------|
| 1 | Show exact process and prompts | Simon Willison | Client case studies, technical blog posts |
| 2 | Lead with proprietary data | Beyond the Hype | Metrics from real consulting engagements |
| 3 | Name your frameworks | Forte Labs, Lenny | Every consulting methodology gets a name |
| 4 | German market contrast angle | t3n | Bridge English AI discourse to German specifics |
| 5 | "I tried X and learned Y" format | Julia Evans | Honest implementation stories |
| 6 | Numbered current-event breakdown | Pragmatic Engineer | Rapid-response analysis for Mittelstand |
| 7 | Practical taxonomy as reference vocabulary | Anthropic | Definitive categorization of AI use cases |
| 8 | "Why this matters" escalation | Import AI | Connect every technical point to business stakes |

### The 5 Content Formats That Emerge

| Format | Inspired By | When to Use |
|--------|-------------|-------------|
| **The Implementation Story** | Willison + Julia Evans | When T-0 builds something for a client |
| **The Framework Post** | Forte + Lenny + Stratechery | When introducing a T-0 methodology |
| **The Rapid Breakdown** | Pragmatic Engineer + Import AI | When a major AI development affects Mittelstand |
| **The Deep Analogy** | Paul Graham + Wait But Why | When making abstract AI concepts tangible |
| **The Data Report** | Intercom + Beyond the Hype | Quarterly/annual "State of AI in Mittelstand" |

### What the Best Writers Have in Common

1. **They do the thinking, not just the reporting.** Original framework, analogy, or interpretation in every piece. The value is the thinking, not the summary.

2. **They earn opinions through visible work.** Willison shows 3,890 photos. Gruber shows 20 years of daily posts. Clark shows 440+ issues of arXiv synthesis. Show the receipts before offering the opinion.

3. **They use consistent structural patterns.** Urban always escalates. Gruber always quotes-then-comments. Clark always includes "Why this matters." Predictability creates trust.

4. **They are personal at precisely calibrated moments.** Strategic disclosure makes analytical points land harder. "When we deployed this, I genuinely didn't expect it to work" beats "Results exceeded expectations by 40%."

5. **They trust their readers' intelligence.** Don't simplify the insight — simplify the path to the insight. The audience isn't looking for "AI for dummies."

6. **They build content that accumulates.** Reference your own previous posts. Track predictions. Show the arc. A blog that can say "In post #47, we predicted X; here's what happened" has a credibility advantage no single brilliant post can match.

### What T-0 Should NOT Do

- Don't adopt t3n's institutional voice — T-0 needs personal voices (Aaron, Julien), not a publication tone
- Don't try Paul Graham's format (no headings, pure essay) — the Mittelstand audience needs scannability
- Don't imitate Forte's philosophical grandiosity — T-0's voice is pragmatic, not aspirational
- Don't copy Intercom's product-marketing blend — T-0 is a consultancy, not a product company

### The T-0 Voice Formula

**Anthropic's taxonomic clarity + Lenny's named frameworks + Intercom's report-driven authority + Julia Evans' honest process transparency + Graham's analogies from non-tech domains + Import AI's "Why this matters" escalation.**

Applied to the German Mittelstand market with proprietary consulting data. That combination would produce a voice that is:
- Authoritative without being academic
- Practical without being superficial
- Personal without being confessional
- Distinctly European without being provincial
- Opinionated without being dismissive

And most importantly: recognizable without a byline.
