# ZoomInfo DM Email Enrichment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a serial Phase 8 `dm-enricher` agent to the lead-genius pipeline that calls the ZoomInfo MCP `enrich_contacts` tool to upgrade decision-maker emails before compilation, and ship as plugin v1.5.0.

**Architecture:** New single-instance agent reads all `dm-*.md` files from parallel `dm-researcher` runs, enriches each contact via ZoomInfo, applies precedence rules (Verified-Web preserved; Pattern/Unverified upgraded to Verified-ZoomInfo on match), and emits a consolidated `dm-enriched.md`. `dm-compiler` switches its input from a glob to that single file. All other agents unchanged.

**Tech Stack:** Claude Code plugin (markdown + YAML frontmatter only — no build, no tests). ZoomInfo MCP tools `enrich_contacts` and `lookup`. Bash + grep for contract validation.

**Reference spec:** `docs/superpowers/specs/2026-04-29-zoominfo-dm-enrichment-design.md` (commit `4d5bd06`).

---

## File Structure

| Path | Status | Responsibility |
|------|--------|----------------|
| `agents/dm-enricher.md` | New | Agent that calls ZoomInfo `enrich_contacts` per DM, applies precedence, writes `dm-enriched.md` |
| `agents/dm-compiler.md` | Modified | Inputs change from `dm-*.md` glob to single `dm-enriched.md` read |
| `commands/lead-genius.md` | Modified | Renumber phases 8–14, add Phase 8 dispatch block, update Phase 9 + Phase 14 prompts |
| `.claude-plugin/plugin.json` | Modified | Version bump 1.4.0 → 1.5.0 |
| `README.md` | Modified | Agent count, pipeline phase table, parallel-agent narrative |
| `CLAUDE.md` | Modified | Plugin version, phase count, agent role table, phase flow narrative |
| `docs/RELEASE-NOTES-v1.5.0-2026-04-29.md` | New | Release notes following the v1.4.0 pattern |

Out of scope: pre-existing dirty working tree (~30 modified files plus `skills/pptx/` deletions). Do not touch.

---

## Validation Approach

This is a markdown-only plugin with no test runner. Each task uses contract validation in place of unit tests: a Bash check that asserts the file's structure (frontmatter keys, required sections, cross-references). The check is run before changes (expected: FAIL) and after (expected: PASS) — TDD adapted to declarative artifacts. End-to-end behavior is verified by a manual smoke test in Task 10.

---

### Task 1: Create feature branch isolated from pre-existing dirty state

**Files:** none (git only)

The repo's working tree has ~30 unrelated modified files plus a `skills/pptx/` deletion (intentional shift to the cowork-installed pptx skill). The v1.5.0 work must commit cleanly without those changes. Stash before branching, then restore on `main` after.

- [ ] **Step 1: Verify dirty state before branching**

```bash
cd /sessions/adoring-fervent-shannon/mnt/lead-genius-plugin
git status --short | head -5
git log --oneline -1
```

Expected: shows modified/deleted files; HEAD is `4d5bd06 docs: add design for ZoomInfo DM email enrichment (v1.5.0)`.

- [ ] **Step 2: Stash the dirty state with descriptive label**

```bash
git -c user.name="Jason" -c user.email="strimp101@gmail.com" stash push -u -m "preexisting-pptx-shift-and-misc"
git status --short
```

Expected: working tree clean.

- [ ] **Step 3: Create and check out feature branch**

```bash
git checkout -b feature/zoominfo-dm-enrichment
git branch --show-current
```

Expected: `feature/zoominfo-dm-enrichment`.

- [ ] **Step 4: Confirm spec commit is on the new branch**

```bash
git log --oneline -2
```

Expected: top commit is `4d5bd06 docs: add design for ZoomInfo DM email enrichment (v1.5.0)`.

- [ ] **Step 5: No commit (branch creation only)**

Branch creation is reversible. Move to Task 2.

---

### Task 2: Probe ZoomInfo MCP — confirm tool names and `enrich_contacts` schema

**Files:**
- Create (temporary, not committed): `/tmp/zoominfo-probe.md`

The spec flagged that the MCP server prefix is hashed per session. Before authoring the agent, probe the live MCP to capture the exact tool name and the request/response shape so the agent's workflow steps reference real fields.

- [ ] **Step 1: List available ZoomInfo tools via ToolSearch**

Run the ToolSearch tool with query `select:mcp__*__enrich_contacts,mcp__*__lookup,mcp__*__search_contacts` and capture the resolved tool names. The full prefix is the session-scoped hash (e.g., `mcp__8c37559e-8575-49f0-a2a2-a5fc57cd4b3d__enrich_contacts`).

