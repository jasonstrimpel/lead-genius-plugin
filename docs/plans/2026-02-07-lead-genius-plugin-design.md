# Lead Genius Plugin Design

## Overview

Repackage the Lead Genius multi-agent GTM research pipeline as a Claude Code plugin installable via `/plugins`. The goal is to allow non-coders to run the full lead generation process from Claude Code or Claude Cowork using a single `/lead-genius` slash command.

## Decisions

- **Single command**: `/lead-genius` walks users through the entire flow conversationally
- **Project-relative I/O**: `./collateral/` and `./senders/` for inputs, `./{slug}/` for outputs
- **File-based coordination**: Parallel researchers work independently; synthesizer agents deduplicate via shared filesystem (no inter-agent communication)
- **All agents visible**: 8 agent `.md` files in `agents/` directory, orchestrated by the slash command
- **Conversational setup**: Command checks for sender profiles and collateral, asks user to pick interactively

## Plugin Structure

```
plugin/
├── .claude-plugin/
│   └── plugin.json                    # Plugin manifest
├── commands/
│   └── lead-genius.md                 # Main slash command (orchestrator)
├── agents/
│   ├── collateral-analyzer.md         # Phase 1: PDF analysis
│   ├── gtm-synthesizer.md            # Phase 3: Combine interview data
│   ├── scoring-strategist.md         # Phase 4: Generate scoring rubrics
│   ├── company-researcher.md         # Phase 5: Web research (x5 parallel)
│   ├── company-synthesizer.md        # Phase 6: Dedupe & rank companies
│   ├── dm-researcher.md              # Phase 7: Find decision makers (x5 parallel)
│   ├── dm-compiler.md                # Phase 8: Compile & rank DMs
│   └── outreach-composer.md          # Phase 9: Generate emails
├── skills/
│   └── executive-outreach/
│       ├── SKILL.md                   # Email generation spec
│       └── references/
│           └── examples.md            # Tier-calibrated examples
├── README.md                          # Installation & usage guide
└── LICENSE
```

## Slash Command Flow (`/lead-genius`)

### Startup
1. Check `./senders/` for `.md` files - list and ask user to pick one (or skip)
2. Check `./collateral/` for PDF files - list and confirm analysis
3. Ask user to describe their offering in 1-2 sentences
4. Generate URL-safe slug from offering description

### Phase 1: Collateral Analysis (optional)
- If PDFs exist, spawn `collateral-analyzer` agent via Task
- Output: `./{slug}/collateral/collateral-analysis.md`
- Skip if no PDFs

### Phase 2: GTM Interview (inline in command)
- The command prompt contains the GTM strategist interview logic
- Asks ~25 adaptive questions (one per turn, plain text)
- Skips [Clear] sections if collateral was analyzed
- Output: `./{slug}/interview/offering.md` and `gtm-strategy.md`

### Phase 3-9: Research Pipeline (delegated to agents)
Sequential/parallel orchestration via Task tool:

| Phase | Agent | Parallelism | Output |
|-------|-------|-------------|--------|
| 3 | gtm-synthesizer | 1x | `go-to-market/research-brief.md` |
| 4 | scoring-strategist | 1x | `go-to-market/scoring-rubrics.md` |
| 5 | company-researcher | 5x parallel | `company-research/companies-{01-05}.md` |
| 6 | company-synthesizer | 1x | `companies/qualified-companies.md` |
| 7 | dm-researcher | 5x parallel | `decision-maker-research/dm-{01-05}.md` |
| 8 | dm-compiler | 1x | `decision-makers/decision-makers.md` |
| 9 | outreach-composer | 1x | `outreach.md` |

### Completion
Print summary with offering name, slug, all file paths, and counts (companies, DMs, emails).

## Agent Definitions

Each agent `.md` file has YAML frontmatter + system prompt:

```yaml
---
name: agent-identifier
description: |
  Use this agent when [condition].
  <example>Context: ... user: "..." assistant: "..."</example>
model: inherit
tools: [Read, Write, WebSearch]
---

System prompt content here...
```

### Tool Allocations

| Agent | Tools |
|-------|-------|
| collateral-analyzer | Read, Write, Bash |
| gtm-synthesizer | Read, Write, Bash |
| scoring-strategist | Read, Write, Bash |
| company-researcher | Read, Write, WebSearch |
| company-synthesizer | Read, Write, Glob, Bash |
| dm-researcher | Read, Write, WebSearch |
| dm-compiler | Read, Write, Glob, Bash |
| outreach-composer | Read, Write, Glob, Skill |

### Path Changes

All file paths updated from `lead_genius/files/{slug}/` to `./{slug}/`.

## Executive Outreach Skill

Bundled verbatim from `.claude/skills/executive-outreach/`. No modifications needed. The `outreach-composer` agent invokes it via the Skill tool for each decision maker email.

## Installation

```
/plugins marketplace add youruser/lead-genius-plugin
/plugins install lead-genius-plugin
```

Or clone locally and point Claude Code at it.

## Usage

```
/lead-genius
```

Prerequisites:
- (Optional) `./senders/your-name.md` with bio/credentials
- (Optional) `./collateral/*.pdf` with sales materials
- Anthropic API key configured
