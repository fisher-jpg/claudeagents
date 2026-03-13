# Workflow Definition: Prep Notes to Demo Plan

---

## Scenario Metadata

| Field | Value |
|---|---|
| **Workflow Name** | Prep Notes to Demo Plan |
| **Description** | When sales rep prep notes are received (via Slack DM or email/Google Doc), processes the notes into a structured one-pager demo plan, saves it to a Notion database, and links it in the corresponding Call Prep calendar event. |
| **Process Outcome** | A structured demo plan page in Notion, linked from the Call Prep calendar event |
| **Trigger** | Sales rep sends prep notes via Slack DM or email (Google Doc) |
| **Type** | Augmented |
| **Business Objective** | Eliminate manual formatting of rep notes into demo plans; ensure every prospect call has a structured, accessible prep doc |
| **Current Owner** | John Fisher, Solutions Consultant |
| **Lens** | Individual |

---

## Refined Steps

### Step 1 — Receive and Extract Prep Notes
- **Action**: Receive sales rep prep notes and extract the raw text content
- **Sub-steps**:
  1. Notes arrive via Slack DM or email (Google Doc link)
  2. If Slack: extract the message text directly
  3. If email/Google Doc: open the Google Doc and extract the document content
  4. Identify the company name from the notes (mentioned in Slack message or Google Doc title)
  5. Identify the rep name from the sender (Slack user or email sender)
- **Data In**: Slack DM text or email with Google Doc URL
- **Data Out**: Raw prep notes text, company name, rep name, source channel (Slack or Email)
- **Context Needs**: Slack MCP (read DMs), Gmail MCP (read emails), Google Docs access (read doc content)
- **Failure Modes**:
  - Google Doc permissions block access → flag user
  - No company name identifiable → ask user to clarify

---

### Step 2 — Match to Call Prep Block
- **Action**: Find the Call Prep calendar block for the company referenced in the notes
- **Sub-steps**:
  1. Search upcoming Google Calendar events for a "Call Prep" event containing the company name in the title
  2. If exactly one match: use it
  3. If multiple matches: present options to user
  4. If no match: flag user
- **Data In**: Company name from Step 1
- **Data Out**: Matched Call Prep calendar event (event ID, date, start time)
- **Context Needs**:
  - Google Calendar MCP (read access)
  - Prep-block-on-booking workflow updated to name prep blocks "Call Prep - [Company Name]" *(not yet implemented — see Dependencies)*
- **Failure Modes**:
  - No matching Call Prep block found → ask user to identify the correct event
  - Multiple matches → ask user to pick

---

### Step 3 — Generate Demo Plan
- **Action**: Run the prep notes through the prep-note-summary skill to produce a structured one-pager demo plan
- **Sub-steps**:
  1. Pass the raw prep notes text to the prep-note-summary skill
  2. Skill processes the notes and generates a structured Markdown demo plan following the template (Company Overview, Tech Stack, Key Contacts, Challenges, Desired Solutions, Competitive Landscape, Key Risks, Discovery Questions, Recommended Demo Flow)
  3. Sections with no relevant information from the notes are omitted
  4. If notes are sparse, an "Info to Gather Before the Call" checklist is appended
- **Data In**: Raw prep notes text from Step 1
- **Data Out**: Structured Markdown demo plan
- **Context Needs**: prep-note-summary skill (exists)
- **Failure Modes**:
  - Notes are too vague to produce anything useful → generate what's possible, lean heavily on Discovery Questions and Info to Gather sections

---

### Step 4 — Save Demo Plan to Notion
- **Action**: Create a new page in the Pre-Call Demo Prep Library with the generated demo plan
- **Sub-steps**:
  1. Create a new page in the Pre-Call Demo Prep Library Notion database
  2. Set the title to the company name
  3. Populate properties: Rep Name, Call Date (from Step 2), Source (Slack or Email)
  4. Populate the page body with the structured Markdown demo plan from Step 3
  5. Capture the Notion page URL for use in subsequent steps
- **Data In**: Structured demo plan (Step 3), company name (Step 1), rep name (Step 1), call date (Step 2), source channel (Step 1)
- **Data Out**: Notion page URL
- **Context Needs**:
  - Notion MCP (write access)
  - Pre-Call Demo Prep Library database *(does not exist yet — needs creation)*