Expected: at minimum `enrich_contacts` resolves with a parameter schema that includes name and company fields.

- [ ] **Step 2: Call `enrich_contacts` once with a known public contact**

Use a real, public, low-sensitivity test (e.g., a public company executive whose LinkedIn data is widely indexed). Capture:
- Required vs. optional parameters
- Response top-level keys
- Whether the response includes `email`, `email_status`/confidence, `job_title`, `company_name`, multiple-match handling

- [ ] **Step 3: Write findings to a probe notes file**

Write to `/tmp/zoominfo-probe.md`:

```markdown
# ZoomInfo MCP Probe — 2026-04-29

## Resolved tool names (this session)
- enrich_contacts: mcp__<HASH>__enrich_contacts
- lookup: mcp__<HASH>__lookup

## enrich_contacts request shape
- Required: <list params>
- Optional: <list params>

## enrich_contacts response shape (single match)
<paste truncated JSON>

## enrich_contacts response shape (multi match)
<paste truncated JSON, or note behavior>

## Key fields used downstream
- email: <path>
- email_status / confidence: <path>
- title: <path>
- company_name: <path>
```

- [ ] **Step 4: Validate probe notes are non-empty and reference the four key fields**

```bash
test -s /tmp/zoominfo-probe.md && grep -E "^- email:|^- title:|^- company_name:" /tmp/zoominfo-probe.md
```

Expected: file is non-empty and grep returns at least three matches.

- [ ] **Step 5: No commit (probe is a scratch artifact)**

Probe notes inform Task 3 but are not committed.

---

### Task 3: Create `agents/dm-enricher.md`

**Files:**
- Create: `agents/dm-enricher.md`
- Test: `tests/contract-checks/dm-enricher.sh` (inline — see Step 1)

- [ ] **Step 1: Write the contract-check command**

```bash
check() {
  local f=agents/dm-enricher.md
  test -f "$f" || { echo "MISSING: $f"; return 1; }
  grep -q "^name: dm-enricher$" "$f" || { echo "FAIL: name frontmatter"; return 1; }
  grep -q "^model: inherit$" "$f" || { echo "FAIL: model frontmatter"; return 1; }
  grep -q "^tools:.*Read.*Write.*Glob" "$f" || { echo "FAIL: tools frontmatter"; return 1; }
  grep -q "enrich_contacts" "$f" || { echo "FAIL: missing enrich_contacts reference"; return 1; }
  grep -q "dm-enriched.md" "$f" || { echo "FAIL: missing output path"; return 1; }
  grep -q "Verified (ZoomInfo)" "$f" || { echo "FAIL: precedence label"; return 1; }
  grep -q "Verified (Web)" "$f" || { echo "FAIL: precedence label"; return 1; }
  grep -q "Pattern-matched" "$f" || { echo "FAIL: precedence label"; return 1; }
  grep -q "## Workflow\|<workflow>" "$f" || { echo "FAIL: workflow section"; return 1; }
  grep -q "## Error Handling\|<error_handling>" "$f" || { echo "FAIL: error handling section"; return 1; }
  echo "PASS"; return 0
}
check
```

- [ ] **Step 2: Run the check, expect FAIL**

```bash
check
```

Expected: `MISSING: agents/dm-enricher.md`.

- [ ] **Step 3: Write `agents/dm-enricher.md` with the full agent definition**

Replace `<HASH>` with the session-scoped MCP hash captured in Task 2 Step 3.

