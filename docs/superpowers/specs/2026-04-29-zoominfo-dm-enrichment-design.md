# ZoomInfo Decision Maker Email Enrichment

**Date:** 2026-04-29
**Target version:** 1.5.0
**Status:** Approved (awaiting implementation plan)

## Summary

Add a serial enrichment phase to the lead-genius pipeline that uses the ZoomInfo MCP to verify and upgrade decision-maker email addresses produced by the parallel `dm-researcher` agents. ZoomInfo is the first attempt at email verification; web-research results are preserved when ZoomInfo finds no match or returns an error.

The deck-generation skill (`/pptx`) remains unchanged.

## Goals

1. Increase the share of decision-maker contacts shipped to outreach with verified emails.
2. Preserve high-confidence web-verified emails when ZoomInfo data is stale or absent.
3. Keep the pipeline robust to ZoomInfo MCP unavailability.
4. Add no new failure modes to downstream agents (`dm-compiler`, `outreach-composer`).

## Non-Goals

- ZoomInfo company enrichment for `company-researcher` / `company-synthesizer`.
- ZoomInfo intent or scoops signals feeding the scoring rubric.
- Replacing the `/pptx` skill or modifying `deck-builder`.
- Caching ZoomInfo results across pipeline runs.

## Architecture

### Phase Insertion

A new agent, `dm-enricher`, runs serially between DM research (parallel) and DM compilation. Phases renumber rather than introducing a fractional `7.5`:

| New | Old | Phase |
|-----|-----|-------|
| 7   | 7   | DM Research (5x parallel) |
| **8** | —   | **DM Enrichment (new, 1x serial)** |
| 9   | 8   | DM Compilation |
| 10  | 9   | Outreach Generation |
| 11  | 10  | Marketing Content |
| 12  | 11  | Deck Script Generation |
| 13  | 12  | Deck Generation |
| 14  | 13  | Completion |

### Data Flow

```
dm-researcher (5x) → dm-01.md ... dm-05.md
                      │
                      ▼
              dm-enricher (1x)  ←─ ZoomInfo MCP (enrich_contacts)
                      │
                      ▼
              dm-enriched.md
                      │
                      ▼
              dm-compiler → decision-makers.md → outreach-composer
```

`dm-compiler`'s sole DM input changes from a glob over `dm-*.md` to a single read of `dm-enriched.md`. The original per-instance files remain on disk for audit.

## Component Specifications

### Agent: `dm-enricher`

**File:** `agents/dm-enricher.md`
**Model:** inherit
**Tools:** `[Read, Write, Glob, mcp__<zoominfo-server>__enrich_contacts, mcp__<zoominfo-server>__lookup]`

**Inputs**
- `./{slug}/decision-maker-research/dm-*.md` (all parallel researcher outputs)

**Output**
- `./{slug}/decision-maker-research/dm-enriched.md`

**Workflow**

1. Glob all `dm-*.md` files. Read each completely. Extract every contact row.
2. For each unique contact (deduplicated by normalized name + company):
   - Call `enrich_contacts` once with `{first_name, last_name, company_name}`.
   - If the response returns multiple candidates, disambiguate by entity-resolved title match against the contact's known title.
   - If still ambiguous, classify as `Match: ambiguous` and retain the dm-researcher email.
   - If a single matched record returns an email, apply the precedence rules in the section below.
3. Track per-contact match status (`zoominfo`, `no-match`, `ambiguous`, `mcp-error`, `skipped`).
4. Compute aggregate enrichment metrics (matched count, override count, etc.).
5. If MCP-wide failure detected at agent start, write the warning banner and pass dm-researcher data through unchanged with `Match: skipped` for all contacts.
6. Write `dm-enriched.md` per the schema below.

**Constraints**

- Maximum one ZoomInfo call per contact. No retries on no-match — this is a "first attempt" check, not exhaustive search.
- No fabrication. If ZoomInfo returns no email, do not synthesize one.
- Do not perform deduplication beyond the input-stage normalized-name match needed to avoid double-billing ZoomInfo for duplicates across researcher files. Final deduplication remains in `dm-compiler`.

### Precedence Rules

| dm-researcher email confidence | ZoomInfo result        | Final email          | Final confidence       |
|-------------------------------|------------------------|----------------------|------------------------|
| Verified (Web)                | match found            | dm-researcher's      | `Verified (Web)`       |
| Verified (Web)                | no match / error       | dm-researcher's      | `Verified (Web)`       |
| Pattern-matched               | match found            | **ZoomInfo's**       | `Verified (ZoomInfo)`  |
| Pattern-matched               | no match / error       | dm-researcher's      | `Pattern-matched`      |
| Unverified                    | match found            | **ZoomInfo's**       | `Verified (ZoomInfo)`  |
| Unverified                    | no match / error       | dm-researcher's      | `Unverified`           |
| (none)                        | match found            | **ZoomInfo's**       | `Verified (ZoomInfo)`  |
| (none)                        | no match / error       | (none)               | `No email`             |

