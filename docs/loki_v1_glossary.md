# LokisWand — Glossary v1

Terms are organized by domain cluster. This glossary covers domain concepts, system-specific terms, and third-party tools used in LokisWand. It is intended as a reference for both implementation and interview preparation.

---

## 1. Project Identity

**LokisWand**
The name of this project. Named following the Norse mythology convention established by Huginn. Loki is the Norse trickster god of cunning and transformation; the wand is the instrument of that cunning. Thematically appropriate for a tool that reads situations and reveals hidden signals.

**Interface contract**
The document specifying every input, output, data flow, prompt directive, schema, and failure behavior in LokisWand before implementation begins. Stored as `loki_v1_interface_contract.md`. The authoritative reference for Claude Code during implementation.

**Design decisions document**
The locked record of every architectural and behavioral choice made during the design phase of LokisWand. Stored as `loki_v1_design_decisions.md`. Claude Code must not deviate from any locked decision during implementation.

**Masterplan**
The implementation roadmap for LokisWand, organized into eight phases with acceptance criteria at each gate. Stored as `loki_v1_masterplan.md`.

---

## 2. User Interaction and Input

**Job description**
The full text of a job listing, pasted by the user into the n8n form trigger. The primary input to the core pipeline. URL fetching is explicitly not used — the user performs Ctrl+A, copy, and paste from the listing page. See: Decision 1.

**Form trigger**
The n8n node that initiates the core pipeline. Generates a hosted form URL with a single long text field for pasting the job description. The user's only recurring interaction point with the system.

**Master resume**
The source document used to generate the candidate profile. A comprehensive markdown or .docx file containing all resume information across all roles and versions. Not passed directly to the LLM — converted to a structured profile document first. See: Decision 5.

**Conversion prompt**
The one-time Claude prompt used to generate the candidate profile document from the master resume. Stored in `profile/loki_conversion_prompt.md`. Instructs Claude to follow the profile document schema exactly and place all inferred fields in the Review Required section.

---

## 3. Candidate Profile Document

**Profile document**
The structured markdown file used as the primary LLM input for all candidate fit assessments. Generated once from the master resume via the conversion prompt, reviewed by the user, and uploaded to n8n as a static reference. Contains identity, skills, experience with technical details, education, highlights with relevance tags, and goals and known gaps. See: Decision 5, Decision 6.

**Review Required**
The first section of the candidate profile document. Contains all fields that Claude inferred during document generation and that require user verification before the profile is used. Always includes hard constraints, goals, and known gaps regardless of Claude's confidence level. Structured separation — rather than color coding or emoji flags — makes inferred fields visually scannable.

**Technical details field**
A field within each experience entry in the candidate profile document. Lists the actual technologies, tools, methods, and frameworks used in that role — distinct from the achievement-focused bullet points. Provides Instance A and Instance B with concrete technical evidence for fit and risk signal identification. See: Decision 8.

**Goals and known gaps**
A section of the candidate profile document capturing target role type, preferred company stage, location constraints, salary floor, known skill gaps actively being closed, and hard constraints. Always placed in the Review Required section regardless of Claude's confidence level. Used by Instance B to distinguish between gaps that are in-progress and gaps that are entirely unaddressed.

**Known gaps**
A subsection of the Goals and known gaps section of the profile document. Lists skill gaps the candidate is actively working to close. Used by Instance B to distinguish between gaps that are in-progress and gaps that are entirely unaddressed, producing more accurate risk assessments.

**Hard constraints**
Fields within the Goals and known gaps section of the profile document representing non-negotiable requirements — location, salary floor, and other deal-breakers. Always placed in the Review Required section for user verification. Used by Instance B to flag genuine disqualifying mismatches.

**Profile version**
A field in the profile document header and in every Airtable record. Tracks which version of the candidate profile was used to produce a given assessment. Allows the user to identify stale assessments after a profile update. Format: `v1.0`, `v1.1`, etc. See: Decision 15.

**Static file reference**
The mechanism by which the candidate profile document is made available to n8n workflows. The profile document is uploaded once to n8n and referenced by both Instance A and Instance B nodes as a fixed input. Must be manually re-uploaded if the profile document is updated.