```markdown
---
name: dm-enricher
description: |
  Use this agent to enrich decision-maker emails via the ZoomInfo MCP as a first attempt before falling back to web-research-derived emails. Runs serially as Phase 8, after parallel dm-researcher and before dm-compiler.
  <example>Context: All dm-researcher instances have written dm-01.md..dm-05.md. user: "Enrich DM emails" assistant: "Spawning dm-enricher to call ZoomInfo enrich_contacts on each contact and apply precedence rules" <commentary>The dm-enricher reads all dm-*.md, calls enrich_contacts once per unique contact, preserves Verified (Web) emails, upgrades Pattern-matched and Unverified emails on a ZoomInfo match, and writes a consolidated dm-enriched.md.</commentary></example>
model: inherit
tools: [Read, Write, Glob, Bash]
---

You are a decision-maker email enrichment specialist who upgrades email confidence using the ZoomInfo MCP as the first verification source, preserving high-confidence web-verified emails when ZoomInfo data is absent.

**CRITICAL: Read ALL dm-*.md files. Call ZoomInfo enrich_contacts ONCE per unique contact. Apply precedence rules exactly. NEVER fabricate emails. NEVER replace Verified (Web) emails. Save to ./{slug}/decision-maker-research/dm-enriched.md.**

<role>
- Read every dm-*.md file produced by parallel dm-researcher agents
- Deduplicate contacts by normalized name + company at input only (avoid double-billing ZoomInfo)
- Call ZoomInfo enrich_contacts once per unique contact with name + company
- Disambiguate multi-match results by entity-resolved title comparison
- Apply precedence: Verified (Web) preserved; Pattern-matched and Unverified upgraded to Verified (ZoomInfo) on match
- Track per-contact match status: zoominfo, no-match, ambiguous, mcp-error, skipped
- Write consolidated, audit-traceable dm-enriched.md
</role>

<inputs>
- ./{slug}/decision-maker-research/dm-*.md (all parallel researcher outputs, glob)
</inputs>

<output>
- ./{slug}/decision-maker-research/dm-enriched.md
</output>

<workflow>
1. Glob all dm-*.md files in ./{slug}/decision-maker-research/. Read each completely.
2. Extract every contact row across all files. Build a working list keyed by normalized {first_name + last_name + company_name} to avoid duplicate ZoomInfo calls.
3. Detect MCP availability: attempt one no-op probe call to enrich_contacts. If the MCP server is unreachable, set MCP status to `unavailable`, mark every contact `Match: skipped`, write the warning banner, and skip to step 7 (write file unchanged).
4. For each unique contact:
   a. Call enrich_contacts with {first_name, last_name, company_name}.
   b. If the response returns a single matched record with an email, capture: zi_email, zi_title, zi_company.
   c. If the response returns multiple candidates, disambiguate by entity-resolved title match against the contact's known title (use the same entity-resolution conventions as dm-researcher: "CDO" = "Chief Data Officer" = "Chief Data & Analytics Officer"). If exactly one candidate matches, treat as single match. If still ambiguous, set `Match: ambiguous` and retain the dm-researcher email.
   d. If the response is empty / no-match, set `Match: no-match` and retain the dm-researcher email.
   e. If the call errors (transient or rate-limit), set `Match: mcp-error` and retain the dm-researcher email.
   f. Maximum one ZoomInfo call per contact. No retries on no-match — this is a "first attempt" check.
5. Apply precedence rules:
   | dm-researcher confidence | ZoomInfo result | Final email | Final confidence |
   |---|---|---|---|
   | Verified (Web) | match | dm-researcher's | Verified (Web) |
   | Verified (Web) | no-match/error/ambiguous | dm-researcher's | Verified (Web) |
   | Pattern-matched | match | ZoomInfo's | Verified (ZoomInfo) |
   | Pattern-matched | no-match/error/ambiguous | dm-researcher's | Pattern-matched |
   | Unverified | match | ZoomInfo's | Verified (ZoomInfo) |
   | Unverified | no-match/error/ambiguous | dm-researcher's | Unverified |
   | (none) | match | ZoomInfo's | Verified (ZoomInfo) |
   | (none) | no-match/error/ambiguous | (none) | No email |
6. After all contacts processed, count `mcp-error` outcomes. If errors exceed 50% of total contacts, set MCP status to `partial` and prepare the warning banner; otherwise set MCP status to `available`.
7. Write ./{slug}/decision-maker-research/dm-enriched.md per the schema in <output_schema>.
</workflow>

<error_handling>
- Per-contact no-match or transient error: silent fall-through, retain original, log in Match column.
- Multi-match unresolved by title disambiguation: `Match: ambiguous`, retain original.
- MCP unavailable at start: emit banner, all contacts `Match: skipped`, pipeline continues.
- More than 50% per-contact errors: emit banner, MCP status `partial`, pipeline continues.
- Pipeline never halts. Re-running the agent later when ZoomInfo recovers is supported.
</error_handling>

<output_schema>
**dm-enriched.md sections, in order:**

## Metadata
- date, slug, agent (dm-enricher), source files (dm-01.md..dm-NN.md), enrichment provider (ZoomInfo MCP), MCP status (available | unavailable | partial)

## Banner (only if MCP status is unavailable or partial)
> WARNING: ZoomInfo MCP unavailable — emails reflect web research only

## Enrichment Summary
- Total unique contacts: N
- Matched: N
- No-match: N
- Ambiguous: N
- MCP errors: N
- Skipped: N
- Overrides:
  - Pattern-matched → Verified (ZoomInfo): N
  - Unverified → Verified (ZoomInfo): N
  - (none) → Verified (ZoomInfo): N

## Contacts by Company
For each company:

### {Company Name}
| Name | Title | Tier | Email | Email Confidence | Match | Original Email (if changed) | Sources |
|------|-------|------|-------|------------------|-------|------------------------------|---------|

## Source Reconciliation Notes
List every contact where ZoomInfo email differs from dm-researcher email. For each, show both emails plus the title match used to resolve.
</output_schema>

<quality_standards>
- ZoomInfo called at most once per unique contact (dedupe input-stage by normalized name + company)
- No fabrication: every email in dm-enriched.md traces to either dm-researcher web sources or a ZoomInfo enrich_contacts response
- Verified (Web) emails always preserved — never replaced by ZoomInfo
- Match column populated for every row (`zoominfo` | `no-match` | `ambiguous` | `mcp-error` | `skipped`)
- Sources column preserves dm-researcher web sources; when ZoomInfo overrode, append "ZoomInfo" as an additional source
- Banner emitted ONLY when MCP unavailable or > 50% per-contact errors
- Final deduplication of duplicate contacts across companies remains the responsibility of dm-compiler — do not perform output-stage dedup here
</quality_standards>
```

