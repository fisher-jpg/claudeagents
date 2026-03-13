# AI Building Block Spec: Post-Call Follow-Up Generation

---

## Context

John Fisher (Solutions Consultant at Gem) spends significant manual effort after every prospect call reviewing the Gong-generated summary, building a to-do list, doing product knowledge research, and crafting a follow-up message. This workflow automates that process: starting from the Gong call transcript, Claude independently analyzes the call, cross-checks product knowledge, and delivers a ready-to-send follow-up message (email or Slack DM) with zero copy-pasting.

---

## Scenario Summary

| Field | Value |
|---|---|
| **Workflow Name** | Post-Call Follow-Up Generation |
| **Lens** | Individual |
| **Trigger** | Sales call ends and Gong transcript becomes available (user-invoked) |
| **Outcome** | Single ready-to-send follow-up message: Q&A bullets / Follow-Ups / Recommended Workflows |
| **Platform** | Claude Code with sub-agents (markdown agent files) |
| **Definition File** | `outputs/post-call-follow-up-generation-definition.md` |

---

## Architecture Decisions

| Dimension | Decision | Rationale |
|---|---|---|
| **Platform** | Claude Code + sub-agents | User's stated preference; markdown agent files, no code required |
| **Tools needed** | Gong API, Glean, Gem help centers (x2), Google Docs, Highspot/Spekit, Dropbox Paper, Gmail MCP, Slack MCP | Extracted from Context Shopping List |
| **Trigger type** | Event-driven, user-invoked | Call ends → user pastes call ID or link to start the agent |
| **Involvement mode** | Augmented | Hard human checkpoint at Step 4; user reviews before drafting begins |
| **Integration availability** | To be researched during Construct | Gong API, Glean are the highest-risk items |

---

## Autonomy Assessment

**Workflow-level autonomy: Guided**

The overall flow follows a mostly fixed sequence (retrieve → analyze → compare → human review → research → draft → deliver). The AI makes bounded judgments within steps (classifying which items need product lookup, reconciling discrepancies, routing to email vs. Slack), but the path itself doesn't change based on AI output. The hard human gate at Step 4 ensures the AI never drafts without approval. This is **Guided** — not Deterministic (tool use + bounded AI decisions required), not Autonomous (no backtracking, no self-directed path changes).

---

## Orchestration Mechanism

**Recommendation: Agent (Augmented)**

- Tool use is required at Steps 1, 5, and 7 (Gong, Glean, Gmail/Slack)
- Multiple steps require orchestration and state passing (transcript → summary → action items → answers → draft)
- Parallel execution of Steps 2 & 3 is possible (agent dispatches both simultaneously)
- A Skill-Powered Prompt cannot manage tool calls or parallel execution

**Involvement mode: Augmented** — user invokes the agent, reviews at Step 4, then sends the final draft.

**Single orchestrator agent** for MVP. Parallelism for Steps 2 & 3 can be achieved via sub-agent dispatch. Start single-agent and add parallelism in a future iteration.

---

## Step-by-Step Decomposition

| Step | Name | Autonomy | Building Blocks | Tools | Skill Candidate | Human Gate |
|------|------|----------|-----------------|-------|-----------------|------------|
| 1 | Retrieve Gong Artifacts | Deterministic | Agent + MCP | Gong API | No | No |
| 2 | Independent Transcript Analysis | Guided | Skill | None | **Yes** | No |
| 3 | Compare to Gong Summary | Guided | Skill | None | **Yes** | No |
| 4 | Human Review Checkpoint | Human | — | None | No | **YES** |
| 5 | Product Knowledge Check | Autonomous | Skill + MCP + Context | Glean, Help Centers, Google Docs | **Yes** | No |
| 6 | Draft Follow-Up Message | Guided | Skill + Context | None | **Yes** | No |
| 7 | Deliver to Gmail or Slack | Deterministic | Agent + MCP | Gmail MCP, Slack MCP | No | No |

> **Phase 1 implementation:** Step 1 is handled by the `gong-call-summary-extractor` skill, which extracts call summaries from Gong notification emails via Gmail MCP (no Gong API required). In Phase 1, Steps 2 and 3 are bypassed — the Gong email summary serves as the sole data source, and the workflow proceeds directly to Step 4 (Human Review). See design spec: `docs/superpowers/specs/2026-03-13-gong-gmail-call-summary-extractor-design.md`.

