# Release Notes

## v1.8.0 (2026-07-09)

### Summary

Three changes: the command is renamed to de-double its invocation, ZoomInfo enrichment is made robust to environment-specific server names, and the collateral analyzer now anonymizes client identities while still mining them for ICP intelligence.

### Command Invocation: `/lead-genius:run`

- Renamed `commands/lead-genius.md` -> `commands/run.md`. The command now invokes as **`/lead-genius:run`** instead of the doubled `/lead-genius:lead-genius`.
- Claude Code namespaces all plugin commands as `/plugin:command`; a bare `/lead-genius` is only possible by abandoning the plugin architecture (standalone `.claude/` config, no marketplace distribution), which we deliberately did not do. De-doubling to `run` matches the ZoomInfo plugin's verb-style naming (`zoominfo:build-list`, etc.).
- Updated every invocation reference in `plugin.json`, `marketplace.json`, `CLAUDE.md`, and `README.md`. Plugin name remains `lead-genius`.

### ZoomInfo Enrichment: robust to server naming

- Removed the `tools:` allowlist from `agents/dm-enricher.md`. The ZoomInfo MCP server resolves to an environment-specific (UUID) name rather than a friendly `mcp__zoominfo__` prefix, so a hardcoded allowlist entry would silently break enrichment across sessions/installs.
- With no allowlist, dm-enricher inherits all available tools and can reach the ZoomInfo MCP (`enrich_contacts`, `lookup`) regardless of how that server is registered. A frontmatter comment documents the intent. Trade-off: broader tool access for that one agent.

### Collateral Analyzer: strip client identities, keep the intelligence

- `agents/collateral-analyzer.md` previously preserved verbatim content, quotes, and named entities — which would pull confidential client names, logos, and attributed quotes from SOWs, proposals, and RFP responses straight into the output.
- New `<entity_anonymization>` rule: for every named entity, (1) STRIP it from the output (client/customer/prospect/third-party names, logos, individuals, project/contract IDs, addresses, or any detail specific enough to identify one org), and (2) ABSTRACT it into ICP-defining attributes (industry/sub-vertical, size band, geography, business model, regulatory context, buyer roles, use cases, deal-size ranges) routed into the ICP, Demand Signals, and Buyer Personas sections.
- The offering and seller remain the subject of the analysis and are not anonymized — only clients, prospects, and third parties are.
- Also updated: the CRITICAL directive, `<role>`, `<workflow>` (added an anonymize step plus a final "zero entity names remain" scan), `<extraction_guidance>` (no client-identifying quotes; job titles/roles stay), the Success Stories section (anonymized outcomes + segment only), the ICP section (fold abstracted client attributes in as segments), and the `sources` metadata (list document type only; redact client names from filenames).

### Other

- `skills/executive-outreach/SKILL.md`: tightened the detail-rich sentence-length guidance from 15-20 words to 10-15 words.

### Updated Files

| File | Change |
|------|--------|
| `commands/run.md` | Renamed from `commands/lead-genius.md`; invocation is now `/lead-genius:run` |
| `agents/dm-enricher.md` | Removed `tools:` allowlist so it reaches the ZoomInfo MCP regardless of server name |
| `agents/collateral-analyzer.md` | Strip-but-abstract anonymization of client/company identities |
| `skills/executive-outreach/SKILL.md` | Sentence-length guidance 15-20 -> 10-15 words |
| `.claude-plugin/plugin.json` | Version 1.7.0 -> 1.8.0; invocation reference |
| `.claude-plugin/marketplace.json` | Version 1.7.0 -> 1.8.0; invocation reference |
| `CLAUDE.md` | Version, `/lead-genius:run`, `commands/run.md` references |
| `README.md` | Invocation references, changelog link |

### Upgrade Notes

- **Invocation changed**: use `/lead-genius:run` (the old `/lead-genius:lead-genius` no longer exists).
- **No functional change** to the research, scoring, enrichment precedence, outreach, marketing, or deck-script pipeline beyond the above.
