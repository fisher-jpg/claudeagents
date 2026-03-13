# Gong Call Summary Extractor — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a skill that extracts Gong call summaries from Gmail notification emails and outputs structured Markdown artifacts for the post-call follow-up workflow.

**Architecture:** A single skill file (`SKILL.md`) that instructs Claude to search Gmail for Gong notification emails using the Gmail MCP, parse the email body to extract call metadata and summary content, and return a structured Markdown artifact. No code — this is a markdown-based skill following the same pattern as `prep-note-summary` and `prep-block-rule-engine`.

**Tech Stack:** Gmail MCP (`gmail_search_messages`, `gmail_read_message`), Markdown skill file with YAML frontmatter

**Spec:** `docs/superpowers/specs/2026-03-13-gong-gmail-call-summary-extractor-design.md`

---

## Chunk 1: Skill File Creation

### Task 1: Create the skill directory and SKILL.md

**Files:**
- Create: `skills/gong-call-summary-extractor/SKILL.md`

**Reference files to read first:**
- `skills/prep-note-summary/SKILL.md` — for YAML frontmatter format and skill structure conventions
- `skills/prep-block-rule-engine/SKILL.md` — for input/output structuring and decision logic patterns
- `docs/superpowers/specs/2026-03-13-gong-gmail-call-summary-extractor-design.md` — the design spec

- [ ] **Step 1: Read reference files**

Read the two existing skills and the design spec to understand the conventions. Key patterns to follow:
- YAML frontmatter with `name` and `description` fields
- `description` should include trigger phrases so the skill activates on relevant user requests
- Clear sections: purpose, inputs, decision logic, outputs, guidelines
- Output section should specify "Return as your response. Do not save to a file."

- [ ] **Step 2: Write the SKILL.md file**

Create `skills/gong-call-summary-extractor/SKILL.md`. Copy everything between the `<!-- SKILL.MD START -->` and `<!-- SKILL.MD END -->` markers below into the file:

<!-- SKILL.MD START -->
````markdown
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
````
<!-- SKILL.MD END -->

- [ ] **Step 3: Verify file structure**

Run: `ls -la skills/gong-call-summary-extractor/`
Expected: `SKILL.md` exists

- [ ] **Step 4: Validate YAML frontmatter**

Read `skills/gong-call-summary-extractor/SKILL.md` and verify:
- `name` field matches directory name
- `description` includes trigger phrases
- `user-invocable: true` is set
- Frontmatter is valid YAML between `---` delimiters

- [ ] **Step 5: Commit**

```bash
git add skills/gong-call-summary-extractor/SKILL.md
git commit -m "feat: add gong-call-summary-extractor skill for Gmail-based extraction"
```

---

## Chunk 2: Manual Validation

### Task 2: Test with a real Gong email

This task validates that the skill works end-to-end with a real Gong notification email in the user's inbox.

**Prerequisites:** Task 1 complete, user has at least one Gong notification email in Gmail

- [ ] **Step 1: Invoke the skill with no filters**

Ask the user to invoke the skill: "Pull my latest Gong call summary"

Verify:
- Gmail search executes with correct query
- Results are returned (or "no results" message if inbox is empty)

- [ ] **Step 2: Verify email parsing**

If a Gong email is found, verify:
- All required fields are extracted (call title, company, date, link, key points, next steps, participants)
- Optional fields are included when present (deal info, duration, call ID)
- No `[Not found in email]` placeholders appear (indicates parsing issue)

- [ ] **Step 3: Verify output format**

Compare the output against the spec's template:
- Markdown headers are correct (`#`, `##`, `###`)
- All sections present and in correct order
- Key points and next steps are numbered and preserved verbatim
- Associated Deal section is included only if deal data exists
- Participants are grouped by company with names and titles

- [ ] **Step 4: Test with company filter**

Invoke: "Get the Gong summary for [known company name]"

Verify:
- Search narrows to that company
- Correct email is found (or appropriate "not found" message)

- [ ] **Step 5: Test error handling**

Invoke: "Pull the Gong summary for NonexistentCompany12345"

Verify:
- Skill reports no results found
- Suggests widening the search

- [ ] **Step 6: Commit any fixes**

If any adjustments were needed to the skill during testing:
```bash
git add skills/gong-call-summary-extractor/SKILL.md
git commit -m "fix: adjust gong-call-summary-extractor based on live testing"
```

---

## Chunk 3: Documentation and Integration

### Task 3: Update project documentation

**Files:**
- Modify: `outputs/post-call-follow-up-generation-building-block-spec.md` (add Phase 1 note)

- [ ] **Step 1: Add Phase 1 integration note to post-call follow-up spec**

In `outputs/post-call-follow-up-generation-building-block-spec.md`, find the row in the Step-by-Step Decomposition table for Step 1 ("Retrieve Gong Artifacts"). Add a note directly below the table:

> **Phase 1 implementation:** Step 1 is handled by the `gong-call-summary-extractor` skill, which extracts call summaries from Gong notification emails via Gmail MCP (no Gong API required). In Phase 1, Steps 2 and 3 are bypassed — the Gong email summary serves as the sole data source, and the workflow proceeds directly to Step 4 (Human Review). See design spec: `docs/superpowers/specs/2026-03-13-gong-gmail-call-summary-extractor-design.md`.

- [ ] **Step 2: Commit**

```bash
git add outputs/post-call-follow-up-generation-building-block-spec.md
git commit -m "docs: add Phase 1 integration note to post-call follow-up spec"
```

### Task 4: Register the skill (if applicable)

If the user's workflow includes registering building blocks in Notion:

- [ ] **Step 1: Check if registration is needed**

Ask the user: "Would you like to register this skill in the Notion AI Building Blocks database?"

- [ ] **Step 2: Register if requested**

Use the `ai-registry:registering-building-blocks` skill to add the new skill to Notion.

- [ ] **Step 3: Commit any changes**

If registration produced any file changes, commit them.
