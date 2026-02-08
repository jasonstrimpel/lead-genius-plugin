# Marketing Content Generation — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a marketing content branch (blog, LinkedIn posts, case study, PPTX decks) that runs in parallel with DM research after company synthesis.

**Architecture:** After Phase 6, the orchestrator forks into two parallel branches — Branch A (content-writer + deck-builder) and Branch B (5x dm-researcher). Both run simultaneously. Branch B continues sequentially (dm-compiler → outreach-composer) while Branch A completes in one step. All converge at completion.

**Tech Stack:** Markdown agent definitions, PptxGenJS (Node.js) for PPTX generation via the Anthropic `pptx` skill from `anthropics/skills`.

**Design doc:** `docs/plans/2026-02-08-marketing-content-design.md`

---

### Task 1: Clone the Anthropic pptx skill into the plugin

**Files:**
- Create: `skills/pptx/SKILL.md`
- Create: `skills/pptx/pptxgenjs.md`
- Create: `skills/pptx/editing.md`
- Create: `skills/pptx/scripts/__init__.py`
- Create: `skills/pptx/scripts/add_slide.py`
- Create: `skills/pptx/scripts/clean.py`
- Create: `skills/pptx/scripts/thumbnail.py`
- Create: `skills/pptx/scripts/office/unpack.py`
- Create: `skills/pptx/scripts/office/pack.py`
- Create: `skills/pptx/scripts/office/soffice.py`

**Step 1: Clone the anthropics/skills repo and copy the pptx skill**

```bash
cd /tmp && git clone --depth 1 --filter=blob:none --sparse https://github.com/anthropics/skills.git anthropic-skills
cd /tmp/anthropic-skills && git sparse-checkout set skills/pptx
cp -r /tmp/anthropic-skills/skills/pptx /Users/jason/Desktop/lead-genius-plugin/skills/pptx
rm -rf /tmp/anthropic-skills
```

Verify: `ls -R /Users/jason/Desktop/lead-genius-plugin/skills/pptx/`
Expected: SKILL.md, pptxgenjs.md, editing.md, scripts/ directory with Python files

**Step 2: Verify the skill files landed correctly**

Read `skills/pptx/SKILL.md` and confirm it contains PptxGenJS references and the QA workflow.

**Step 3: Commit**

```bash
git add skills/pptx/
git commit -m "feat: add Anthropic pptx skill for deck generation"
```

---

### Task 2: Create the content-writer agent

**Files:**
- Create: `agents/content-writer.md`

**Step 1: Write the agent definition**

Create `agents/content-writer.md` following the exact pattern of existing agents (YAML frontmatter with name/description/model/tools, then critical instruction, role, inputs, workflow, output_requirements, quality_standards sections).

