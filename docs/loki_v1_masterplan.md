# LokisWand — Masterplan v1

**Status:** Implementation complete. All eight phases passed. Phase 7 feedback loop complete. Next phase: v2 design.

## Overview
LokisWand is a job application intelligence pipeline built in n8n. It accepts a pasted job description, evaluates candidate fit using dual Claude assessments and a synthesis call, logs all results to Airtable, and delivers follow-up nudges via Slack at defined intervals. The system is designed for minimum viable user input: one-time profile setup and per-application job description paste.

---

## Reference Documents
All implementation decisions are locked in the following documents. Claude Code must not deviate from them without explicit instruction.

- `loki_v1_interface_contract.md` — all inputs, outputs, schemas, and prompt directives
- `loki_v1_design_decisions.md` — all locked decisions with rationale and rejected alternatives

---

## Folder Structure

```
lokiswand/
├── profile/
│   ├── loki_candidate_profile.md        # Generated profile document
│   └── loki_conversion_prompt.md        # One-time Claude conversion prompt
├── prompts/
│   ├── general/                         # Reusable methodology prompts
│   │   ├── 01_design_methodology.md
│   │   ├── 02_readme_generation.md
│   │   ├── 03_generate_walkthrough.md
│   │   ├── 04_rewrite_glossary.md
│   │   ├── 05_feedback_loop.md
│   │   └── 06_generate_diagrams.md
│   ├── project/                         # LokisWand-specific pipeline prompts
│   │   ├── loki_instance_a_prompt.md    # Fit signals prompt directive
│   │   ├── loki_instance_b_prompt.md    # Risk signals prompt directive
│   │   └── loki_synthesis_prompt.md     # Synthesis prompt directive
│   └── README.md
├── walkthrough/                         # Annotated design rationale (post-implementation)
│   ├── README.md
│   ├── loki_core_pipeline.md
│   ├── loki_followup_nudge.md
│   ├── loki_candidate_profile.md
│   ├── loki_conversion_prompt.md
│   ├── loki_instance_a_prompt.md
│   ├── loki_instance_b_prompt.md
│   └── loki_synthesis_prompt.md
├── workflows/
│   ├── loki_core_pipeline.json          # n8n core pipeline workflow export
│   └── loki_followup_nudge.json         # n8n follow-up nudge workflow export
├── docs/
│   ├── loki_v1_design_decisions.md
│   ├── loki_v1_glossary.md
│   ├── loki_v1_interface_contract.md
│   ├── loki_v1_masterplan.md
│   └── loki_v1_setup_notes.md
├── DESIGN.md
├── DESIGN_DECISIONS.md
├── INTERFACE_CONTRACT.md
└── README.md

[Updated post-implementation: Actual folder structure differs from the original plan. The prompts/ directory was reorganized into general/ (reusable methodology prompts) and project/ (LokisWand-specific pipeline prompts). A walkthrough/ directory was added post-implementation. Root-level DESIGN.md, DESIGN_DECISIONS.md, and INTERFACE_CONTRACT.md were added as reader-accessible documentation artifacts. The docs/ directory was preserved for the canonical versioned design documents.]
```

---

## Phase 1 — Environment Setup
**Goal:** All accounts, credentials, and tools configured before any workflow is built.

### Steps
1. Create an n8n cloud account at n8n.io — cloud instance is required for reliable follow-up nudge scheduling
2. Create an Airtable account at airtable.com
3. Create a personal Slack workspace at slack.com
4. Obtain a Claude API key from console.anthropic.com
5. Store all credentials securely — they will be entered into n8n's credential manager, never hardcoded

### Acceptance criteria
- n8n instance is accessible and running
- Airtable base exists with the correct schema (see Phase 2)
- Slack workspace exists with a dedicated job-search channel
- Claude API key is valid and stored in n8n credential manager

---

## Phase 2 — Airtable Schema Setup
**Goal:** Airtable base configured exactly as specified in the interface contract before any workflow writes to it.

### Airtable base name
LokisWand

### Table name
Applications

