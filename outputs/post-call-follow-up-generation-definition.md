# Workflow Definition: Post-Call Follow-Up Generation

---

## Scenario Metadata

| Field | Value |
|---|---|
| **Workflow Name** | Post-Call Follow-Up Generation |
| **Description** | Takes a Gong call transcript and summary, independently analyzes the conversation, cross-checks product knowledge, and generates a single ready-to-send follow-up message (email or Slack DM) addressed to the prospect or an internal stakeholder. |
| **Process Outcome** | A single, ready-to-send follow-up message with three sections: Q&A (question → answer bullets), Follow-Ups (action items), and Recommended Workflows (tailored to the prospect's needs) |
| **Trigger** | Sales call with prospect ends and the Gong transcript becomes available |
| **Type** | Augmented |
| **Business Objective** | Reduce post-call follow-up effort for the Solutions Consultant, ensure no action items or product questions are missed, and deliver a high-quality, personalized follow-up faster |
| **Current Owner** | John Fisher, Solutions Consultant |
| **Lens** | Individual |

---

## Refined Steps

### Step 1 — Retrieve Call Artifacts from Gong
- **Action**: Access the Gong call record to retrieve both the full transcript and the Gong-generated summary
- **Sub-steps**:
  1. Identify the call record (by prospect name, date, or call ID)
  2. Pull the full transcript
  3. Wait for the Gong summary to be generated (slight delay after transcript availability)
  4. Pull the Gong summary
- **Data In**: Gong call record identifier
- **Data Out**: Raw transcript, Gong summary
- **Context Needs**: Gong API access (transcript + summary)
- **Failure Modes**:
  - Transcript not yet available → wait and retry
  - Gong summary delayed → proceed with transcript analysis only, flag that comparison is pending
  - API access fails → prompt user to manually provide transcript and/or summary

---

### Step 2 — Independent Transcript Analysis
- **Action**: Claude reads the full transcript and independently generates a summary of the call and a list of action items / follow-up questions
- **Sub-steps**:
  1. Parse the transcript
  2. Identify key topics discussed (product questions, business goals, technical needs)
  3. Extract action items and follow-up questions
  4. Generate a structured call summary
- **Data In**: Full Gong transcript
- **Data Out**: Claude-generated call summary, action item list, follow-up questions list
- **Context Needs**: None beyond the transcript
- **Failure Modes**:
  - Transcript too long or poorly formatted → summarize in chunks
  - Missing speaker attribution → flag ambiguity to user

*Note: Steps 2 and 3 can run in parallel.*

---

### Step 3 — Compare to Gong Summary
- **Action**: Claude compares its independently generated summary to the Gong-generated summary and flags any discrepancies or items that appear in one but not the other
- **Sub-steps**:
  1. Identify items in Claude's summary not present in Gong's summary
  2. Identify items in Gong's summary not present in Claude's summary
  3. Produce a consolidated action item list with discrepancies highlighted
- **Data In**: Claude-generated summary (Step 2), Gong summary (Step 1)
- **Data Out**: Consolidated action item list with discrepancy flags
- **Context Needs**: None
- **Failure Modes**:
  - Gong summary not yet available → skip comparison, proceed with Claude's summary only, note to user

*Note: Steps 2 and 3 can run in parallel; Step 3 depends on both Step 1 (Gong summary) and Step 2.*

---

### Step 4 — Human Review Checkpoint
- **Action**: Present the consolidated action item list to John for review before proceeding to draft
- **Sub-steps**:
  1. Display consolidated action items and any flagged discrepancies
  2. Allow John to add, remove, or reprioritize items
  3. Allow John to call out specific items as high priority or requiring special attention
  4. Confirm the final action item list before proceeding
- **Data In**: Consolidated action item list
- **Data Out**: Confirmed, user-approved action item list
- **Context Needs**: None
- **Failure Modes**:
  - User makes no changes → proceed as-is
  - User significantly changes scope → re-confirm before proceeding

---

### Step 5 — Product Knowledge Check
- **Action**: For each question or action item requiring product knowledge, Claude queries the available product knowledge sources to generate accurate, sourced answers
- **Sub-steps**:
  1. Identify which action items require product knowledge (vs. logistics or scheduling)
  2. Query Glean as primary aggregator — but always inspect the source behind Glean's answer, never accept it at face value
  3. Apply source trust hierarchy to each Glean result:
     - Support tickets → Trustworthy ✓
     - Internal documentation → Check the date; recent = trustworthy, undated or old = suspect
     - Previous call transcripts → Highly suspect — do not use as a source
  4. Cross-reference Glean's answer against the Gem external help center; if both agree → High confidence
  5. Apply specificity check: a document directly addressing the question outranks a tangential mention
  6. Flag any questions that could not be answered with confidence as Unresolved
- **Data In**: Confirmed action item list (Step 4)
- **Data Out**: Each question paired with: answer text, source name, source type, confidence level (High / Medium / Unresolved)
- **Context Needs**:
  - Glean (primary aggregator)
  - Gem external help center (URL) — used for cross-reference validation
  - Gem internal help center (URL)
  - Google Docs documentation (access needed)
  - Highspot / Spekit (access uncertain — may come via Glean)
  - Dropbox Paper (access uncertain — may come via Glean)
- **Decision Rules**:
  - Multi-source agreement (2+ independent sources) → High confidence
  - Single source only → Medium confidence at best
  - Conflicting sources → surface both to user, do not synthesize
  - Suspect source only (previous call, undated doc) → mark Unresolved, surface source name to user
  - No source found → mark Unresolved, never hallucinate
- **Failure Modes**:
  - Source not accessible → flag the question as Unresolved with placeholder ("Following up on this separately")
  - Conflicting information across sources → surface both sources with confidence ratings to user

---

### Step 6 — Draft Follow-Up Message
- **Action**: Claude drafts a single follow-up message using the confirmed action items, sourced answers, call summary highlights, and recommended workflows
- **Sub-steps**:
  1. Determine message type (email = external prospect; Slack DM = internal)
  2. Draft Section 1 — Q&A: bullet format "Here's your question → Here's your answer"
  3. Draft Section 2 — Follow-Ups: action items from the call
  4. Draft Section 3 — Recommended Workflows: tailored workflow/solution recommendations based on prospect's stated needs
  5. Apply appropriate tone (professional for email, conversational for Slack)
- **Data In**: Confirmed action item list + sourced answers (Step 5), call summary (Step 2), prospect context from transcript
- **Data Out**: Draft follow-up message (structured, three-section format)
- **Context Needs**:
  - Recommended Workflows Library *(Needs Creation — high-value future artifact)*
  - In the absence of the library, Claude infers recommendations from transcript context
- **Failure Modes**:
  - Unanswered product questions → include flagged placeholder in Q&A section (e.g., "Following up on this separately")
  - Unclear recipient type → ask user before drafting

---

### Step 7 — Deliver Draft to Gmail or Slack
- **Action**: Push the drafted message directly into Gmail as an email draft (external) or Slack as a DM draft (internal), addressed to the appropriate recipient
- **Sub-steps**:
  1. Identify recipient (from transcript context or user input)
  2. If email: create Gmail draft addressed to prospect
  3. If Slack: create Slack DM draft to internal contact
  4. Notify user that the draft is ready for review and send
- **Data In**: Draft follow-up message (Step 6), recipient identity
- **Data Out**: Draft staged in Gmail or Slack
- **Context Needs**: Gmail MCP access, Slack MCP access
- **Failure Modes**:
  - Recipient email/Slack handle not identified → ask user before creating draft
  - Draft tool unavailable → output message as formatted text for manual copy-paste

---

## Step Sequence and Dependencies

```
Step 1 (Retrieve Gong Artifacts)
    ├── Step 2 (Claude Transcript Analysis)  ─┐
    └── Step 3 (Compare to Gong Summary)    ──┤ (parallel)
                                              ↓
                                   Step 4 (Human Review Checkpoint)
                                              ↓
                                   Step 5 (Product Knowledge Check)
                                              ↓
                                   Step 6 (Draft Follow-Up Message)
                                              ↓
                                   Step 7 (Deliver to Gmail or Slack)
```

- **Sequential**: Steps 1 → 4 → 5 → 6 → 7
- **Parallel**: Steps 2 and 3 run concurrently after Step 1; Step 3 waits for Gong summary
- **Critical Path**: Step 1 → Step 2 → Step 4 → Step 5 → Step 6 → Step 7
- **Human Gate**: Step 4 is a required checkpoint before any drafting begins

---

## Context Shopping List

| Artifact | Description | Used By Steps | Status | Notes |
|---|---|---|---|---|
| Gong call transcript | Full verbatim transcript of the prospect call | 1, 2 | Exists | Gong API access needed |
| Gong call summary | Auto-generated summary from Gong | 1, 3 | Exists | Gong API access needed; arrives after transcript |
| Gem external help center | Public-facing product documentation | 5 | Exists | URL to be provided |
| Gem internal help center | Internal product knowledge base | 5 | Exists | URL to be provided; internal access only |
| Glean | Product knowledge aggregator connecting internal sources | 5 | Exists | Integration setup needed; primary source for product Q&A |
| Google Docs documentation | Internal product and process docs | 5 | Exists | Access needed |
| Highspot / Spekit | Sales enablement content | 5 | Exists | Not live yet; access uncertain — may come via Glean |
| Dropbox Paper | Additional internal documentation | 5 | Exists | Access uncertain — may come via Glean |
| Recommended Workflows Library | Library of common Gem workflow templates mapped to customer use cases | 6 | **Needs Creation** | High-value artifact; currently inferred on the fly by Solutions Consultant |
| Gmail draft access | Ability to create email drafts in Gmail | 7 | Exists | Gmail MCP connected |
| Slack DM access | Ability to send or draft Slack messages | 7 | Exists | Slack MCP connected |
