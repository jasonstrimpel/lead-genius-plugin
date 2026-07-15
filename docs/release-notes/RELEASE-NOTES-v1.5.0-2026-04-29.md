# Release Notes

## v1.5.0 (2026-04-29)

### Summary

ZoomInfo decision-maker email enrichment. A new single-instance `dm-enricher` agent runs serially as Phase 8, between the parallel `dm-researcher` and `dm-compiler` phases. It calls the ZoomInfo MCP `enrich_contacts` tool once per unique contact as a first-attempt email check. High-confidence web-verified emails are preserved; pattern-matched and unverified emails upgrade to `Verified (ZoomInfo)` when ZoomInfo returns a match.

### New Feature

- **dm-enricher agent (Phase 8)**: Reads all `dm-*.md` files produced by parallel `dm-researcher` instances, calls ZoomInfo `enrich_contacts` once per unique contact, and writes a consolidated `dm-enriched.md` consumed by `dm-compiler`.
- **Precedence rules**: Verified (Web) emails are never replaced. Pattern-matched and Unverified emails upgrade to Verified (ZoomInfo) on a single ZoomInfo match. Multi-match candidates are disambiguated by entity-resolved title comparison; unresolved cases retain the original email.
- **Resilience**: Pipeline continues on ZoomInfo MCP unavailability or partial outage. A warning banner in `dm-enriched.md` flags reduced data quality. Phase 8 can be re-run independently after MCP recovery.
- **Audit trail**: New `Match` column (`zoominfo` | `no-match` | `ambiguous` | `mcp-error` | `skipped`) and `Original Email` column document every override.

### Why This Matters

1. **Higher verified-email share** sent to outreach without sacrificing strong web-sourced verifications.
2. **Bounded ZoomInfo cost**: at most one `enrich_contacts` call per unique contact, no retries on no-match.
3. **No new failure modes** for downstream `dm-compiler` and `outreach-composer` — soft fall-through preserves existing behavior when ZoomInfo is absent.
4. **Phase isolation**: enrichment is a discrete, re-runnable phase rather than embedded in research or compilation.

### Updated Files

| File | Change |
|------|--------|
| `agents/dm-enricher.md` | New agent: ZoomInfo email enrichment with precedence and audit |
| `agents/dm-compiler.md` | Inputs change from `dm-*.md` glob to single `dm-enriched.md` read; Sources column appends ZoomInfo when overridden |
| `commands/lead-genius.md` | New Phase 8 dispatch; phases 8–13 renumbered to 9–14; Phase 14 summary lists `dm-enriched.md` |
| `.claude-plugin/plugin.json` | Version 1.4.0 → 1.5.0 |
| `CLAUDE.md` | Plugin version, 13-phase narrative, agent role table, key design constraints |
| `README.md` | Pipeline phase table updated to 13 phases including Phase 8 enrichment |

### Phase Renumbering

| New | Old | Phase |
|-----|-----|-------|
| 7 | 7 | DM Research (5x parallel) |
| **8** | — | **DM Enrichment (new, 1x serial)** |
| 9 | 8 | DM Compilation |
| 10 | 9 | Outreach Generation |
| 11 | 10 | Marketing Content |
| 12 | 11 | Deck Script Generation |
| 13 | 12 | Deck Generation |
| 14 | 13 | Completion |

### Upgrade Notes

- **Required**: ZoomInfo MCP must be installed and authenticated for full benefit. If absent, Phase 8 emits a warning banner and the pipeline continues using web-research emails only — no breakage.
- **Compatibility**: Existing `dm-*.md` outputs from `dm-researcher` are not modified; they remain on disk for audit.
- **No breaking changes** to `outreach-composer`, `content-writer`, `deck-scripter`, or `deck-builder`.