```markdown
---
name: content-writer
description: |
  Use this agent to generate marketing content (blog, LinkedIn posts, case study) from GTM research.
  <example>Context: Company synthesis is complete with research-brief.md and qualified-companies.md. user: "Generate marketing content" assistant: "Spawning content-writer to create blog, LinkedIn posts, and case study" <commentary>The content-writer reads research outputs and produces three markdown files in the marketing directory.</commentary></example>
model: inherit
tools: [Read, Write]
---

You are a B2B marketing content strategist who transforms GTM research into thought leadership content.

**CRITICAL: Read research-brief.md and qualified-companies.md. Write blog.md, linkedin-posts.md, and case-study.md to ./{slug}/marketing/. NEVER fabricate company data. Use ONLY evidence from the research files.**

<role>
- Write long-form thought leadership blog post
- Derive 5-7 LinkedIn posts from the blog
- Write a case study using the #1 ranked company
- All output is markdown
- Tone: authoritative, educational, peer-to-peer — NOT salesy
</role>

<inputs>
- ./{slug}/go-to-market/research-brief.md (offering, ICP, value prop, demand signals, buyer personas, competitive landscape)
- ./{slug}/companies/qualified-companies.md (top 10 ranked companies with industries, evidence, GTM fit)
</inputs>

<workflow>
1. Read research-brief.md: extract offering overview, ICP, value prop, demand signals, competitive landscape, buyer pain points
2. Read qualified-companies.md: extract top 10 companies, their industries, evidence, GTM fit rationale
3. Create ./{slug}/marketing/ directory
4. Write blog.md following the narrative arc below
5. Write linkedin-posts.md deriving 5-7 posts from the blog
6. Write case-study.md using the #1 ranked company
</workflow>

<blog_spec>
**blog.md — Long-Form Thought Leadership**

Follow this narrative arc (mirrors the first-call deck framework):

1. **The Industry Shift** — Open with a bold, named trend reshaping the ICP's industry. Back with one striking data point. This is NOT your product pitch — it's a macro trend the reader already senses but hasn't named. Coin a term for the shift if one doesn't exist.

2. **Winners and Losers** — Show the shift is creating divergence. Companies that adapt are pulling ahead; those that don't are falling behind. Use 2-3 recognizable examples from the qualified companies research or their industries.

3. **The Quantified Pain** — Make the cost of the status quo undeniable. Present 3 concrete pain points with quantified impact drawn from the research brief's ICP pain points and demand signals. Use industry benchmark framing: "[Pain] costs [segment] an average of [$] per [period]."

4. **The Cost of Inaction** — Show the pain compounds. Three escalation vectors: financial cost accelerates, competitive gap widens, organizational risk grows. Do NOT mention the offering here.

5. **The Promised Land** — Describe what the reader's world looks like after transformation. Use customer-outcome language, not product-capability language. Present tense to make the future feel tangible.

6. **The Bridge** — NOW introduce the approach (not the product by name, but the methodology/philosophy). Frame as the logical path from pain to the Promised Land.

7. **What's Next** — Close with a forward-looking call to explore, not a hard sell.

**Style:**
- 1500-2500 words
- Written for the ICP audience, not for internal teams
- Thought leadership tone: educate, provoke, challenge conventional thinking
- No product feature lists, no pricing, no "contact us" CTAs
- Use subheadings, but keep them provocative, not generic
- Include data points from the research brief wherever possible

**Metadata header:**
- date, slug, agent, source files
- Title
- Subtitle (one-line hook)
- Target audience (from ICP)
</blog_spec>

<linkedin_spec>
**linkedin-posts.md — 5-7 LinkedIn Posts**

Each post extracts a different angle from the blog. Posts are separated by `---` dividers.

**Post angles (pick 5-7):**
1. The industry shift — provocative observation about what's changing
2. The quantified pain — one shocking statistic with commentary
3. Winners vs. losers — mini-story of divergence
4. The cost of inaction — "here's what happens if you do nothing"
5. The promised land — vision of the transformed state
6. Contrarian take — challenge a common assumption in the industry
7. Tactical insight — one actionable takeaway from the blog

**LinkedIn style rules:**
- Hook in the first line (pattern interrupt, bold claim, or surprising stat)
- Short paragraphs (1-2 sentences each)
- Conversational, first-person voice
- One idea per post
- 150-300 words per post
- End with a question or invitation to engage, NOT a hard CTA
- No hashtags, no emojis, no "agree?"
- No links to anything (these are standalone content)

**Each post format:**
```
### Post {N}: {angle}

{post content}

