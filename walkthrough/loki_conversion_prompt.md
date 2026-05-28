# Walkthrough: loki_conversion_prompt.md

**Source file:** `profile/loki_conversion_prompt.md`
**Role:** One-time setup prompt — converts a master resume into the structured candidate profile document

---

## Purpose

The conversion prompt exists because the master resume cannot be used as a direct LLM input. Resume formatting varies, content organization varies, and inference about hard constraints and known gaps will differ across calls without explicit instructions to surface them. The conversion prompt is the mechanism that transforms an uncontrolled input (the master resume) into a controlled one (the profile document). It runs once — or whenever the master resume changes significantly — and the output is the document that governs all downstream assessments.

---

## Relationships

**Used in:** A standalone Claude conversation (not in n8n). The user pastes the conversion prompt and their master resume into a Claude conversation, reviews the output, corrects it, and saves the result as `profile/loki_candidate_profile.md`.

**Produces:** `profile/loki_candidate_profile.md`

**Not referenced by:** Either n8n workflow. The workflows consume the profile document, not the conversion prompt.

---

## Decision rationale

### Hard constraints, goals, and known gaps always go in Review Required — unconditionally
The prompt states this three times in three different forms: as a list item in the instructions, as a bolded note alongside each field type, and again in the Constraints section. The repetition is deliberate. Claude will tend to place fields it's confident about outside Review Required — for example, if a resume has a clear New York address, Claude might omit location constraints from Review Required because it's "obvious." The unconditional instruction overrides this tendency. These fields must always be verified by a human before the profile is used, regardless of Claude's confidence level, because they are the fields most costly to have wrong.

### Schema reproduced in full inside the prompt
The profile document schema appears twice: once in the interface contract and once embedded in the conversion prompt. This redundancy is intentional. Claude generates schema-compliant output more reliably when the target schema is visible in the current conversation context rather than referenced by name or described abstractly. Including the full schema in the prompt eliminates the gap between what the interface contract specifies and what Claude actually produces.

### "Inferred" defined explicitly
The prompt defines "inferred" precisely: "any field where you are drawing a conclusion from context rather than reading a stated value." Without this definition, Claude applies an inconsistent standard — it might place only very uncertain fields in Review Required and leave other inferred fields (like a salary floor derived from market rate patterns) in the profile body where they would not be verified. The explicit definition creates a clear threshold: stated value → profile body; drawn conclusion → Review Required.

### Output constraints remove friction from the save step
"Output the profile document only. No preamble, no explanation, no commentary after the document." Without this constraint, Claude wraps the profile in a conversational frame ("Here is the candidate profile I've generated based on your resume...") that the user must manually strip out before saving. The constraint produces a clean document that can be saved directly.

### "[PASTE MASTER RESUME HERE]" placeholder
The conversion prompt ends with a literal placeholder rather than an instruction like "the resume will be pasted below." This is the simplest possible user interaction: replace the placeholder text with the resume content before running the prompt. It is less ambiguous than asking the user to add content in a specific location or format.

---

## What it does not do

- **Does not validate resume format** — it accepts any resume structure and relies on Claude to handle format variance, which Claude does more reliably than format-specific parsing logic
- **Does not run in n8n** — it is a standalone Claude conversation step. Running it in n8n would add automation for a one-time operation and create dependency on n8n for profile generation
- **Does not check the previous profile version** — when re-running after a resume update, the conversion prompt generates a fresh profile from scratch. The user is responsible for comparing the new output to the existing profile and incrementing the version field
- **Does not produce any output other than the profile document** — error cases (e.g., a resume with insufficient information to fill a required field) result in placeholder values and Review Required entries, not error messages

---

## v2 touch points

- **Schema coupling**: The schema block in this prompt and the profile document schema in the interface contract are the same schema. Any v2 change to the profile document schema (adding a field, changing a section name, adding a new section) must update both the interface contract and this prompt atomically. They will drift if updated independently.
- **Version field**: The conversion prompt instructs Claude to set the version field to "v1.0" and last updated to today's date. When a v2 profile schema is introduced, this hardcoded instruction must change to match the new version format.
