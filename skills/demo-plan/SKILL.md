---
name: demo-plan
description: "Converts a sales rep's free-form prep notes into a structured one-pager pre-call demo plan as a formatted Markdown file. Use this skill whenever the user mentions demo prep, demo plan, pre-call planning, call prep notes, or wants to turn raw prospect notes into a structured document for an upcoming sales demo or discovery call. Also trigger when the user pastes unstructured notes about a prospect and asks for them to be organized, formatted, or turned into a plan. Even if they don't say 'demo plan' explicitly — if they're prepping for a sales call and want structure, use this skill."
---

# Pre-Call Demo Plan Generator

Turn a sales rep's free-form prep notes into a polished, one-pager Markdown demo plan.

## Who this is for

Solutions Consultants and sales reps preparing for prospect demos. The rep pastes raw, unstructured notes — company info, who they're meeting, pain points, whatever they've gathered — and gets back a clean, scannable one-pager they can reference before and during the call.

## Core principles

1. **Only use what the rep provides.** Never infer, guess, or fill in gaps. If the notes don't mention tech stack, leave that section out entirely. The rep needs to trust that everything on the plan came from their own research.
2. **Keep it to one page.** This is a quick-reference document, not a deep-dive report. Every section should be concise — bullets over paragraphs, short phrases over full sentences where appropriate.
3. **Make it scannable.** The rep will glance at this minutes before the call. Use clear headers, tight bullets, and bold key terms so the important stuff jumps out.

## How to process the input

The rep will paste free-form text — it might be bullet points, sentence fragments, copy-pasted CRM fields, Slack messages, or a stream-of-consciousness dump. Your job is to:

1. Read through all of it and identify which pieces of information map to which sections
2. Reorganize and clean up the language (fix typos, normalize formatting) without changing meaning
3. Drop the content into the template structure below
4. Omit any section where the rep provided no relevant information — don't include empty sections or placeholder text like "Not provided"

## Output template

Generate a Markdown file with the filename format: `demo-plan-[company-name].md`

Use this structure. Every section that has content should appear in this order:

```markdown
# Pre-Call Demo Plan: [Company Name]

**Date:** [Call date if mentioned, otherwise omit this line]
**Prepared by:** [Rep name if mentioned, otherwise omit this line]

---

## Company Overview
Bulleted summary of the company. Include any of the following that the rep provided:
- **Industry:** What the company does / their sector
- **Size:** Total headcount, stage, funding
- **HQ / Locations:** Where they're based, any global presence
- **Recruiting team:** Size and composition — break out by role if provided (e.g., "12-person TA team: 6 recruiters, 3 sourcers, 2 coordinators, 1 TA manager"). This is high-value context for an SC walking into a demo.
- Any other relevant context (e.g., hiring plans, recent leadership changes, growth stage)

Omit any bullet where no information was provided.

## Tech Stack
Bullet list of tools, platforms, and systems the prospect currently uses. Group by category if there are many (e.g., ATS, CRM, HRIS). When the rep's notes include additional context about a tool — satisfaction level, contract status, integration issues, how it's being used — include that as a note after the tool name. For example:
- **ATS:** Greenhouse — happy with it, not looking to replace
- **CRM:** Salesforce — only used by sales, not recruiting
- **Sourcing:** LinkedIn Recruiter — contract renews in Q4, exploring alternatives

## Key Contacts
For each person the rep will be meeting with:
- **[Name]** — [Title/Role] · [Any relevant notes about this person, e.g., "decision maker", "technical evaluator", "referred by X"]

## Challenges
Numbered list of the prospect's pain points. Each item starts with a short bolded label (2-5 words) summarizing the challenge, followed by a brief description. Use the rep's own framing — these should sound like things the prospect actually said or the rep observed, not generic industry problems.

Example format:
1. **Manual outreach tracking** — Recruiters spend hours logging touchpoints across LinkedIn and email with no unified view
2. **Reporting bottleneck** — Pipeline metrics require manual spreadsheet exports from the ATS

Order challenges by priority/severity if the notes give any indication. The numbering here matters — it should align with Desired Solutions so the reader can mentally map challenge #1 to solution #1.

## Desired Solutions
Numbered list of what the prospect is hoping to achieve, aligned to the Challenges above. Each item starts with a short bolded label, followed by a brief description focused on outcomes and goals (not features). Where possible, each solution should correspond to the challenge with the same number.

Example format:
1. **Unified engagement platform** — A single tool that combines sourcing, outreach, and candidate tracking
2. **Automated reporting** — Dashboards that can be shared with hiring managers without manual work

## Competitive Landscape
Any competing tools the prospect is currently using or evaluating. Include context if provided (e.g., "Currently on Lever, contract up in Q3" or "Also evaluating Ashby"). If the notes give any indication of which competitor the prospect is most interested in or leaning toward, call that out explicitly — this helps the SC know where to focus competitive positioning during the demo.

## Discovery Questions
2-4 targeted questions derived from gaps or ambiguities in the prep notes. These should help the rep validate assumptions or uncover deeper needs during the call. Frame them as open-ended questions the rep can actually ask.

## Recommended Demo Flow
A numbered sequence of demo segments tailored to the prospect's challenges and goals. Each step should:
- Name the feature area or workflow to show
- Connect it to a specific challenge or goal from the notes
- Be brief — one line per step is ideal

Keep this to 4-6 steps. The demo should tell a story that maps to the prospect's priorities, not just walk through a feature list.
```

## Section-specific guidance

### Discovery Questions
These are the one section where you do light creative work — the rep didn't explicitly provide questions, so you synthesize them from the notes. The questions should:
- Target areas where the notes are thin or ambiguous (e.g., if challenges are vague, ask a question that digs deeper)
- Be phrased conversationally, the way a real SC would ask them on a call
- Avoid generic questions like "What does success look like?" — make them specific to what the rep shared

### Recommended Demo Flow
Build the flow around the prospect's priorities, not a default product walkthrough. The order should reflect what matters most to this prospect based on their challenges and goals. Lead with the highest-impact area.

## Formatting rules

- Use `#` for the document title, `##` for section headers
- Use `**bold**` for names, key terms, and emphasis
- Use `-` for unordered lists, `1.` for numbered lists (Challenges, Desired Solutions, and Recommended Demo Flow)
- Use `---` as a divider after the header metadata
- No emojis, no decorative formatting
- Keep the entire document tight enough to print on a single page if needed

## What to do when information is sparse

If the rep gives you very little to work with (e.g., just a company name and one pain point), still produce the plan with whatever sections you can fill. Don't pad it. A 3-section plan from real information is better than an 8-section plan with guesses. But do generate Discovery Questions in this case — when notes are sparse, that section becomes even more valuable because it highlights what the rep still needs to learn on the call.

## Output

Save the Markdown file to the outputs directory and present it to the user.
