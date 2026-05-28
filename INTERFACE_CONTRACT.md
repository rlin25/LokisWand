# INTERFACE_CONTRACT.md — LokisWand Interface Contract v1

This document specifies every component boundary in LokisWand: what each component accepts as input, what it returns as output, what data types are involved, and how it behaves on failure. It was written before any workflow node was built, and the implementation was built against it — not the other way around. Use it as the authoritative reference for understanding what the system does: if the workflow and this document disagree, the document is right and the workflow needs to be corrected. Readers evaluating the system design should start here; readers evaluating architectural choices should follow the links to [DESIGN_DECISIONS.md](DESIGN_DECISIONS.md).

---

## User Interaction Surface

The entire system exposes exactly two points of user interaction:

1. **One-time setup:** Paste master resume into a Claude conversation, review the generated profile document, save
2. **Per application:** Open the n8n form, paste the full job description text, submit

No other user input is required at any point in the system.

---

## Section 1 — Profile Document Conversion Workflow

*Governing decisions: Decision 5, Decision 6, Decision 7, Decision 8, Decision 9*

### Purpose
Convert a candidate's master resume into a structured profile document for use as a controlled LLM input throughout the core pipeline.

### Input
| Field | Format | Source |
|---|---|---|
| Master resume | Markdown or .docx | User paste into Claude conversation |

### Process
1. Claude receives the master resume and a conversion prompt
2. Claude drafts the full profile document following the defined schema
3. All inferred fields that require verification are surfaced in the Review Required section at the top of the document
4. User reviews the Review Required section, corrects any inaccuracies, and saves the document as a markdown file
5. The saved profile document is uploaded to n8n as a static reference file

### Output
| Field | Format | Destination |
|---|---|---|
| Structured profile document | Markdown (.md) | Saved locally, uploaded to n8n as a static reference file |

### Profile Document Schema

```markdown
# Candidate Profile
**Last updated:** [date]
**Version:** v1.0

---

## Review Required
*All fields in this section were inferred by Claude.
Verify before first use.*

### Hard Constraints
- **Location constraints:** [inferred]
- **Salary floor:** [inferred]
- **Other hard constraints:** [inferred]

### Goals
- **Target role type:** [inferred]
- **Preferred company stage:** [inferred]

### Known Gaps
- [inferred gap]
- [inferred gap]

---

## Profile

### Identity
- **Name:**
- **Headline:**
- **Summary:**

### Skills
#### [Category]
- Skill, Skill, Skill

### Experience
#### [Company] — [Title] — [Dates]
**Achievements:**
- Achievement bullet

**Technical details:**
[Technologies, tools, methods, frameworks actually used in this role]

### Education
#### [Institution] — [Degree] — [Year]

### Highlights and Accolades
#### [Name]
**Description:**
**Tags:** [tag, tag, tag]
```

### Constraints
- Output must follow the defined schema exactly — no freeform sections
- Review Required section must appear first in every generated document
- Hard constraints, goals, and known gaps must always appear in Review Required regardless of Claude's confidence level
- No fields outside the schema may be added without a schema version update

---

## Section 2 — Core Pipeline

*Governing decisions: Decision 1, Decision 2, Decision 3, Decision 4, Decision 10, Decision 11, Decision 12, Decision 13, Decision 14, Decision 15*

### Purpose
Accept a pasted job description, evaluate candidate fit using dual Claude assessments and a synthesis call, log all results to Airtable, and trigger follow-up nudges via Slack at defined intervals.

### Trigger
| Field | Format | Source |
|---|---|---|
| Job description text | Plain text | n8n hosted form, user paste |

### Step 1 — Metadata Extraction

*Governing decisions: Decision 1, Decision 14*

The pasted job description text is passed to Claude to extract structured metadata before assessment begins.

**Input:**
- Raw job description text

**Extraction directive:**
Extract the following fields from the job description. If a field cannot be determined, return null.
- Company name
- Role title
- Location
- Employment type (full-time, part-time, contract)

**Output:**
| Field | Format | Notes |
|---|---|---|
| Company | Text | Extracted by Claude |
| Role title | Text | Extracted by Claude |
| Location | Text | Extracted by Claude |
| Employment type | Text | Extracted by Claude |

**Failure behavior:** If extraction produces null for both company and role title, log error to Airtable, notify user via Slack, halt pipeline.

---

### Step 2 — Dual Assessment

*Governing decisions: Decision 2, Decision 3*

Two Claude instances receive identical context but differentiated prompts. Calls are made in parallel.

**Shared context passed to both instances:**
- Structured profile document (full text)
- Raw job description text

**Instance A prompt directive:**
Identify the strongest signals that this candidate is a strong fit for this role. Be specific — reference exact skills, experiences, or highlights from the profile that map to explicit requirements in the listing. Format your response as a numbered list. Each item must include a signal name and a one to two sentence explanation.

If fewer than three genuine fit signals exist, return only the signals that are real — do not manufacture signals to meet a count. If no genuine fit signals exist, return a single statement deferring to the synthesis agent with a brief explanation of why.

