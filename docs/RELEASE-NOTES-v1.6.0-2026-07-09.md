# Release Notes

## v1.6.0 (2026-07-09)

### Summary

Removal of PPTX deck generation. The pipeline no longer renders PowerPoint files. The `deck-builder` agent and the bundled `pptx` skill (vendored from `anthropics/skills`) are removed entirely. The `deck-scripter` agent is retained: it continues to produce reviewable markdown "deck scripts" (an 8-slide first-call narrative for the general deck plus a 2-slide script per prospect) as standalone content deliverables. This release also fixes several scaffolding defects surfaced in an audit.

### Removed

- **`deck-builder` agent**: The PPTX renderer is deleted. Phase 13 (Deck Generation) and its `npm install -g pptxgenjs` pre-dispatch are removed from the orchestrator.
- **`skills/pptx/` skill**: The entire vendored PPTX generation skill is removed from the plugin. The pipeline no longer depends on PptxGenJS, LibreOffice, or `pdftoppm`.
- All `/pptx` references across the runtime (`commands/lead-genius.md`, `agents/deck-scripter.md`) and primary docs (`CLAUDE.md`, `README.md`).

### Retained

- **`deck-scripter` agent (Phase 12)**: Unchanged in purpose — writes `deck-script.md` (general, 8 slides) and `prospect-specific/{company-slug}-deck-script.md` (2 slides per company). Its "Visual Direction" blocks are now framed as design intent for building the deck by hand or with a slide tool, not as input to a renderer.

### Fixes

- **`dm-enricher` tool grant**: Restored the ZoomInfo MCP tools (`mcp__zoominfo__enrich_contacts`, `mcp__zoominfo__lookup`) to the agent's `tools` allowlist. The agent previously could not call the tool it was built around, so Phase 8 always degraded to "skipped." (The MCP server prefix is environment-dependent — adjust if your ZoomInfo server is registered under a different name.)
- **Version consistency**: `marketplace.json` was stuck at `1.2.0` while `plugin.json` read `1.5.0`. Both now read `1.6.0`.
- **Placeholder author email**: `jason@example.com` replaced with `jason.strimpel@andersenconsulting.com` in both manifests.
- **`deck-scripter` metadata bug**: Generated deck scripts stamped `agent: deck-builder`; now correctly `agent: deck-scripter`.
- **Doc drift**: Corrected the phase count (now 14), agent count (now 11), the agent role table (removed `deck-builder`, added `deck-scripter`), the output-directory tree, and the deck-output descriptions across `CLAUDE.md` and `README.md`.
- **Minor**: Corrected deck-script output file counts and a `Compliace` → `Compliance` typo in the `deck-scripter` worked example.

### Updated Files

| File | Change |
|------|--------|
| `agents/deck-builder.md` | Removed |
| `skills/pptx/**` | Removed (119 files) |
| `commands/lead-genius.md` | Removed Phase 13 (Deck Generation); Completion renumbered 14 → 13; summary no longer lists `.pptx` outputs |
| `agents/deck-scripter.md` | Scrubbed `/pptx` references; fixed `agent:` metadata; corrected output counts and typo |
| `agents/dm-enricher.md` | Restored ZoomInfo MCP tools to the `tools` allowlist |
| `.claude-plugin/plugin.json` | Version 1.5.0 → 1.6.0; author email fixed |
| `.claude-plugin/marketplace.json` | Version 1.2.0 → 1.6.0; owner/author email fixed |
| `CLAUDE.md` | Version, 14-phase narrative, 11-agent structure, agent role table, removed `skills/pptx/` and `/pptx` references |
| `README.md` | Description, 14-phase table, output tree, key files, "Sales Deck Scripts" section, changelog link |

### Phase Renumbering

| New | Old | Phase |
|-----|-----|-------|
| 11 | 11 | Marketing Content |
| 12 | 12 | Deck Script Generation |
| **13** | 14 | **Completion** |
| — | 13 | ~~Deck Generation~~ (removed) |

### Upgrade Notes

- **Breaking**: No `.pptx` files are produced. Consumers expecting `deck-presentation.pptx` / `deck-reading.pptx` (or prospect `*.pptx`) will now find only `deck-script.md` markdown deliverables in `marketing/` and `marketing/prospect-specific/`.
- **No external binaries required**: PptxGenJS, LibreOffice, and `pdftoppm` are no longer part of the pipeline.
- **No changes** to research, scoring, enrichment, outreach, or the blog/LinkedIn/case-study marketing content.