- **Failure Modes**:
  - Notion API write fails → retry once, then flag user

---

### Step 5 — Link Demo Plan to Call Prep Block
- **Action**: Add the Notion page URL to the Call Prep calendar event description
- **Sub-steps**:
  1. Take the Notion page URL from Step 4
  2. Update the Call Prep calendar event description with the link (e.g., "Demo Plan: [Notion URL]")
  3. Confirm the calendar event was updated
- **Data In**: Notion page URL (Step 4), Call Prep event ID (Step 2)
- **Data Out**: Updated Call Prep calendar event
- **Context Needs**: Google Calendar MCP (write access)
- **Failure Modes**:
  - Calendar write fails → retry once, then flag user
  - Call Prep event was deleted between Step 2 and now → flag user

---

### Step 6 — Notify User via Slack
- **Action**: Send a Slack DM confirming the demo plan is ready with key details and a link
- **Sub-steps**:
  1. Send a Slack DM to John Fisher containing:
     - Company Name
     - Rep Name
     - Call Date
     - Source (Slack or Email)
     - Direct link to the Notion demo plan page
- **Data In**: Company name (Step 1), rep name (Step 1), call date (Step 2), source (Step 1), Notion page URL (Step 4)
- **Data Out**: Slack DM sent
- **Context Needs**: Slack MCP (send DM)
- **Failure Modes**:
  - Slack send fails → retry once, then log for manual follow-up

---

## Step Sequence and Dependencies

```
Step 1 (Receive and Extract Prep Notes)
    ↓
Step 2 (Match to Call Prep Block)
    ↓
Step 3 (Generate Demo Plan)
    ↓
Step 4 (Save to Notion)
    ↓
Step 5 (Link to Call Prep Block)
    ↓
Step 6 (Notify User via Slack)
```

- **All sequential**: Each step depends on output from the previous step
- **Critical path**: 1 → 2 → 3 → 4 → 5 → 6
- **No parallel branches**
- **Mutually exclusive outcomes**: None — all steps run in the happy path

---

## Context Shopping List

| Artifact | Description | Used By Steps | Status | Notes |
|---|---|---|---|---|
| Slack MCP | Read DMs to extract prep notes; send completion notification | 1, 6 | Exists | Read + send access |
| Gmail MCP | Read emails containing Google Doc links | 1 | Exists | Read access |
| Google Docs access | Extract content from Google Docs shared by reps | 1 | Exists | Read access; may require permissions per doc |
| Google Calendar MCP | Find Call Prep blocks; update event description with Notion link | 2, 5 | Exists | Read + write access |
| Notion MCP | Create demo plan pages in the Pre-Call Demo Prep Library | 4 | Exists | Write access |
| Pre-Call Demo Prep Library (Notion DB) | Database to store generated demo plans | 4 | **Needs Creation** | Properties: Company Name (title), Rep Name (text), Call Date (date), Source (select: Slack / Email) |
| prep-note-summary skill | Generates structured demo plan from raw notes | 3 | Exists | Located at `skills/prep-note-summary/SKILL.md` |
| Prep block naming convention | Call Prep blocks named "Call Prep - [Company Name]" for reliable matching | 2 | **Needs Update** | Requires change to prep-block-on-booking workflow and agent |

### Dependencies on Other Workflows

- **prep-block-on-booking**: Must be updated to name Call Prep blocks "Call Prep - [Company Name]" instead of just "Call Prep". Without this, Step 2 matching will be unreliable. Update needed in both `agents/prep-block-on-booking.md` and `skills/prep-block-rule-engine/SKILL.md`.

### Future Enhancements (Wishlist)

- **Prospect/Client field**: Add a "Type" property (New Business / Expansion) to the Pre-Call Demo Prep Library. Requires integration with an internal system (CRM or similar) to determine whether the company is a current client or a new prospect. Not included in v1 to keep scope manageable.

### Related Workflows

- **Prep Block on Booking** — Creates the Call Prep calendar block that this workflow links the demo plan to
- **Weekly Sweep** (to be deconstructed) — Batch version of prep block creation for the full upcoming week