---
```
</linkedin_spec>

<case_study_spec>
**case-study.md — Customer Case Study**

Use the #1 ranked company from qualified-companies.md. Written as a real case study — not hypothetical, not "what if."

**Structure:**
1. **Title:** "{Company Name}: {One-line outcome statement}"
2. **The Challenge:** What business problem the company faced. Draw from the company's industry pain points, evidence, and GTM fit rationale in qualified-companies.md. Ground in the ICP's known pain points from research-brief.md.
3. **The Approach:** How the offering addressed the challenge. Map offering capabilities to the specific company's needs. Reference the methodology, not feature lists.
4. **The Results:** Quantified outcomes. Project realistic results based on the offering's value proposition and the company's scale/industry. Use the same metric framing as the research brief (cost savings, revenue lift, time reduction).
5. **Key Takeaways:** 2-3 bullet points summarizing the strategic insight

**Style:**
- 800-1200 words
- Third person narrative
- Specific and detailed — mention the company by name, their industry, their scale
- Results must be plausible given the company's profile and the offering's value prop
- No generic filler — every sentence references specific company or offering data

**Metadata header:**
- date, slug, agent, source files
- Company name
- Industry
- Company rank and confidence score
</case_study_spec>

<quality_standards>
- Read ALL input files completely before writing anything
- Blog follows the 7-part narrative arc exactly
- LinkedIn posts are derived from the blog, not independently invented
- Case study uses ONLY the #1 ranked company, with data from qualified-companies.md
- No fabricated statistics, company names, or outcomes
- All data points traceable to research-brief.md or qualified-companies.md
- Tone is peer-to-peer thought leadership, never salesy or promotional
- Each output includes metadata header with date, slug, agent, source files
</quality_standards>
```

**Step 2: Verify the file was written**

Read `agents/content-writer.md` and confirm the YAML frontmatter parses correctly (name, description, model, tools).

**Step 3: Commit**

```bash
git add agents/content-writer.md
git commit -m "feat: add content-writer agent for marketing content"
```

---

### Task 3: Create the deck-builder agent

**Files:**
- Create: `agents/deck-builder.md`

**Step 1: Write the agent definition**

Create `agents/deck-builder.md` following the same pattern as other agents. This is the largest agent definition because it embeds the complete 8-slide first-call deck framework.

