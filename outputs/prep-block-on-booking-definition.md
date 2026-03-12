# Workflow Definition: Prep Block on Booking

---

## Scenario Metadata

| Field | Value |
|---|---|
| **Workflow Name** | Prep Block on Booking |
| **Description** | When a new prospect call is accepted on the calendar, automatically scans the day for conflicts, applies prep block rules, and creates a "Call Prep" event in Google Calendar — no manual review required. |
| **Process Outcome** | A correctly placed "Call Prep" event created in Google Calendar for the newly accepted prospect call, or a flag to the user if no valid slot could be found |
| **Trigger** | A new demo/prospect call calendar invite is accepted by the user |
| **Type** | Automated |
| **Business Objective** | Eliminate manual prep block creation for every prospect call, ensure prep time is always protected, and automatically handle back-to-back scenarios without user intervention |
| **Current Owner** | John Fisher, Solutions Consultant |
| **Lens** | Individual |

---

## Refined Steps

### Step 1 — Detect New Prospect Call Booking
- **Action**: When a calendar invite is accepted, determine whether it is a prospect call (vs. an internal meeting)
- **Sub-steps**:
  1. Monitor for newly accepted calendar invites
  2. Check invite attendee list for external (non-@gem.com) email addresses
  3. If at least one external email is present → classify as prospect call and proceed
  4. If all attendees are internal → ignore, do not create prep block
- **Data In**: Accepted calendar invite (title, time, attendees)
- **Data Out**: Confirmed prospect call event (date, start time, end time, attendee list)
- **Context Needs**: Google Calendar read access
- **Failure Modes**:
  - Invite has no attendee list → treat as internal, skip
  - Cannot determine email domain → flag to user for manual review

---

### Step 2 — Scan Calendar for the Day
- **Action**: Pull all existing events on the same day as the newly booked call to understand what's already scheduled
- **Sub-steps**:
  1. Fetch all calendar events for the same day within working hours (8am–6pm)
  2. Identify other prospect calls (using same external email detection from Step 1)
  3. Identify internal meetings and other busy blocks
  4. Check if the new booking creates a back-to-back demo situation (demo immediately before or after another demo with no gap)
- **Data In**: Date of the new booking
- **Data Out**: Full day schedule — list of events with times, types (prospect call / internal / other), and back-to-back flags
- **Context Needs**: Google Calendar read access
- **Failure Modes**:
  - Calendar unavailable → abort and flag user

---

### Step 3 — Apply Prep Block Rules
- **Action**: Use the day schedule and prep block rules to determine the correct prep block placement and duration
- **Sub-steps**:
  1. **Back-to-back check**: If the new booking is immediately adjacent to another prospect call (no gap between them):
     - Identify the first demo in the pair
     - Delete any existing "Call Prep" block associated with the first demo
     - Plan a new 60-min prep block before the first demo
  2. **Standard check**: If no back-to-back situation, plan a 30-min prep block ending at the new demo's start time
  3. **Slot availability check**: Starting from the planned prep block time, scan backwards for a free 30-min (or 60-min) window:
     - Slot must not overlap any existing event
     - Slot must fall within working hours (8am–6pm)
  4. If a valid slot is found → proceed to Step 4
  5. If no valid slot exists within working hours → proceed to Step 5 (flag)
- **Data In**: Day schedule (Step 2), new booking details
- **Data Out**: Prep block specification (start time, end time, duration, type) OR flag signal
- **Context Needs**:
  - Working hours rule: 8am–6pm
  - Prep block rules:
    - Standard: 30-min block, placed immediately before the demo if slot is free
    - Back-to-back (2 demos): 60-min block before the first demo; delete existing 30-min block first
    - Back-to-back (3+ demos): flag user — do not create block (handled by weekly sweep)
    - Never place a prep block over an existing event
    - Never place a prep block outside 8am–6pm working hours
- **Failure Modes**:
  - No free slot within working hours → skip to Step 5
  - 3+ back-to-back demos detected → flag user, do not create block

---

### Step 4 — Create Prep Block in Google Calendar
- **Action**: Create the "Call Prep" calendar event at the calculated time
- **Sub-steps**:
  1. If replacing a back-to-back: delete the existing 30-min prep block
  2. Create new calendar event with:
     - Title: "Call Prep"
     - Start time / End time: as calculated in Step 3
     - Status: Busy
     - Description: (blank — to be populated by downstream workflow)
- **Data In**: Prep block specification (Step 3)
- **Data Out**: Created Google Calendar event
- **Context Needs**: Google Calendar write access
- **Failure Modes**:
  - Calendar write fails → retry once; if still failing, flag user

---

### Step 5 — Flag User (Conditional)
- **Action**: Notify the user that no valid prep block slot could be found for the newly booked call
- **Sub-steps**:
  1. Send notification with: call details (title, date, time), reason for failure (no slot within working hours / 3+ back-to-back)
  2. Prompt user to manually place a prep block or adjust their schedule
- **Data In**: New booking details, reason for failure
- **Data Out**: Notification to user
- **Context Needs**: Notification mechanism (email, Slack DM, or calendar notification)
- **Failure Modes**:
  - Notification fails → log for manual follow-up

*Note: Step 5 only runs if Step 3 finds no valid slot. Steps 4 and 5 are mutually exclusive.*

---

## Step Sequence and Dependencies

```
Step 1 (Detect New Prospect Call Booking)
    ↓
Step 2 (Scan Calendar for the Day)
    ↓
Step 3 (Apply Prep Block Rules)
    ├── Valid slot found → Step 4 (Create Prep Block)
    └── No valid slot   → Step 5 (Flag User)
```

- **Sequential**: Steps 1 → 2 → 3 → 4 (happy path)
- **Conditional branch**: Step 3 → Step 5 (failure path)
- **Mutually exclusive**: Steps 4 and 5 never both run
- **Critical Path**: 1 → 2 → 3 → 4

---

## Context Shopping List

| Artifact | Description | Used By Steps | Status | Notes |
|---|---|---|---|---|
| Google Calendar read access | Read events, attendee lists, and availability | 1, 2 | Exists | Google Calendar MCP available |
| Google Calendar write access | Create and delete calendar events | 4 | Exists | Google Calendar MCP available |
| External email detection rule | Non-@gem.com email = external attendee = prospect call | 1, 2 | Exists (rule) | Logic to be implemented in agent |
| Working hours rule | 8am–6pm; no prep blocks outside this window | 3 | Exists (rule) | Confirmed by user |
| Prep block rules | 30-min standard / 60-min back-to-back / delete-and-recreate / no overlap / no 3+ back-to-back | 3 | Exists (rule) | Documented in Notion: Demo Prep Calendar Blocking |
| Notification mechanism | Channel to alert user when no slot is found | 5 | TBD | Email, Slack DM, or calendar notification — to be determined |

### Related Workflows
- **Weekly Sweep** (to be deconstructed) — batch version of this workflow; runs Sunday/Monday across the full upcoming week
- **Downstream: Demo Prep Notes** — populates the "Call Prep" event description with briefing notes from the sales team
