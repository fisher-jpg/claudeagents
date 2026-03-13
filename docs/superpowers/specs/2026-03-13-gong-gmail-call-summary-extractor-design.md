# Design Spec: Gong Call Summary Extractor (Gmail-Triggered)

## Overview

A skill that detects Gong call completion emails in Gmail, extracts the call summary and metadata, and outputs a structured Markdown artifact. This serves as the data ingestion layer for the post-call follow-up generation workflow.

**Phase 1** of a two-phase approach:
- **Phase 1 (this spec):** Gmail email parsing — no Gong API required
- **Phase 2 (future):** Gong MCP/API integration for raw transcript access and independent analysis

---

## Problem Statement

John Fisher (Solutions Consultant at Gem) needs call summary data extracted from Gong to feed into post-call follow-up workflows. Gong sends a rich notification email when call recording and analysis completes. Currently, John manually opens these emails, reviews the summary, and re-enters the information. This skill automates the extraction step.

A key motivation for Phase 2: Gong's AI-generated next steps sometimes miss follow-up items that are present in the raw transcript. Phase 2 will add Claude's independent transcript analysis to catch these gaps.

---

## Architecture

### Flow

```
Gong finishes processing call
        ↓
Email arrives from do-not-reply@gong.io
        ↓
Skill invoked (manually now, autonomous agent trigger later)
        ↓
Search Gmail for Gong notification emails
        ↓
Parse email body → extract structured data
        ↓
Output structured Markdown artifact
        ↓
Handoff to post-call follow-up generation workflow
```

### Trigger

- **Phase 1 (manual):** User invokes the skill — e.g., "Pull my latest Gong call summary" or "Get the Gong summary for the Fandom call"
- **Phase 1+ (autonomous):** A Gmail-monitoring agent (similar to `prep-block-on-booking`) detects new Gong emails and auto-triggers this skill
- **Phase 2:** Gong MCP becomes the data source; Gmail trigger may remain as the timing signal

### Tools Required

| Tool | Purpose | Status |
|---|---|---|
| Gmail MCP | Search for and read Gong notification emails | Configured |

No additional tools or API access required for Phase 1.

---

## Gmail Search Strategy

### Search Parameters

- **From:** `do-not-reply@gong.io`
- **Subject contains:** `Call recording and analysis is ready`
- **Time window:** Last 7 days (default), configurable

### Filtering

- **By company name:** Optional — user can specify a company to narrow results (e.g., "Pull the Gong summary for Acme")
- **Multiple results:** Present a list with call title, company, and date for user to select
- **Single result:** Proceed automatically

---

## Email Structure & Parsing

Based on confirmed Gong email format, the skill extracts the following sections:

### Email Format Reference

```
Subject: [Call Title]: Call recording and analysis is ready
From: Gong <do-not-reply@gong.io>

Body:
  "Your call is ready"

  [Call Card]
    - Call title (linked)
    - Company name
    - Date
    - Duration
    - Participants summary (e.g., "Gem: Andres Ortiz + 1 more · Customer: Ryan Borello")

  [Go to call] button — direct link to Gong
  [Generate follow-up email] link

  Key points
    1. ...
    2. ...
    (numbered list)

  Next steps
    1. ...
    2. ...
    (numbered list)

  Associated deals
    - Deal name (linked)
    - Amount
    - Call stage
    - Close date

  Participants
    [Your Company]
      - Name (role indicator) · Title
    [Customer Company]
      - Name · Title
```

### Extracted Fields

| Field | Source Location | Required |
|---|---|---|
| Call title | Call card / subject line | Yes |
| Company name | Call card + Participants section | Yes |
| Call date | Call card | Yes |
| Duration | Call card | Yes |
| Gong call link | "Go to call" button URL | Yes |
| Key points | "Key points" numbered list | Yes |
| Next steps | "Next steps" numbered list | Yes |
| Deal name | Associated deals section | No (may not always be present) |
| Deal amount | Associated deals section | No |
| Deal stage | Associated deals section | No |
| Deal close date | Associated deals section | No |
| Gem participants | Participants section | Yes |
| Customer participants | Participants section | Yes |

---

## Output Format

The skill produces a structured Markdown artifact:

```markdown
# Call Summary: [Call Title]

## Call Details
- **Company:** [Company Name]
- **Date:** [Date]
- **Duration:** [Duration]
- **Gong Call ID:** [extracted from URL]
- **Gong Link:** [Go to call](URL)

## Participants
### Gem
- [Name] ([role]) · [Title]
- [Name] · [Title]

### [Customer Company]
- [Name] · [Title]

## Associated Deal
- **Deal:** [Deal Name]
- **Amount:** [Amount]
- **Stage:** [Stage]
- **Close Date:** [Close Date]

## Key Points
1. [Key point 1]
2. [Key point 2]
...

## Next Steps
1. [Next step 1]
2. [Next step 2]
...
```

- The "Associated Deal" section is omitted if no deal data is present in the email
- All numbered items are preserved exactly as Gong generated them

---

## Invocation

### Manual Invocation (Phase 1)

The skill accepts optional parameters:

| Parameter | Description | Default |
|---|---|---|
| Company name | Filter results to a specific company | None (return all) |
| Time window | How far back to search | Last 7 days |

**Example invocations:**
- "Pull my latest Gong call summary"
- "Get the Gong summary for the Fandom call"
- "Pull Gong summaries from the last week"

### Autonomous Trigger (Phase 1+)

A future agent wrapping this skill would:
1. Periodically search Gmail for new Gong notification emails (or respond to a Gmail push notification)
2. Check against previously processed calls (deduplication)
3. Auto-invoke this skill for each new email
4. Pass the output to the post-call follow-up generation workflow