---

## Skill Candidates

### Skill 1 — Transcript Analyzer
- **Purpose**: Parse a raw call transcript and extract a structured call summary, action item list, and product questions
- **Inputs**: Full Gong transcript (text)
- **Outputs**: Structured summary (topics covered, business goals discussed), action item list, product questions list
- **Decision logic**: Identify speaker roles (SC vs. prospect), classify each item as action item / product question / logistics / background context, flag ambiguous speaker attribution
- **Failure modes**: Transcript too long → chunk and summarize; missing speaker labels → flag to user

### Skill 2 — Summary Comparator
- **Purpose**: Compare Claude's transcript analysis to the Gong-generated summary and produce a consolidated, discrepancy-flagged action item list
- **Inputs**: Claude-generated summary (Skill 1 output), Gong summary (text)
- **Outputs**: Consolidated action item list with items tagged as: [Both] [Claude Only] [Gong Only]
- **Decision logic**: Deduplicate semantically equivalent items; flag items present in only one source; merge into single ordered list
- **Failure modes**: Gong summary unavailable → return Claude summary only with a note

### Skill 3 — Product Knowledge Resolver
- **Purpose**: For each product question in the action item list, query available knowledge sources and return sourced answers with verified source confidence
- **Inputs**: List of product questions
- **Outputs**: Each question paired with: answer text, source name, source type, confidence level (High/Medium/Unresolved)
- **Decision logic**:
  - **Prioritize Glean** as primary aggregator, but always inspect the source behind Glean's answer — do not accept Glean's response at face value
  - **Source trust hierarchy:**
    - Support tickets → Trustworthy ✓
    - Internal documentation → Check the date; recent = trustworthy, undated or old = suspect
    - Previous call transcripts → Highly suspect — do not use as a source
  - **Cross-reference rule**: Validate Glean's answer against the Gem external help center. If both agree → High confidence. Single-source only → Medium confidence at best.
  - **Multi-source agreement**: If two or more independent sources align → High confidence. Disagreement → surface the conflict to the user, do not synthesize.
  - **Specificity check**: A document that directly addresses the question outranks one that tangentially mentions it. Tangential matches → flag as lower confidence.
  - **Hard rule**: Never synthesize an answer from a suspect source (previous calls, undated docs). Mark as Unresolved and surface the source name to the user so they can decide.
  - **Never hallucinate**: If no confident source is found, mark as Unresolved
- **Failure modes**: Source unavailable → mark question as Unresolved with placeholder text ("Following up on this separately"); conflicting answers → surface both sources to user with confidence ratings

### Skill 4 — Follow-Up Drafter
- **Purpose**: Draft a single follow-up message in three sections using confirmed action items, sourced answers, and call context
- **Inputs**: Confirmed action item list, sourced answers (Skill 3 output), call summary highlights, message type (email vs. Slack), recipient name
- **Outputs**: Formatted draft message with three sections: Q&A bullets / Follow-Ups / Recommended Workflows
- **Decision logic**: Email = professional tone, full formatting; Slack = conversational, concise; Q&A format = "**[Question]** → [Answer]"; Recommended Workflows inferred from transcript until library exists; Unresolved questions use placeholder text
- **Failure modes**: Recipient type unclear → ask user before drafting; no product questions → omit Q&A section

---

## Agent Configuration

### Orchestrator: Post-Call Follow-Up Agent

| Component | Specification |
|---|---|
| **Name** | post-call-followup |
| **Description** | Analyzes a Gong call transcript, cross-checks product knowledge, and delivers a ready-to-send follow-up message (email or Slack DM). Invoked after a sales call ends. |
| **Goal / Invocation** | User runs the agent and provides a Gong call ID or link |
| **Model** | claude-sonnet-4-6 (reasoning + tool use balance) |
| **Tools** | Gong API (MCP or manual input fallback), Glean, Gem help centers, Google Docs, Gmail MCP, Slack MCP |