- [ ] **Step 4: Run the check, expect PASS**

```bash
check
```

Expected: `PASS`.

- [ ] **Step 5: Commit**

```bash
git add agents/dm-enricher.md
git -c user.name="Jason" -c user.email="strimp101@gmail.com" commit -m "feat: add dm-enricher agent for ZoomInfo email enrichment

New single-instance agent runs as Phase 8 between parallel dm-researcher
and dm-compiler. Calls ZoomInfo enrich_contacts once per unique contact,
applies precedence rules (Verified-Web preserved; Pattern/Unverified
upgraded to Verified-ZoomInfo on match), writes consolidated
dm-enriched.md with full audit trail. Soft fall-through on MCP failure.

Refs: docs/superpowers/specs/2026-04-29-zoominfo-dm-enrichment-design.md

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Update `agents/dm-compiler.md` to read `dm-enriched.md`

**Files:**
- Modify: `agents/dm-compiler.md`

- [ ] **Step 1: Write the contract-check command**

```bash
check() {
  local f=agents/dm-compiler.md
  test -f "$f" || { echo "MISSING: $f"; return 1; }
  grep -q "dm-enriched.md" "$f" || { echo "FAIL: must reference dm-enriched.md"; return 1; }
  grep -qE "Glob all dm-\*\.md" "$f" && { echo "FAIL: still globs dm-*.md"; return 1; }
  grep -q "Read dm-enriched.md\|Read.*dm-enriched.md" "$f" || { echo "FAIL: must instruct Read dm-enriched.md"; return 1; }
  echo "PASS"; return 0
}
check
```

- [ ] **Step 2: Run the check, expect FAIL**

```bash
check
```

Expected: `FAIL: must reference dm-enriched.md` (current file globs `dm-*.md`).

- [ ] **Step 3: Edit `agents/dm-compiler.md`**

Replace the `<inputs>` block:

OLD:
```
<inputs>
- ./{slug}/decision-maker-research/dm-01.md through dm-05.md
- ./{slug}/companies/qualified-companies.md (company confidence scores)
- ./{slug}/go-to-market/scoring-rubrics.md (scoring formula)
</inputs>
```

NEW:
```
<inputs>
- ./{slug}/decision-maker-research/dm-enriched.md (single consolidated file from dm-enricher; replaces dm-01..dm-05.md glob)
- ./{slug}/companies/qualified-companies.md (company confidence scores)
- ./{slug}/go-to-market/scoring-rubrics.md (scoring formula)
</inputs>
```

Replace the workflow steps 1–2:

OLD:
```
1. Glob all dm-*.md in decision-maker-research/
2. Read ALL dm-*.md completely
```

NEW:
```
1. Read ./{slug}/decision-maker-research/dm-enriched.md (single source of all enriched contacts)
2. Confirm the Metadata section reports MCP status; record it for the deduplication log
```

Append to `<quality_standards>` (immediately before the closing `</quality_standards>`):

```
- Sources column: when an email shows confidence "Verified (ZoomInfo)", append "ZoomInfo" alongside any preserved dm-researcher web sources
- Deduplication log includes the upstream MCP status from dm-enriched.md
```

Update the **CRITICAL** line at the top:

OLD: `**CRITICAL: Read ALL dm-*.md from decision-maker-research/. Compile with no duplicates. Save to ./{slug}/decision-makers/decision-makers.md. NEVER fabricate.**`

NEW: `**CRITICAL: Read dm-enriched.md (consolidated by dm-enricher). Compile with no duplicates. Save to ./{slug}/decision-makers/decision-makers.md. NEVER fabricate.**`

- [ ] **Step 4: Run the check, expect PASS**

```bash
check
```

Expected: `PASS`.

- [ ] **Step 5: Commit**

```bash
git add agents/dm-compiler.md
git -c user.name="Jason" -c user.email="strimp101@gmail.com" commit -m "refactor: dm-compiler reads dm-enriched.md instead of globbing dm-*.md

