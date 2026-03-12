---
name: prep-block-rule-engine
description: >
  This skill should be used when you have a new prospect call booking and a day's
  calendar schedule and need to calculate where to place a prep block. Given the new
  booking details, the full day's events, and working hours, it applies the prep block
  rules and returns either a prep block specification (start time, end time, duration,
  type, delete_existing flag) or a flag signal explaining why no slot could be found.
  Designed for automated use by the prep-block-on-booking agent and reuse by the
  Weekly Sweep workflow.
user-invocable: true
metadata:
  author: handsonai
  version: "1.0.0"
---

# Prep Block Rule Engine

Given a new prospect call booking and a full day's calendar schedule, calculate the correct prep block placement and duration — or return a flag signal if no valid slot exists.

## Inputs

You will receive:

- **New booking**: date, start time, end time
- **Day schedule**: list of events, each with start time, end time, and type (one of: `prospect_call`, `internal`, `busy`)
- **Working hours**: 8:00 AM – 6:00 PM (hardcoded; do not deviate)

If the day schedule is missing or empty, treat as a no-events day (standard path applies). If event types are ambiguous or unclassifiable, treat as `busy` (conservative default).

## Decision Logic

Apply the following steps in order:

### Step 1 — Back-to-Back Check

Count how many `prospect_call` events are immediately adjacent to the new booking (no gap between end of one and start of another, in either direction).

- **3 or more prospect calls back-to-back** (including the new booking): Return a **flag signal** with reason `"3+ back-to-back detected"`. Stop here.
- **Exactly 2 prospect calls back-to-back** (the new booking plus one other): Identify the earlier of the two calls. Plan a **60-minute prep block** ending at the start of the earlier call. Set `delete_existing: true` — any existing "Call Prep" event immediately before the earlier call must be deleted before the new block is created.
- **No back-to-back**: Plan a **30-minute prep block** ending at the start of the new booking. Set `delete_existing: false`.

### Step 2 — Slot Availability Check

Starting from the planned prep block end time, scan backwards to find the latest available slot of the required duration (30 or 60 min):

1. The slot must not overlap any event in the day schedule (including the new booking)
2. The slot must fall entirely within working hours: start ≥ 8:00 AM, end ≤ new booking start time
3. Place the slot as close as possible to the demo start (maximize buffer time earlier in the day)

If a valid slot is found: return the **prep block spec**.

If no valid slot exists within working hours: return a **flag signal** with reason `"no slot within working hours"`.

## Outputs

Return exactly one of the following:

### Prep Block Spec (success)

```
result: prep_block
start_time: [HH:MM AM/PM]
end_time: [HH:MM AM/PM]
duration_minutes: [30 or 60]
type: [standard | back-to-back]
delete_existing: [true | false]
```

- `type: standard` — 30-minute block, no back-to-back situation
- `type: back-to-back` — 60-minute block, replaces two individual 30-minute blocks

### Flag Signal (failure)

```
result: flag
reason: [no slot within working hours | 3+ back-to-back detected]
new_booking_title: [title]
new_booking_date: [date]
new_booking_start: [HH:MM AM/PM]
new_booking_end: [HH:MM AM/PM]
```

## Examples

### Example 1 — Standard (no back-to-back)

New booking: 2:00 PM – 3:00 PM
Day schedule: 10:00 AM internal, 11:00 AM internal, 1:00 PM internal (ends 1:45 PM)
Working hours: 8:00 AM – 6:00 PM

Decision:
- No back-to-back → 30-min block
- Target slot: 1:30 PM – 2:00 PM
- 1:00 PM internal ends at 1:45 PM → overlaps, scan back
- Next candidate: 1:15 PM – 1:45 PM → still overlaps
- Next candidate: 1:00 PM – 1:30 PM → overlaps (internal meeting ends 1:45)
- Next free slot: before 1:00 PM internal → 12:30 PM – 1:00 PM ✓

Output:
```
result: prep_block
start_time: 12:30 PM
end_time: 1:00 PM
duration_minutes: 30
type: standard
delete_existing: false
```

### Example 2 — Back-to-back (2 demos)

New booking: 2:00 PM – 3:00 PM
Day schedule includes: 3:00 PM – 4:00 PM prospect_call (immediately after new booking)
Earlier call is new booking (2:00 PM). Plan 60-min block before 2:00 PM.
Slot 12:00 PM – 1:00 PM is free ✓

Output:
```
result: prep_block
start_time: 12:00 PM
end_time: 1:00 PM
duration_minutes: 60
type: back-to-back
delete_existing: true
```

### Example 3 — No slot

New booking: 9:00 AM – 10:00 AM
Day schedule: 8:00 AM – 9:00 AM internal (no free 30-min slot before 9 AM within working hours)

Output:
```
result: flag
reason: no slot within working hours
new_booking_title: Demo - Prospect Co
new_booking_date: 2026-03-15
new_booking_start: 9:00 AM
new_booking_end: 10:00 AM
```

## Guidelines

- Working hours are fixed at 8:00 AM – 6:00 PM. Never place a prep block outside this window, even if a free slot exists.
- "Immediately adjacent" means zero gap — if demo A ends at 2:00 PM and demo B starts at 2:00 PM, they are back-to-back. A 1-minute gap means they are not back-to-back.
- When `delete_existing: true`, the existing "Call Prep" event to delete is the 30-minute block immediately preceding the earlier back-to-back demo. The calling agent is responsible for performing the deletion.
- The skill only calculates placement — it does not create or delete calendar events. That is the agent's responsibility.
- If event type cannot be determined (missing attendee info, no type label), treat as `busy`.
- Always return structured output in the exact format above — the agent parses this output programmatically.
