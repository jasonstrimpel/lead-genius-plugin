# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code plugin (v1.1.0) that provides the `/lead-genius` command — a 10-phase conversational lead generation pipeline. It interviews users about their offering/GTM strategy, dispatches parallel research agents to find companies and decision makers, then generates personalized outreach emails.

There is no build system, package manager, or test suite. The codebase is entirely markdown files: agent definitions, commands, and skills interpreted by the Claude Code plugin runtime.

## Plugin Structure

```
.claude-plugin/plugin.json   # Plugin manifest (name, version, tools)
commands/lead-genius.md       # Main orchestrator — the /lead-genius command
agents/                       # 8 specialized agent prompts (markdown + YAML frontmatter)
skills/executive-outreach/    # Email generation skill with SKILL.md + reference examples
```

## Architecture

The orchestrator (`commands/lead-genius.md`) drives a linear 10-phase pipeline. It never does research or writes output itself — it delegates everything to agents via the Task tool.

**Phase flow:** Setup → Collateral Analysis → GTM Interview → Synthesis → Scoring Rubrics → Company Research (5 parallel agents) → Company Synthesis → DM Research (5 parallel agents) → DM Compilation → Outreach Generation

**Coordination model:** File-based. Parallel researchers write independent output files; synthesizer agents read all files, deduplicate, and merge. No inter-agent messaging.

**Output directory:** `./{slug}/` at project root, where slug is URL-safe derived from the offering. Each phase writes to a specific subdirectory (e.g., `company-research/`, `decision-makers/`).

### Agent Roles

| Agent | Purpose | Parallelism |
|-------|---------|-------------|
| `collateral-analyzer` | Extract GTM content from sales PDFs | 1x |
| `gtm-synthesizer` | Combine interview outputs into research brief | 1x |
| `scoring-strategist` | Generate deterministic scoring rubrics | 1x |
| `company-researcher` | Web search for qualifying companies | 5x parallel |
| `company-synthesizer` | Dedupe and rank top 10 companies | 1x |
| `dm-researcher` | Find decision makers at target companies | 5x parallel |
| `dm-compiler` | Compile and priority-rank all contacts | 1x |
| `outreach-composer` | Generate tier-matched emails via `/executive-outreach` skill | 1x |

### Key Design Constraints

- The orchestrator must keep responses to 2-3 sentences between phases, ask one question per message during the interview, and use plain text (no markdown) for interview questions.
- Researcher agents (company-researcher, dm-researcher) must make 5-10 WebSearch calls each and cite all sources with URLs.
- Scoring uses a deterministic formula: `(Tier × Multiplier) + (Role Points) + (Activity Bonuses)` — defined by the scoring-strategist and applied by synthesizers.
- Buyer tier distribution targets 60% business/economic, 25% bridge/champion, 15% technical.
- Outreach emails must be under 120 words, tier-matched in tone, and reference company-specific evidence.
- Collateral analysis marks sections as `[Clear]`, `[Inferred]`, or `[Gap]` — the interview adapts by skipping `[Clear]` topics.

## Making Changes

When modifying agent prompts, keep these constraints in mind:
- Each agent's YAML frontmatter defines its `allowed_tools` — only tools listed there are available to that agent.
- Agent input/output file paths are hardcoded in both the agent definition and the orchestrator. If you change one, update the other.
- The orchestrator references agents by their filename (minus `.md`). Renaming an agent file requires updating `commands/lead-genius.md`.
- The `outreach-composer` agent invokes the `/executive-outreach` skill, which lives at `skills/executive-outreach/SKILL.md`. The skill's reference examples are in `skills/executive-outreach/references/examples.md`.

## Optional User Inputs

- `senders/{name}.md` — Sender professional bio for credibility bridges in outreach emails.
- `collateral/*.pdf` — Sales materials analyzed in Phase 1 to pre-fill interview answers.
