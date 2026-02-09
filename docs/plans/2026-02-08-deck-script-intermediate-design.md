# Deck Script Intermediate Artifact — Design Document

**Date:** 2026-02-08
**Branch:** `feature/deck-script-intermediate`
**Version:** 1.3.0 → 1.4.0

## Overview

Add an intermediate markdown "deck script" step to the deck-builder agent. Currently, the deck-builder reads the research brief and qualified companies, then directly generates PPTX files via the `/pptx` skill. This creates a black-box jump from research data to rendered slides — making it hard to review, iterate on, or reuse the narrative decisions independently of the visual output.

The deck script separates content and narrative decisions (Pass 1) from PPTX rendering (Pass 2). The markdown artifact captures both **what goes on each slide** (content) and **how it should look** (visual direction) — enough for the `/pptx` skill to render without re-interpreting the research.

## Problem

The current deck-builder workflow has three issues:

1. **No reviewable intermediate.** The agent goes from research files straight to `.pptx`. If the narrative framing is wrong, the user can only see this after rendering — and fixing it means regenerating the entire deck.
2. **Content and rendering coupled.** Narrative decisions (which stat to lead with, how to frame the pain) are made inside the same pass that handles layout, colors, and fonts. Changes to either require re-doing both.
3. **No reuse.** The narrative structure is locked inside binary PPTX files. It can't be easily repurposed for other formats (Google Slides, Keynote, HTML) or reviewed in a diff.

## Deck Script Format

Each deck script is a markdown file with one section per slide. Every section has two parts:

### Content Block
The actual words, data points, and narrative for the slide. This is the "what" — the substance that goes on the slide.

### Visual Direction Block
Layout guidance, emphasis cues, and visual element descriptions. This is the "how" — enough direction for the `/pptx` skill to render the slide without making content decisions.

### Example: One Slide Section

```markdown
## Slide 3: The Quantified Pain

### Content

**Headline:** The Hidden Cost of Manual Compliance

Three pain points with quantified impact:

1. **$2.4M per year** in manual review labor for mid-market insurers (industry benchmark)
2. **23 days average** from filing to resolution — 4x the customer-expected timeline
3. **34% of compliance decisions** require rework due to inconsistent human judgment

The single most impactful number: **$2.4M/year** — bold and dominant.

Source framing: "Manual KYC review costs mid-market insurers an average of $2.4M per year in direct labor alone."

### Visual Direction

- Layout: Three-column with icon above each pain point
- Dominant element: The $2.4M figure at 2x size, top-center
- Color: Use the accent color for all dollar/percentage figures
- Icons: Clock (time), dollar sign (cost), warning triangle (risk)
- Presentation version: Numbers only, no source framing text
- Reading version: Include source framing below each data point
```

### What the Visual Direction Captures

The visual direction is **not** a pixel-perfect layout spec. It captures:

- **Layout pattern** (columns, split, full-bleed, etc.)
- **Emphasis hierarchy** (what's biggest, what's secondary)
- **Visual element types** (icons, charts, shapes, accent bars)
- **Presentation vs. reading differences** (what to include/exclude per version)
- **Color intent** (where to use accent, dominant, supporting colors)

What it does **not** capture (left to the `/pptx` skill):
- Exact coordinates and dimensions
- Font sizes and specific typefaces
- Exact color hex values
- PptxGenJS API details

## Deck Script File Structure

### General Deck Script

```markdown
---
type: deck-script
deck: general-offering
slug: {slug}
date: YYYY-MM-DD
agent: deck-builder
source_files:
  - ./{slug}/go-to-market/research-brief.md
  - ./{slug}/companies/qualified-companies.md
---

# General Offering Deck — Deck Script

## Design Direction

- Color palette: {palette name and rationale}
- Visual motif: {motif description}
- Font pairing: {title font / body font}
- Industry tone: {e.g., "enterprise financial services — authoritative, restrained"}

## Slide 1: The Industry Shift

### Content
{headline, data point, hook}

### Visual Direction
{layout, emphasis, visual elements}

## Slide 2: Winners and Losers
...

## Slide 8: Next Steps
...
```