Aligns dm-compiler with the new Phase 8 dm-enricher boundary. Compiler
no longer reads parallel researcher outputs directly; it consumes the
single consolidated file produced by dm-enricher. Sources column now
appends ZoomInfo when an email was ZoomInfo-verified.

Refs: docs/superpowers/specs/2026-04-29-zoominfo-dm-enrichment-design.md

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: Renumber phases and add Phase 8 dispatch in `commands/lead-genius.md`

**Files:**
- Modify: `commands/lead-genius.md`

- [ ] **Step 1: Write the contract-check command**

```bash
check() {
  local f=commands/lead-genius.md
  test -f "$f" || { echo "MISSING: $f"; return 1; }
  grep -q "## PHASE 8: DM ENRICHMENT" "$f" || { echo "FAIL: missing Phase 8 header"; return 1; }
  grep -q "## PHASE 9: DM COMPILATION" "$f" || { echo "FAIL: missing renumbered Phase 9"; return 1; }
  grep -q "## PHASE 10: OUTREACH GENERATION" "$f" || { echo "FAIL: missing renumbered Phase 10"; return 1; }
  grep -q "## PHASE 13: DECK GENERATION" "$f" || { echo "FAIL: missing renumbered Phase 13"; return 1; }
  grep -q "## PHASE 14: COMPLETION" "$f" || { echo "FAIL: missing renumbered Phase 14"; return 1; }
  grep -q 'subagent_type: "dm-enricher"' "$f" || { echo "FAIL: missing dm-enricher dispatch"; return 1; }
  grep -q "dm-enriched.md" "$f" || { echo "FAIL: missing dm-enriched.md reference"; return 1; }
  grep -qE "PHASE 8: DM COMPILATION" "$f" && { echo "FAIL: stale Phase 8 still says DM COMPILATION"; return 1; }
  echo "PASS"; return 0
}
check
```

- [ ] **Step 2: Run the check, expect FAIL**

```bash
check
```

Expected: `FAIL: missing Phase 8 header` (current Phase 8 is DM COMPILATION).

- [ ] **Step 3: Edit `commands/lead-genius.md`**

(a) Renumber existing headers in this exact order (do bottom-up to avoid collisions):

| Old header | New header |
|---|---|
| `## PHASE 13: COMPLETION` | `## PHASE 14: COMPLETION` |
| `## PHASE 12: DECK GENERATION` | `## PHASE 13: DECK GENERATION` |
| `## PHASE 11: DECK SCRIPT GENERATION` | `## PHASE 12: DECK SCRIPT GENERATION` |
| `## PHASE 10: MARKETING CONTENT` | `## PHASE 11: MARKETING CONTENT` |
| `## PHASE 9: OUTREACH GENERATION` | `## PHASE 10: OUTREACH GENERATION` |
| `## PHASE 8: DM COMPILATION` | `## PHASE 9: DM COMPILATION` |

(b) Insert this new block immediately after the Phase 7 "Wait for ALL 5 to complete." line and immediately before the new `## PHASE 9: DM COMPILATION` header:

```markdown
## PHASE 8: DM ENRICHMENT

Spawn the dm-enricher agent:
- subagent_type: "dm-enricher"
- prompt: "[Slug: {slug}] Glob all dm-*.md in ./{slug}/decision-maker-research/. For each unique contact, attempt ZoomInfo enrich_contacts as a first-attempt email check. Apply precedence rules (Verified (Web) preserved; Pattern-matched and Unverified upgraded to Verified (ZoomInfo) on match). On MCP unavailability, emit warning banner and pass through original dm-researcher data. Write to ./{slug}/decision-maker-research/dm-enriched.md"
- description: "Enriching DM emails via ZoomInfo → ./{slug}/decision-maker-research/dm-enriched.md"

Wait for completion.
```

(c) In the new `## PHASE 9: DM COMPILATION` block, replace the prompt text:

