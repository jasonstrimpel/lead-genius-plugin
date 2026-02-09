# Release Notes

## v1.4.0 (2026-02-08)

### Summary

Deck script intermediate artifacts. The deck-builder agent now uses a two-pass workflow: Pass 1 writes a markdown "deck script" capturing content and visual direction per slide, Pass 2 renders PPTX from the script. Deck scripts are reviewable, editable, and format-portable intermediates that separate narrative decisions from rendering.

### New Feature

- **Deck scripts**: Before generating PPTX files, the deck-builder writes a markdown deck script for each deck:
  - **General deck script** (`deck-script.md`): Metadata header, design direction (palette, motif, fonts, tone), and one section per slide with `### Content` (words, data, narrative) and `### Visual Direction` (layout, emphasis, visuals, version-specific notes)
  - **Prospect deck scripts** (`{company-slug}-deck-script.md`): Same format, 2 slides per company (industry impact + company ROI)
  - Pass 2 renders PPTX exclusively from the deck script — never re-reads research files
  - Deck scripts live alongside their PPTX counterparts in the `marketing/` directory

### Why This Matters

1. **Reviewable narrative** — Read the markdown to verify framing, stats, and messaging before any rendering happens
2. **Faster iteration** — Edit the deck script and re-render without re-extracting from research files
3. **Format-portable** — Deck scripts can feed Google Slides, Keynote, or HTML renderers in the future
4. **Debuggable** — Compare deck script to rendered output to isolate content issues from rendering issues

### Updated Files

| File | Change |
|------|--------|
| `agents/deck-builder.md` | Workflow rewritten for two-pass approach (Pass 1: write deck scripts, Pass 2: render PPTX from scripts). Output section updated to include deck script files. |
| `commands/lead-genius.md` | Phase 10 completion summary includes deck script files and counts |
| `.claude-plugin/plugin.json` | Version 1.3.0 → 1.4.0 |
| `CLAUDE.md` | Version, deck-builder description, output directory note |
| `README.md` | Deck scripts in output structure, pipeline table, marketing content section, key files list, changelog link |

### Output Directory (Updated)

```
{slug}/marketing/
├── blog.md
├── linkedin-posts.md
├── case-study.md
├── deck-script.md                              ← NEW
├── deck-presentation.pptx
├── deck-reading.pptx
└── prospect-specific/
    ├── {company-slug}-deck-script.md            ← NEW
    ├── {company-slug}-presentation.pptx
    ├── {company-slug}-reading.pptx
    └── ... (~30 files for 10 companies)
```

### Upgrade Notes

No breaking changes. The deck-builder agent now produces additional markdown files alongside the existing PPTX outputs. All other agents, phases, and pipeline behavior are unchanged.