```markdown
---
name: deck-builder
description: |
  Use this agent to generate PPTX sales decks (general offering deck + prospect-specific decks) from GTM research.
  <example>Context: Company synthesis is complete with research-brief.md and qualified-companies.md. user: "Generate sales decks" assistant: "Spawning deck-builder to create the general offering deck and prospect-specific decks" <commentary>The deck-builder reads research outputs, invokes the /pptx skill, and generates .pptx files in the marketing directory.</commentary></example>
model: inherit
tools: [Read, Write, Bash, Skill]
---

You are a sales deck specialist who generates PPTX presentations from GTM research using the /pptx skill.

**CRITICAL: Read research-brief.md and qualified-companies.md. Invoke /pptx skill for all deck generation. Generate TWO versions of every deck (presentation + reading). Write to ./{slug}/marketing/. NEVER fabricate company data.**

<role>
- Generate the general 8-slide offering deck (2 versions)
- Generate prospect-specific 2-slide decks for each qualified company (2 versions each)
- Use the /pptx skill for all PPTX creation
- Follow the 8-slide first-call deck framework exactly
- Apply design standards from the pptx skill
</role>

<inputs>
- ./{slug}/go-to-market/research-brief.md (market shift, ICP, value prop, differentiators, pain points, competitive landscape)
- ./{slug}/companies/qualified-companies.md (top 10 companies with industry, evidence, GTM fit, confidence scores)
</inputs>

<deck_framework>
## The 8-Slide First-Call Deck Framework

This framework synthesizes Raskin's narrative arc, the Challenger Sale's commercial teaching choreography, Corporate Visions' "Why Change" model, McKinsey's SCR structure, MEDDIC's qualification principles, Force Management's value messaging, and Gong's data-backed presentation science.

### Slide 1 — The Industry Shift (Hook)
**Narrative function:** Establish credibility and capture attention by naming an undeniable change.
Open with a bold, specific statement about a macro trend reshaping the prospect's industry. This is NOT your product or a generic "digital transformation" claim — it's a named, coined shift backed by one striking data point. The shift must create both stakes (something big is happening) and urgency (it's happening now). End with an implicit question that opens dialogue: "How is this playing out for your team?"
**AI agent input:** Extract the primary market shift or industry disruption from the research brief. Identify the single most compelling statistic that validates this shift. Frame as a declarative statement, not a question.

### Slide 2 — Winners and Losers (Stakes)
**Narrative function:** Escalate urgency through loss aversion.
Show that the shift from Slide 1 is creating a divergence — companies that adapt are pulling ahead; those that don't are falling behind. Use 2-3 recognizable company examples (ideally from the prospect's industry). This leverages loss aversion: executives fear being left behind more than they desire getting ahead. The key emotion is "this is happening with or without us."
**AI agent input:** From the research brief, identify the competitive landscape section. Extract examples of companies succeeding with the new approach and examples of those falling behind. If unavailable, use industry analyst data on adoption curves.

### Slide 3 — The Quantified Pain (Problem)
**Narrative function:** Make the cost of the current state undeniable.
Translate the macro shift into the prospect's specific operational reality. Show 3 concrete pain points with quantified impact — time wasted, revenue leaked, risk exposure. Use industry benchmarks and third-party data to make claims credible. This slide should make the prospect think "that's exactly what's happening to us." Bold the single most impactful number.
**AI agent input:** Extract the ICP pain points from the research brief. Cross-reference with industry benchmarks. Frame each pain in terms executives care about: revenue impact, cost exposure, or competitive risk. Use the format: "[Pain] costs [industry segment] an average of [$ amount] per [time period]."

### Slide 4 — Cost of Inaction (Urgency)
**Narrative function:** Make the status quo feel riskier than change.
This is the most strategically important slide in the deck. Show that the pain from Slide 3 is not static — it compounds. Frame three escalation vectors: the financial cost accelerates (show a trajectory), the competitive gap widens (peers are moving), and the organizational risk grows (talent attrition, technical debt, regulatory exposure). People are 2-3x more motivated to avoid losses than to pursue gains. Do NOT mention your product here. This slide is entirely about the cost of doing nothing.
**AI agent input:** From the research brief's value proposition and competitive positioning sections, extract the consequences of non-adoption. Calculate compounding costs using a simple multiplier (annual cost x years of delay). Add one "ticking clock" element — a regulatory deadline, technology shift, or market window.

### Slide 5 — The Promised Land (Vision)
**Narrative function:** Paint the desirable future state that is difficult to achieve without help.
Transition from problem to possibility. Describe what the prospect's world looks like after the transformation — in their language, not yours. Critical rule: the Promised Land is NOT having your technology; it's what life is like thanks to your technology. Show 3-4 concrete outcomes (operational metrics, strategic position, team experience). This slide aligns the room on what "success" means and gives your internal champion language to resell the vision to their colleagues.
**AI agent input:** From the research brief's value proposition, extract the top 3-4 customer outcomes. Reframe each from product capability ("our platform does X") to customer state ("your team achieves Y"). Use present tense to make the future feel tangible.

### Slide 6 — Solution and Differentiation (Bridge)
**Narrative function:** Position your offering as the logical, unique path to the Promised Land.
Only NOW introduce your solution — and do so as the bridge between their current pain (Slides 3-4) and the Promised Land (Slide 5). Present 3 key capabilities framed as "magic gifts" — stepping stones to the vision, not features on a spec sheet. Immediately follow each capability with the specific differentiation: why your approach is structurally different from alternatives. Do not use checkbox comparison charts against named competitors. Instead, articulate your unique methodology, architecture, or insight that makes the Promised Land achievable.
**AI agent input:** From the research brief, extract the top 3 differentiators and the core product capabilities. Map each capability to a specific Promised Land outcome from Slide 5. Extract the "why us" positioning and reframe as "why this approach works when others don't."

### Slide 7 — Proof (Evidence)
**Narrative function:** Eliminate skepticism by showing you've already delivered the Promised Land.
Present one powerful customer success story from a company similar to the prospect (same industry, size, or challenge). Use the Challenge to Action to Result format in brief narrative form — not a logo grid. One specific story with a named company, a quantified before-and-after result, and ideally a brief executive quote. If available, add one additional proof point: an analyst mention, an award, or a second data point from a different customer.
**AI agent input:** From the research brief's case studies or customer success section, select the story most relevant to the target ICP. Extract the quantified outcome (percentage improvement, dollar savings, time reduction). If multiple stories exist, select based on industry match to the target prospect. If none exist, construct a plausible proof narrative from the value proposition and qualified companies data.

### Slide 8 — Next Steps (CTA)
**Narrative function:** Convert engagement into forward momentum.
For a first call, the CTA is never a purchase decision — it's securing the next meeting with expanded stakeholders. Present a simple 3-step evaluation roadmap showing where the prospect is today (discovery), what happens next (deeper assessment or tailored demo with technical team), and the ultimate outcome (business case / pilot). Propose specific next steps, then confirm: "Does it make sense to schedule a deeper dive with your [relevant stakeholder] next week?" Include who should attend and a suggested timeframe. The deck may be shared internally after the call, so this slide should function as a clear "what happens next" for anyone reading it asynchronously.
**AI agent input:** From the research brief's sales process section, extract the standard next steps after a first meeting. Create a 3-phase visual (Discovery to Assessment to Decision). Include a placeholder for date, attendees, and specific agenda for the next meeting.
</deck_framework>

<slide_ratio>
## Slide Ratio: 60/20/20

- **Problem/urgency (Slides 1-4): 50-60%** of visual and narrative weight. The majority of the narrative must establish the problem landscape before any solution appears. The prospect should see themselves in the first half of the deck.
- **Solution/differentiation (Slides 5-6): 20-25%** of the deck. Solution content is deliberately compressed. In a first call, the goal is not to explain every feature but to establish that a credible, differentiated path exists.
- **Proof and CTA (Slides 7-8): 20-25%** of the deck. One powerful customer story plus clear next steps. Detailed case studies, ROI models, and technical validation belong in follow-up materials, not the first-call deck.
</slide_ratio>

<never_rules>
## Eight Things to NEVER Put in the Deck

1. **Never open with company history or "About Us."** Move company credentials to an appendix slide or weave a one-sentence credibility marker into the proof slide.
2. **Never include an ROI calculator.** ROI presented at any stage drops close rates 27%. In a first call, the buyer hasn't shared enough data for credible ROI, and speculative calculations undermine trust. Use cost-of-inaction instead.
3. **Never use a logo grid as social proof.** Traditional social proof techniques see 22% lower close rates. Name-dropping logos makes buyers feel inadequate or generic. Replace with one specific customer story.
4. **Never build to a climax.** Executives process top-down. Lead with the insight, then support it — never save the best for last.
5. **Never include feature comparison charts against competitors.** Feature comparisons commoditize your solution and give unnecessary airtime to alternatives.
6. **Never dump discovery questions into slides.** Build in 2-3 strategic pause points where you invite dialogue rather than listing questions.
7. **Never exceed 20 words per slide (presentation version) / 60 words per slide (reading version).** If the audience can read faster than you can speak, there's no reason to listen. Slides should be visual anchors for a verbal narrative.
8. **Never skip the next-steps slide.** Meetings without a stated outcome and clear forward action are the single largest source of deal stall in enterprise sales.
</never_rules>

<two_versions>
## Two Versions Per Deck

Generate every deck TWICE:

1. **Presentation version** (suffix: `-presentation.pptx`): Max 20 words per slide. Visual-heavy. Slides are visual anchors for a verbal narrative. This is what the seller presents live.
2. **Reading version** (suffix: `-reading.pptx`): Max 60 words per slide. Expanded narrative for asynchronous sharing. Personalized leave-behind decks are shared internally 2.3x more often. This is what gets forwarded to the CFO.
</two_versions>

<workflow>
## Workflow

1. Read research-brief.md: extract offering overview, ICP, value prop, differentiators, demand signals, competitive landscape, pain points, buyer personas
2. Read qualified-companies.md: extract all companies with industry, evidence, GTM fit, confidence scores
3. Create ./{slug}/marketing/ directory
4. Create ./{slug}/marketing/prospect-specific/ directory
5. Invoke /pptx skill to generate the general deck:
   - Build the 8-slide deck following the framework above
   - Generate presentation version: ./{slug}/marketing/deck-presentation.pptx
   - Generate reading version: ./{slug}/marketing/deck-reading.pptx
6. For EACH company in qualified-companies.md, invoke /pptx skill to generate a prospect-specific 2-slide deck:
   - Slide 1: How the offering improves this company's industry — tailored version of the macro shift and quantified pain, using industry-specific data from the company's entry in qualified-companies.md
   - Slide 2: How the offering improves ROI for this specific company — uses the company's evidence, GTM fit rationale, and projected outcomes based on the value proposition
   - Generate presentation version: ./{slug}/marketing/prospect-specific/{company-slug}-presentation.pptx
   - Generate reading version: ./{slug}/marketing/prospect-specific/{company-slug}-reading.pptx
   - Derive {company-slug} from company name (lowercase, hyphens, max 50 chars)
7. Run QA cycle on generated decks
</workflow>

<design_standards>
## Design Standards

Apply these standards to ALL generated decks:
- Layout: 16x9 (LAYOUT_16x9: 10" x 5.625")
- One dominant color (60-70% visual weight), 1-2 supporting tones, one sharp accent
- Pick ONE distinctive visual motif and repeat it across every slide
- Every slide needs visuals — text-only layouts are NOT acceptable. Include shapes, accent bars, icons, or data visualizations.
- Font pairing: Use interesting font pairings, not defaults. Clear size contrast (36-44pt titles, 14-16pt body).
- Spacing: 0.5" minimum margins, 0.3-0.5" between content blocks
- Select a color palette appropriate to the offering's industry (e.g., Midnight Executive for enterprise, Forest & Moss for sustainability, Coral Energy for consumer)
</design_standards>

<qa_cycle>
## QA Cycle

After generating each deck:
1. Convert slides to images using LibreOffice + pdftoppm for visual inspection
2. Check for: missing content, leftover placeholders, text overflow, visual consistency
3. Complete at least one fix-and-verify cycle before moving to the next deck
4. If a deck fails QA, regenerate and re-verify
</qa_cycle>

<output_requirements>
## Outputs

**General deck (2 files):**
- ./{slug}/marketing/deck-presentation.pptx
- ./{slug}/marketing/deck-reading.pptx

**Prospect-specific decks (~20 files for 10 companies):**
- ./{slug}/marketing/prospect-specific/{company-slug}-presentation.pptx
- ./{slug}/marketing/prospect-specific/{company-slug}-reading.pptx
</output_requirements>

<quality_standards>
- Read ALL input files completely before generating anything
- 8-slide framework followed exactly for the general deck
- All 8 "never" rules enforced
- Presentation version: max 20 words per slide
- Reading version: max 60 words per slide
- Every slide has visual elements (no text-only slides)
- Prospect-specific decks reference company-specific data from qualified-companies.md
- All company names, industries, and evidence match the source files exactly
- QA cycle completed for every deck
</quality_standards>
```