---

## 4. Assessment Pipeline

**Core pipeline**
The primary n8n workflow in LokisWand. Triggered by form submission, it runs metadata extraction, duplicate detection, dual assessment, synthesis, Airtable write, and Slack confirmation in sequence. Stored as `workflows/loki_core_pipeline.json`.

**Metadata extraction**
Step 1 of the core pipeline. A Claude API call that extracts company name, role title, location, and employment type from the pasted job description text. Returns null for any field that cannot be determined. See: Decision 14.

**Duplicate detection**
The pre-write check performed before creating a new Airtable record. n8n queries for an existing record matching the same company name and role title. If a match is found, the pipeline halts and sends a Slack notification rather than creating a duplicate record. See: Decision 13.

**Parallel assessment**
The step in the core pipeline where Instance A and Instance B are called simultaneously rather than sequentially. Reduces total pipeline execution time by running both Claude API calls at the same time.

**Complementary framing**
The design pattern used to differentiate Instance A and Instance B. Rather than adversarial personas (optimist vs. skeptic), the two instances are given complementary tasks — fit signals and risk signals respectively — that produce genuine variance without manufacturing conflict. See: Decision 2.

**Instance A**
The first of two parallel Claude API calls in the core pipeline. Prompted to identify the strongest fit signals between the candidate profile and the job description. May defer to synthesis if no genuine fit signals exist. See: Decision 2, Decision 3.

**Instance B**
The second of two parallel Claude API calls in the core pipeline. Prompted to identify the strongest risk signals between the candidate profile and the job description. Explicitly distinguishes between in-progress gaps and unaddressed gaps. May defer to synthesis if no genuine risk signals exist. See: Decision 2, Decision 3.

**Fit signals**
The output of Instance A. A numbered list of the strongest indicators that the candidate is a good match for the role, drawn from the candidate profile and compared against the job description. Stored in the Fit signals field of the Airtable record.

**Risk signals**
The output of Instance B. A numbered list of the strongest indicators that the candidate may not be a good match for the role, drawn from gaps between the candidate profile and the job description. Distinguishes between in-progress gaps and unaddressed gaps. Stored in the Risk signals field of the Airtable record.

**Deferral**
The behavior of Instance A or Instance B when no genuine signals exist to report. Rather than manufacturing signals to meet a count, the instance returns a statement deferring to the synthesis agent with a brief explanation. The synthesis prompt treats a deferral as meaningful evidence — a deferral from Instance B signals strong fit; a deferral from Instance A signals weak fit. See: Decision 3.

**Assessment**
The output of a single Claude instance evaluation of a candidate's profile against a job description. LokisWand produces two assessments per submission — one fit assessment from Instance A and one risk assessment from Instance B — before passing both to the synthesis call.

**Synthesis**
The third Claude API call in the core pipeline. Receives the outputs of Instance A and Instance B and produces a unified recommendation. Resolves genuine conflicts between assessments, collapses false conflicts into confident statements, incorporates deferrals as meaningful evidence, and concludes with a fixed enum recommendation and one paragraph justification. See: Decision 4.

**Recommendation**
The final output of the synthesis call. A fixed enum value: Strong Fit, Moderate Fit, Weak Fit, or Do Not Apply. Stored as a single select field in Airtable and included in the Slack confirmation message on pipeline completion.

**Strong Fit**
One of four possible synthesis recommendation values. Indicates the synthesis agent has determined the candidate is a strong match for the role with minimal meaningful gaps.

**Moderate Fit**
One of four possible synthesis recommendation values. Indicates the candidate has meaningful fit with the role but notable gaps or risks exist.

**Weak Fit**
One of four possible synthesis recommendation values. Indicates the candidate has some relevant experience but significant gaps exist that make the role a poor match.

**Do Not Apply**
One of four possible synthesis recommendation values. Indicates the synthesis agent has determined the candidate should not apply to this role. The strongest negative recommendation.

---

## 5. Follow-up and Notifications