`outreach-composer` treats `Verified (ZoomInfo)` as a clean verified email — no warning banner. Existing handling of `Pattern-matched`, `Unverified`, and missing emails carries forward unchanged.

### Output Schema: `dm-enriched.md`

Sections, in order:

1. **Metadata** — date, slug, agent, source files (`dm-01.md`..`dm-05.md`), enrichment provider (`ZoomInfo MCP`), MCP status (`available` | `unavailable` | `partial`).
2. **Banner** — present only when MCP is unavailable at start or when more than 50% of contacts return `mcp-error`. Text: `WARNING: ZoomInfo MCP unavailable — emails reflect web research only`.
3. **Enrichment Summary** — total contacts, matched, no-match, ambiguous, errors, skipped, override count broken out by transition (`Pattern → ZoomInfo`, `Unverified → ZoomInfo`, `(none) → ZoomInfo`).
4. **Contacts by Company** — for each company, a table:
   `Name | Title | Tier | Email | Email Confidence | Match | Original Email (if changed) | Sources`
5. **Source Reconciliation Notes** — list of contacts where ZoomInfo email differs from dm-researcher email. Captures both for traceability.

### Error Handling

- **Per-contact no-match or API error.** Fall through silently. Retain the dm-researcher email and confidence. Record the result in the `Match` column.
- **Ambiguous match** (multiple ZoomInfo candidates after title disambiguation). Retain dm-researcher email. Record `Match: ambiguous`.
- **MCP-wide unavailability detected at agent start.** Emit banner, pass through all dm-researcher data unchanged with `Match: skipped`. Pipeline continues.
- **Partial outage mid-run.** Count `mcp-error` entries; if >50% of contacts hit errors, emit the same banner. Otherwise proceed without banner.
- **Pipeline never halts on ZoomInfo failures.** A user can re-run Phase 8 in isolation to backfill enrichment after MCP recovery.

## Orchestrator Changes

### `commands/lead-genius.md`

- Renumber phases per the table in Architecture.
- Add the Phase 8 dispatch block:
  ```
  Spawn the dm-enricher agent:
  - subagent_type: "dm-enricher"
  - prompt: "[Slug: {slug}] Glob all dm-*.md in
             ./{slug}/decision-maker-research/. For each contact, attempt
             ZoomInfo enrich_contacts. Apply precedence rules. Write to
             ./{slug}/decision-maker-research/dm-enriched.md"
  - description: "Enriching DM emails via ZoomInfo →
                  ./{slug}/decision-maker-research/dm-enriched.md"

  Wait for completion.
  ```
- Update Phase 9 (compilation) prompt: change `Glob all dm-*.md` to `Read dm-enriched.md`.
- Update Phase 14 (completion summary) to reference the new enrichment file.

### `agents/dm-compiler.md`

- Change inputs section: replace `dm-*.md` glob with single read of `dm-enriched.md`.
- Update workflow step 1 from `Glob all dm-*.md in decision-maker-research/` to `Read dm-enriched.md`.
- Add to deduplication notes: when ZoomInfo overrode an email, surface both ZoomInfo and dm-researcher web sources in the `Sources` column of `decision-makers.md`.
- No changes to scoring, ranking, or tier-distribution logic.

### `agents/dm-researcher.md`

No changes. The researcher continues to produce the same output schema; enrichment is additive downstream.

## Versioning and Documentation

- `plugin.json`: bump `1.4.0` → `1.5.0`.
- New release notes file: `docs/RELEASE-NOTES-v1.5.0-2026-04-29.md`.
- `README.md` updates: agent table (add `dm-enricher`), pipeline phase list.
- `CLAUDE.md` updates: agent role table, phase-flow narrative, key design constraints (note ZoomInfo "first attempt" semantics and precedence rules).

## Testing and Validation

- Manual end-to-end run with a small slug (3-5 companies, 10-15 contacts) to confirm:
  - At least one `Pattern → ZoomInfo` override occurs.
  - At least one `Verified (Web)` email is preserved despite a ZoomInfo match.
  - `dm-enriched.md` schema validates against the structure above.
  - `dm-compiler` reads the enriched file without error and produces a properly ranked `decision-makers.md`.
- Simulate MCP-down: run with ZoomInfo disconnected; confirm banner appears and pipeline completes successfully.
- Simulate partial outage: inject errors in a fraction of contacts; confirm threshold logic.

## Open Items

None at design time. Implementation plan will resolve concrete ZoomInfo MCP tool-name strings (the server prefix differs per session) and the exact `enrich_contacts` request/response shape from a probe.