### Fields
| Field name | Field type | Notes |
|---|---|---|
| Company | Single line text | |
| Role title | Single line text | |
| Location | Single line text | |
| Employment type | Single line text | |
| Date submitted | Date | Include time |
| Fit signals | Long text | |
| Risk signals | Long text | |
| Synthesis | Long text | |
| Recommendation | Single select | Options: Strong Fit, Moderate Fit, Weak Fit, Do Not Apply |
| Status | Single select | Options: Researching, Applied, Following Up, Interviewing, Offer, Rejected, Skipped |
| Follow-up sent day 7 | Checkbox | |
| Follow-up sent day 14 | Checkbox | |
| Profile version | Single line text | |

### Kanban view
Create a kanban view grouped by the Status field. This is the primary pipeline visualization surface.

### Acceptance criteria
- All fields exist with correct types
- All single select options are configured exactly as specified
- Kanban view is configured and displays correctly
- Airtable API key and base ID are stored in n8n credential manager

---

## Phase 3 — Profile Document Generation
**Goal:** Structured candidate profile document generated, reviewed, and ready for use as a static n8n input.

### Steps
1. Draft the conversion prompt in `profile/loki_conversion_prompt.md` following the interface contract schema
2. Paste the master resume into a Claude conversation along with the conversion prompt
3. Save the generated output as `profile/loki_candidate_profile.md`
4. Review the Review Required section — verify all inferred fields and correct any inaccuracies
5. Set the version field to `v1.0`
6. Upload `loki_candidate_profile.md` to n8n as a static file reference

### Conversion prompt structure
The conversion prompt must instruct Claude to:
- Follow the profile document schema exactly as defined in the interface contract
- Place all inferred fields in the Review Required section regardless of confidence level
- Draft technical detail fields for each experience entry from resume content
- Infer highlights tags from the nature of each project and its relevance dimensions
- Infer goals, known gaps, and hard constraints from resume trajectory and target role signals
- Flag nothing outside the schema

### Acceptance criteria
- Profile document follows the schema exactly
- Review Required section is complete and has been verified by the user
- Version field is set to `v1.0`
- File is uploaded and accessible in n8n

---

## Phase 4 — Prompt Directives
**Goal:** All three Claude prompt directives drafted, tested in isolation, and saved before workflow construction begins.

### Prompts to draft

**`prompts/loki_instance_a_prompt.md`**
Fit signals directive as specified in the interface contract. Must include the silence and deferral instruction — if no genuine fit signals exist, defer to synthesis with explanation rather than manufacturing signals.

**`prompts/loki_instance_b_prompt.md`**
Risk signals directive as specified in the interface contract. Must include the silence and deferral instruction. Must explicitly distinguish between gaps marked as in-progress in the profile and gaps that are entirely unaddressed.

**`prompts/loki_synthesis_prompt.md`**
Synthesis directive as specified in the interface contract. Must handle deferral from either or both instances as meaningful evidence. Must conclude with a fixed enum recommendation followed by a one paragraph justification.

### Testing
Before proceeding to workflow construction, test each prompt in isolation using the Claude API or a Claude conversation:
- Test Instance A against a strong match listing — verify it produces genuine fit signals, not manufactured ones
- Test Instance B against a weak match listing — verify it produces genuine risk signals
- Test Instance B against a strong match listing — verify it defers rather than manufacturing risk signals
- Test synthesis with a deferral input from Instance B — verify it incorporates the deferral as positive evidence

### Acceptance criteria
- All three prompts produce correct output against test inputs
- Deferral behavior is confirmed working for both Instance A and Instance B
- Synthesis correctly handles deferral inputs

---

## Phase 5 — Core Pipeline Workflow
**Goal:** The core n8n workflow is built, tested end-to-end, and producing correct Airtable records.

### Node sequence
1. **Form trigger** — hosted n8n form with a single long text field: job description
2. **Metadata extraction** — HTTP request node calling Claude API with metadata extraction prompt; parses company, role title, location, employment type from job description text
3. **Duplicate check** — Airtable search node queries for existing records matching company and role title; if match found, send Slack notification and halt
4. **Parallel assessment** — two simultaneous HTTP request nodes calling Claude API with Instance A and Instance B prompts respectively; both receive profile document text and job description text as context
5. **Synthesis** — HTTP request node calling Claude API with synthesis prompt; receives Instance A output and Instance B output
6. **Airtable write** — Airtable create record node writing all fields including profile version read from the static profile document
7. **Slack confirmation** — Slack node posting a confirmation message to the job-search channel with company, role title, and recommendation enum

