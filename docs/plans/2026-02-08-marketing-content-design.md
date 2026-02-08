# Marketing Content Generation — Design Document

**Date:** 2026-02-08
**Branch:** `feature/marketing-content-generation`
**Version:** 1.2.0 → 1.3.0

## Overview

Add a marketing content generation branch to the lead-genius pipeline. After company synthesis (Phase 6), the pipeline forks into two parallel branches: Branch A generates marketing content (blog, LinkedIn posts, case study, PPTX decks), while Branch B continues the existing DM research and outreach flow. Both branches run simultaneously and converge at the completion phase.

## Pipeline Architecture

The current 10-phase linear pipeline becomes a branching pipeline after Phase 6:

```
Phase 0: Setup
Phase 1: Collateral Analysis
Phase 2: GTM Interview
Phase 3: Synthesis
Phase 4: Scoring Rubrics
Phase 5: Company Research (5x parallel)
Phase 6: Company Synthesis
         │
         ├─── Branch A: Marketing ──────────────────────────┐
         │    Phase 7A-i:  content-writer (blog, LinkedIn,  │
         │                 case study — all markdown)        │
         │    Phase 7A-ii: deck-builder (general 8-slide     │
         │                 deck × 2 versions + ~10 prospect  │
         │                 2-slide decks × 2 versions)       │
         │    (7A-i and 7A-ii run in parallel)               │
         │                                                   │
         ├─── Branch B: DM + Outreach ─────────────────────┐│
         │    Phase 7B: DM Research (5x parallel)          ││
         │    Phase 8B: DM Compilation                     ││
         │    Phase 9B: Outreach Generation                ││
         │                                                  ││
         └──── Phase 10: Completion (both branches done) ◄──┘┘
```

After Phase 6, the orchestrator launches 7 parallel agents in a single dispatch:
- 5x `dm-researcher` (existing)
- 1x `content-writer` (new)
- 1x `deck-builder` (new)

None depend on each other. The orchestrator waits for all 7 to return, then continues Branch B's sequential phases (DM Compilation → Outreach) while Branch A is already done.

## New Agent: content-writer

**File:** `agents/content-writer.md`

**Frontmatter:**
```yaml
---
name: content-writer
description: |
  Use this agent to generate marketing content from GTM research.
model: inherit
tools: [Read, Write]
---
```

**Inputs:**
- `{slug}/go-to-market/research-brief.md` — offering, ICP, value prop, demand signals, buyer personas
- `{slug}/companies/qualified-companies.md` — top 10 ranked companies with industries, evidence, GTM fit

**Outputs:**
- `{slug}/marketing/blog.md` — Long-form thought leadership post following the narrative arc: industry shift → stakes → pain → vision → bridge to solution. Written for the ICP audience, not as a sales pitch. Establishes authority on the problem space.
- `{slug}/marketing/linkedin-posts.md` — 5-7 posts in a single file, each separated by `---`. Each post extracts a different angle from the blog (industry shift, quantified pain, vision, etc.). Short paragraphs, conversational, hook in the first line, one idea per post.
- `{slug}/marketing/case-study.md` — Uses the #1 ranked company from qualified-companies.md. Written as a real case study with Challenge → Action → Result structure. Draws on the company's actual industry, pain points, and the offering's value prop to construct a compelling narrative with quantified outcomes.

**No web search needed** — all source material exists in pipeline output files.

## New Agent: deck-builder

**File:** `agents/deck-builder.md`

**Frontmatter:**
```yaml
---
name: deck-builder
description: |
  Use this agent to generate PPTX sales decks from GTM research.
model: inherit
tools: [Read, Write, Bash, Skill]
---
```

**Inputs:**
- `{slug}/go-to-market/research-brief.md` — market shift, ICP, value prop, differentiators, pain points, competitive landscape
- `{slug}/companies/qualified-companies.md` — top 10 companies with industry, evidence, GTM fit, confidence scores

**Tools:**
- `Skill` to invoke the `/pptx` skill for generation
- `Bash` to run PptxGenJS scripts and QA commands

### General Offering Deck — The 8-Slide First-Call Framework

The agent's prompt embeds the complete 8-slide framework, which synthesizes Raskin's narrative arc, the Challenger Sale's commercial teaching choreography, Corporate Visions' "Why Change" model, McKinsey's SCR structure, MEDDIC's qualification principles, Force Management's value messaging, and Gong's data-backed presentation science.

**Slide 1 — The Industry Shift (Hook)**
Establish credibility and capture attention by naming an undeniable change. Open with a bold, specific statement about a macro trend reshaping the prospect's industry — a named, coined shift backed by one striking data point. End with an implicit question that opens dialogue.
*AI agent input:* Extract the primary market shift from the research brief. Identify the single most compelling statistic. Frame as a declarative statement.

**Slide 2 — Winners and Losers (Stakes)**
Escalate urgency through loss aversion. Show the shift is creating divergence — companies that adapt pull ahead; those that don't fall behind. Use 2-3 recognizable company examples. The key emotion: "this is happening with or without us."
*AI agent input:* Extract competitive landscape data. Identify companies succeeding and those falling behind.

