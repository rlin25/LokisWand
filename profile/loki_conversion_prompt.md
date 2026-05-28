# LokisWand — Candidate Profile Conversion Prompt

You are converting a master resume into a structured candidate profile document
for use as a controlled LLM input in an automated job application assessment
pipeline. Your output will be read by other Claude instances evaluating candidate
fit against job descriptions. Precision and schema compliance are more important
than prose quality.

---

## Instructions

### Step 1 — Draft the full profile document

Using the master resume pasted below, generate a complete candidate profile
document following the schema defined in Step 3 exactly. Do not add sections,
rename sections, or deviate from the schema in any way.

### Step 2 — Place all inferred fields in Review Required

The Review Required section must appear first in the document and must contain
every field you inferred rather than read directly from the resume. This includes:

- **Hard constraints** (location constraints, salary floor, other deal-breakers)
  — always place in Review Required regardless of how confident you are
- **Goals** (target role type, preferred company stage) — always place in Review
  Required regardless of how confident you are
- **Known gaps** (skill gaps the candidate is actively working to close) — always
  place in Review Required regardless of how confident you are

"Inferred" means any field where you are drawing a conclusion from context rather
than reading a stated value. If the resume does not explicitly state a salary
floor, you infer it. If the resume does not explicitly state a preferred company
stage, you infer it from trajectory signals. Place all of these in Review
Required. Do not omit them because they are uncertain — that is exactly why they
need review.

### Step 3 — Draft technical detail fields from resume content

For each experience entry, populate the Technical details field with the actual
technologies, tools, methods, and frameworks used in that role. Draw these
directly from the resume text. Do not infer technologies not mentioned. Do not
duplicate achievement bullets — the Technical details field is distinct from the
Achievements field and focuses on what was used, not what was accomplished.

### Step 4 — Infer highlights tags

For each highlight or accolade in the Highlights and Accolades section, infer
tags indicating what role types, skills, or competencies it is most relevant to.
Tags should be short (one to three words each) and specific enough to be useful
for matching against job descriptions (e.g. "RAG pipelines", "multi-agent
systems", "CI/CD", "financial domain", "evaluation pipelines"). Do not use
generic tags like "engineering" or "AI".

### Step 5 — Set the version field

Set the Version field to v1.0 and the Last updated field to today's date.

---

## Schema

Output the profile document following this schema exactly. Every section and
field must be present. Do not add fields. Do not rename fields. Do not reorder
sections.

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

---

## Constraints

- Output the profile document only. No preamble, no explanation, no commentary
  after the document.
- Do not add any section not present in the schema.
- Do not omit any section present in the schema, even if the resume provides
  no content for it — use a placeholder and flag it in Review Required.
- If a field is ambiguous, make your best inference and place it in Review
  Required. Do not leave fields blank outside of Review Required.
- The Review Required section must appear first, before the Profile section.
- Hard constraints, goals, and known gaps must always appear in Review Required —
  this is unconditional, regardless of your confidence level.

---

## Master Resume

[PASTE MASTER RESUME HERE]