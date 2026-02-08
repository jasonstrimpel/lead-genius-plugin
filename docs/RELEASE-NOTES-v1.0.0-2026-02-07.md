# Release Notes

## v1.0.0 (2026-02-07)

Initial release of the Lead Genius plugin for Claude Code and Claude Cowork.

### Features

- **Single command pipeline**: Run `/lead-genius` to execute the full lead generation workflow from interview to personalized outreach
- **Collateral analysis**: Automatically extract GTM content from sales PDFs with confidence-level tagging
- **Adaptive interview**: ~25 questions across 7 GTM areas, with intelligent skip logic when collateral provides clear answers
- **Parallel company research**: 5 researcher agents search simultaneously for qualified prospects
- **Parallel decision maker research**: 5 researcher agents identify accessible business buyers across target companies
- **Deterministic scoring**: Company confidence scores (1-5) and decision maker priority rankings based on offering-specific rubrics
- **Personalized outreach**: Tier-matched cold emails under 120 words, each referencing company-specific evidence and demand signals
- **Sender credential integration**: Optional sender bio for authentic credibility bridges in outreach emails

### Agents

| Agent | Role |
|-------|------|
| collateral-analyzer | Extract GTM content from sales PDFs |
| gtm-synthesizer | Combine interview and collateral into research brief |
| scoring-strategist | Generate deterministic scoring rubrics |
| company-researcher | Web research for qualified companies (5x parallel) |
| company-synthesizer | Deduplicate and rank top 10 companies |
| dm-researcher | Identify decision makers at target companies (5x parallel) |
| dm-compiler | Compile and priority-rank all decision makers |
| outreach-composer | Generate personalized outreach emails |

### Skills

- **executive-outreach**: Tier-calibrated email generation with 9-component structure, style rules, and anti-pattern detection

### Output

Full pipeline produces ~20 files across 7 directories in `./{slug}/`, including:
- Research brief and scoring rubrics
- Top 10 qualified companies with evidence
- Priority-ranked decision makers with contact details
- Personalized outreach emails ready for review and sending