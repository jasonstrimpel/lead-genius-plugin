# Release Notes

## v1.9.0 (2026-07-09)

### Summary

Adds a final mail-merge export step. A new `mail-merge-builder` agent turns the finished outreach emails into a clean, five-column Excel file (`mail-merge.xlsx`) ready to drop into a mail-merge tool.

### New Feature: Mail Merge Export (Phase 13)

- **`mail-merge-builder` agent**: reads `./{slug}/outreach.md` and writes `./{slug}/mail-merge.xlsx` in the same directory.
- **Exactly five columns**:

  | Column | Contents |
  |--------|----------|
  | First | Recipient first name |
  | Last | Recipient last name |
  | Email | `First Last <email@example.com>` (confidence tags stripped) |
  | Subject | One common, compelling subject line — identical in every row |
  | Body | The email body with no greeting and no sign-off |

- **One common subject**: synthesized from the per-recipient subjects and the shared offering, so a single subject line applies to all recipients.
- **Body cleaning**: the opening salutation and the closing sender signature are removed; interior paragraphs (including the forward-looking closing sentence) are preserved with line breaks in a wrapped cell.
- **Emailable recipients only**: one row per contact that has an email address; contacts with no email are skipped (and the count reported), since they cannot be merged.
- **XLSX generation**: writes and runs a small openpyxl script (installing openpyxl if needed), then removes the temporary script. Falls back to the `/xlsx` skill if Python is unavailable.

### Pipeline Change

- New **Phase 13: Mail Merge Export**, running last (after Deck Script Generation). Completion moves to **Phase 14**; the pipeline is now 15 phases.

### Housekeeping

- Release notes now live in `docs/release-notes/` (relocated from `docs/`).

### Updated Files

| File | Change |
|------|--------|
| `agents/mail-merge-builder.md` | New agent: exports outreach emails to `mail-merge.xlsx` |
| `commands/run.md` | New Phase 13 (Mail Merge Export); Completion renumbered to 14; summary lists `mail-merge.xlsx` |
| `.claude-plugin/plugin.json` | Version 1.8.0 -> 1.9.0 |
| `.claude-plugin/marketplace.json` | Version 1.8.0 -> 1.9.0 |
| `CLAUDE.md` | Version, 15-phase pipeline, 12 agents, phase flow, agent table |
| `README.md` | Phase table, output tree, Key Files, new "Mail Merge Export" section, changelog link |

### Upgrade Notes

- **Dependency**: real `.xlsx` output requires Python + openpyxl in the run environment (or the `/xlsx` skill as a fallback). If neither is available, this phase cannot produce the file.
- **No functional change** to earlier phases; the mail-merge file is derived entirely from `outreach.md`.
