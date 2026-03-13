# AI Building Block Spec: Prep Notes to Demo Plan

---

## Context

John Fisher (Solutions Consultant) receives sales rep prep notes via Slack DM or email (Google Doc) before prospect calls, then manually reformats them into structured demo plans. This workflow automates the end-to-end process: extract notes, match to the Call Prep calendar block, generate a structured demo plan via the existing `prep-note-summary` skill, save it to Notion, link it to the calendar event, and confirm via Slack.

---

## Scenario Summary

| Field | Value |
|---|---|
| **Workflow Name** | Prep Notes to Demo Plan |
| **Lens** | Individual |
| **Trigger** | Sales rep sends prep notes via Slack DM or email (Google Doc link) — user-invoked |
| **Outcome** | Structured demo plan page in Notion, linked from the Call Prep calendar event, with Slack confirmation |
| **Platform** | Claude Code (markdown agent file) |
| **Definition File** | `outputs/prep-notes-to-demo-plan-definition.md` |

---

## Architecture Decisions

| Dimension | Decision | Rationale |
|---|---|---|
| **Platform** | Claude Code (markdown agent file) | User's standard platform; no code required |
| **Tools needed** | Slack MCP, Gmail MCP, Google Calendar MCP, Notion MCP | All connected; no new integrations required |
| **Trigger type** | User-invoked | User pastes or forwards rep notes to start the agent |
| **Involvement mode** | Automated | No human gate — fixed sequence with bounded AI judgment in extraction and generation steps only |
| **Integration availability** | All available | Every tool is already connected via MCP |

---

## Autonomy Assessment

**Workflow-level autonomy: Guided**

The flow follows a fixed 6-step sequence (extract → match → generate → save → link → notify). The AI makes bounded judgments in two places: Step 1 (extracting company name, rep name, and raw text from unstructured input) and Step 3 (generating the demo plan via the `prep-note-summary` skill). All other steps are deterministic tool calls. The path does not change based on AI output — it always runs the same sequence. This is **Guided**, not Deterministic (extraction and generation require AI judgment) and not Autonomous (no self-directed path changes or backtracking).

---

## Orchestration Mechanism

**Recommendation: Agent (Automated)**

- Tool use is required at Steps 1, 2, 4, 5, and 6 (Slack, Gmail, Google Calendar, Notion)
- Sequential state passing is required (notes → company name → calendar event → demo plan → Notion URL → notification)
- A Skill-Powered Prompt cannot manage tool calls or multi-step state
- No human gate — the agent runs end-to-end once invoked

**Pattern:** Single agent, same as `prep-block-on-booking` — linear sequence, no sub-agents, no parallel branches.

---

## Step-by-Step Decomposition

| Step | Name | Autonomy | Building Block | Tools | Skill Candidate | Human Gate |
|------|------|----------|----------------|-------|-----------------|------------|
| 1 | Receive and Extract Prep Notes | Guided | Agent + MCP | Slack MCP, Gmail MCP | No | No |
| 2 | Match to Call Prep Block | Deterministic | Agent + MCP | Google Calendar MCP | No | No |
| 3 | Generate Demo Plan | Guided | Skill | None (text processing) | **Exists: prep-note-summary** | No |
| 4 | Save Demo Plan to Notion | Deterministic | Agent + MCP | Notion MCP | No | No |
| 5 | Link Demo Plan to Call Prep Block | Deterministic | Agent + MCP | Google Calendar MCP | No | No |
| 6 | Notify User via Slack | Deterministic | Agent + MCP | Slack MCP | No | No |

---

## Skill Candidates

### Existing Skill — prep-note-summary

- **Location**: `skills/prep-note-summary/SKILL.md`
- **Purpose**: Converts free-form sales rep prep notes into a structured one-pager demo plan (Markdown)
- **Inputs**: Raw prep notes text
- **Outputs**: Structured Markdown demo plan with sections: Company Overview, Tech Stack, Key Contacts, Challenges, Desired Solutions, Competitive Landscape, Key Risks, Discovery Questions, Recommended Demo Flow
- **No changes needed**: The skill already handles sparse notes (adds "Info to Gather Before the Call" section) and omits empty sections

No new skills are required for this workflow. All other steps are tool calls or direct text operations.

---

## Agent Configuration

### Orchestrator: Prep Notes to Demo Plan Agent

| Component | Specification |
|---|---|
| **Name** | prep-notes-to-demo-plan |
| **Description** | Processes sales rep prep notes (from Slack DM or email/Google Doc) into a structured demo plan, saves it to Notion, links it to the Call Prep calendar event, and sends a Slack confirmation. |
| **Goal / Invocation** | User provides rep notes (pastes text, forwards Slack message, or shares Google Doc link) |
| **Model** | sonnet — fast execution, sufficient reasoning for text extraction and template matching |
| **Skills** | prep-note-summary |
| **Tools** | Slack MCP, Gmail MCP, Google Calendar MCP, Notion MCP |