**Instructions summary:**
1. Retrieve transcript + Gong summary from Gong (fallback: ask user to paste)
2. Run Transcript Analyzer skill → run Summary Comparator skill
3. Present consolidated action items to user — wait for confirmation before proceeding
4. Run Product Knowledge Resolver skill on all product questions
5. Run Follow-Up Drafter skill with confirmed items + answers
6. Identify recipient and route draft to Gmail (email) or Slack (DM)
7. Notify user the draft is ready

---

## Step Sequence and Dependencies

```
Step 1 (Retrieve Gong Artifacts)
    ├── Step 2 (Transcript Analyzer skill)   ─┐
    └── [Gong summary available]             ─┤→ Step 3 (Summary Comparator skill)
                                               ↓
                              Step 4 ── HUMAN GATE (review + confirm action items)
                                               ↓
                              Step 5 (Product Knowledge Resolver skill)
                                               ↓
                              Step 6 (Follow-Up Drafter skill)
                                               ↓
                              Step 7 (Deliver → Gmail or Slack MCP)
```

---

## Integration Research Needed (Construct Phase)

| Tool | Used For | Steps | Risk |
|---|---|---|---|
| **Gong API** | Retrieve transcript + summary | 1 | High — API access must be confirmed; fallback = manual paste |
| **Glean** | Primary product knowledge aggregator | 5 | High — MCP or API integration needed |
| **Gem internal help center** | Product knowledge validation | 5 | Medium — internal URL + auth needed |
| **Gem external help center** | Product knowledge validation | 5 | Low — public URL |
| **Google Docs** | Product/process documentation | 5 | Medium — Google Drive MCP or manual access |
| **Highspot / Spekit** | Sales enablement content | 5 | High — not live yet; may come via Glean |
| **Dropbox Paper** | Internal documentation | 5 | Medium — uncertain; may come via Glean |
| **Gmail MCP** | Email draft creation | 7 | Low — already connected |
| **Slack MCP** | Slack DM draft | 7 | Low — already connected |

---

## Model Recommendation

**claude-sonnet-4-6** for all steps.

- Steps 2, 3, 5, 6 require multi-step reasoning (analysis, comparison, knowledge synthesis, drafting) — Sonnet 4.6 handles these well
- Steps 1 and 7 are deterministic tool calls — Sonnet is fine; Haiku could be used to optimize cost in a future iteration
- Opus is not needed — this workflow is reasoning + retrieval, not deep research or complex multi-step deduction

---

## Context Inventory

| Artifact | Status | Risk |
|---|---|---|
| Gong call transcript | Exists — API access needed | High |
| Gong call summary | Exists — API access needed | High |
| Gem external help center | Exists — URL to be provided | Low |
| Gem internal help center | Exists — internal access needed | Medium |
| Glean | Exists — integration needed | High |
| Google Docs documentation | Exists — access needed | Medium |
| Highspot / Spekit | Exists — not live yet | High |
| Dropbox Paper | Exists — access uncertain | Medium |
| Recommended Workflows Library | **Needs Creation** | High (long-term value) |
| Gmail MCP | Connected | Low |
| Slack MCP | Connected | Low |

---

## Recommended Implementation Order

**Phase 1 — Quick Win (no integrations needed)**
- Build Skill 4 (Follow-Up Drafter) — test with sample action items + answers
- Build Skill 1 (Transcript Analyzer) — test with a pasted transcript

**Phase 2 — Core Analysis Loop**
- Build Skill 2 (Summary Comparator) — test with sample Gong summary vs. Claude summary
- Wire the orchestrator agent: Steps 1 (manual input) → 2 → 3 → 4 (human gate) → 6 → 7

**Phase 3 — Integrations**
- Connect Gmail MCP (Step 7 — email delivery)
- Connect Slack MCP (Step 7 — DM delivery)
- Research Gong API access (Step 1 automation)

**Phase 4 — Product Knowledge**
- Build Skill 3 (Product Knowledge Resolver)
- Connect Glean + Gem help centers
- Research/connect Google Docs, Highspot, Dropbox Paper

**Phase 5 — Enhancements**
- Build Recommended Workflows Library
- Add parallel sub-agent dispatch for Steps 2 & 3

---

## Where to Run

**Claude Code (CLI)** — user invokes the `post-call-followup` agent by name after a call ends. No scheduled/automated execution at launch. Human is in the loop at Step 4.

