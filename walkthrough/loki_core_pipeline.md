# Walkthrough: loki_core_pipeline.json

**Source file:** `workflows/loki_core_pipeline.json`
**Role:** Primary n8n workflow — the main execution path from job description paste to Airtable record

---

## Purpose

The core pipeline is the central workflow. It implements the five-step assessment sequence defined in the interface contract: metadata extraction, duplicate detection, parallel dual assessment, synthesis, and Airtable write with Slack notification. Every design decision in the system has an implementation artifact in this file — understanding why each node is structured the way it is requires tracing back to those decisions.

---

## Relationships

**Consumes:**
- Job description text (from the n8n form trigger — the user's only recurring input)
- Candidate profile (embedded as a hardcoded string in the Load Profile Document code node — see v1 departure note below)
- Instance A, Instance B, and synthesis prompt directives (embedded in HTTP Request node bodies)
- Airtable credentials and Anthropic API key (referenced via n8n credential manager)

**Produces:**
- An Airtable record in the Applications table with all pipeline output fields populated
- A Slack confirmation message on success
- Slack error notifications on specific failure conditions (metadata null, assessment failure, synthesis failure, Airtable write failure)

**Does not touch:**
- The follow-up nudge workflow — that is a completely separate workflow on its own schedule
- Any Airtable fields other than the 13 fields defined in the interface contract schema
- The candidate profile document on disk — it reads an embedded copy, not the file

---

## Decision rationale

### HTTP Request nodes for all Claude API calls
All three Claude calls (metadata extraction, Instance A, Instance B, synthesis) use n8n's HTTP Request node rather than any built-in AI node. This keeps API parameters explicit — model, max_tokens, message structure, and system/user role separation are all visible and version-stable in the node configuration. n8n's native AI nodes abstract these details in ways that can change across n8n updates. Explicit HTTP Requests make the Claude calls predictable and debuggable.

### Profile document embedded in a Code node
The interface contract specified that the profile document would be "uploaded to n8n as a static reference file." The implementation departed from this: the profile text is embedded as a hardcoded string in a Code node called Load Profile Document. This happened because n8n's file reference handling introduced more complexity than anticipated during implementation. Embedding the profile directly eliminates the file reference dependency at the cost of requiring a workflow edit when the profile updates. This is noted as an implementation decision with a v2 remediation path.

### Duplicate detection placed before assessment calls
The duplicate check (an Airtable search) runs before Instance A and Instance B are called. This is Decision 13 applied to execution order: running the check first prevents spending three Claude API calls on a submission that will be rejected. The check is inexpensive (one Airtable query with continueOnFail enabled); rejecting after assessment would waste API credits.

### Process Duplicate Check code node
The Airtable search node returns an empty array when no match is found. A raw empty array is ambiguous in n8n's IF node logic. The Process Duplicate Check code node translates the array result into an explicit boolean (duplicateFound: true/false), making the downstream IF condition reliable rather than dependent on n8n's type coercion behavior.

### Merge node (typeVersion 3.2) for parallel assessment
Instance A and Instance B run on parallel branches. Their outputs must be joined before being passed to synthesis. The Merge node at typeVersion 3.2 was required specifically — earlier typeVersions had different behavior for joining parallel branches and were not compatible with this workflow's execution order.

### Parse Synthesis Output code node
The synthesis prompt instructs Claude to return four labeled sections (Conflict Resolution, Collapsed Signals, Deferral Handling, Recommendation). The Parse Synthesis Output node uses regex to extract each section and the recommendation enum value. The regex targets exact section labels as written in the synthesis prompt — the section labels in the prompt and the parse patterns are tightly coupled. Any edit to the synthesis prompt section headers would break the parse step.

### Airtable write uses native Airtable node (create)
For the final write, the native n8n Airtable node is used with the create operation. This is appropriate for creating new records by field name — the native node handles field name mapping cleanly. This is distinct from the nudge workflow's update operations, which bypass the native node due to the batchUpdate bug (see loki_followup_nudge.md).

### Error branches and continueOnFail
Each stage that can fail has a corresponding error branch. The Metadata Valid? IF node checks whether both company and role title are null. The Assessment Error? IF node checks whether both Instance A and Instance B produced outputs. The Synthesis Error? IF node checks for a parse failure. The Airtable Write Failed? IF node checks whether the record was created. Each branch routes to a Slack notification node specific to that failure mode.

---

## What it does not do

- **Does not generate or manage the candidate profile** — profile creation is a manual one-time operation using loki_conversion_prompt.md
- **Does not send follow-up nudges** — that is loki_followup_nudge.json's responsibility
- **Does not retry failed Claude calls** — failures route to error branches; retrying is not specified in v1
- **Does not log successful metadata when the metadata check fails** — the halt is clean, not partial

---

## v2 touch points

- **Profile document embedding** is the primary maintenance pain point. v2 should externalize the profile into an n8n variable or properly configured static file reference so profile updates don't require a workflow edit.
- **Model hardcoded as claude-opus-4-7** in each HTTP Request node. v2 would benefit from centralizing model selection in an n8n variable, allowing model changes without editing three nodes.
- **No retry logic** for transient Claude API failures. v2 could add retry nodes at the assessment and synthesis stages for resilience.
