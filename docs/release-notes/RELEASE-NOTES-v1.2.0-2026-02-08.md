# Release Notes

## v1.2.0 (2026-02-08)

### Changes

- **Email confidence indicators**: Outreach emails now surface email confidence with three layered indicators for non-verified contacts. Verified emails stay clean with no extra noise.
  - **Metadata field**: `**Email Confidence:**` added as the last item in the metadata block (omitted for verified)
  - **Warning banner**: Placed between metadata and email line with actionable guidance (omitted for verified)
  - **Inline tag**: `[Pattern-matched]` or `[Unverified]` appended to the email address (omitted for verified)
- **No-email outreach**: Contacts with no email found now get full outreach messages with lookup guidance (LinkedIn profile URL, company email pattern) so the message is ready when the user finds the address.
- **Outreach output format**: Updated outreach-composer to use header + metadata block format with continuous email body (no internal section labels).

### Updated Files

- `skills/executive-outreach/SKILL.md`: Added email confidence to input requirements, output structure (metadata field, warning banner, inline tag), and no-email handling
- `skills/executive-outreach/references/examples.md`: Updated Tier 3 example to show pattern-matched indicators, added no-email example with lookup guidance
- `agents/outreach-composer.md`: Extracts and passes email confidence, generates outreach for all DMs including those with no email
- `CLAUDE.md`: Version bump, email confidence design constraint
- `README.md`: Updated outreach quality section and changelog link

### Upgrade Notes

No breaking changes. Existing pipeline output is unaffected. New pipeline runs will include email confidence indicators in the outreach output.