**Instructions summary:**

1. **Extract** — Parse the input to get raw prep notes text, company name, and rep name. Handle Slack DM text directly; for email/Google Doc, extract the document content.
2. **Match** — Search upcoming Google Calendar events for "Call Prep - [Company Name]". Exactly one match → use it. Multiple → present options. None → flag user.
3. **Generate** — Invoke the `prep-note-summary` skill with the raw notes text. Receive structured Markdown demo plan.
4. **Save** — Create a new page in the Pre-Call Demo Prep Library (Notion database). Set title to company name. Populate properties: Rep Name, Call Date (from Step 2), Source (Slack or Email). Populate page body with the demo plan Markdown. Capture the Notion page URL.
5. **Link** — Update the matched Call Prep calendar event description to include "Demo Plan: [Notion URL]".
6. **Notify** — Send a Slack DM to John Fisher with: Company Name, Rep Name, Call Date, Source, and direct link to the Notion demo plan page.

**Error handling:**

- Google Doc permissions block access → flag user via Slack DM
- No company name identifiable → ask user to clarify
- No matching Call Prep block → ask user to identify the correct event
- Notion or Calendar write fails → retry once, then flag user via Slack DM

---

## Step Sequence and Dependencies

```
Step 1 (Receive and Extract Prep Notes)
    ↓
Step 2 (Match to Call Prep Block)
    ↓
Step 3 (Generate Demo Plan — prep-note-summary skill)
    ↓
Step 4 (Save Demo Plan to Notion)
    ↓
Step 5 (Link Demo Plan to Call Prep Block)
    ↓
Step 6 (Notify User via Slack)
```

- **All sequential**: Each step depends on output from the previous step
- **Critical path**: 1 → 2 → 3 → 4 → 5 → 6
- **No parallel branches**

---

## Model Recommendation

**sonnet** for all steps.

- Steps 1 and 3 require text extraction and generation — Sonnet handles these well
- Steps 2, 4, 5, 6 are deterministic tool calls — Sonnet is sufficient; Haiku could optimize cost in a future iteration
- Opus is not needed — this workflow is extraction + template generation + tool calls, not deep research

---

## Context Inventory

| Artifact | Status | Risk |
|---|---|---|
| Slack MCP | Connected | Low |
| Gmail MCP | Connected | Low |
| Google Calendar MCP | Connected | Low |
| Notion MCP | Connected | Low |
| prep-note-summary skill | Exists at `skills/prep-note-summary/SKILL.md` | Low |
| Pre-Call Demo Prep Library (Notion DB) | **Needs Creation** | Low — straightforward database |
| Prep block naming convention ("Call Prep - [Company Name]") | **Needs Update** in `prep-block-on-booking` | Medium — dependency on another workflow |

---

## Prerequisites

### 1. Create Pre-Call Demo Prep Library (Notion Database)

Create a new Notion database with these properties:

| Property | Type | Notes |
|---|---|---|
| **Company Name** | Title | Primary identifier |
| **Rep Name** | Text | Name of the sales rep who sent the notes |
| **Call Date** | Date | Date of the prospect call (from the matched calendar event) |
| **Source** | Select | Options: `Slack`, `Email` |

Page body will contain the full structured demo plan Markdown.

### 2. Update prep-block-on-booking Naming Convention

Update `agents/prep-block-on-booking.md` and `skills/prep-block-rule-engine/SKILL.md` so that Call Prep blocks are named **"Call Prep - [Company Name]"** instead of just "Call Prep". The company name should be extracted from the prospect call event title.

Without this, Step 2 matching will be unreliable — there's no way to associate a generic "Call Prep" event with a specific company.

---

## Recommended Implementation Order

**Phase 1 — Prerequisites**
1. Create the Pre-Call Demo Prep Library Notion database
2. Update `prep-block-on-booking` agent and `prep-block-rule-engine` skill to name blocks "Call Prep - [Company Name]"

**Phase 2 — Build**
3. Build the `prep-notes-to-demo-plan` agent markdown file (`agents/prep-notes-to-demo-plan.md`)

**Phase 3 — Test**
4. Manually invoke the agent with sample rep notes
5. Verify: Notion page created with correct properties and demo plan content
6. Verify: Call Prep calendar event updated with Notion link in description
7. Verify: Slack notification received with correct fields and link

---

## Where to Run

**Claude Code (CLI)** — user invokes the `prep-notes-to-demo-plan` agent by name when rep notes arrive. No scheduled/automated execution at launch. Future enhancement: trigger automatically when a Slack DM from a known rep is received.
