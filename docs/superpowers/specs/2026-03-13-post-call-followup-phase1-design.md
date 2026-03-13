# Design Spec: Post-Call Follow-Up Generation (Phase 1)

## Overview

A single orchestrator skill that takes a Gong call summary (from the `gong-call-summary-extractor` skill), identifies product questions, researches answers using the Gem external help center, and delivers a formatted follow-up message via Gmail (external) or Slack (internal).

**Phase 1 scope:** Product question answering only. Action item tracking, internal help center search, and GitHub source integration are deferred to future phases.

---

## Problem Statement

After every prospect call, John Fisher (Solutions Consultant at Gem) manually reviews the Gong summary, identifies product questions that came up, searches the help center for answers, and drafts a follow-up message. This workflow automates that process: Claude classifies the questions, researches answers, drafts the message, and delivers it — with a human review gate before research begins.

---

## Architecture

### End-to-End Flow

```
Step 1: gong-call-summary-extractor (existing skill)
    → outputs key points, next steps, participants, deal info
                    ↓
Step 2: post-call-followup (this skill)
    a. Classify product questions from key points + next steps
    b. HUMAN GATE: confirm questions + choose internal vs. external
    c. Research answers via help.gem.com (WebFetch)
    d. Draft message in the appropriate format
    e. Deliver: Gmail draft (external) or Slack message (internal)
```

### Invocation

The two skills are invoked separately in Phase 1:
1. User runs `gong-call-summary-extractor` → gets structured call summary
2. User runs `post-call-followup` → passes the call summary output

In a future phase, an orchestrator agent can chain these automatically.

### Tools Required

| Tool | Purpose | Status |
|---|---|---|
| WebFetch | Search Gem external help center (help.gem.com) | Configured |
| Google Calendar MCP (`gcal_list_events`) | Look up calendar invite to find attendee email addresses | Configured |
| Gmail MCP (`gmail_create_draft`) | Create external follow-up email draft | Configured |
| Slack MCP (`slack_send_message`) | Send internal follow-up to AE/rep | Configured |

---

## Input Contract

This skill consumes the structured Markdown output from the `gong-call-summary-extractor` skill. The expected sections and how each step uses them:

| Gong Extractor Section | Used By | Purpose |
|---|---|---|
| `## Key Points` (numbered list) | Step A | Scan for product questions |
| `## Next Steps` (numbered list) | Step A | Scan for product questions |
| `## Call Details` (Company, Date, Duration) | Step D | Call date for external message greeting |
| `## Participants` (Gem + Customer groups) | Step E | Recipient identification |
| `## Associated Deal` (optional) | Not used | Available for future extensions |

The call title is extracted from the `# Call Summary: [Title]` heading.

**Note on participants:** The Gong extractor output includes participant names and titles but NOT email addresses. Email addresses can be found on the original calendar invite for the call (via Google Calendar MCP). For internal Slack delivery, the skill will attempt to match Gem participant names to Slack users but may need to ask.

---

## Step-by-Step Design

### Step A: Product Question Classification

**Input:** The `## Key Points` and `## Next Steps` sections from the Gong extractor output.

**Classification logic:**

A product question is any item where:
- A prospect asked about a Gem capability, feature, integration, or limitation
- A Gem team member committed to follow up with product information
- Something was discussed but the answer was uncertain or incomplete on the call

Items that are NOT product questions:
- Logistics (scheduling next call, sending pricing)
- Customer-side commitments (legal review, internal discussions)
- Administrative actions (sending a draft order form, sharing screenshots)

**Output:** A numbered list of product questions, rephrased as clear, answerable questions. For example:

```
1. Does Gem support two-way integration with Rippling for job openings?
2. Can job ID prefixes be customized differently per role type?
3. Is the offer generation page mobile-optimized?
4. Can an admin user with super user rights override and approve offers?
```

Each question is derived from the key points/next steps but rephrased from a summary statement into a clear question that can be searched against the help center.

---

### Step B: Human Gate

After classification, the skill pauses and presents:

```
I found [N] product questions from this call:

1. Does Gem support two-way integration with Rippling?
2. Can job ID prefixes be customized per role type?
3. Is the offer generation page mobile-optimized?
4. Can an admin with super user rights override offers?

- Edit this list (add, remove, or rephrase any questions)
- Is this follow-up internal (Slack to AE/rep) or external (Gmail to customer)?
```

**The skill waits for the user's response before proceeding.**

The user can:
- Confirm the list as-is
- Remove questions they don't need answered
- Add questions that were missed
- Rephrase questions for clarity
- Specify internal or external delivery

**Gate output:** Confirmed question list + delivery target (internal or external).

---

### Step C: Product Knowledge Research

For each confirmed question, search the Gem external help center.

**Source:** help.gem.com (via WebFetch)

**Research process per question:**

1. Construct a search query from the question's key terms (e.g., "Rippling integration" from "Does Gem support two-way integration with Rippling?")
2. Fetch `https://help.gem.com/hc/en-us/search?utf8=%E2%9C%93&query=[search terms]` via WebFetch to find relevant articles
3. Read the top 1-2 most relevant article pages to extract the answer
4. If the first search yields no relevant results, rephrase the query with alternative terms and retry once (e.g., try "Rippling sync" instead of "Rippling integration")
5. Extract the answer text and the article URL

Research questions sequentially (one at a time) to avoid overwhelming the help center with concurrent requests.

**Research output per question:**

```
Question: Does Gem support two-way integration with Rippling?
Answer: Yes, Gem supports a two-way integration with Rippling...
Source: https://help.gem.com/en-us/articles/rippling-integration
Status: Resolved
```

or

```
Question: Can job ID prefixes vary per role type?
Answer: [none found]
Source: [none]
Status: Unresolved
```

**Unresolved questions:** Collected separately and presented to the user after delivery. Not included in the follow-up message. A separate workflow for handling unresolved questions is a planned future extension.

---

### Step D: Message Drafting

Based on the delivery target chosen at the human gate, draft the message in the appropriate format.

**External format (Gmail draft to customer):**

```
Hi [Customer Name],

Thanks for taking the time to connect on [call date]. Following up on a few
questions that came up during our conversation:

**[Question 1]**
[Answer text]
Learn more: [help center article link]

**[Question 2]**
[Answer text]
Learn more: [help center article link]

Let me know if any other questions come up.

Best,
[Your name]
```

- "Learn more" links point to the help.gem.com article where the answer was found
- Only resolved questions are included in the message
- Tone: professional, concise

**Internal format (Slack message to AE/rep):**

```
Product Q&A from [Call Title]:

**[Question 1]**
[Answer text]

**[Question 2]**
[Answer text]
```

- No greeting, no call framing — just Q&A pairs
- Source links omitted from internal messages
- Tone: direct, concise

---

### Step E: Delivery

**External (Gmail):**
- Use `gmail_create_draft` to create an email draft
- Recipient: customer's email address (extracted from the call summary's participants section if available; otherwise ask the user)
- Subject line: "Following up — [Call Title]"
- The draft is NOT sent automatically — the user reviews and sends manually

**Internal (Slack):**
- Use `slack_send_message` to send a DM
- Recipient: the AE or sales rep from the call (identified from participants; otherwise ask the user)
- Sent directly (not a draft) — this is intentional: internal messages are lower-stakes (going to a colleague, not a customer), and the user already reviewed and confirmed the question list at the human gate. If a draft workflow is preferred in the future, this can be changed to `slack_send_message_draft`.

**After delivery, present unresolved questions (if any):**

```
Delivered to [Gmail draft / Slack].

These questions couldn't be answered from the help center — handle separately:
1. [Unresolved question 1]
2. [Unresolved question 2]
```

---

## Recipient Identification

The skill extracts recipient information from the call summary's Participants section:

**External delivery:**
- **Default: include all customer participants.** Unless the call summary indicates the follow-up should go to specific people only, all customer participants from the Participants section should be recipients.
- **If the call explicitly states a narrower audience** (e.g., "send the follow-up just to Ryan"), include only those named individuals. Others may be CC'd at the user's discretion.
- **Email address lookup:** The Gong extractor output includes names but not email addresses. To find emails, search for the original calendar invite via Google Calendar MCP (match by call title and date from `## Call Details`). Extract attendee email addresses from the invite. If the calendar invite can't be found, ask the user to provide email addresses.

**Internal delivery:**
- Look for the Gem AE or sales rep from the Participants section (non-SC roles)
- If multiple Gem participants, ask which one to send to
- If the Slack handle can't be determined, ask the user

---

## Error Handling

| Scenario | Behavior |
|---|---|
| No product questions found in the call summary | Report "No product questions identified in this call summary." Offer to let the user add questions manually. |
| WebFetch to help.gem.com fails | Warn the user. Mark all questions as unresolved. Still offer to draft a message with placeholder text. |
| All questions are unresolved | Report that no answers were found. List all questions for manual follow-up. Do not draft a message. |
| Calendar invite not found for email lookup | Ask the user to provide recipient email addresses |
| Calendar invite found but missing some attendee emails | Use what's available, ask the user for the rest |
| Slack recipient can't be identified | Ask the user to provide the recipient's Slack handle or name |
| Gmail draft creation fails | Output the formatted message as text for manual copy-paste |
| Slack message send fails | Output the formatted message as text for manual copy-paste |

---

## Relationship to Existing Specs

| Spec | Relationship |
|---|---|
| `gong-call-summary-extractor` skill | Upstream — provides the call summary input for this skill |
| `post-call-follow-up-generation-building-block-spec.md` | Parent spec — this Phase 1 skill implements a simplified version (Steps 4-7, product questions only) |
| `post-call-follow-up-generation-definition.md` | Detailed step definitions — this skill draws from Steps 5, 6, and 7 |

### Mapping to Original Spec Steps

| Original Step | Phase 1 Status |
|---|---|
| Step 1: Retrieve Gong Artifacts | Handled by `gong-call-summary-extractor` skill |
| Step 2: Independent Transcript Analysis | Skipped (no raw transcript in Phase 1) |
| Step 3: Compare to Gong Summary | Skipped (no independent analysis to compare) |
| Step 4: Human Review Checkpoint | Implemented as Step B (scoped to product questions + delivery choice) |
| Step 5: Product Knowledge Check | Implemented as Step C (external help center only) |
| Step 6: Draft Follow-Up Message | Implemented as Step D (Q&A format only, no action items) |
| Step 7: Deliver to Gmail or Slack | Implemented as Step E |

---

## Implementation Location

- **Skill file:** `skills/post-call-followup/SKILL.md`
- **Follows existing pattern:** Same YAML frontmatter format as other skills in the project
- **Output behavior:** The skill's primary output is the delivered message (Gmail draft or Slack message). Unresolved questions are presented in the conversation after delivery.

---

## What This Skill Does NOT Do (Phase 1)

- Analyze raw call transcripts (Phase 2 — requires Gong MCP)
- Compare Claude's analysis to Gong's summary (Phase 2)
- Track or include action items in the follow-up (future extension)
- Search the Gem internal help center (removed — too risky for auto-inclusion in messages)
- Search GitHub repositories for product answers (future extension)
- Handle unresolved questions automatically (future workflow)
- Chain with the Gong extractor automatically (future agent orchestration)

---

## Future Extensions

| Extension | Description | Trigger |
|---|---|---|
| Action items in external messages | Add Gem-side and customer-side action item recap to external follow-ups | User request |
| Internal help center as a source | Add Notion help center search with `[Source: Internal]` flagging and review step | After guardrail design |
| GitHub as a source | Add GitHub repo search for product answers | Colleague's bot already does this |
| Unresolved questions workflow | Separate workflow for routing and tracking unanswered questions | After Phase 1 validation |
| Automatic chaining | Agent that runs Gong extractor → this skill in sequence | After Phase 1 validation |
| Colleague adoption | Modular decomposition for reuse by other SCs | After workflow is proven |
| Raw transcript analysis | Claude independently analyzes transcript, catches missed questions | Gong MCP availability |