This agent is not in scope for this spec but follows the same pattern as `prep-block-on-booking`.

---

## Handoff to Post-Call Follow-Up Workflow

The structured Markdown output becomes the input for the `post-call-follow-up-generation` workflow (spec at repo root: `outputs/post-call-follow-up-generation-building-block-spec.md`).

### Phase 1 Workflow Simplification

The existing post-call follow-up workflow was designed for full transcript access. In Phase 1 (email-only), **Steps 2 and 3 are bypassed**:

- **Step 2 (Transcript Analyzer):** Requires the raw transcript for independent analysis. In Phase 1, no transcript is available — this step is skipped. The Gong summary from the email serves as the sole source of call content.
- **Step 3 (Summary Comparator):** Compares Claude's analysis to Gong's summary to catch discrepancies. With no independent analysis (Step 2 skipped), there is nothing to compare — this step is skipped.

**Phase 1 effective flow:**
```
This Skill (extract from Gmail) → Step 4 (Human Review) → Step 5 (Product Knowledge) → Step 6 (Draft) → Step 7 (Deliver)
```

**Phase 2 restores the full flow** by adding transcript access, enabling Steps 2 and 3.

### Output Mapping

| This Skill's Output | Feeds Into |
|---|---|
| Key Points + Next Steps | Step 4 (Human Review) — user reviews Gong's summary as the single source |
| Participants | Step 6 (Follow-Up Drafter) — recipient and attendee context |
| Deal info | Step 6 (Follow-Up Drafter) — deal context for the follow-up |
| Gong Link | Preserved as reference in the follow-up |
| Gong Call ID | Extracted from Gong link URL — used for Phase 2 API calls |

---

## Phase 2 Migration Path

When the Gong MCP (currently being developed by the admin team) becomes available:

1. **Data source swap:** Replace Gmail parsing with Gong MCP API calls to fetch transcripts and summaries directly
2. **New output section:** Add `## Transcript Analysis` with Claude's independent findings from the raw transcript
3. **New output section:** Add `## Discrepancies` highlighting items Claude found that Gong's summary missed
4. **Output contract preserved:** All existing fields remain; Phase 2 adds sections without breaking downstream consumers
5. **Gmail trigger retained:** The email arrival still serves as the timing signal that a transcript is ready, even though data comes from the API

---

## Email Body Parsing Approach

The Gmail MCP `gmail_read_message` tool returns email content which may be HTML (Gong uses a styled email template). The parsing approach:

1. **Prefer plain text:** If the Gmail MCP returns a plain-text version (multipart alternative), use it — the structure is simpler to parse
2. **HTML fallback:** If only HTML is available, extract text content by identifying Gong's structural elements (headings like "Key points", "Next steps", "Associated deals", "Participants" and their associated content blocks)
3. **Section-based extraction:** Parse by looking for known section headers rather than relying on specific HTML classes or CSS (which Gong may change). The section headers ("Key points", "Next steps", etc.) are stable content strings.
4. **Numbered list preservation:** Extract numbered items maintaining their original order and full text

### Gong Call ID Extraction

The "Go to call" link URL contains a Gong call identifier. Extract and include this in the output as a stable ID for Phase 2 API correlation.

---

## Error Handling

| Scenario | Behavior |
|---|---|
| No matching emails found | Report "No Gong call emails found in the last [time window]" and suggest widening the search |
| Email found but a required field cannot be parsed | Extract what is available, flag missing fields with `[Not found in email]`, and warn the user |
| Email body format differs from expected structure | Return the raw email content to the user with a note that parsing failed — do not guess |
| Gmail MCP returns truncated content | Warn the user and attempt extraction from what is available |
| Multiple emails match and no company filter provided | Present a numbered list of matches (title, company, date) for the user to select |

**Idempotency:** Re-extracting the same email is safe and expected. The skill does not track previously processed emails — that is the responsibility of the future autonomous agent wrapper (Phase 1+).

---

## Constraints & Assumptions

- Gong email format is consistent (confirmed by user)
- Emails always come from `do-not-reply@gong.io`
- Subject line always ends with `: Call recording and analysis is ready`
- The email body contains the full Gong summary (confirmed — same content as in the Gong UI)
- Gmail MCP is already configured and working
- Gmail MCP `gmail_search_messages` supports Gmail query syntax (e.g., `from:do-not-reply@gong.io subject:"Call recording and analysis is ready" newer_than:7d`)
- Not all calls will have associated deal data
- The call card in the email shows a condensed participant summary (e.g., "+ 1 more"); full participant details with names and titles are in the dedicated Participants section at the bottom of the email

---

## Relationship to Existing Specs

| Existing Spec | Relationship |
|---|---|
| `post-call-follow-up-generation-building-block-spec.md` | This skill replaces Step 1 (Retrieve Gong Artifacts) for Phase 1 |
| `post-call-follow-up-generation-definition.md` | This skill's output feeds the workflow defined here |
| `prep-block-on-booking` agent | Pattern reference for the future autonomous Gmail-monitoring agent |

---

## Implementation Location

- **Skill file:** `skills/gong-call-summary-extractor/SKILL.md`
- **Follows existing pattern:** Same YAML frontmatter format as `skills/prep-note-summary/SKILL.md`
- **Output behavior:** Return the structured Markdown artifact as the response (do not auto-save to a file)

---

## What This Skill Does NOT Do

- Generate follow-up messages (that's the post-call follow-up workflow)
- Analyze the raw transcript (Phase 2)
- Compare Claude's analysis to Gong's summary (Phase 2)
- Query product knowledge sources (separate skill in the workflow)
- Send any messages or drafts
