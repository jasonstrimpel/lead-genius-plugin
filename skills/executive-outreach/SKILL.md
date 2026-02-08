---
name: executive-outreach
description: Generate context-relevant, executive-level cold outreach emails optimized for response rates. Use when creating personalized outreach emails that reference specific triggers (news, promotions, initiatives), incorporate sender credentials, and match tone to buyer tier. Triggers on requests to write cold emails, outreach emails, prospecting emails, or business development emails targeting executives or decision makers.
---

# Executive Outreach Email Generator

Generate high-conversion cold outreach emails that are concise, value-focused, and structured for time-constrained decision makers.

## Input Requirements

Expect structured data containing:

- **Contact**: Name, title, company, email, buyer tier (1-3), accessibility level, personalization hooks, recommended approach
- **Offering**: Service/product description and value proposition
- **Company**: Fit rationale, strategic priorities, recent activity, demand signal (the observable trigger making outreach timely)
- **Sender**: Name, credentials, relevant experience, shared connections

## Output Structure

For each contact, generate output with these sections in exact order:

### 1. Header and Metadata

**Header line:**

`### {Company} - {Contact Name} ({Title})`

**Metadata block** (bullet list immediately below header):
- **Overall Confidence Score:** 1-5 rating from scoring rubrics
- **Tier:** Buyer tier designation (e.g., T1, T2, T3, or hybrid like T3/T1)
- **Hook**: Personalization hook used in the email opening
- **Offering**: The offering being pitched
- **Signal**: Demand signal making outreach timely (e.g., "Series B funding", "VP Sales hire", "AWS migration announced")
- **Evidence Used**: Company-specific facts referenced in the message
- **Sender Context**: Which sender credentials/experience used (if applicable)

### 2. Email

**Format:**

```
**Email:** Name <email@domain.com>

**Subject:** 6-10 word subject referencing ONE specific hook

{First name},

{Continuous email body}

{Forward-looking closing line}

{Sender first name, if sender profile provided}
```

The email body is continuous prose with no internal section labels. Compose it following this flow:

**Opening (2-3 sentences):** Lead with the trigger from personalization hooks. Name the initiative, announcement, or recognition. Establish relevance within first 15 words.

**Credibility Bridge (1 sentence):** Single sentence connecting sender background to recipient's context. Only use to establish relevance, not to make it about the sender. Use ONE:
- Shared industry experience: "[X] years in [their domain]"
- Relevant client work: "My work with [comparable situation]"
- Shared connection: "[Mutual contact] suggested I reach out"
- Domain credential: "As a [relevant background]"

Skip if no authentic relevance exists. Never fabricate.

**Value Bridge (2-3 sentences):** Connect offering to their stated priority. Quantify ONE benefit. Match language to tier:
- **Tier 1**: Strategic outcomes, competitive advantage, business impact
- **Tier 2**: Operational improvement, process enhancement, team enablement
- **Tier 3**: Technical validation, implementation support, evaluation criteria

**Proof Point (1-2 sentences):** Reference capability, case study, or credential with specifics: outcomes, timeframes, comparable situations. Never mention attachments or offer materials.

**Call to Action (1-2 sentences):** One clear next step matched to accessibility:
- **High**: Direct meeting request with timeframe. Reference their location, not yours.
- **Medium**: Brief call or introduction offer
- **Low**: Permission to send additional materials

**Closing (1 sentence):** Forward-looking statement tied to their priority. Sender first name only (if provided), no title block.

## Style Rules

**Voice**: Peer-to-peer professional. Confident and direct. Respectful. Relationship-focused.

**Structure**: Alternate short sentences (5-8 words) with detail-rich ones (15-20 words). Max 2-3 sentences per paragraph. Line breaks between sections. One idea per sentence. Value focused. Recipient focused.

**Word choice**: Active voice only. No hedging. Match technical depth to tier. Always include pronouns ("I'm reaching out" not "Reaching out"). Always use full sentences to preserve formality ("Is it worth a 15 minute call to discuss?" not "Worth a 15 minute call to discuss?")

**Length**: 100-150 words total. 5-6 paragraphs.

## Prohibited

- Emdashes (—), semicolons (;), and hyphens (-) to separate ideas in a sentence
- Generic openers ("I hope this finds you well")
- Superlatives ("best-in-class," "industry-leading," "cutting-edge")
- Pressure tactics or false urgency
- Questions in opening line
- Multiple CTAs
- The phrase "I wanted to"
- Repeating recipient's title back to them
- Fabricated connections or credentials
- References to attachments or offers to send materials
- Mentioning sender's location or travel plans
- Mentioning anything personal about the target

## Required

- At least one quantified benefit
- Direct reference to ONE personalization hook in opening
- Preserve all proper nouns exactly as provided
- CTA matched to accessibility level
- Credibility bridge connecting sender to recipient context (if authentic)
- Header and metadata block before each email

## Tier Calibration

| Tier | Focus | Tone | Credibility | CTA |
|------|-------|------|-------------|-----|
| 1 (C-Suite, VPs, Directors) | Business outcomes, ROI, differentiation | Strategic peer | Industry tenure, exec experience, client outcomes | Meeting to discuss application |
| 2 (Operations, Program Managers) | Efficiency, process improvement, outcomes | Collaborative expert | Domain expertise, methodology, hands-on experience | Methodology discussion |
| 3 (Technical, Analysts, BD) | Technical fit, implementation, evaluation | Resource and ally | Technical credentials, implementation experience | Share framework or support evaluation |

## Examples

See `references/examples.md` for effective vs ineffective patterns.