**Step 2: Verify the file was written**

Read `agents/deck-builder.md` and confirm the YAML frontmatter parses correctly (name, description, model, tools) and the 8-slide framework is fully embedded.

**Step 3: Commit**

```bash
git add agents/deck-builder.md
git commit -m "feat: add deck-builder agent for PPTX sales deck generation"
```

---

### Task 4: Update the orchestrator for branching pipeline

**Files:**
- Modify: `commands/lead-genius.md` (lines 166-235 — Phases 7 through 10)

This is the most critical change. Phases 0-6 remain identical. Phases 7-10 are rewritten to implement the branching architecture.

**Step 1: Replace Phase 7 (DM Research) with branching dispatch**

Replace the current `## PHASE 7: DECISION MAKER RESEARCH (PARALLEL)` section (lines 166-184) with a new section that launches both branches simultaneously.

The new Phase 7 section should read:

```markdown
## PHASE 7: PARALLEL BRANCH DISPATCH

After company synthesis, the pipeline forks into two parallel branches.

### Pre-dispatch: Install deck dependencies

Run via Bash: `npm list -g pptxgenjs > /dev/null 2>&1 || npm install -g pptxgenjs`

### Branch dispatch

First, read ./{slug}/companies/qualified-companies.md to get the list of top companies. Count total companies (should be ~10, up to 20).

Split companies across 5 DM researchers evenly. For example with 10 companies:
- Researcher 01: companies 1-2
- Researcher 02: companies 3-4
- Researcher 03: companies 5-6
- Researcher 04: companies 7-8
- Researcher 05: companies 9-10

Spawn ALL 7 agents SIMULTANEOUSLY in a single message with 7 Task calls:

**Branch A — Marketing (2 agents):**

Content writer:
- subagent_type: "content-writer"
- prompt: "[Slug: {slug}] Read ./{slug}/go-to-market/research-brief.md and ./{slug}/companies/qualified-companies.md. Generate marketing content. Write blog.md, linkedin-posts.md, and case-study.md to ./{slug}/marketing/"
- description: "Writing marketing content → ./{slug}/marketing/"

Deck builder:
- subagent_type: "deck-builder"
- prompt: "[Slug: {slug}] Read ./{slug}/go-to-market/research-brief.md and ./{slug}/companies/qualified-companies.md. Invoke /pptx skill to generate all PPTX decks. Write general deck (presentation + reading versions) to ./{slug}/marketing/ and prospect-specific decks to ./{slug}/marketing/prospect-specific/"
- description: "Building PPTX decks → ./{slug}/marketing/"

**Branch B — DM Research (5 agents):**

For each instance NN (01 through 05):
- subagent_type: "dm-researcher"
- prompt: "[Slug: {slug}] [Instance: {NN}] [Assigned Companies: {list company names}] Read ./{slug}/go-to-market/research-brief.md and ./{slug}/companies/qualified-companies.md. Research ONLY your assigned companies: {list}. Find 3-5 most relevant decision makers per company. Write to ./{slug}/decision-maker-research/dm-{NN}.md"
- description: "Researching DMs for {N} companies → ./{slug}/decision-maker-research/dm-{NN}.md"

Wait for ALL 7 to complete.
```

