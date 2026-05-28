# LokisWand — Glossary v1

This glossary serves two audiences: recruiters (technical and non-technical) reading the repository, and the developer returning to the codebase after time away.

**Part 1** covers domain terms and high-level system concepts. Read this if you have already read the README and want to understand the vocabulary before diving deeper.

**Part 2** covers the architecture and execution model. Read this if you want to understand how the components connect and what each processing step produces.

**Part 3** is an implementation reference, organized by source file. Read this if you are reading specific source files and need terminology explained.

---

## Part 1 — Domain and System Overview

*Readable without technical background. Assumes you have read the README.*

---

**Job description**
The full text of a job listing, copied from a job board and pasted into the n8n form by the user. This is the only recurring user input to the system — no URL, no browser extension, no structured form beyond the text field. The user submits one job description per application they are evaluating.

**Job application**
A single candidate submission for a specific open role at a specific company. LokisWand processes one job application per form submission and stores the result as a single record in the Airtable tracker.

**Application status**
The current state of a job application in the Airtable tracker. Updated manually by the candidate. Options: Researching, Applied, Following Up, Interviewing, Offer, Rejected, Skipped. Only applications in "Applied" status receive follow-up nudges.

**Kanban board**
The Airtable view where all job applications are displayed as cards organized by status column. The candidate moves cards between columns as applications progress. No custom UI is required — Airtable's native kanban view handles this.

**Master resume**
The candidate's comprehensive source document containing all resume content. Not passed directly to any Claude assessment — converted once into the candidate profile document first.

**Candidate profile**
A structured markdown document derived from the master resume. Contains the candidate's identity, skills, experience (with separate achievement and technical detail fields), education, highlights with relevance tags, hard constraints, goals, and known gaps. Generated once, verified by the candidate, and used as context for every assessment. The quality of this document directly determines the quality of every downstream assessment.

**Fit assessment**
The output of Instance A: a numbered list identifying the strongest signals that the candidate is a good match for a specific role. Based on evidence from the candidate profile matched against the job description.

**Risk assessment**
The output of Instance B: a numbered list identifying gaps or mismatches between the candidate's profile and the role requirements. Each signal distinguishes whether the gap is something the candidate is actively closing (in-progress) or has not addressed (unaddressed).

**Recommendation**
The final output of the synthesis call. A single value from a fixed set of four: **Strong Fit**, **Moderate Fit**, **Weak Fit**, **Do Not Apply**. This is actionable — it is a direct answer to the question "should I apply to this role?"

**Follow-up**
The action of contacting a company after submitting an application to check on its status. LokisWand sends a Slack reminder at 7 days and 14 days after the application status is changed to Applied, so no follow-up deadline is missed.

**Follow-up nudge**
A Slack message sent to the candidate when an application is overdue for follow-up. Contains the company name, role title, date applied, and which follow-up interval (day 7 or day 14) is due.

---

## Part 2 — Architecture and Execution

*For readers who have read the README and want to understand how the system works before examining source files.*

---

**Three-call LLM orchestration**
The core architectural pattern: each job submission triggers exactly three Claude API calls. Call 1 extracts metadata from the job description. Calls 2a and 2b run in parallel — one identifies fit signals, the other identifies risk signals. Call 3 reconciles the two assessments and produces the final recommendation. Each call has a single, constrained task. No call attempts to evaluate both fit and risk.

**Complementary framing**
The design pattern that differentiates the two parallel assessment calls. Instance A is assigned fit signals; Instance B is assigned risk signals. The two instances are complementary — each has a distinct task — rather than adversarial (optimist vs. skeptic). Adversarial framing produces manufactured disagreement; complementary framing produces genuine variance that synthesis can reason about. See Decision 2.

**Parallel assessment**
Instance A and Instance B execute simultaneously rather than sequentially. Both receive the same inputs (candidate profile + job description) at the same time. Neither sees the other's output. This reduces total pipeline execution time and ensures each assessment is independent.

**Deferral**
The designed behavior of Instance A or Instance B when no genuine signals exist in their assigned category. Instead of manufacturing signals to meet a count, the instance returns a specific deferral statement. The synthesis call treats a deferral from Instance B as evidence of strong fit (no meaningful risks found) and a deferral from Instance A as evidence of weak fit (no meaningful fit evidence found). Deferral is a meaningful output, not an error. See Decision 3.

**Metadata extraction**
Step 1 of the core pipeline: a Claude API call that extracts company name, role title, location, and employment type from the raw job description text. Uses Claude rather than regex because job description formatting varies significantly across companies and job boards. Returns explicit null for fields it cannot determine, enabling clean failure handling. See Decision 14.

**Duplicate detection**
A guard that runs before assessment begins. Queries Airtable for an existing record matching the same company name and role title. If a match exists, the pipeline halts with a Slack notification — no duplicate record is created and no API calls are made for assessment. See Decision 13.