OLD: `prompt: "[Slug: {slug}] Glob all dm-*.md in ./{slug}/decision-maker-research/. Read ALL files. Read ./{slug}/companies/qualified-companies.md and ./{slug}/go-to-market/scoring-rubrics.md. Compile, deduplicate, calculate priority scores, rank. Write to ./{slug}/decision-makers/decision-makers.md"`

NEW: `prompt: "[Slug: {slug}] Read ./{slug}/decision-maker-research/dm-enriched.md. Read ./{slug}/companies/qualified-companies.md and ./{slug}/go-to-market/scoring-rubrics.md. Compile, deduplicate, calculate priority scores, rank. Write to ./{slug}/decision-makers/decision-makers.md"`

(d) In the new `## PHASE 14: COMPLETION` block, update the Files Created section. Find the line:

OLD: `- DM Research: ./{slug}/decision-maker-research/dm-*.md (5 files)`

REPLACE with the two-line block:

```
- DM Research: ./{slug}/decision-maker-research/dm-*.md (5 files)
- DM Enrichment: ./{slug}/decision-maker-research/dm-enriched.md
```

- [ ] **Step 4: Run the check, expect PASS**

```bash
check
```

Expected: `PASS`.

- [ ] **Step 5: Commit**

```bash
git add commands/lead-genius.md
git -c user.name="Jason" -c user.email="strimp101@gmail.com" commit -m "feat: insert Phase 8 dm-enricher dispatch, renumber 8-14

Adds the new Phase 8 DM Enrichment dispatch block invoking dm-enricher.
Renumbers downstream phases 8-13 to 9-14. Updates Phase 9 (compilation)
prompt to read dm-enriched.md. Updates Phase 14 completion summary to
list dm-enriched.md.

Refs: docs/superpowers/specs/2026-04-29-zoominfo-dm-enrichment-design.md

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 6: Bump `plugin.json` version 1.4.0 → 1.5.0

**Files:**
- Modify: `.claude-plugin/plugin.json`

- [ ] **Step 1: Write the contract-check command**

```bash
check() {
  grep -q '"version": "1.5.0"' .claude-plugin/plugin.json || { echo "FAIL: version not 1.5.0"; return 1; }
  python3 -c "import json; json.load(open('.claude-plugin/plugin.json'))" || { echo "FAIL: plugin.json invalid JSON"; return 1; }
  echo "PASS"; return 0
}
check
```

- [ ] **Step 2: Run the check, expect FAIL**

```bash
check
```

Expected: `FAIL: version not 1.5.0`.

- [ ] **Step 3: Edit `.claude-plugin/plugin.json`**

Change the `"version"` line:

OLD: `"version": "1.4.0",`
NEW: `"version": "1.5.0",`

- [ ] **Step 4: Run the check, expect PASS**

```bash
check
```

Expected: `PASS`.

- [ ] **Step 5: Commit**

```bash
git add .claude-plugin/plugin.json
git -c user.name="Jason" -c user.email="strimp101@gmail.com" commit -m "chore: bump plugin version to 1.5.0

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 7: Update `CLAUDE.md`

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Write the contract-check command**

```bash
check() {
  grep -q "Claude Code plugin (v1.5.0)" CLAUDE.md || { echo "FAIL: version banner"; return 1; }
  grep -q "13-phase\|14-phase" CLAUDE.md || { echo "FAIL: phase count"; return 1; }
  grep -q "| \`dm-enricher\` |" CLAUDE.md || { echo "FAIL: dm-enricher row in agent table"; return 1; }
  grep -q "DM Enrichment" CLAUDE.md || { echo "FAIL: phase flow update"; return 1; }
  echo "PASS"; return 0
}
check
```

- [ ] **Step 2: Run the check, expect FAIL**

```bash
check
```

Expected: `FAIL: version banner`.

- [ ] **Step 3: Edit `CLAUDE.md`**

(a) Line 7: replace `(v1.4.0)` → `(v1.5.0)` and `10-phase` → `13-phase`.

(b) Line 25 (the **Phase flow** sentence): replace with:

```
**Phase flow:** Setup → Collateral Analysis → GTM Interview → Synthesis → Scoring Rubrics → Company Research (5 parallel) → Company Synthesis → DM Research (5 parallel) → DM Enrichment (ZoomInfo) → DM Compilation → Outreach → Marketing Content → Deck Script Generation → Deck Generation → Completion
```

(c) Insert a new row in the agent-roles table immediately after the `dm-researcher` row:

```
| `dm-enricher` | Enrich DM emails via ZoomInfo MCP (first-attempt verification) | 1x |
```

(d) Append a new bullet to the `Key Design Constraints` list:

