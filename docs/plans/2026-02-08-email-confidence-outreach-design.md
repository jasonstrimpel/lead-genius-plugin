# Email Confidence Indicators in Outreach Output

## Problem

Some decision makers have no attached email or have low-confidence emails (pattern-matched, unverified). The outreach.md output currently has no way to indicate this, leaving users unaware of which emails need verification before sending.

## Design Decisions

- Generate outreach for ALL decision makers, including those with no email. The message is ready when the user finds the address.
- Three layered indicators for non-verified emails. Verified emails stay clean with no extra noise.
- Inline only per contact. No separate summary section at the end.

## Output Format

### Three Layers (non-verified only)

**1. Metadata field** — `**Email Confidence:**` added as the last item in the metadata block. Omitted for verified emails.

```
- **Email Confidence:** Pattern-matched (company format: first initial + last name)
```

```
- **Email Confidence:** Unverified (guessed from public sources)
```

```
- **Email Confidence:** No email found
```

**2. Warning banner** — Placed between the metadata block and the `**Email:**` line. Omitted for verified emails.

For pattern-matched or unverified:

```
> ⚠ **Email unverified — confirm before sending**
```

For no email found:

```
> ⚠ **No email found — lookup required.** Try LinkedIn (linkedin.com/in/name) or company directory pattern (firstlast@company.com).
```

**3. Inline tag on Email line** — Appended after the email address. Omitted for verified emails.

```
**Email:** Mike Alberts <malberts@coherecapital.com> `[Pattern-matched]`
```

```
**Email:** Mike Alberts <malberts@coherecapital.com> `[Unverified]`
```

For no email:

```
**Email:** Jordan Rivera <— lookup required>
```

### Placement Order

Email Confidence is the last metadata field, warning banner sits between metadata and Email line. A user scanning top-to-bottom hits the warning before reaching the copy-paste email address.

## Complete Examples

### Verified (clean — no extra elements)

```
### GuidePost Growth - Chris Cavanagh (EVP, Portfolio Management)
- **Overall Confidence Score:** 5
- **Tier:** T3/T1
- **Hook**: Fund IV close ($521M oversubscribed)
- **Offering**: Intangible asset valuation for PE
- **Signal**: Fund IV oversubscribed close
- **Evidence Used**: $521M fund close, software thesis
- **Sender Context**: Growth equity intangible asset valuation experience

**Email:** Chris Cavanagh <ccavanagh@guidepostgrowth.com>

**Subject:** Fund IV Data Asset Valuation Opportunity

Chris,
...
```

### Pattern-matched (all three indicators)

```
### Cohere Capital - Mike Alberts (Principal, Head of Business Development)
- **Overall Confidence Score:** 3
- **Tier:** T3
- **Hook**: ACG Boston Advisory Board
- **Offering**: Intangible asset valuation for PE
- **Signal**: Active partner evaluation role
- **Evidence Used**: Role evaluating new service partners
- **Sender Context**: 40+ PE transaction valuations
- **Email Confidence:** Pattern-matched (company format: first initial + last name)

> ⚠ **Email unverified — confirm before sending**

**Email:** Mike Alberts <malberts@coherecapital.com> `[Pattern-matched]`

**Subject:** Intangible Valuation Partnership for Cohere Deal Flow

Mike,
...
```

### No email found (warning with lookup guidance)

```
### Apex Partners - Jordan Rivera (VP, Corporate Development)
- **Overall Confidence Score:** 4
- **Tier:** T1
- **Hook**: Q3 earnings call, M&A pipeline expansion
- **Offering**: Intangible asset valuation for PE
- **Signal**: M&A pipeline expansion announced
- **Evidence Used**: Q3 earnings call commentary, 3 acquisitions in 12 months
- **Sender Context**: PE exit valuation experience
- **Email Confidence:** No email found

> ⚠ **No email found — lookup required.** Try LinkedIn (linkedin.com/in/jordanrivera) or company directory pattern (firstlast@apexpartners.com).

**Email:** Jordan Rivera <— lookup required>

**Subject:** Apex M&A Pipeline Valuation Opportunity

Jordan,
...
```

## Files to Change

### 1. `skills/executive-outreach/SKILL.md`

- Add `**Email Confidence:**` as optional metadata field (omitted for verified)
- Add warning banner placement rule between metadata and Email line
- Add inline tag format for the Email line
- Add no-email format: `Name <— lookup required>` with lookup guidance in banner

### 2. `skills/executive-outreach/references/examples.md`

- Add pattern-matched example showing all three indicators
- Add no-email example showing lookup guidance
- Keep existing Tier 1 example as the verified/clean baseline

### 3. `agents/outreach-composer.md`

- Step 1: also extract email confidence per DM
- Step 4 (per-DM loop): pass email confidence to the skill invocation
- Add rule: generate outreach for ALL decision makers regardless of email status
- For no-email contacts: include lookup suggestions (LinkedIn URL if found during research, company email pattern)

### 4. No changes needed

`agents/dm-researcher.md` and `agents/dm-compiler.md` already track and pass through Email Confidence (Verified/Pattern-matched/Unverified). The no-email case is implicit when no email is found.