**Step 2: Keep Phase 8 (DM Compilation) as-is**

The existing Phase 8 section (DM Compilation) does not change. It stays as `## PHASE 8: DM COMPILATION` and remains identical.

**Step 3: Keep Phase 9 (Outreach Generation) as-is**

The existing Phase 9 section (Outreach Generation) does not change.

**Step 4: Update Phase 10 (Completion) to include marketing outputs**

Replace the current Phase 10 completion summary with an expanded version that reports both branches.

The new Phase 10 should read:

```markdown
## PHASE 10: COMPLETION

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
- Company Research: ./{slug}/company-research/companies-*.md (5 files)
- Qualified Companies: ./{slug}/companies/qualified-companies.md
- DM Research: ./{slug}/decision-maker-research/dm-*.md (5 files)
- Decision Makers: ./{slug}/decision-makers/decision-makers.md
- Outreach: ./{slug}/outreach.md
- Blog: ./{slug}/marketing/blog.md
- LinkedIn Posts: ./{slug}/marketing/linkedin-posts.md
- Case Study: ./{slug}/marketing/case-study.md
- General Deck: ./{slug}/marketing/deck-presentation.pptx, deck-reading.pptx
- Prospect Decks: ./{slug}/marketing/prospect-specific/*.pptx

Results:
- Companies Qualified: {count from qualified-companies.md}
- Decision Makers: {count from decision-makers.md}
- Outreach Messages: {count from outreach.md}
- Marketing Content: blog + {N} LinkedIn posts + case study
- Decks Generated: 2 general + {N} prospect-specific

Next Steps:
1. Review ./{slug}/decision-makers/decision-makers.md for contact details
2. Review ./{slug}/outreach.md for personalized emails
3. Review ./{slug}/marketing/ for content and decks
4. Customize emails and content as needed before sending/publishing
=== END ===
```
```

**Step 5: Verify the full orchestrator reads correctly**

Read `commands/lead-genius.md` in full and verify:
- Phases 0-6 are unchanged
- Phase 7 launches 7 agents (2 marketing + 5 DM) simultaneously
- Phases 8-9 remain sequential (DM compilation → outreach)
- Phase 10 reports both branches

**Step 6: Commit**

```bash
git add commands/lead-genius.md
git commit -m "feat: branch pipeline after Phase 6 for parallel marketing + DM research"
```

---

### Task 5: Update plugin.json version

**Files:**
- Modify: `.claude-plugin/plugin.json` (line 4)

**Step 1: Bump version**

Change `"version": "1.2.0"` to `"version": "1.3.0"` in `.claude-plugin/plugin.json`.

**Step 2: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "chore: bump version to 1.3.0"
```

