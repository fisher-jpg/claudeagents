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
