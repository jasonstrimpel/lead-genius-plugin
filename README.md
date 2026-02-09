# Lead Genius Plugin

AI-powered lead generation pipeline for Claude Code and Claude Cowork. Find qualified companies, identify decision makers, generate personalized outreach emails, and produce marketing content (blog, LinkedIn posts, executive case study, PPTX sales decks) through a single conversational command.

## Prerequisites

- **Claude Code** (terminal) or **Claude Cowork** (desktop) installed and running
- **Anthropic API key** configured in your environment (required for web search during research phases)
- **Claude Opus model** recommended for best results (set automatically by the command)

### Installing Claude Code

**macOS (Homebrew):**
```bash
brew install claude-code
```

**macOS / Linux (npm):**
```bash
npm install -g @anthropic-ai/claude-code
```

**Windows (npm):**
```powershell
npm install -g @anthropic-ai/claude-code
```

**Windows (winget):**
```powershell
winget install Anthropic.ClaudeCode
```

After installing, run `claude` in your terminal and complete the authentication flow to set up your API key.

### Installing Claude Cowork (Desktop)

Download from [claude.ai/download](https://claude.ai/download) for macOS or Windows. Claude Cowork includes built-in plugin support.

## Plugin Installation

Open Claude Code or Claude Cowork and run:

```
/plugins marketplace add jasonstrimpel/lead-genius-plugin
/plugins install lead-genius-plugin
```

### Local / Development Install

Clone the repository and install from your local copy:

**macOS / Linux:**
```bash
git clone https://github.com/jasonstrimpel/lead-genius-plugin.git ~/lead-genius-plugin
```

**Windows (PowerShell):**
```powershell
git clone https://github.com/jasonstrimpel/lead-genius-plugin.git $HOME\lead-genius-plugin
```

Then in Claude Code or Claude Cowork:
```
/plugins install ~/lead-genius-plugin
```

## Updating

If you already have the plugin installed, update to the latest version in Claude Code or Claude Cowork:

```
/plugins update lead-genius-plugin
```

For local/development installs, pull the latest changes and reinstall:

**macOS / Linux:**
```bash
cd ~/lead-genius-plugin && git pull
```

**Windows (PowerShell):**
```powershell
cd $HOME\lead-genius-plugin; git pull
```

Then in Claude Code or Claude Cowork:
```
/plugins install ~/lead-genius-plugin
```

## Quick Start

1. Open a terminal (or Claude Cowork) and navigate to a project directory where you want your lead generation output
2. Run the command:
   ```
   /lead-genius
   ```
3. Answer the interview questions about your offering (~10-15 minutes)
4. Wait while agent teams research companies and decision makers
5. Review your results in `./{slug}/`

## Setup (Optional)

These optional steps improve the quality of your results. You can skip them and the pipeline will still work.

### Sender Profile

A sender profile lets the plugin personalize outreach emails with your credentials and experience. Create a `senders/` directory in your project and add a markdown file with your professional bio.

**macOS / Linux:**
```bash
mkdir -p senders
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Path senders -Force
```

Create `senders/your-name.md` using any text editor:

```markdown
{Your Name} is a {Title} at {Company} where they {primary responsibility}.
Previously at {Previous Company} from {dates}, they {key accomplishment with
specific metrics}. They {another accomplishment with quantified outcomes}.

Key credentials: {certifications, publications, speaking engagements}

Notable results:
- {Quantified achievement 1}
- {Quantified achievement 2}
- {Quantified achievement 3}
```

The more specific and quantified your bio, the better the credibility bridges in your outreach emails.

### Sales Collateral

If you have sales materials (pitch decks, case studies, whitepapers), the plugin can analyze them before the interview. This pre-fills answers and skips questions it already has answers to.

**macOS / Linux:**
```bash
mkdir -p collateral
cp ~/Documents/pitch-deck.pdf ./collateral/
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Path collateral -Force
Copy-Item "$HOME\Documents\pitch-deck.pdf" -Destination collateral\
```

## How It Works

The pipeline runs 12 phases using teams of specialized AI agents:

| Phase | What Happens | Output |
|-------|-------------|--------|
| 0 | Setup: detect sender profile and collateral | - |
| 1 | Analyze sales collateral PDFs (if provided) | `collateral-analysis.md` |
| 2 | Interview you about offering and GTM strategy | `offering.md`, `gtm-strategy.md` |
| 3 | Synthesize interview into research brief | `research-brief.md` |
| 4 | Generate scoring rubrics | `scoring-rubrics.md` |
| 5 | **5 agents in parallel** search for companies | `companies-01..05.md` |
| 6 | Deduplicate and rank top 10 companies | `qualified-companies.md` |
| 7 | **5 agents in parallel** research decision makers | `dm-01..05.md` |
| 8 | Compile and priority-rank all decision makers | `decision-makers.md` |
| 9 | Generate personalized outreach for each DM | `outreach.md` |
| 10 | Generate marketing content (blog, LinkedIn, case study) | `blog.md`, `linkedin-posts.md`, `case-study.md` |
| 11 | Generate deck scripts and PPTX sales decks | `deck-script.md`, `*.pptx` |

### Parallel Agent Teams

Phases 5 and 7 use parallel agents for speed. Phase 5 spawns 5 company researchers. Phase 7 spawns 5 DM researchers. All other phases run sequentially, each spawning a single agent.

## Output Structure

All output goes to `./{slug}/`:

```
{slug}/
├── collateral/
│   └── collateral-analysis.md           # Extracted GTM content from PDFs
├── interview/
│   ├── offering.md                      # Offering definition
│   └── gtm-strategy.md                 # GTM strategy and buyer personas
├── go-to-market/
│   ├── research-brief.md               # Combined research brief
│   └── scoring-rubrics.md              # Scoring criteria
├── company-research/
│   ├── companies-01.md                  # Raw research (5 files)
│   ├── companies-02.md
│   ├── companies-03.md
│   ├── companies-04.md
│   └── companies-05.md
├── companies/
│   └── qualified-companies.md           # Top 10 ranked companies
├── decision-maker-research/
│   ├── dm-01.md                         # Raw research (5 files)
│   ├── dm-02.md
│   ├── dm-03.md
│   ├── dm-04.md
│   └── dm-05.md
├── decision-makers/
│   └── decision-makers.md               # Priority-ranked contacts
├── outreach.md                          # Personalized emails
└── marketing/                           # Marketing content (Branch A)
    ├── blog.md                          # Thought leadership blog post
    ├── linkedin-posts.md                # 5-7 LinkedIn posts
    ├── case-study.md                    # Executive case study
    ├── deck-script.md                   # General deck script (content + visual direction)
    ├── deck-presentation.pptx           # 8-slide sales deck (live version)
    ├── deck-reading.pptx                # 8-slide sales deck (leave-behind)
    └── prospect-specific/               # Per-company 2-slide decks
        ├── {company}-deck-script.md     # Prospect deck script
        ├── {company}-presentation.pptx
        └── {company}-reading.pptx
```

### Key Files to Review

- **`decision-makers.md`** - Your ranked list of contacts with titles, companies, tiers, and email addresses
- **`outreach.md`** - Ready-to-send personalized emails for each contact
- **`qualified-companies.md`** - The top 10 companies with evidence and fit rationale
- **`marketing/blog.md`** - Thought leadership blog post following a 7-part narrative arc
- **`marketing/case-study.md`** - Publication-ready executive case study (Andersen Consulting format)
- **`marketing/deck-script.md`** - Reviewable deck script with content and visual direction per slide
- **`marketing/deck-presentation.pptx`** - 8-slide first-call sales deck for live presentation
- **`marketing/prospect-specific/`** - Customized 2-slide decks for each qualified company

## The Interview

The interview phase asks ~25 questions across 7 areas:

1. **Market Definition** - Problem, industries, firmographics, buying triggers
2. **Value Proposition** - Outcomes, alternatives, differentiation, use cases
3. **Pricing & Packaging** - Model, tiers, competitive positioning
4. **Sales Motion** - Cycle length, qualification, proof-of-value, team
5. **Marketing Engine** - Channels, content, demand strategy
6. **Buyer Personas** - Economic buyer, champion, technical evaluator
7. **Disqualifiers** - Who NOT to target

If you provide sales collateral PDFs, the interview adapts: it skips questions already answered in the collateral and digs deeper on gaps.

## Outreach Email Quality

Each outreach email is:
- Under 120 words
- Personalized with company-specific evidence and demand signals
- Tier-matched (C-suite gets business outcomes, technical contacts get implementation detail)
- Structured with a clear call-to-action (15-min call, not vague "connect")
- Enhanced with sender credentials when a bio is provided
- Flagged with email confidence indicators (verified emails stay clean; pattern-matched and unverified emails get a warning banner and inline tag; contacts with no email get lookup guidance)

## Marketing Content

The pipeline automatically generates marketing materials alongside outreach emails:

### Blog Post
A 1500-2500 word thought leadership piece following the same narrative arc as the sales deck: industry shift, winners/losers, quantified pain, cost of inaction, promised land, bridge to solution, and next steps. Written for the ICP audience, not as a sales pitch.

### LinkedIn Posts
5-7 posts derived from the blog, each pulling a different angle (industry shift, shocking statistic, contrarian take, etc.). Ready to copy-paste into LinkedIn.

### Executive Case Study
Publication-ready case study in the Andersen Consulting executive format. Anonymous client, mandatory heading structure, strict cadence requirements. Includes verbatim firm description and contact section.

### Sales Decks (PPTX)
Real PowerPoint files generated using the Anthropic `pptx` skill (PptxGenJS). The deck-builder uses a two-pass workflow: first it writes a markdown "deck script" capturing content and visual direction per slide, then renders PPTX from the script. Deck scripts are reviewable intermediates — you can edit narrative framing before re-rendering.

Two versions of every deck:
- **Presentation version** (≤20 words/slide) - Visual-heavy, for live delivery
- **Reading version** (≤60 words/slide) - Expanded narrative, for async sharing

The general deck follows an 8-slide first-call framework backed by sales methodology research (Raskin, Challenger Sale, Corporate Visions, Gong). Prospect-specific 2-slide decks are generated for each qualified company.

## Troubleshooting

**No collateral PDFs?** That's fine. Skip Phase 1 entirely. The interview covers everything the pipeline needs.

**No sender profile?** Outreach emails will still work but won't include credibility bridges from your bio.

**Interview feels long?** If you have detailed collateral, many questions get skipped automatically.

**Want to re-run just outreach?** Currently the pipeline runs end-to-end. If your research files exist from a previous run, you can manually invoke the outreach-composer agent.

**Plugin not showing up?** Make sure you restarted Claude Code or Claude Cowork after installing. Run `/help` to see if `/lead-genius` appears in your available commands.

**WebSearch errors?** Ensure your Anthropic API key is configured. The research agents need web access to find companies and decision makers.

## Changelog

See [docs/RELEASE-NOTES-v1.4.0-2026-02-08.md](docs/RELEASE-NOTES-v1.4.0-2026-02-08.md) for the latest release notes.

## License

MIT