**Slide 3 — The Quantified Pain (Problem)**
Make the cost of the current state undeniable. Show 3 concrete pain points with quantified impact — time wasted, revenue leaked, risk exposure. Use industry benchmarks and third-party data. Bold the single most impactful number.
*AI agent input:* Extract ICP pain points. Cross-reference with industry benchmarks. Frame as: "[Pain] costs [segment] an average of [$] per [period]."

**Slide 4 — Cost of Inaction (Urgency)**
The most strategically important slide. Show pain is not static — it compounds. Frame three escalation vectors: financial cost accelerates, competitive gap widens, organizational risk grows. Do NOT mention your product. This slide is entirely about the cost of doing nothing.
*AI agent input:* Extract consequences of non-adoption. Calculate compounding costs (annual cost × years of delay). Add one "ticking clock" element.

**Slide 5 — The Promised Land (Vision)**
Paint the desirable future state. Describe what the prospect's world looks like after the transformation — in their language, not yours. The Promised Land is NOT having your technology; it's what life is like thanks to your technology. Show 3-4 concrete outcomes.
*AI agent input:* Extract top 3-4 customer outcomes. Reframe from product capability to customer state. Use present tense.

**Slide 6 — Solution and Differentiation (Bridge)**
Only NOW introduce the solution — as the bridge between pain and the Promised Land. Present 3 key capabilities framed as "magic gifts" — stepping stones to the vision, not features. Follow each with specific differentiation. No checkbox comparison charts against competitors.
*AI agent input:* Extract top 3 differentiators and capabilities. Map each to a Promised Land outcome. Reframe as "why this approach works when others don't."

**Slide 7 — Proof (Evidence)**
Eliminate skepticism with one powerful customer success story. Challenge → Action → Result format in brief narrative — not a logo grid. Named company, quantified before-and-after, ideally a brief executive quote.
*AI agent input:* Select the story most relevant to the target ICP. Extract the quantified outcome.

**Slide 8 — Next Steps (CTA)**
Convert engagement into forward momentum. Present a 3-step evaluation roadmap: Discovery → Assessment → Decision. Propose specific next steps. Include who should attend and suggested timeframe. Must function as a clear "what happens next" for async readers.
*AI agent input:* Extract standard next steps from the research brief's sales process section.

### Slide Ratio: 60/20/20

- **Problem/urgency (Slides 1-4): 50-60%** — Majority of narrative establishes the problem landscape before any solution appears
- **Solution/differentiation (Slides 5-6): 20-25%** — Deliberately compressed; goal is to establish a credible path, not explain every feature
- **Proof and CTA (Slides 7-8): 20-25%** — One story plus clear next steps

### Eight Things to Never Put in the Deck

1. Never open with company history or "About Us"
2. Never include an ROI calculator (27% close rate drop per Gong)
3. Never use a logo grid as social proof (22% lower close rates per Gong)
4. Never build to a climax — lead with the insight, then support it
5. Never include feature comparison charts against competitors
6. Never dump discovery questions into slides
7. Never exceed 20 words per slide (presentation version) / 60 words (reading version)
8. Never skip the next-steps slide

### Two Versions Per Deck

Every deck is generated twice:
- **Presentation version** (≤20 words/slide): Visual-heavy, for live delivery. Slides are visual anchors for a verbal narrative.
- **Reading version** (≤60 words/slide): Expanded narrative, for asynchronous sharing. Personalized leave-behind decks are shared internally 2.3x more often.

### General Deck Outputs

- `{slug}/marketing/deck-presentation.pptx`
- `{slug}/marketing/deck-reading.pptx`

### Prospect-Specific Decks

For each of the ~10 companies in `qualified-companies.md`, generate a 2-slide deck in both versions:

- **Slide 1:** How the offering improves this company's industry — tailored version of the macro shift and pain, using industry-specific data from qualified companies research
- **Slide 2:** How the offering improves ROI for this specific company — uses the company's evidence, GTM fit rationale, and projected outcomes

These are designed to be inserted into or appended to the general deck when presenting to a specific prospect.

**Outputs:**
- `{slug}/marketing/prospect-specific/{company-slug}-presentation.pptx`
- `{slug}/marketing/prospect-specific/{company-slug}-reading.pptx`

### Design Standards (from Anthropic pptx skill)

- One dominant color (60-70% visual weight), 1-2 supporting tones, one sharp accent
- One distinctive visual motif repeated across every slide
- Every slide needs visuals — no text-only layouts
- Interesting font pairings, not defaults; clear size contrast (36-44pt titles, 14-16pt body)
- 0.5" minimum margins, 0.3-0.5" between content blocks

### QA Cycle

1. Convert slides to images using LibreOffice + pdftoppm
2. Visually inspect for missing content, leftover placeholders, overflow
3. At least one fix-and-verify cycle before declaring success

## PPTX Generation Method

The deck-builder agent uses the **Anthropic `pptx` skill** from the `anthropics/skills` repository.

**Dependencies (installed before branch point):**
- `npm install -g pptxgenjs` — JavaScript library for creating presentations
- `pip install markitdown[pptx] Pillow` — for QA thumbnail generation
- LibreOffice + Poppler — for slide-to-image conversion during QA

