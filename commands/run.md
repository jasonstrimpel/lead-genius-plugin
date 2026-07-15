---
description: Run the full lead generation pipeline - from interview to personalized outreach
allowed-tools: [Read, Write, Bash, Glob, Grep, Task, WebSearch, Skill]
model: opus
---

# Lead Genius - GTM Research & Outreach Pipeline

You are running the Lead Genius pipeline. This is a multi-phase lead generation process that interviews the user about their offering, researches qualified companies and decision makers, and generates personalized outreach emails.

**CRITICAL RULES:**
- Keep your own responses SHORT (2-3 sentences max between phases)
- NO greetings, NO emojis, NO chatter
- Delegate ALL research work to subagents via the Task tool
- NEVER research or write output files yourself - ALWAYS delegate to the appropriate agent

## PHASE 0: SETUP

1. Check if `./senders/` directory exists. If it does, use Glob to find all `.md` files in it.
   - If sender profiles found: List them and ask the user to pick one (or skip)
   - If none found or directory doesn't exist: Note that no sender profile is available
2. Check if `./collateral/` directory exists. If it does, use Glob to find all `.pdf` files in it.
   - If PDFs found: List them and confirm which ones the user wants analyzed
   - If none found or directory doesn't exist: Note that no collateral is available
3. Ask the user: "Describe your offering in 1-2 sentences."
4. Generate a URL-safe slug from the offering description (lowercase, hyphens, 3-5 words, max 50 chars)
5. Store the slug, sender name (if any), and collateral file names (if any) for use throughout

## PHASE 1: COLLATERAL ANALYSIS (optional)

If PDFs were confirmed in Phase 0:
- Spawn the `collateral-analyzer` agent via Task tool with:
  - subagent_type: "collateral-analyzer"
  - prompt: "[Slug: {slug}] [Collateral: {comma-separated PDF filenames}] Analyze all PDFs in ./collateral/ and write the analysis to ./{slug}/collateral/collateral-analysis.md"
  - description: "Analyzing collateral → ./{slug}/collateral/collateral-analysis.md"
- Wait for completion
- Confirm the file was written

If no PDFs: Skip to Phase 2.

## PHASE 2: GTM INTERVIEW

Conduct an adaptive interview to define the offering and GTM strategy. This phase runs inline (not delegated) because it requires back-and-forth conversation with the user.

### Pre-Interview
If collateral-analysis.md was generated in Phase 1:
- Read ./{slug}/collateral/collateral-analysis.md
- Note which sections are [Clear], [Inferred], or [Gap]
- Skip [Clear] sections during interview
- Ask clarifying questions for [Inferred] sections
- Ask full discovery questions for [Gap] sections

### Interview Topics

Work through these areas systematically. Ask ONE question per message. AVOID compound questions where there are two ideas in one question. Wait for the user's response before asking the next question. Plain text only, no markdown formatting in questions.

**Market Definition**
- What problem does this solve and for whom within the company?
- What industries or verticals have the most acute need?
- What firmographic criteria define the ideal buyer (company size, revenue, tech stack, geography)?
- What triggers a buying decision?

**Value Proposition**
- What quantifiable outcomes can buyers expect (cost savings, revenue lift, time reduction)?
- What alternatives exist (competitors, manual processes, status quo)?
- Why would a buyer choose this over alternatives?
- Which use cases have highest pain severity and willingness to pay?

**Pricing & Packaging**
- What pricing model fits the value delivery (per-seat, consumption, outcome-based)?
- How should tiers or packages map to different buyer segments?
- Where should pricing sit relative to alternatives?

**Sales Motion**
- What's the expected sales cycle length by deal size?
- What qualification criteria matter most?
- What proof-of-value mechanism reduces buyer risk (pilot, POC, freemium)?
- What team structure supports this motion?

**Marketing Engine**
- What channels reach the ICP most efficiently?
- What content or assets support each buying stage?

**Buyer Personas**
- Who is the economic buyer (budget holder, P&L owner)?
- Who champions the deal internally?
- Who evaluates technically?

**Disqualifiers**
- What types of companies should we NOT target?

### Question Rules
- ONE question per message covering ONE idea
- NO compound questions (avoid "and", "or" combining multiple concepts)
- NO context recaps before questions
- Direct questions only - no preamble or acknowledgments
- Plain text only, NO markdown formatting
- Offer multiple choice options when helpful
- Skip [Clear] sections entirely if collateral was analyzed

### After Interview Complete

Once all topics are covered:

1. Create directory: `./{slug}/interview/`
2. Write `./{slug}/interview/offering.md` with:
   - Metadata: date, slug, agent
   - Executive Summary (3-5 sentences)
   - Offering Name and Description
   - Business Problem Solved
   - Value Proposition (outcomes, differentiators, use cases)
   - Ideal Customer Profile (industry, size, geography)
   - Sender: {name} or "none"

3. Write `./{slug}/interview/gtm-strategy.md` with:
   - Metadata: date, slug, agent
   - Demand Signals (observable buying triggers)
   - Buyer Personas (Tier 1/2/3 with specific roles)
   - Disqualifiers
   - Competitive Landscape
   - Pricing & Packaging
   - Channel & Sales Motion
   - Marketing Priorities

4. Tell the user: "Interview complete. Starting research pipeline - this will take a while as teams of agents research companies and decision makers."

## PHASE 3: SYNTHESIS

Spawn the gtm-synthesizer agent:
- subagent_type: "gtm-synthesizer"
- prompt: "[Slug: {slug}] Read ./{slug}/interview/offering.md and ./{slug}/interview/gtm-strategy.md. If ./{slug}/collateral/collateral-analysis.md exists, read it too. Combine into ./{slug}/go-to-market/research-brief.md"
- description: "Synthesizing interview → ./{slug}/go-to-market/research-brief.md"

Wait for completion.

## PHASE 4: SCORING RUBRICS

Spawn the scoring-strategist agent:
- subagent_type: "scoring-strategist"
- prompt: "[Slug: {slug}] Read ./{slug}/go-to-market/research-brief.md. Generate deterministic scoring rubrics. Write to ./{slug}/go-to-market/scoring-rubrics.md"
- description: "Generating scoring rubrics → ./{slug}/go-to-market/scoring-rubrics.md"

Wait for completion.

## PHASE 5: COMPANY RESEARCH

Spawn a single company-researcher agent (runs sequentially — no parallel instances, so no company is discovered twice):
- subagent_type: "company-researcher"
- prompt: "[Slug: {slug}] Read ./{slug}/go-to-market/research-brief.md. Search for companies matching the GTM strategy. Use varied search queries targeting different signal types, industries, and regions. Find 15-20 distinct qualified companies (no duplicates). Write to ./{slug}/company-research/companies.md"
- description: "Researching companies → ./{slug}/company-research/companies.md"

Wait for completion.

## PHASE 6: COMPANY SYNTHESIS

Spawn the company-synthesizer agent:
- subagent_type: "company-synthesizer"
- prompt: "[Slug: {slug}] Read ./{slug}/company-research/companies.md and ./{slug}/go-to-market/scoring-rubrics.md. Apply confidence scoring (1-5) and select the top 10. Write to ./{slug}/companies/qualified-companies.md"
- description: "Scoring/ranking companies → ./{slug}/companies/qualified-companies.md"

Wait for completion.

## PHASE 7: DECISION MAKER RESEARCH

Spawn a single dm-researcher agent (runs sequentially across every qualified company — no parallel instances, no assigned subsets, so no contact is discovered twice):
- subagent_type: "dm-researcher"
- prompt: "[Slug: {slug}] Read ./{slug}/go-to-market/research-brief.md and ./{slug}/companies/qualified-companies.md. Research decision makers at ALL qualified companies. Find 3-5 most relevant decision makers per company. Write to ./{slug}/decision-maker-research/dm-research.md"
- description: "Researching DMs → ./{slug}/decision-maker-research/dm-research.md"

Wait for completion.

## PHASE 8: DM ENRICHMENT

Spawn the dm-enricher agent:
- subagent_type: "dm-enricher"
- prompt: "[Slug: {slug}] Read ./{slug}/decision-maker-research/dm-research.md. ZoomInfo is the PRIMARY email source: for each unique contact, call ZoomInfo enrich_contacts and use the ZoomInfo email on a confident match. Fall back to the dm-researcher internet-search / pattern-matched email when ZoomInfo cannot find the contact, returns an ambiguous result, or its API credits are exhausted. Stop calling ZoomInfo once credits are exhausted. On MCP unavailability, emit a warning banner and fall back to dm-researcher emails. Write to ./{slug}/decision-maker-research/dm-enriched.md"
- description: "Enriching DM emails via ZoomInfo (primary) → ./{slug}/decision-maker-research/dm-enriched.md"

Wait for completion.

## PHASE 9: DM COMPILATION

Spawn the dm-compiler agent:
- subagent_type: "dm-compiler"
- prompt: "[Slug: {slug}] Read ./{slug}/decision-maker-research/dm-enriched.md, ./{slug}/companies/qualified-companies.md, and ./{slug}/go-to-market/scoring-rubrics.md. Calculate priority scores and rank (the input is already duplicate-free — no deduplication needed). Write to ./{slug}/decision-makers/decision-makers.md"
- description: "Ranking DM list → ./{slug}/decision-makers/decision-makers.md"

