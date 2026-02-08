# Release Notes

## v1.3.0 (2026-02-08)

### Summary

Marketing content generation branch. After company synthesis, the pipeline now forks into two parallel branches: one generates marketing content (blog, LinkedIn posts, executive case study, PPTX sales decks), the other continues the existing DM research and outreach flow. Both run simultaneously, reducing total pipeline time.

### New Features

- **Branching pipeline**: After Phase 6 (Company Synthesis), the orchestrator launches 7 agents in parallel — 2 marketing agents + 5 DM researchers — instead of running DM research sequentially before marketing.
- **Content writer agent**: Generates three marketing markdown files from the research brief and qualified companies:
  - **Blog post** (`blog.md`): 1500-2500 word thought leadership piece following a 7-part narrative arc (industry shift → stakes → pain → cost of inaction → promised land → bridge → next steps)
  - **LinkedIn posts** (`linkedin-posts.md`): 5-7 posts derived from the blog, each extracting a different angle (industry shift, quantified pain, contrarian take, etc.)
  - **Executive case study** (`case-study.md`): Andersen Consulting format — anonymous client, mandatory heading structure, strict cadence requirements, publication-ready with verbatim closing section
- **Deck builder agent**: Generates PPTX sales decks using the Anthropic `pptx` skill (PptxGenJS):
  - **General offering deck**: 8-slide first-call framework (Industry Shift → Winners/Losers → Quantified Pain → Cost of Inaction → Promised Land → Solution/Differentiation → Proof → Next Steps) with 60/20/20 ratio
  - **Two versions per deck**: Presentation (≤20 words/slide, visual-heavy) and Reading (≤60 words/slide, leave-behind)
  - **Prospect-specific decks**: 2-slide deck per qualified company (industry impact + company ROI), both versions — ~20 files for 10 companies
- **Anthropic pptx skill**: Full `pptx` skill from `anthropics/skills` repository integrated into the plugin at `skills/pptx/`, including PptxGenJS documentation, template editing workflow, and Python helper scripts

### New Agents

| Agent | Purpose | Tools |
|-------|---------|-------|
| `content-writer` | Blog, LinkedIn posts, executive case study | Read, Write |
| `deck-builder` | General 8-slide deck + prospect-specific decks | Read, Write, Bash, Skill |

### Pipeline Architecture Change

```
Phases 0-6: unchanged (linear)

Phase 7 (was DM Research, now Parallel Branch Dispatch):
  Branch A: content-writer + deck-builder (parallel)
  Branch B: 5x dm-researcher (parallel, unchanged)
  All 7 agents launch simultaneously

Phases 8-9: unchanged (DM Compilation → Outreach, sequential)
Phase 10: updated completion summary includes marketing outputs
```

### 8-Slide First-Call Deck Framework

The deck builder embeds a research-backed framework synthesizing Raskin's narrative arc, the Challenger Sale, Corporate Visions' "Why Change" model, McKinsey's SCR structure, MEDDIC, Force Management, and Gong's presentation science. Eight rules enforced: no About Us opener, no ROI calculator, no logo grids, no climax build, no competitor comparison charts, no discovery question dumps, word count limits, mandatory next-steps slide.

### Executive Case Study Format

The content writer produces case studies in the Andersen Consulting executive format:
- Anonymous client derived from ICP firmographics
- Mandatory heading structure: Title, Executive Summary, Introduction (Across the Industry / The Client's Challenge), Solution (5 subsections), Results and Impact, A New Standard, Looking Ahead
- Input mapping from research brief sections to case study sections
- Strict cadence requirements (varied sentence/paragraph length, active voice, no AI-sounding rhythm)
- 800-1,100 words, plain text, no vendor platform names
- Verbatim closing section with firm description and contact info

### New Files

| File | Purpose |
|------|---------|
| `agents/content-writer.md` | Marketing content agent definition |
| `agents/deck-builder.md` | PPTX deck agent definition with embedded 8-slide framework |
| `skills/pptx/*` | Anthropic pptx skill (SKILL.md, pptxgenjs.md, editing.md, scripts/) |

### Updated Files

| File | Change |
|------|--------|
| `commands/lead-genius.md` | Phase 7 rewritten for branching dispatch (7 parallel agents), Phase 10 updated with marketing outputs |
| `.claude-plugin/plugin.json` | Version 1.2.0 → 1.3.0 |
| `CLAUDE.md` | Version, plugin structure (10 agents, pptx skill), architecture (branching pipeline), agent roles table, making changes section |
| `README.md` | Marketing content section, updated pipeline table, output structure, changelog link |

### Output Directory (New)

```
{slug}/marketing/
├── blog.md
├── linkedin-posts.md
├── case-study.md
├── deck-presentation.pptx
├── deck-reading.pptx
└── prospect-specific/
    ├── {company-slug}-presentation.pptx
    ├── {company-slug}-reading.pptx
    └── ... (~20 files for 10 companies)
```

### Upgrade Notes

No breaking changes. Existing pipeline phases 0-6 and 8-9 are unchanged. Phase 7 now launches 7 agents instead of 5 (the 5 DM researchers still run identically). Phase 10 completion summary now includes marketing output counts. The `pptxgenjs` npm package will be installed automatically during the pipeline run if not already present.