### Error handling nodes
Add error handling branches at the following points per the interface contract failure handling table:
- Metadata extraction returns null for both company and role title → Slack alert, halt
- Instance A or B call fails → Slack alert, halt before synthesis
- Synthesis call fails → log partial results to Airtable, Slack alert, halt
- Airtable write fails → Slack notification with raw output

### Acceptance criteria
- Form submission triggers the full pipeline end-to-end
- A correctly structured Airtable record is created for each submission
- Duplicate submissions are caught and halted with a Slack notification
- All error branches trigger correct notifications
- Slack confirmation message fires on successful completion

---

## Phase 6 — Follow-up Nudge Workflow
**Goal:** The follow-up nudge workflow is built, scheduled, and correctly querying Airtable.

### Node sequence
1. **Schedule trigger** — runs daily at 9:00 AM user local time
2. **Airtable query — day 7** — search for records where status is Applied, date submitted is 7 or more days ago and less than 21 days ago, and follow-up sent day 7 is false
3. **Slack nudge — day 7** — for each matching record, post a Slack message to the job-search channel with company, role title, date applied, and day 7 follow-up prompt
4. **Airtable update — day 7** — set follow-up sent day 7 to true for each notified record
5. **Airtable query — day 14** — search for records where status is Applied, date submitted is 14 or more days ago and less than 21 days ago, and follow-up sent day 14 is false
6. **Slack nudge — day 14** — for each matching record, post a Slack message with day 14 follow-up prompt
7. **Airtable update — day 14** — set follow-up sent day 14 to true for each notified record

### Slack message format
```
Follow-up Reminder
Company: [company]
Role: [role title]
Applied: [date submitted]
Day [7 or 14] follow-up due — have you heard back?
```

### Acceptance criteria
- Workflow runs on schedule without manual trigger
- Day 7 and day 14 queries return correct records
- Slack nudges fire for each qualifying record
- Airtable checkboxes are updated to true after each nudge
- Records with checkboxes already set to true are not re-notified
- Workflow failure sends a Slack notification and retries next day

---

## Phase 7 — End-to-End Testing
**Goal:** Full system validated against real job descriptions before use.

### Test cases
1. **Strong match** — submit a job description closely matching the candidate profile; verify recommendation is Strong Fit and fit signals are genuine
2. **Weak match** — submit a job description with significant gaps; verify recommendation is Weak Fit or Do Not Apply and risk signals are genuine
3. **Instance B deferral** — submit a near-perfect match; verify Instance B defers rather than manufacturing risk signals
4. **Duplicate detection** — submit the same job description twice; verify the second submission is halted with a Slack notification
5. **Follow-up nudge** — manually set a test record's date submitted to 7 days ago and status to Applied; trigger the nudge workflow manually and verify Slack message fires and checkbox updates
6. **Profile version logging** — verify all Airtable records contain the correct profile version field value

### Acceptance criteria
- All six test cases pass
- No manufactured signals observed in any assessment output
- All Airtable records are correctly structured
- Kanban view displays all test records in correct status columns

---

## Phase 8 — README
**Goal:** Project documented for portfolio presentation.

### README structure
1. **What LokisWand does** — plain English, no jargon; the problem it solves and the workflow from job description paste to Airtable record
2. **Why the design matters** — dual assessment pattern, deferral behavior, synthesis, minimum viable user input principle
3. **How it was built** — design-first methodology, interface contract before workflow construction, Socratic design process
4. **Stack** — n8n, Claude API, Airtable, Slack
5. **Companion documents** — link to interface contract and design decisions with one-line descriptions

### Acceptance criteria
- README is complete and accurately describes the system
- Companion documents are linked and accessible
- No implementation details that contradict the interface contract or design decisions

---

## Implementation Notes for Claude Code
- Read `loki_v1_interface_contract.md` and `loki_v1_design_decisions.md` in full before beginning any phase
- All prompt directives must match the interface contract exactly — do not paraphrase or simplify
- All Airtable field names must match the schema exactly — case sensitive
- Do not add nodes, fields, or behaviors not specified in this masterplan without explicit instruction
- If a locked decision conflicts with an n8n implementation constraint, surface the conflict explicitly and halt