**Creation workflow:**
1. Agent writes a Node.js script using PptxGenJS to build each deck
2. Script defines layout (16x9), color palette, typography, visual motifs per skill's design standards
3. Script generates the `.pptx` file to the output path
4. Agent runs the script via `Bash`

**The skill ships with the plugin** at `skills/pptx/`, referenced by the deck-builder agent the same way outreach-composer references `/executive-outreach`.

## Output Directory Structure

Complete filesystem tree after all phases:

```
{slug}/
├── collateral/
│   └── collateral-analysis.md
├── interview/
│   ├── offering.md
│   └── gtm-strategy.md
├── go-to-market/
│   ├── research-brief.md
│   └── scoring-rubrics.md
├── company-research/
│   ├── companies-01.md
│   ├── companies-02.md
│   ├── companies-03.md
│   ├── companies-04.md
│   └── companies-05.md
├── companies/
│   └── qualified-companies.md
├── decision-maker-research/              ← Branch B
│   ├── dm-01.md
│   ├── dm-02.md
│   ├── dm-03.md
│   ├── dm-04.md
│   └── dm-05.md
├── decision-makers/                      ← Branch B
│   └── decision-makers.md
├── outreach.md                           ← Branch B
└── marketing/                            ← Branch A (new)
    ├── blog.md
    ├── linkedin-posts.md
    ├── case-study.md
    ├── deck-presentation.pptx
    ├── deck-reading.pptx
    └── prospect-specific/
        ├── {company-1-slug}-presentation.pptx
        ├── {company-1-slug}-reading.pptx
        ├── {company-2-slug}-presentation.pptx
        ├── {company-2-slug}-reading.pptx
        ├── ...
        ├── {company-10-slug}-presentation.pptx
        └── {company-10-slug}-reading.pptx
```

~25 new files in the marketing directory per run.

## Orchestrator Changes

**File:** `commands/lead-genius.md`

1. **Phase 6 → Branch point.** After Company Synthesis, launch 7 parallel agents in a single dispatch: 5x `dm-researcher` + 1x `content-writer` + 1x `deck-builder`.
2. **Dependency setup.** Before the branch point, install PptxGenJS and QA dependencies.
3. **Convergence.** Wait for all 7 agents, then continue Branch B's sequential phases (DM Compilation → Outreach) while Branch A is already done.
4. **Phase 10 update.** Completion summary reports both branches: DM/outreach counts (existing) + marketing content listing (new).

## Skill Integration

The Anthropic `pptx` skill is copied into the plugin:

```
skills/
├── executive-outreach/
│   ├── SKILL.md
│   └── references/examples.md
└── pptx/                                 ← new (from anthropics/skills)
    ├── SKILL.md
    ├── pptxgenjs.md
    ├── editing.md
    └── scripts/
        ├── __init__.py
        ├── add_slide.py
        ├── clean.py
        ├── thumbnail.py
        └── office/
            ├── unpack.py
            ├── pack.py
            └── soffice.py
```

## Files Changed

| File | Change |
|------|--------|
| `commands/lead-genius.md` | Rewrite Phase 6→10 for branching, add dependency install, update completion summary |
| `.claude-plugin/plugin.json` | Bump version 1.2.0 → 1.3.0, register pptx skill |
| `CLAUDE.md` | Update architecture docs: new agents, branching pipeline, marketing output directory |

## Files Added

| File | Purpose |
|------|---------|
| `agents/content-writer.md` | Marketing content agent (blog, LinkedIn, case study) |
| `agents/deck-builder.md` | PPTX deck agent (general + prospect-specific) |
| `skills/pptx/*` | Anthropic pptx skill (SKILL.md, pptxgenjs.md, editing.md, scripts/) |

## Files Unchanged

All existing agents (`collateral-analyzer`, `gtm-synthesizer`, `scoring-strategist`, `company-researcher`, `company-synthesizer`, `dm-researcher`, `dm-compiler`, `outreach-composer`) and the `executive-outreach` skill remain untouched.

## Agent Summary Table (Updated)

| Agent | Purpose | Parallelism | Branch |
|-------|---------|-------------|--------|
| `collateral-analyzer` | Extract GTM content from sales PDFs | 1x | Linear |
| `gtm-synthesizer` | Combine interview outputs into research brief | 1x | Linear |
| `scoring-strategist` | Generate deterministic scoring rubrics | 1x | Linear |
| `company-researcher` | Web search for qualifying companies | 5x parallel | Linear |
| `company-synthesizer` | Dedupe and rank top 10 companies | 1x | Linear |
| `content-writer` | Generate blog, LinkedIn posts, case study | 1x | A (Marketing) |
| `deck-builder` | Generate PPTX decks (general + prospect-specific) | 1x | A (Marketing) |
| `dm-researcher` | Find decision makers at target companies | 5x parallel | B (DM/Outreach) |
| `dm-compiler` | Compile and priority-rank all contacts | 1x | B (DM/Outreach) |
| `outreach-composer` | Generate tier-matched emails | 1x | B (DM/Outreach) |