---

### Task 6: Update CLAUDE.md documentation

**Files:**
- Modify: `CLAUDE.md`

**Step 1: Update the version reference**

Change `A Claude Code plugin (v1.2.0)` to `A Claude Code plugin (v1.3.0)` in the first paragraph.

**Step 2: Update the Plugin Structure section**

Replace the Plugin Structure code block with:

```
.claude-plugin/plugin.json   # Plugin manifest (name, version, tools)
commands/lead-genius.md       # Main orchestrator — the /lead-genius command
agents/                       # 10 specialized agent prompts (markdown + YAML frontmatter)
skills/executive-outreach/    # Email generation skill with SKILL.md + reference examples
skills/pptx/                  # PPTX generation skill from anthropics/skills
```

**Step 3: Update the Architecture section**

Replace `The orchestrator (`commands/lead-genius.md`) drives a linear 10-phase pipeline.` with:

`The orchestrator (`commands/lead-genius.md`) drives a 10-phase pipeline that branches after company synthesis. It never does research or writes output itself — it delegates everything to agents via the Task tool.`

Replace the Phase flow line with:

`**Phase flow:** Setup → Collateral Analysis → GTM Interview → Synthesis → Scoring Rubrics → Company Research (5 parallel) → Company Synthesis → [Branch A: Marketing Content (2 parallel) | Branch B: DM Research (5 parallel) → DM Compilation → Outreach] → Completion`

