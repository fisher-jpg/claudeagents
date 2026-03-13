---
name: gong-call-summary-extractor
description: >
  Extracts Gong call summaries from Gmail notification emails and returns a
  structured Markdown artifact. Use when the user asks to pull a Gong call
  summary, get call notes from Gong, check for recent Gong recordings, or
  mentions "Gong summary", "call recording", "call transcript", or "pull my
  last call". Also trigger when the user asks about follow-ups from a recent
  call and no Gong data has been extracted yet.
user-invocable: true
metadata:
  author: handsonai
  version: "1.0.0"
---

# Gong Call Summary Extractor

Extract call summaries from Gong notification emails in Gmail and return a
structured Markdown artifact ready for the post-call follow-up workflow.

## When to use this skill

- User asks to pull a Gong call summary
- User wants to review a recent call's key points or next steps
- User mentions a specific call and needs the summary extracted
- A workflow needs Gong call data as input (e.g., post-call follow-up generation)

## Tools required

- `gmail_search_messages` — to find Gong notification emails
- `gmail_read_message` — to read the full email content

## Inputs

The user may provide:
- **Company name** (optional) — filter results to a specific company
- **Time window** (optional) — how far back to search; defaults to last 7 days

If the user says something like "pull my latest Gong call" with no filters,
search the last 7 days with no company filter.

## Step-by-step process

### Step 1 — Search Gmail for Gong notification emails

Use `gmail_search_messages` with this query:

```
from:do-not-reply@gong.io subject:"Call recording and analysis is ready" newer_than:7d
```

Adjust the query based on user input:
- If a company name is provided, add it as a separate subject term: `from:do-not-reply@gong.io subject:"Call recording and analysis is ready" subject:[Company] newer_than:7d`
  - Note: the company name appears before the colon in the subject line, but may be part of a longer call title. Keep it as a separate `subject:` term rather than embedding it in the quoted string.
- If a different time window is requested, adjust `newer_than` accordingly (e.g., `newer_than:1d`, `newer_than:30d`)

### Step 2 — Handle search results

- **No results:** Report "No Gong call emails found in the last [time window]. Try widening the search or checking the company name."
- **One result:** Proceed to Step 3 with that email.
- **Multiple results:** Present a numbered list to the user:
  ```
  Found [N] Gong call summaries. Which one would you like?
  1. [Subject line excerpt] — [Date]
  2. [Subject line excerpt] — [Date]
  ...
  ```
  Wait for the user to select before proceeding.

### Step 3 — Read the email

Use `gmail_read_message` with the message ID from Step 2.

### Step 4 — Parse the email body

The Gong notification email has a consistent structure. Extract these fields:

**From the call card (top of email):**
- **Call title** — the linked text at the top of the card (also available from the subject line, before `: Call recording and analysis is ready`)
- **Company name** — listed on the second line of the call card
- **Call date** — date shown in the call card
- **Duration** — shown in the call card (e.g., "50 minutes")
- **Gong call link** — the URL behind the "Go to call" button
- **Gong call ID** — extract from the Gong call link URL

**From the body sections:**
- **Key points** — numbered list under the "Key points" heading. Preserve all items exactly as written.
- **Next steps** — numbered list under the "Next steps" heading. Preserve all items exactly as written.

**From the Associated deals section (may not be present):**
- **Deal name** — linked text
- **Amount** — dollar amount
- **Call stage** — stage label
- **Close date** — date

**From the Participants section:**
- **Gem participants** — names, role indicators (e.g., "host"), and titles listed under the "Gem" heading
- **Customer participants** — names and titles listed under the customer company heading

**Parsing notes:**
- The email may be HTML. Look for section headers ("Key points", "Next steps", "Associated deals", "Participants") as structural anchors.
- If a plain-text version is available, prefer it.
- The call card's participant summary (e.g., "Gem: Andres Ortiz + 1 more · Customer: Ryan Borello") is a condensed version. Use the full Participants section at the bottom for complete details.

### Step 5 — Handle parsing failures

If any **required** field cannot be found:
- Still extract everything that is available
- For missing required fields, include `[Not found in email]` as the value
- Warn the user: "Note: Could not extract [field name] from the email. The email format may have changed."

If the email structure is completely unrecognizable:
- Return the raw email content to the user
- Note: "Could not parse this Gong email. The format may have changed. Here is the raw content."
- Do not attempt to guess or fabricate data.

Required fields: call title, company name, call date, duration, Gong call link, key points, next steps, participants
Optional fields: deal name, deal amount, deal stage, deal close date, Gong call ID

If the email content appears truncated (cuts off mid-section), warn the user and extract what is available from the complete sections.

### Step 6 — Format and return the output

Return a structured Markdown artifact in this exact format:

```
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

**Output rules:**
- Omit the "Associated Deal" section entirely if no deal data was found in the email
- Preserve all numbered items exactly as Gong generated them — do not summarize, reorder, or edit
- Do not save the output to a file — return it as your response. The caller decides where it goes.

## Guidelines

- **Never fabricate data.** Every field in the output must come directly from the email. If it's not in the email, it's not in the output.
- **Preserve Gong's content exactly.** Key points and next steps should be word-for-word from the email. Do not paraphrase, summarize, or reorder.
- **This skill only extracts data.** It does not generate follow-up messages, analyze the call, or send anything. Those are separate workflow steps.
- **Idempotent.** Running this skill on the same email twice produces the same output. There is no state tracking or deduplication — that is the responsibility of the calling agent or workflow.

## Phase 2 note

This skill currently extracts from Gmail only. When the Gong MCP becomes available, this skill will be updated to:
1. Use the Gong MCP to fetch the full transcript and summary directly
2. Add a `## Transcript Analysis` section with Claude's independent findings
3. Add a `## Discrepancies` section highlighting items Claude found that Gong's summary missed
4. The existing output format will be preserved — Phase 2 adds sections without breaking downstream consumers
