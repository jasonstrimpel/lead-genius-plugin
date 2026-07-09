# Release Notes

## v1.7.0 (2026-07-09)

### Summary

Two changes to the research pipeline. First, company and decision-maker research now run as **single sequential agents** instead of 5 parallel instances, eliminating the duplicate discovery that forced downstream agents to spend tokens de-duplicating. Second, `dm-enricher` is reworked so **ZoomInfo is the primary email source**, with a fallback to internet-search and pattern-matched emails when ZoomInfo cannot find a contact or its API credits are exhausted.

### Sequential Research (no wasted deduplication)

- **`company-researcher`**: was 5 parallel instances (`companies-01.md`..`companies-05.md`); now one sequential agent that finds 15-20 distinct, non-duplicate companies and writes a single `companies.md`.
- **`dm-researcher`**: was 5 parallel instances with assigned company subsets (`dm-01.md`..`dm-05.md`); now one sequential agent that covers all qualified companies in a single pass and writes a single `dm-research.md`.
- **`company-synthesizer`** and **`dm-compiler`** are retained (they do more than deduplicate) but their prompts had all deduplication/merge language removed. They now read a single, already-duplicate-free file and focus on scoring, ranking, and selection. The "Deduplication Log" output sections and the now-unused `Glob` tool grants were removed from both.

Why: parallel researchers repeatedly rediscovered the same companies and contacts, and a follow-up agent burned tokens reconciling the overlap. A single sequential pass never produces duplicates, so the reconciliation work — and its token cost — is gone.

### ZoomInfo Primary with Fallback

- **ZoomInfo is now primary**: a confident ZoomInfo match is used even over a web-verified email (previously web-verified was preserved over ZoomInfo).
- **Fallback layer**: when ZoomInfo returns no-match, an ambiguous result, an error, is unavailable, or its **API credits are exhausted**, the pipeline falls back to the internet-search / pattern-matched email discovered by `dm-researcher`, at that email's original confidence.
- **Credit-exhaustion short-circuit**: on a credit/quota-exhausted error, `dm-enricher` stops issuing further ZoomInfo calls, marks remaining contacts `credits-exhausted`, and falls back — no wasted calls once credits are gone.
- New `credits-exhausted` MCP status and updated warning banner; new precedence table in `agents/dm-enricher.md`.

### Updated Files

| File | Change |
|------|--------|
| `agents/company-researcher.md` | Single sequential agent; finds 15-20 distinct companies; writes `companies.md` |
| `agents/company-synthesizer.md` | Dedup/merge language removed; reads single `companies.md`; scoring/ranking/top-10 retained; dropped `Glob` |
| `agents/dm-researcher.md` | Single sequential agent across all companies; writes `dm-research.md`; emails framed as enrichment fallback layer |
| `agents/dm-enricher.md` | ZoomInfo primary; fallback on no-match/ambiguous/error/unavailable/credit-exhaustion; credit short-circuit; single-file input |
| `agents/dm-compiler.md` | Dedup language removed; reads single `dm-enriched.md`; priority scoring/ranking/tier retained; dropped `Glob` |
| `commands/lead-genius.md` | Phases 5-9 rewritten: single-agent dispatch, new file names, ZoomInfo-primary enrichment prompt |
| `.claude-plugin/plugin.json` | Version 1.6.0 → 1.7.0 |
| `.claude-plugin/marketplace.json` | Version 1.6.0 → 1.7.0 |
| `CLAUDE.md` | Sequential coordination model, agent table, WebSearch-count and ZoomInfo constraints |
| `README.md` | Phase table, "Sequential Agents" section, output tree (`companies.md`, `dm-research.md`, `dm-enriched.md`) |

### File Naming Changes

| Old | New |
|-----|-----|
| `company-research/companies-01.md`..`-05.md` | `company-research/companies.md` |
| `decision-maker-research/dm-01.md`..`-05.md` | `decision-maker-research/dm-research.md` |

### Email Enrichment Precedence

| ZoomInfo result | Fallback (web/pattern) email | Final email | Final confidence |
|---|---|---|---|
| confident match | (any, or none) | ZoomInfo's | `Verified (ZoomInfo)` |
| no-match / ambiguous | Verified (Web) / Pattern / Unverified | dm-researcher's | unchanged |
| no-match / ambiguous | (none) | (none) | `No email` |
| credits-exhausted / mcp-error / unavailable | Verified (Web) / Pattern / Unverified | dm-researcher's | unchanged |
| credits-exhausted / mcp-error / unavailable | (none) | (none) | `No email` |

### Upgrade Notes

- **Trade-off**: sequential research is slower in wall-clock than 5x parallel and may surface somewhat fewer raw candidates, in exchange for eliminated deduplication and lower token cost.
- **Behavior change**: ZoomInfo matches now override web-verified emails. If you preferred web-verified precedence, this is a reversal from v1.5.0/v1.6.0.
- **ZoomInfo MCP server name**: `dm-enricher` grants `mcp__zoominfo__enrich_contacts` and `mcp__zoominfo__lookup`. If your ZoomInfo MCP server is registered under a different name, adjust the `tools` list accordingly.
- **No changes** to interview, synthesis, scoring, outreach, marketing content, or deck-script generation.