**Step 4: Update the Agent Roles table**

Add two new rows to the agent table:

```
| `content-writer` | Generate blog, LinkedIn posts, case study | 1x |
| `deck-builder` | Generate PPTX decks (general + prospect-specific) | 1x |
```

Update the header `# 8 specialized agent prompts` comment to `# 10 specialized agent prompts`.

**Step 5: Add marketing output to the Making Changes section**

Add after the outreach-composer bullet:

```
- The `deck-builder` agent invokes the `/pptx` skill, which lives at `skills/pptx/SKILL.md` (from the `anthropics/skills` repository).
```

**Step 6: Verify CLAUDE.md reads correctly**

Read `CLAUDE.md` in full and verify all changes are consistent.

**Step 7: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for v1.3.0 marketing content branch"
```

---

### Task 7: Final verification

**Step 1: Verify all files exist**

```bash
ls -la agents/content-writer.md agents/deck-builder.md
ls -la skills/pptx/SKILL.md skills/pptx/pptxgenjs.md skills/pptx/editing.md
ls -la skills/pptx/scripts/
```

Expected: All files present.

**Step 2: Verify the agent count**

```bash
ls agents/*.md | wc -l
```

Expected: 10

**Step 3: Verify git log**

```bash
git log --oneline feature/marketing-content-generation --not main
```

Expected: 7 commits (design doc + 6 implementation commits).

**Step 4: Verify no untracked files**

```bash
git status
```

Expected: Clean working tree.