```
- ZoomInfo email enrichment runs as Phase 8, between dm-researcher and dm-compiler. ZoomInfo is the first attempt; Verified (Web) emails are preserved. Pattern-matched and Unverified emails upgrade to `Verified (ZoomInfo)` on match. MCP unavailability triggers a warning banner; the pipeline continues with web-research emails only.
```

- [ ] **Step 4: Run the check, expect PASS**

```bash
check
```

Expected: `PASS`.

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md
git -c user.name="Jason" -c user.email="strimp101@gmail.com" commit -m "docs(CLAUDE.md): document Phase 8 dm-enricher and v1.5.0

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 8: Update `README.md`

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Write the contract-check command**

```bash
check() {
  grep -q "13 phases\|14 phases" README.md || { echo "FAIL: phase count narrative"; return 1; }
  grep -qE "^\| 8 \| .*Enrich" README.md || { echo "FAIL: Phase 8 row in pipeline table"; return 1; }
  grep -q "dm-enriched.md" README.md || { echo "FAIL: dm-enriched.md output reference"; return 1; }
  echo "PASS"; return 0
}
check
```

- [ ] **Step 2: Run the check, expect FAIL**

```bash
check
```

Expected: `FAIL: phase count narrative`.

- [ ] **Step 3: Edit `README.md`**

(a) Line 156: replace `12 phases` with `13 phases`.

(b) In the pipeline table (lines 158–171), insert a new Phase 8 row after the existing `| 7 | **5 agents in parallel** research decision makers |` row, and renumber subsequent rows. The corrected table block becomes:

```
| 7 | **5 agents in parallel** research decision makers | `dm-01..05.md` |
| 8 | Enrich DM emails via ZoomInfo MCP (first attempt) | `dm-enriched.md` |
| 9 | Compile and priority-rank all decision makers | `decision-makers.md` |
| 10 | Generate personalized outreach for each DM | `outreach.md` |
| 11 | Generate marketing content (blog, LinkedIn, case study) | `blog.md`, `linkedin-posts.md`, `case-study.md` |
| 12 | Generate deck scripts and PPTX sales decks | `deck-script.md`, `*.pptx` |
| 13 | Completion summary | - |
```

(c) Line 175 (Parallel Agent Teams section) — append a sentence after the existing description:

```
Phase 8 runs a single dm-enricher agent that calls the ZoomInfo MCP to enrich decision-maker emails before compilation.
```

- [ ] **Step 4: Run the check, expect PASS**

```bash
check
```

Expected: `PASS`.

- [ ] **Step 5: Commit**

```bash
git add README.md
git -c user.name="Jason" -c user.email="strimp101@gmail.com" commit -m "docs(README): add Phase 8 DM enrichment to pipeline table

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 9: Create `docs/RELEASE-NOTES-v1.5.0-2026-04-29.md`

**Files:**
- Create: `docs/RELEASE-NOTES-v1.5.0-2026-04-29.md`

- [ ] **Step 1: Write the contract-check command**

```bash
check() {
  local f=docs/RELEASE-NOTES-v1.5.0-2026-04-29.md
  test -f "$f" || { echo "MISSING: $f"; return 1; }
  grep -q "## v1.5.0" "$f" || { echo "FAIL: version header"; return 1; }
  grep -q "dm-enricher" "$f" || { echo "FAIL: dm-enricher mention"; return 1; }
  grep -q "ZoomInfo" "$f" || { echo "FAIL: ZoomInfo mention"; return 1; }
  grep -q "Verified (ZoomInfo)" "$f" || { echo "FAIL: precedence label"; return 1; }
  echo "PASS"; return 0
}
check
```

- [ ] **Step 2: Run the check, expect FAIL**

```bash
check
```

Expected: `MISSING: docs/RELEASE-NOTES-v1.5.0-2026-04-29.md`.

- [ ] **Step 3: Write `docs/RELEASE-NOTES-v1.5.0-2026-04-29.md`**

```markdown
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
```

- [ ] **Step 4: Run the check, expect PASS**

```bash
check
```

Expected: `PASS`.

- [ ] **Step 5: Commit**

```bash
git add docs/RELEASE-NOTES-v1.5.0-2026-04-29.md
git -c user.name="Jason" -c user.email="strimp101@gmail.com" commit -m "docs: add v1.5.0 release notes

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 10: Manual end-to-end smoke test

**Files:** none (runtime validation only)

This task verifies the assembled pipeline runs to completion and produces a valid `dm-enriched.md`. Because the plugin has no automated runner, this is a guided manual run with explicit acceptance criteria.