**Instance B prompt directive:**
Identify the strongest risk signals for this candidate against this role. Be specific — reference exact gaps, mismatches, or missing requirements relative to the listing. Distinguish explicitly between gaps marked as in-progress in the candidate's profile and gaps that are entirely unaddressed. Format your response as a numbered list. Each item must include a signal name and a one to two sentence explanation.

If fewer than three genuine risk signals exist, return only the signals that are real — do not manufacture signals to meet a count. If no genuine risk signals exist, return a single statement deferring to the synthesis agent with a brief explanation of why.

**Output per instance:**
| Field | Format | Notes |
|---|---|---|
| Assessment | Numbered list of three items | Each item is a named signal with a one to two sentence explanation |
| Instance identifier | String | "fit" or "risk" |

---

### Step 3 — Synthesis

*Governing decisions: Decision 3, Decision 4*

A third Claude instance receives both assessments and produces a unified output.

**Input:**
- Instance A output (fit signals)
- Instance B output (risk signals)
- Synthesis prompt directive

**Synthesis prompt directive:**
You have received two assessments of the same candidate against the same role. One identifies fit signals. The other identifies risk signals. Either or both assessments may have deferred to you if no genuine signals were found — treat a deferral as meaningful signal in itself, not as an error.

First, identify any points where the two assessments appear to conflict and resolve each conflict with explicit reasoning.

Second, identify any apparent conflicts that are not genuine — where both assessments are effectively saying the same thing from different angles — and collapse each into a single confident statement.

Third, if one or both instances deferred, incorporate that deferral into your reasoning explicitly — a deferral from Instance A suggests weak fit evidence; a deferral from Instance B suggests weak risk evidence.

Fourth, conclude with a clear recommendation using exactly one of the following: Strong Fit, Moderate Fit, Weak Fit, Do Not Apply. Follow the recommendation with a justification of one paragraph maximum.

**Output:**
| Field | Format | Notes |
|---|---|---|
| Conflict resolution | Prose | Addresses genuine conflicts between assessments |
| Collapsed signals | Prose | Resolves false conflicts into confident statements |
| Recommendation | Enum | One of: Strong Fit, Moderate Fit, Weak Fit, Do Not Apply |
| Justification | Prose | One paragraph maximum |

---

### Step 4 — Log to Airtable

*Governing decisions: Decision 10, Decision 15*

All pipeline outputs are written to a single Airtable record on pipeline completion.

**Airtable schema:**
| Field | Type | Source |
|---|---|---|
| Company | Text | Extracted by Claude in Step 1 |
| Role title | Text | Extracted by Claude in Step 1 |
| Location | Text | Extracted by Claude in Step 1 |
| Employment type | Text | Extracted by Claude in Step 1 |
| Date submitted | Date | System timestamp |
| Fit signals | Long text | Instance A output |
| Risk signals | Long text | Instance B output |
| Synthesis | Long text | Instance C output |
| Recommendation | Single select | Strong Fit, Moderate Fit, Weak Fit, Do Not Apply |
| Status | Single select | Researching, Applied, Following Up, Interviewing, Offer, Rejected, Skipped |
| Follow-up sent day 7 | Checkbox | Updated by follow-up workflow |
| Follow-up sent day 14 | Checkbox | Updated by follow-up workflow |
| Profile version | Text | Read from profile document version field |

**Default values on record creation:**
- Status: Researching
- Follow-up sent day 7: false
- Follow-up sent day 14: false

---

### Step 5 — Follow-up Nudge Workflow

*Governing decisions: Decision 11, Decision 12*

A separate n8n workflow runs on a daily schedule and checks Airtable for records meeting nudge criteria.

**Trigger:** Daily schedule — runs once per day at 9:00 AM user local time

**Query logic:**
- Status is "Applied"
- AND one of the following:
  - Date submitted is 7 or more days ago (and less than 21 days ago) AND Follow-up sent day 7 is false
  - Date submitted is 14 or more days ago (and less than 21 days ago) AND Follow-up sent day 14 is false

**Slack message format:**
```
Follow-up Reminder
Company: [company]
Role: [role title]
Applied: [date submitted]
Day [7 or 14] follow-up due — have you heard back?
```

**Post-nudge behavior:**
- n8n updates the corresponding Airtable checkbox to true
- No further nudges are sent for that interval regardless of application status

---

### Failure Handling

| Failure point | Behavior |
|---|---|
| Metadata extraction returns null for company and role title | Log error to Airtable, Slack notification to user, halt |
| Instance A or B fails | Log error to Airtable, Slack notification, halt before synthesis |
| Synthesis fails | Log partial results to Airtable, Slack notification, halt |
| Airtable write fails | Slack notification with raw output, user saves manually |
| Follow-up query fails | Slack notification, retry next day |

---

## Versioning
This is Interface Contract v1. Any change to input schema, output schema, Airtable fields, or prompt directives constitutes a version increment. The profile document schema version and the pipeline contract version are tracked independently.