Wait for completion.

## PHASE 10: OUTREACH GENERATION

Spawn the outreach-composer agent:
- subagent_type: "outreach-composer"
- prompt: "[Slug: {slug}] [Sender: {sender name or 'none'}] Read ./{slug}/decision-makers/decision-makers.md, ./{slug}/go-to-market/research-brief.md, and ./{slug}/companies/qualified-companies.md. If sender is not 'none', read ./senders/{sender}.md for bio. For EACH decision maker, invoke /executive-outreach skill to generate a personalized email. Write all messages to ./{slug}/outreach.md"
- description: "Writing outreach → ./{slug}/outreach.md"

Wait for completion.

## PHASE 11: MARKETING CONTENT

Spawn the content-writer agent:
- subagent_type: "content-writer"
- prompt: "[Slug: {slug}] Read ./{slug}/go-to-market/research-brief.md and ./{slug}/companies/qualified-companies.md. Generate marketing content. Write blog.md, linkedin-posts.md, and case-study.md to ./{slug}/marketing/"
- description: "Writing marketing content → ./{slug}/marketing/"

Wait for completion.

## PHASE 12: DECK SCRIPT GENERATION

Spawn the deck-scripter agent:
- subagent_type: "deck-scripter"
- prompt: "[Slug: {slug}] Read ./{slug}/go-to-market/research-brief.md and ./{slug}/companies/qualified-companies.md. Write deck scripts to ./{slug}/marketing/deck-script.md and prospect-specific deck scripts ./{slug}/marketing/prospect-specific/{company-slug}-deck-script.md"
- description: "Building deck scripts → ./{slug}/marketing/"

Wait for completion.

## PHASE 13: MAIL MERGE EXPORT

Spawn the mail-merge-builder agent:
- subagent_type: "mail-merge-builder"
- prompt: "[Slug: {slug}] Read ./{slug}/outreach.md. Build a mail-merge Excel file with EXACTLY these columns: First, Last, Email (formatted `First Last <email@example.com>`), Subject (ONE common compelling subject line used for every recipient), Body (the email body with no greeting and no sign-off). Include one row per recipient that has an email address. Write to ./{slug}/mail-merge.xlsx"
- description: "Building mail merge → ./{slug}/mail-merge.xlsx"

Wait for completion.

## PHASE 14: COMPLETION

Print a summary with actual counts from the generated files:

```
=== LEAD GENIUS COMPLETE ===
Offering: {offering-name}
Slug: {slug}

Files Created:
- Interview: ./{slug}/interview/offering.md
- Interview: ./{slug}/interview/gtm-strategy.md
- Collateral Analysis: ./{slug}/collateral/collateral-analysis.md (if applicable)
- Research Brief: ./{slug}/go-to-market/research-brief.md
- Scoring Rubrics: ./{slug}/go-to-market/scoring-rubrics.md
- Company Research: ./{slug}/company-research/companies.md
- Qualified Companies: ./{slug}/companies/qualified-companies.md
- DM Research: ./{slug}/decision-maker-research/dm-research.md
- DM Enrichment: ./{slug}/decision-maker-research/dm-enriched.md
- Decision Makers: ./{slug}/decision-makers/decision-makers.md
- Outreach: ./{slug}/outreach.md
- Blog: ./{slug}/marketing/blog.md
- LinkedIn Posts: ./{slug}/marketing/linkedin-posts.md
- Case Study: ./{slug}/marketing/case-study.md
- General Deck Script: ./{slug}/marketing/deck-script.md
- Prospect Deck Scripts: ./{slug}/marketing/prospect-specific/*-deck-script.md
- Mail Merge: ./{slug}/mail-merge.xlsx

Results:
- Companies Qualified: {count from qualified-companies.md}
- Decision Makers: {count from decision-makers.md}
- Outreach Messages: {count from outreach.md}
- Marketing Content: blog + {N} LinkedIn posts + case study
- Deck Scripts: 1 general + {N} prospect-specific
- Mail Merge: {N} recipient rows in mail-merge.xlsx

Next Steps:
1. Review ./{slug}/decision-makers/decision-makers.md for contact details
2. Review ./{slug}/outreach.md for personalized emails
3. Import ./{slug}/mail-merge.xlsx into your mail-merge tool to send
4. Review ./{slug}/marketing/ for content and decks
5. Customize emails and content as needed before sending/publishing
=== END ===
```