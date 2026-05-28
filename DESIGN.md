# DESIGN.md — LokisWand System Design

## System Overview

LokisWand is a job application intelligence pipeline built in n8n. It accepts a pasted job description, evaluates candidate fit using dual Claude assessments and a synthesis call, logs all results to Airtable, and delivers follow-up nudges via Slack at defined intervals. The system is designed for minimum viable user input: one-time profile setup and per-application job description paste.

---

## The Components

### Profile Document Conversion
A one-time operation that converts a master resume into a structured markdown profile document for use as a controlled LLM input throughout the core pipeline.

**Accepts:** Master resume (markdown or .docx), pasted into a Claude conversation along with the conversion prompt.
**Returns:** Structured profile document following a defined schema, with all inferred fields grouped into a Review Required section at the top.
**Does not:** Run inside n8n. This is a human-in-the-loop operation performed once via a Claude conversation. The resulting file is embedded in the core pipeline workflow as static text.

The Review Required section always surfaces hard constraints, goals, and known gaps for user verification regardless of Claude's confidence level. The profile document is version-stamped (e.g., `v1.0`) and that version is logged in every Airtable record it produces.

---

### Core Pipeline
The primary n8n workflow, triggered by form submission. Runs five sequential steps with parallel execution at the assessment stage.

**Accepts:** A pasted job description submitted via an n8n hosted form.
**Returns:** An Airtable record containing metadata, fit signals, risk signals, synthesis output, and recommendation; a Slack confirmation message on success.
**Does not:** Fetch URLs, parse HTML, or accept any input other than pasted text.

#### Step 1 — Metadata Extraction
Claude extracts company name, role title, location, and employment type from the raw job description text. Returns null for any field that cannot be determined. If both company and role title return null, the pipeline halts with a Slack alert.

#### Step 2 — Duplicate Detection
Before assessment begins, n8n queries Airtable for an existing record matching the same company name and role title. If a match is found, the pipeline halts with a Slack notification. No duplicate record is created.

#### Step 3 — Dual Assessment (parallel)
Two Claude API calls run simultaneously:

- **Instance A** identifies the strongest fit signals — evidence the candidate is a strong match for the role
- **Instance B** identifies the strongest risk signals — evidence of gaps or mismatches

Both instances receive the full profile document and the raw job description. Either instance may defer to synthesis if no genuine signals exist; a deferral is transmitted as meaningful evidence, not treated as an error.

#### Step 4 — Synthesis
A third Claude call receives both assessment outputs and produces a unified recommendation. It resolves genuine conflicts between the two assessments, collapses false conflicts into confident statements, incorporates any deferrals as evidence, and concludes with a fixed enum recommendation: Strong Fit, Moderate Fit, Weak Fit, or Do Not Apply. The recommendation is followed by a one-paragraph justification.

#### Step 5 — Airtable Write and Notification
All outputs are written to a single Airtable record. A Slack confirmation message fires on success. On partial failure (synthesis completes but Airtable write fails), the raw output is sent to Slack for manual recovery.

---

### Follow-up Nudge Workflow
A separate n8n workflow that runs on a daily schedule and delivers Slack reminders for overdue application follow-ups.

**Accepts:** Nothing — triggered on a daily timer at 9:00 AM.
**Returns:** Slack messages for qualifying records; Airtable checkbox updates after each nudge.
**Does not:** Modify any fields other than Follow-up sent day 7 and Follow-up sent day 14.

Queries Airtable for records where Status is Applied, the corresponding follow-up checkbox is false, and the date submitted falls within the relevant window (7–21 days for day 7, 14–21 days for day 14). The range query rather than exact date match ensures nudges are not permanently lost if the workflow fails to run on a given day.

---

## Key Architectural Patterns

### Three-call LLM orchestration
Each job submission triggers exactly three Claude API calls: metadata extraction, parallel dual assessment, and synthesis. The three-call structure creates a genuine evaluation chain where each stage has a constrained task, rather than one call attempting to do everything.

The two parallel calls in the assessment stage reduce total pipeline latency — Instance A and Instance B execute simultaneously rather than sequentially.

### Complementary framing
Instance A and Instance B are not adversarial (optimist vs. skeptic). Each has a constrained, concrete task: fit signals and risk signals respectively. This produces genuine variance without manufacturing conflict. If a candidate is a strong fit, Instance B should produce few or no risk signals — not invent them to appear balanced. The synthesis call then resolves real inputs rather than performed disagreement.