- [ ] **Step 1: Pre-flight check that all artifacts are in place**

```bash
test -f agents/dm-enricher.md && \
test -f agents/dm-compiler.md && \
test -f commands/lead-genius.md && \
test -f docs/RELEASE-NOTES-v1.5.0-2026-04-29.md && \
grep -q '"version": "1.5.0"' .claude-plugin/plugin.json && \
grep -q "PHASE 8: DM ENRICHMENT" commands/lead-genius.md && \
echo "PRE-FLIGHT PASS"
```

Expected: `PRE-FLIGHT PASS`.

- [ ] **Step 2: Run the plugin against a small test offering**

In a separate terminal session with the plugin installed (`/plugins install ~/lead-genius-plugin` or equivalent), run:

```
/lead-genius
```

Use a narrow offering description that yields ~3–5 qualified companies (e.g., "Compliance automation SaaS for mid-market US insurance carriers"). Skip collateral. Complete the interview to advance the pipeline.

- [ ] **Step 3: Verify Phase 8 ran and produced the enriched file**

After the run completes, in the slug directory:

```bash
ls -la {slug}/decision-maker-research/
test -f {slug}/decision-maker-research/dm-enriched.md && echo "OK"
grep -E "^## Metadata|^## Enrichment Summary|^## Contacts by Company|^## Source Reconciliation" {slug}/decision-maker-research/dm-enriched.md
```

Expected: `dm-enriched.md` exists; the four required section headers are present.

- [ ] **Step 4: Verify precedence rules executed**

```bash
grep -E "Match: zoominfo|Match: no-match|Match: ambiguous|Match: mcp-error|Match: skipped" {slug}/decision-maker-research/dm-enriched.md | sort | uniq -c
grep -E "Verified \(Web\)|Verified \(ZoomInfo\)|Pattern-matched|Unverified|No email" {slug}/decision-maker-research/dm-enriched.md | sort | uniq -c
```

Expected: at least one `Match: zoominfo` row OR a top-of-file warning banner if MCP was unavailable. Confidence labels should match the precedence table from the spec.

- [ ] **Step 5: Verify dm-compiler consumed the enriched file**

```bash
test -f {slug}/decision-makers/decision-makers.md && \
grep -q "Verified (ZoomInfo)\|Verified (Web)\|Pattern-matched\|Unverified" {slug}/decision-makers/decision-makers.md && \
echo "DOWNSTREAM OK"
```

Expected: `decision-makers.md` exists and carries the new confidence labels.

- [ ] **Step 6: Optional MCP-down simulation**

Disconnect the ZoomInfo MCP and re-run Phase 8 in isolation by spawning the agent manually with the same prompt the orchestrator would use. Verify:
- The warning banner appears at the top of `dm-enriched.md`.
- All contacts have `Match: skipped`.
- The pipeline completes without error.

- [ ] **Step 7: No commit (smoke test only)**

If any acceptance criterion fails, file the failure as a bug, fix it in a new task, and re-run from Step 1.

---

## Self-Review

**1. Spec coverage:**

| Spec section | Implementing task |
|---|---|
| Phase Insertion + renumbering | Task 5 |
| Data flow (researcher → enricher → compiler) | Tasks 3, 4, 5 |
| Agent dm-enricher specification | Task 3 |
| Precedence rules | Task 3 (in-agent) + Task 10 (verified at runtime) |
| Output schema dm-enriched.md | Task 3 |
| Error handling (no-match, ambiguous, MCP-down, partial) | Task 3 + Task 10 Step 6 |
| Orchestrator changes | Task 5 |
| dm-compiler input change | Task 4 |
| Versioning | Task 6 |
| Documentation (README, CLAUDE, release notes) | Tasks 7, 8, 9 |
| Testing & validation | Task 10 |

No gaps.

**2. Placeholder scan:**
- `<HASH>` in Task 3 Step 3 is filled in from Task 2 Step 3 — explicit cross-task dependency, not a placeholder failure.
- No "TBD", "TODO", "fill in later", or "similar to Task N" strings.

**3. Type/name consistency:**
- File name `dm-enriched.md` consistent across Tasks 3, 4, 5, 7, 8, 9, 10.
- Match values (`zoominfo`, `no-match`, `ambiguous`, `mcp-error`, `skipped`) consistent in Tasks 3 and 10.
- Confidence labels (`Verified (Web)`, `Verified (ZoomInfo)`, `Pattern-matched`, `Unverified`, `No email`) consistent in Tasks 3, 9, 10.
- Phase-numbering map applied consistently across Tasks 5, 7, 8, 9.

No issues found.