**Follow-up nudge workflow**
The secondary n8n workflow in LokisWand. Runs on a daily schedule at 9:00 AM user local time. Queries Airtable for Applied records where day 7 or day 14 follow-up nudges are due and have not yet been sent. Posts Slack reminders and updates Airtable checkboxes. Stored as `workflows/loki_followup_nudge.json`.

**Nudge**
A Slack message sent by the follow-up nudge workflow reminding the user to follow up on a job application. Sent at day 7 and day 14 after the application status is set to Applied.

**Range query**
The follow-up nudge workflow's approach to querying Airtable for eligible records. Queries for records where date submitted is 7 or more days ago (capped at 21 days) rather than exactly 7 days ago. Prevents permanent nudge loss if the workflow fails to run on the exact target day. See: Decision 12.

**Schedule trigger**
The n8n node that initiates the follow-up nudge workflow. Configured to run once daily at 9:00 AM user local time.

**Follow-up sent day 7**
An Airtable checkbox field. Set to true after the day 7 follow-up nudge has been sent for a given record. Prevents duplicate nudges on subsequent workflow runs.

**Follow-up sent day 14**
An Airtable checkbox field. Set to true after the day 14 follow-up nudge has been sent for a given record. Prevents duplicate nudges on subsequent workflow runs.

---

## 6. Data Storage and Visualization

**Airtable**
A cloud-based database and spreadsheet hybrid used as the persistent data store for LokisWand. Chosen for its native kanban view, date field filtering, and n8n integration. All job application records are written to and read from Airtable.

**Airtable base**
The top-level container in Airtable, analogous to a database. LokisWand uses a single base named LokisWand containing one table: Applications.

**Airtable kanban view**
A visual board view in Airtable that groups records by a single select field — in LokisWand, the Status field. Provides the pipeline visualization surface without requiring custom UI development.

**Status**
An Airtable single select field tracking the current state of a job application. Options: Researching, Applied, Following Up, Interviewing, Offer, Rejected, Skipped. Updated manually by the user. Records in Applied status are eligible for follow-up nudge evaluation.

**Applied**
An application status value indicating the user has submitted an application for this role. Records in Applied status are eligible for follow-up nudge evaluation by the nudge workflow.

**Single select field**
An Airtable field type storing one value from a predefined list of options. Used in LokisWand for Recommendation and Status.

**Checkbox field**
An Airtable field type storing a boolean value. Used in LokisWand for Follow-up sent day 7 and Follow-up sent day 14 — set to true after each nudge is sent to prevent duplicate notifications.

**Employment type**
A metadata field extracted from the job description by Claude in Step 1 of the core pipeline. Values include full-time, part-time, and contract. Written to the Airtable record.

---

## 7. Infrastructure and Tooling

**n8n**
An open-source workflow automation tool used to build and run both the core pipeline and the follow-up nudge workflow. Chosen for its native Claude API integration, Airtable integration, Slack integration, form trigger capability, and scheduled workflow support.

**n8n cloud**
The hosted version of n8n used by LokisWand. Required for reliable follow-up nudge scheduling — self-hosted instances may have uptime constraints that cause missed nudge windows.

**Node**
The basic unit of an n8n workflow. Each node performs a single action — triggering, transforming data, calling an API, writing to a database, or sending a message. The core pipeline is composed of nodes connected in sequence with parallel branches for dual assessment.

**Workflow**
An n8n automation composed of connected nodes. LokisWand contains two workflows: the core pipeline and the follow-up nudge workflow.

**HTTP request node**
The n8n node type used to make Claude API calls within the core pipeline. Used for metadata extraction, Instance A, Instance B, and synthesis calls.

**Claude API**
The Anthropic API used to make programmatic calls to Claude models within n8n workflows. LokisWand makes three Claude API calls per job submission: Instance A, Instance B, and synthesis.

**Slack**
The messaging platform used for follow-up nudges and pipeline notifications in LokisWand. A personal Slack workspace with a dedicated job-search channel receives all system messages. Chosen over email for signal-to-noise ratio and relevance to Alloy's internal tooling context. See: Decision 11.