### Deferral as signal
Both assessment instances may return a deferral statement instead of a list of signals when no genuine signals exist. The synthesis prompt treats a deferral from Instance B as positive evidence (no meaningful risks found) and a deferral from Instance A as negative evidence (no meaningful fit found). Deferral is a designed behavior, not an error condition.

### Range query with idempotent state
The follow-up nudge workflow uses a range query (7–21 days ago) rather than an exact date match. This prevents permanent nudge loss on workflow failure days. The checkbox updated immediately after each nudge sent ensures re-runs do not produce duplicate notifications. The range query plus the idempotent checkbox together produce exactly-once nudge delivery across workflow failure scenarios.

### Pre-write guard pattern
Duplicate detection runs before any assessment begins. This guards the Airtable tracker against data corruption from repeat submissions without requiring post-hoc deduplication logic. The check is cheap (one Airtable query) and placed at the earliest possible gate.

### Structured LLM input
Rather than passing raw resume text to the assessment instances, a one-time profile document generation step produces a structured markdown file that both instances use as their primary context. This eliminates input format variance across runs and gives the assessment instances concrete technical evidence (via the technical details field in each experience entry) rather than requiring inference from achievement language.

---

## Key Design Decisions

### Complementary framing over adversarial personas (Decision 2)
The most significant LLM orchestration decision in the system. Adversarial personas (optimist vs. skeptic) produce manufactured disagreement — instances invent conflict on points where no genuine ambiguity exists, which pollutes the synthesis input. Complementary framing gives each instance a task it can fail gracefully at (deferral) rather than one it must always produce output for. The synthesis call receives cleaner, more honest inputs.

*Rejected:* Optimist/skeptic persona framing — produces artificial disagreement that degrades synthesis quality.

### Deferral over forced output (Decision 3)
Both instances are explicitly permitted to return no signals if no genuine signals exist. This decision protects assessment reliability at the cost of occasionally sparse output. A forced three-signal output rule would produce manufactured assessments that look complete but mislead. The synthesis prompt is written to treat deferral as evidence rather than an error, which makes the full system more reliable.

*Rejected:* Mandatory signal count — produces manufactured assessments, degrades reliability.

### Claude extraction over regex for metadata (Decision 14)
Job description formatting varies significantly across companies and job boards. Regex patterns that work for one format fail silently on another. Claude handles format variance gracefully and returns explicit null for fields it cannot determine, enabling clean failure handling downstream. The explicit null is more reliable than a confident wrong answer.

*Rejected:* Regex or string parsing — brittle across format variance, fails silently.

### Range query over exact date match (Decision 12)
An exact date match drops follow-up nudges permanently on any day the workflow fails to run. A range query (7+ days, capped at 21) recovers missed nudges on the next successful run without sending duplicates, because the checkbox is set to true immediately after each nudge. The range plus idempotent state combination produces exactly-once delivery.

*Rejected:* Exact date match — permanent nudge loss on workflow failure days.

### Duplicate detection before assessment (Decision 13)
Running duplicate detection before the three Claude API calls prevents tracker corruption while also avoiding unnecessary API spend on submissions the pipeline would reject. The check is inexpensive and placed at the earliest viable gate.

*Rejected:* No duplicate detection — allows silent tracker corruption on repeat submissions.

---

## What This Version Deliberately Excludes

The following were evaluated and explicitly deferred. They are not omissions — they are scope decisions.

- **URL fetching** — job listing pages are unreliable targets for automated fetching due to JavaScript rendering, robots.txt restrictions, and authentication walls. Paste-based input works consistently across all job boards. (Decision 1)
- **Automated profile conversion workflow** — the conversion is a one-time, infrequent operation. A Claude conversation with a conversion prompt is sufficient for v1 and avoids adding infrastructure for an operation that happens rarely. (Decision 6)
- **Skill proficiency tagging** — self-reported proficiency labels degrade in accuracy over time. Inaccurate proficiency tags produce confidently wrong assessments. Claude infers relative skill strength from the experience and highlights sections, which carry higher-quality signal. (Decision 7)
- **High-volume scalability** — three Claude API calls per submission is not cost-efficient at high volume. Acceptable for personal use at v1 scale; a production version would require call consolidation or caching.