**Synthesis**
The third Claude call. Receives both assessment outputs and does four things in order: (1) resolves genuine conflicts between the two assessments, (2) collapses apparent conflicts that are not genuine into confident statements, (3) incorporates any deferral evidence from either instance, (4) concludes with a fixed enum recommendation and one-paragraph justification. Synthesis is analytical work, not summarization. See Decision 4.

**Conflict resolution**
The first analytical step in synthesis: identifying points where the fit assessment and risk assessment appear to directly contradict each other, then resolving each conflict with explicit reasoning. Unresolved conflicts produce hedged synthesis output, which is not acceptable — a hedged output is not a recommendation.

**Signal collapse**
The second analytical step in synthesis: identifying apparent conflicts that are not genuine — cases where both agents are observing the same thing from different angles — and collapsing each into a single confident statement. For example, if Instance A notes a strong ML background and Instance B notes that the role requires ML experience the candidate may not have, this is likely the same observation from two angles, not a genuine conflict.

**Recommendation enum**
The fixed set of four values that the synthesis call must conclude with: Strong Fit, Moderate Fit, Weak Fit, Do Not Apply. The recommendation must be exactly one of these — no qualifications appended, no variations. The fixed enum forces a concrete, actionable output. Stored as a single select field in Airtable.

**Pre-write guard pattern**
The structural pattern of running duplicate detection before the three Claude API calls. Cheap operations that can reject a submission run first; expensive operations (API calls) only run if the cheap guards pass.

**Structured LLM input**
The architectural principle underlying the candidate profile design: rather than passing raw resume text to assessment instances, a one-time conversion step produces a structured document with explicit fields for skills, achievements, technical details, and relevance tags. This eliminates input format variance across submissions and gives assessment instances concrete evidence rather than requiring them to infer from unstructured text.

**Range query with idempotent checkbox**
The two-part pattern used by the follow-up nudge workflow to achieve exactly-once nudge delivery. The range query (7 or more days ago, capped at 21) catches missed workflow execution days. The checkbox (set to true after each nudge sent) prevents the range query from sending duplicates on subsequent runs. Together they handle workflow failure gracefully. See Decision 12.

**Profile version**
The version identifier in the candidate profile header (e.g., v1.0). Logged in every Airtable record at assessment time. Makes assessments auditable — if the profile is updated, older records can be identified by their profile version as potentially reflecting outdated information. See Decision 15.

**Review Required**
The first section of the candidate profile document. Contains all fields inferred by Claude during profile generation — specifically hard constraints, goals, and known gaps — that require candidate verification before the profile is used. Always appears first, regardless of how confident Claude is in its inferences. See Decision 5.

**Known gaps**
A subsection of the Review Required section in the candidate profile. Lists skill gaps the candidate is actively working to close. Instance B uses this to distinguish between in-progress gaps and unaddressed gaps when identifying risk signals.

**Technical details field**
A field in each experience entry in the candidate profile, separate from achievement bullets. Lists the actual technologies, tools, methods, and frameworks used in that role. Provides Instance A and Instance B with concrete stack evidence rather than requiring inference from achievement language. See Decision 8.

**Highlight tags**
Explicit tags on each highlight in the Highlights and Accolades section of the candidate profile, indicating which role types, skills, or competencies the highlight is most relevant to. Eliminate the need for Instance A to infer relevance from descriptions on every assessment call. See Decision 9.

---

## Part 3 — Implementation Reference

*For developers reading specific source files. Organized by source file.*

---

### `workflows/loki_core_pipeline.json`

**Load Profile Document**
n8n Code node that provides the candidate profile text to downstream nodes. In v1, the profile is embedded as a hardcoded string in this node's JavaScript rather than fetched from an external file. This is a departure from the interface contract's specification of a "static file reference" — driven by n8n file handling complexity discovered during implementation. A v2 remediation item: the profile should be externalized to an n8n variable or properly configured file reference.

**Process Duplicate Check**
n8n Code node that translates the Airtable search result into an explicit `duplicateFound` boolean. Required because the Airtable search node returns an empty array when no match is found — a raw empty array is ambiguous in n8n's IF node logic, while an explicit boolean is not.

**Merge node (typeVersion 3.2)**
n8n node that joins the parallel Instance A and Instance B branches before synthesis. TypeVersion 3.2 was required specifically — earlier typeVersions had different behavior for combining parallel branch outputs that was not compatible with this workflow's execution order.

**Parse Synthesis Output**
n8n Code node that uses regex to extract the four labeled sections and recommendation enum value from the synthesis call's text response. The section label patterns are tightly coupled to the synthesis prompt's output format. Any rename of a section in the synthesis prompt must be updated in this node atomically.

**Metadata Valid? / Assessment Error? / Synthesis Error? / Airtable Write Failed?**
Four IF nodes, each gating a specific failure condition. Each routes to a distinct Slack error notification node for its failure mode. All four are required — omitting any one means a specific failure type goes unnoticed.

