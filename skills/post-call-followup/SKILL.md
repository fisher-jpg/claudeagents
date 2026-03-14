---
name: post-call-followup
description: >
  Generates a post-call follow-up message from a Gong call summary. Identifies
  product questions, researches answers from the Gem help center, and delivers
  the follow-up via Gmail draft (external) or Slack message (internal). Use when
  the user says "follow up on that call", "draft a follow-up", "send the
  customer answers", "post-call follow-up", or has just run the
  gong-call-summary-extractor and wants to act on the results. Also trigger
  when the user asks to answer product questions from a call, research help
  center answers for a prospect, or draft a Q&A email after a demo.
user-invocable: true
metadata:
  author: handsonai
  version: "1.0.0"
---

# Post-Call Follow-Up Generator

Take a Gong call summary, identify product questions, research answers from the
Gem external help center, and deliver a formatted follow-up message.

## When to use this skill

- User has a Gong call summary and wants to draft a follow-up
- User asks to answer product questions that came up on a call
- User wants to send a customer Q&A based on a recent call
- User ran `gong-call-summary-extractor` and wants to act on the output
- User asks to "follow up on that call" or "draft a follow-up"

## Tools required

- `WebFetch` — search Gem external help center (help.gem.com)
- `gcal_list_events` — look up calendar invite for attendee email addresses
- `gmail_create_draft` — create external follow-up email draft
- `slack_send_message` — send internal follow-up to AE/rep

## Input

This skill consumes the structured Markdown output from the `gong-call-summary-extractor` skill. The expected format is:

```
# Call Summary: [Call Title]

## Call Details
- **Company:** [Company Name]
- **Date:** [Date]
- **Duration:** [Duration]

## Participants
### Gem
- [Name] ([role]) · [Title]

### [Customer Company]
- [Name] · [Title]

## Associated Deal (optional)
...

## Key Points
1. [numbered items]

## Next Steps
1. [numbered items]
```

If the user pastes this output directly, use it as-is. If the user references a
call but hasn't extracted the summary yet, tell them to run the
`gong-call-summary-extractor` skill first.

## Step-by-step process

### Step A — Classify product questions

Read the `## Key Points` and `## Next Steps` sections from the call summary.

**Identify product questions.** A product question is any item where:
- A prospect asked about a Gem capability, feature, integration, or limitation
- A Gem team member committed to follow up with product information
- Something was discussed but the answer was uncertain or incomplete on the call

**Not product questions — skip these:**
- Logistics (scheduling next call, sending pricing)
- Customer-side commitments (legal review, internal discussions)
- Administrative actions (sending a draft order form, sharing screenshots)

**Rephrase each product question** as a clear, answerable question. Transform
summary statements into searchable questions.

Example transformation:
- Key point: "Discussed Rippling integration capabilities and two-way sync"
  → Question: "Does Gem support two-way integration with Rippling for job openings?"
- Next step: "John to follow up on whether job ID prefixes can vary by role type"
  → Question: "Can job ID prefixes be customized differently per role type?"

If no product questions are found in the call summary, report:
"No product questions identified in this call summary."
Then offer to let the user add questions manually. If the user declines, stop here.

### Step B — Human gate

Present the classified questions and ask the user to confirm before proceeding.
**Do not continue until the user responds.**

Present in this format:

```
I found [N] product questions from this call:

1. [Question 1]
2. [Question 2]
3. [Question 3]

Before I research answers:
- Edit this list if needed (add, remove, or rephrase any questions)
- Is this follow-up **internal** (Slack to AE/rep) or **external** (Gmail to customer)?
```

Wait for the user's response. The user may:
- Confirm the list as-is and specify delivery target
- Remove questions they don't need answered
- Add questions that were missed
- Rephrase questions for clarity
- Specify internal or external delivery

After this gate, you have: a confirmed question list + delivery target (internal or external).

### Step C — Research answers from the Gem help center

For each confirmed question, search the Gem external help center.

**Source:** help.gem.com only. Do NOT search the internal Notion help center, GitHub, or any other source. All answers must be safe to include in customer-facing messages.

**Research process for each question:**

1. Extract 2-4 key terms from the question (e.g., "Rippling integration" from "Does Gem support two-way integration with Rippling?")
2. Fetch the search results page:
   ```
   https://help.gem.com/hc/en-us/search?utf8=%E2%9C%93&query=[key terms]
   ```
   Use `WebFetch` to retrieve this page.
3. From the search results, identify the top 1-2 most relevant article links.
4. Fetch each relevant article page with `WebFetch` and extract the answer.
5. If the first search yields no relevant results, rephrase the query with alternative terms and retry once (e.g., try "Rippling sync" instead of "Rippling integration").

**Research questions sequentially** — one at a time, not in parallel.

**Track results for each question:**

```
Question: [the question]
Answer: [extracted answer text, concise but complete]
Source: [full URL of the help center article]
Status: Resolved
```

or, if no answer was found after the retry:

```
Question: [the question]
Answer: [none found]
Source: [none]
Status: Unresolved
```

**Important:**
- Only include information that is explicitly stated in the help center article. Do not infer, extrapolate, or combine information from multiple articles to construct an answer.
- Keep answers concise — 2-4 sentences that directly address the question.
- Preserve the source URL for each answer (used in the external message format).

### Step D — Draft the follow-up message

Based on the delivery target chosen at the human gate, draft the message using the appropriate format. Only include resolved questions — unresolved questions are handled separately after delivery.

**If all questions are unresolved:** Do not draft a message. Report that no answers were found and list all questions for manual follow-up. Stop here.

**External format (Gmail draft to customer):**