### Prospect-Specific Deck Script

```markdown
---
type: deck-script
deck: prospect-specific
company: {company name}
company_slug: {company-slug}
slug: {slug}
date: YYYY-MM-DD
agent: deck-builder
source_files:
  - ./{slug}/go-to-market/research-brief.md
  - ./{slug}/companies/qualified-companies.md
---

# {Company Name} — Prospect Deck Script

## Slide 1: Industry Impact
### Content
{how the offering reshapes this company's industry}

### Visual Direction
{layout guidance}

## Slide 2: Company ROI
### Content
{company-specific projected outcomes}

### Visual Direction
{layout guidance}
```

## Updated Deck-Builder Workflow

The current single-pass workflow becomes two passes:

### Pass 1: Write Deck Scripts (Content + Visual Direction)

1. Read research-brief.md and qualified-companies.md
2. Apply the 8-slide framework to extract content for each slide
3. Make all narrative decisions: which stats to lead with, how to frame pain, which examples to use
4. Determine visual direction: layout patterns, emphasis hierarchy, motif
5. Write `deck-script.md` for the general deck
6. Write `{company-slug}-deck-script.md` for each prospect-specific deck

### Pass 2: Render PPTX from Deck Scripts

7. Read `deck-script.md`
8. Invoke `/pptx` skill to generate presentation version (`deck-presentation.pptx`)
9. Invoke `/pptx` skill to generate reading version (`deck-reading.pptx`)
10. For each prospect deck script, invoke `/pptx` skill to generate both versions
11. Run QA cycle on rendered decks

### Key Principle

Pass 2 should **never** need to re-read the research files. The deck script contains everything needed to render — content, emphasis, layout direction, and version-specific instructions.

## Output Directory

Deck scripts live alongside their PPTX counterparts in the marketing directory:

```
{slug}/marketing/
├── blog.md
├── linkedin-posts.md
├── case-study.md
├── deck-script.md                              ← NEW (general deck script)
├── deck-presentation.pptx
├── deck-reading.pptx
└── prospect-specific/
    ├── {company-slug}-deck-script.md            ← NEW (prospect deck script)
    ├── {company-slug}-presentation.pptx
    ├── {company-slug}-reading.pptx
    └── ...
```

Deck scripts are placed next to the PPTX files they produce. This makes the relationship obvious and keeps the marketing directory self-contained.

## Files Changed

| File | Change |
|------|--------|
| `agents/deck-builder.md` | Rewrite workflow section for two-pass approach. Add deck script format spec. Pass 1 writes markdown scripts, Pass 2 renders PPTX from scripts. |
| `commands/lead-genius.md` | Update Phase 10 completion summary to include deck script files |
| `CLAUDE.md` | Update deck-builder description and output directory listing |
| `README.md` | Add deck scripts to output structure and marketing content section |
| `.claude-plugin/plugin.json` | Bump version 1.3.0 → 1.4.0 |
| `docs/RELEASE-NOTES-v1.4.0-2026-02-08.md` | New release notes |

## Files Unchanged

All other agents, skills, and the orchestrator's Phase 7 branch dispatch logic remain untouched. The deck-builder is still dispatched the same way — it just does more internally.

## Benefits

1. **Reviewable narrative.** Users can read the deck script markdown to verify framing, stats, and messaging before any rendering happens.
2. **Faster iteration.** If a narrative choice is wrong, edit the markdown and re-render — no need to re-extract from research files.
3. **Format-portable.** The deck script can feed into Google Slides, Keynote, or HTML renderers in the future without re-running the content extraction.
4. **Debuggable.** When a PPTX looks wrong, compare the deck script to the rendered output to isolate whether the issue is content (Pass 1) or rendering (Pass 2).