**Slack nodes** (Metadata Error, Duplicate Detected, Assessment Error, Synthesis Error, Write Failed, Pipeline Complete)
Six HTTP Request nodes posting to the Slack webhook. Each has a distinct message body for its context. All reference `$vars.SLACK_WEBHOOK_URL` — an n8n variable that must be configured in Settings → Variables before the workflow will function.

---

### `workflows/loki_followup_nudge.json`

**Daily Schedule**
Schedule trigger node set to 9:00 AM daily. Fires both the day 7 and day 14 query branches in parallel from a single trigger.

**Prepare Day 7 Message / Prepare Day 14 Message**
Code nodes that loop over Airtable query results and construct formatted Slack message bodies for each record. Adds a `slackBody` field to each item. Required because the Slack message requires JavaScript string interpolation of multiple fields — cleaner in a Code node than in n8n expression syntax in the HTTP Request body.

**Update Day 7 Checkbox / Update Day 14 Checkbox**
HTTP Request PATCH nodes that call the Airtable REST API directly to set the checkbox field to true. These use the HTTP Request node rather than the native n8n Airtable node because of an n8n Airtable v2 bug: the native node's batchUpdate operation treats "id" in "Columns to match on: id" as a user-created column name rather than the internal Airtable record ID, producing a 422 INVALID_RECORDS error. Direct HTTP PATCH with the proper `{ records: [{ id: <record_id>, fields: { ... } }] }` payload bypasses this bug.

**Cross-node reference** (`$('Prepare Day X Message').item.json.id`)
Used in the Update nodes to retrieve the original Airtable record ID. After the Slack HTTP Request executes, `$json` contains the Slack API response — no longer the Airtable record. The cross-node reference reaches back to the Code node's output to get the record ID. This reference and the node ordering are tightly coupled.

**`$vars.SLACK_WEBHOOK_URL`**
n8n environment variable referenced in all Slack-sending HTTP Request nodes in both workflows. Must be set in n8n Settings → Variables before importing either workflow. Keeps the webhook URL out of the committed workflow JSON files.

---

### `profile/loki_candidate_profile.md`

**Version field** (`v1.0`)
The version identifier in the profile header. Read by the Load Profile Document code node in the core pipeline and written to the Profile version field of every Airtable record.

**Hard constraints** (in Review Required)
Non-negotiable requirements: location constraints, salary floor, and other deal-breakers. Always placed in Review Required regardless of inference confidence. Directly affects which roles Instance B flags as disqualifying mismatches.

---

### `profile/loki_conversion_prompt.md`

**Schema block**
The full profile document schema reproduced inline in the conversion prompt (in addition to its definition in the interface contract). Present in the prompt because Claude generates more schema-compliant output when the schema is visible in the current conversation context. Both the conversion prompt schema block and the interface contract schema definition must be updated atomically when the schema changes.

**[PASTE MASTER RESUME HERE]**
The literal placeholder at the end of the conversion prompt. The user replaces this with their resume text before running the prompt in a Claude conversation.

---

### `prompts/project/loki_instance_a_prompt.md` and `loki_instance_b_prompt.md`

**Deferral statement**
The exact text an assessment instance returns when no genuine signals exist. Frozen — the synthesis prompt was written to recognize this specific format. Any paraphrase or reformatting of the deferral statement breaks the synthesis agent's Step 3 interpretation.

**[CANDIDATE PROFILE INSERTED HERE BY n8n]** / **[JOB DESCRIPTION INSERTED HERE BY n8n]**
Literal placeholder markers in the prompt files. Replaced at runtime by n8n expression interpolation in the HTTP Request node body before the API call is made.

**In-progress gap** (Instance B only)
Label applied to a risk signal when the candidate's profile marks the gap as something they are actively closing. Informs the synthesis agent that this risk is bounded, not open-ended.

**Unaddressed gap** (Instance B only)
Label applied to a risk signal when the gap is not acknowledged or being actively closed in the profile. Carries stronger negative weight in synthesis interpretation than an in-progress gap.

---

### `prompts/project/loki_synthesis_prompt.md`

**Four labeled sections**
The required output structure: Conflict Resolution, Collapsed Signals, Deferral Handling, Recommendation. These exact labels are the regex targets in the Parse Synthesis Output code node. Cannot be renamed without updating the parse node atomically.

**Justification**
The one-paragraph explanation that follows the recommendation enum in the synthesis output. Must be specific to the candidate and the role — the synthesis prompt explicitly prohibits generic statements. Stored in the Synthesis field of the Airtable record.

**[INSTANCE A OUTPUT INSERTED HERE BY n8n]** / **[INSTANCE B OUTPUT INSERTED HERE BY n8n]**
Placeholder markers replaced at runtime by n8n expression interpolation in the Run Synthesis HTTP Request node body.