```
Hi [Customer First Name(s)],

Thanks for taking the time to connect on [call date from ## Call Details]. Following up on a few questions that came up during our conversation:

**[Question 1]**
[Answer text — 2-4 sentences]
Learn more: [help center article URL]

**[Question 2]**
[Answer text — 2-4 sentences]
Learn more: [help center article URL]

Let me know if any other questions come up.

Best,
John
```

External format rules:
- Address the customer by first name. If multiple customer participants, use all first names (e.g., "Hi Ryan and Sarah,").
- "Learn more" links point to the help.gem.com article where the answer was found.
- Tone: professional, concise, helpful. Not overly formal.
- Only include resolved questions. Do not mention unresolved questions in the message.

**Internal format (Slack message to AE/rep):**

```
Product Q&A from [Call Title]:

**[Question 1]**
[Answer text]

**[Question 2]**
[Answer text]
```

Internal format rules:
- No greeting, no sign-off — just the Q&A pairs.
- No "Learn more" links — source links are omitted from internal messages.
- Tone: direct, concise.
- Only include resolved questions.

### Step E — Deliver the message

**External delivery (Gmail draft):**

1. **Find recipient email addresses.** The Gong extractor output has participant names but not email addresses. To find emails:
   - Search for the original calendar invite using `gcal_list_events`. Match by call title and date from the `## Call Details` section.
   - Extract attendee email addresses from the calendar invite.
   - Default: include **all customer participants** as recipients unless the user specified otherwise at the human gate.
   - If the calendar invite can't be found, ask the user to provide email addresses.
   - If the calendar invite is found but missing some attendee emails, use what's available and ask the user for the rest.

2. **Create the Gmail draft** using `gmail_create_draft`:
   - **To:** customer email address(es)
   - **Subject:** `Following up — [Call Title]`
   - **Body:** the external format message from Step D

   The draft is NOT sent automatically. The user reviews and sends manually.

3. If `gmail_create_draft` fails, output the formatted message as text so the user can copy-paste it manually.

**Internal delivery (Slack message):**

1. **Identify the recipient.** Look for the Gem AE or sales rep in the Participants section (non-SC roles). If multiple Gem participants, ask which one to send to. If the Slack handle can't be determined from the name, ask the user.

2. **Send the Slack message** using `slack_send_message` as a DM to the identified recipient.

   This is sent directly, not as a draft. Internal messages are lower-stakes (going to a colleague) and the user already reviewed the question list at the human gate.

3. If `slack_send_message` fails, output the formatted message as text so the user can copy-paste it manually.

**After delivery, report unresolved questions (if any):**

```
Delivered to [Gmail draft / Slack DM to [Name]].

These questions couldn't be answered from the help center — you'll need to handle these separately:
1. [Unresolved question 1]
2. [Unresolved question 2]
```

If all questions were resolved, just confirm delivery:

```
Delivered to [Gmail draft / Slack DM to [Name]]. All questions answered.
```

## Guidelines

- **Only use help.gem.com for product answers.** Do NOT search the internal Notion help center, GitHub, or any other source. All answers must be safe for customer-facing messages.
- **Never fabricate answers.** Every answer must come directly from a help center article. If you can't find it, mark the question as unresolved.
- **Preserve the user's control.** The human gate exists so the user can shape the output. Don't skip it, rush through it, or proceed without explicit confirmation.
- **Keep answers concise.** 2-4 sentences per answer. The customer wants a quick, useful response — not a knowledge base dump.
- **Don't include unresolved questions in the message.** They go in the post-delivery report, not the follow-up message itself.
- **This skill only handles product questions.** Action items, scheduling follow-ups, and other call outcomes are out of scope for Phase 1.

## Error handling

| Scenario | What to do |
|---|---|
| No product questions found in call summary | Report "No product questions identified." Offer to let the user add questions manually. |
| WebFetch to help.gem.com fails | Warn the user. Mark all questions as unresolved. Offer to draft a message with placeholder text. |
| All questions are unresolved | Report that no answers were found. List all questions for manual follow-up. Do not draft a message. |
| Calendar invite not found for email lookup | Ask the user to provide recipient email addresses. |
| Calendar invite found but missing some attendee emails | Use what's available, ask for the rest. |
| Slack recipient can't be identified | Ask the user for the recipient's Slack handle or name. |
| Gmail draft creation fails | Output the formatted message as text for manual copy-paste. |
| Slack message send fails | Output the formatted message as text for manual copy-paste. |
| Call summary input is missing expected sections | Warn the user which sections are missing. Proceed with what's available if Key Points or Next Steps exist. If both are missing, ask the user to provide the call summary. |

## Phase 1 scope

This skill implements a simplified version of the full post-call follow-up workflow:

| Full Workflow Step | Phase 1 Status |
|---|---|
| Step 1: Retrieve Gong Artifacts | Handled by `gong-call-summary-extractor` (separate skill) |
| Step 2: Independent Transcript Analysis | Skipped — no raw transcript in Phase 1 |
| Step 3: Compare to Gong Summary | Skipped — no independent analysis to compare |
| Step 4: Human Review Checkpoint | Implemented as Step B |
| Step 5: Product Knowledge Check | Implemented as Step C (external help center only) |
| Step 6: Draft Follow-Up Message | Implemented as Step D |
| Step 7: Deliver to Gmail or Slack | Implemented as Step E |

## What this skill does NOT do (Phase 1)

- Analyze raw call transcripts (Phase 2 — requires Gong MCP)
- Compare Claude's analysis to Gong's summary (Phase 2)
- Track or include action items in the follow-up (future extension)
- Search the Gem internal help center (removed — too risky for customer messages)
- Search GitHub repositories for product answers (future extension)
- Handle unresolved questions automatically (future workflow)
- Chain with the Gong extractor automatically (future agent orchestration)
