# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code plugin (v1.7.0) that provides the `/lead-genius` command — a 14-phase conversational lead generation pipeline. It interviews users about their offering/GTM strategy, dispatches sequential research agents to find companies and decision makers, then generates personalized outreach emails and marketing content (blog, LinkedIn posts, case study, sales deck scripts).

There is no build system, package manager, or test suite. The codebase is entirely markdown files: agent definitions, commands, and skills interpreted by the Claude Code plugin runtime.

## Plugin Structure

```
.claude-plugin/plugin.json   # Plugin manifest (name, version, tools)
commands/lead-genius.md       # Main orchestrator — the /lead-genius command
agents/                       # 11 specialized agent prompts (markdown + YAML frontmatter)
skills/executive-outreach/    # Email generation skill with SKILL.md + reference examples
```

## Architecture

The orchestrator (`commands/lead-genius.md`) drives a 14-phase sequential pipeline. It never does research or writes output itself — it delegates everything to agents via the Task tool.

**Phase flow:** Setup → Collateral Analysis → GTM Interview → Synthesis → Scoring Rubrics → Company Research → Company Synthesis → DM Research → DM Enrichment (ZoomInfo) → DM Compilation → Outreach → Marketing Content → Deck Script Generation → Completion

**Coordination model:** File-based and fully sequential. Each research stage runs as a single agent that writes one output file (`companies.md`, `dm-research.md`); downstream agents score, enrich, and rank without de-duplicating, because a single sequential pass never produces duplicate entries. No inter-agent messaging.

**Output directory:** `./{slug}/` at project root, where slug is URL-safe derived from the offering. Each phase writes to a specific subdirectory (e.g., `company-research/`, `decision-makers/`). The deck-scripter writes markdown deck scripts in `marketing/`.

### Agent Roles

| Agent | Purpose | Parallelism |
|-------|---------|-------------|
| `collateral-analyzer` | Extract GTM content from sales PDFs | 1x |
| `gtm-synthesizer` | Combine interview outputs into research brief | 1x |
| `scoring-strategist` | Generate deterministic scoring rubrics | 1x |
| `company-researcher` | Web search for qualifying companies (one sequential pass) | 1x |
| `company-synthesizer` | Score and rank top 10 companies | 1x |
| `dm-researcher` | Find decision makers across all companies (one sequential pass) | 1x |
| `dm-enricher` | Enrich DM emails: ZoomInfo primary, web/pattern fallback | 1x |
| `dm-compiler` | Priority-rank all contacts | 1x |
| `content-writer` | Generate blog, LinkedIn posts, case study | 1x |
| `deck-scripter` | Write deck scripts (general + prospect-specific) from GTM research | 1x |
| `outreach-composer` | Generate tier-matched emails via `/executive-outreach` skill | 1x |

### Key Design Constraints

- The orchestrator must keep responses to 2-3 sentences between phases, ask one question per message during the interview, and use plain text (no markdown) for interview questions.
- Research runs sequentially, not in parallel: a single `company-researcher` makes 10-20 WebSearch calls and finds 15-20 distinct companies; a single `dm-researcher` makes 5-10 WebSearch calls per company across all qualified companies. Both cite all sources with URLs. Because each stage is one pass, downstream agents never deduplicate.
- Scoring uses a deterministic formula: `(Tier × Multiplier) + (Role Points) + (Activity Bonuses)` — defined by the scoring-strategist and applied by synthesizers.
- Buyer tier distribution targets 60% business/economic, 25% bridge/champion, 15% technical.
- Outreach emails must be under 120 words, tier-matched in tone, and reference company-specific evidence. Emails are generated for all decision makers including those with no email. Non-verified emails get three layered indicators: metadata field, warning banner, and inline tag. Verified emails stay clean.
- Collateral analysis marks sections as `[Clear]`, `[Inferred]`, or `[Gap]` — the interview adapts by skipping `[Clear]` topics.
- ZoomInfo email enrichment runs as Phase 8, between dm-researcher and dm-compiler. ZoomInfo is the PRIMARY source: a confident ZoomInfo match is used (`Verified (ZoomInfo)`) even over a web-verified email. When ZoomInfo cannot find a contact, returns an ambiguous result, or its API credits are exhausted, the pipeline falls back to the dm-researcher internet-search / pattern-matched email at its original confidence. Credit exhaustion short-circuits further ZoomInfo calls. MCP unavailability, partial outage, or credit exhaustion triggers a warning banner; the pipeline continues on the fallback layer.

## Making Changes

When modifying agent prompts, keep these constraints in mind:
- Each agent's YAML frontmatter defines its `allowed_tools` — only tools listed there are available to that agent.
- Agent input/output file paths are hardcoded in both the agent definition and the orchestrator. If you change one, update the other.
- The orchestrator references agents by their filename (minus `.md`). Renaming an agent file requires updating `commands/lead-genius.md`.
- The `outreach-composer` agent invokes the `/executive-outreach` skill, which lives at `skills/executive-outreach/SKILL.md`. The skill's reference examples are in `skills/executive-outreach/references/examples.md`.

## Optional User Inputs

- `senders/{name}.md` — Sender professional bio for credibility bridges in outreach emails.
- `collateral/*.pdf` — Sales materials analyzed in Phase 1 to pre-fill interview answers.
