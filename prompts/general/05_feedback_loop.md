# Prompt: Phase 7 — Feedback Loop Documentation Update

**Phase:** Post-implementation feedback loop
**When to use:** After all subplans are complete and all tests pass. Run this before beginning next-version design.
**Produces:** Updated versions of all design documents, reconciled against the implemented codebase.

---

## Before Running This Prompt

Fill in the following placeholders before handing this to Claude Code:

| Placeholder | Replace with |
|---|---|
| `{PROJECT_NAME}` | The name of the project (e.g. LokisWand) |
| `{VERSION}` | The current version (e.g. v1) |
| `{NEXT_VERSION}` | The next planned version (e.g. v2) |
| `{SOURCE_FILES}` | The full list of source files in the project |
| `{TEST_FILES}` | The full list of test files in the project |
| `{INTERFACE_CHECKS}` | The specific interface specifications to verify — drawn from the interface contract, one bullet per component |
| `{DECISION_CHECKS}` | The specific locked decisions most likely to have shifted under implementation pressure — drawn from the design decisions document |
| `{DEFERRED_IMPLEMENTATIONS}` | Items the interface contract explicitly deferred to implementation — draw from any "deferred to implementation" notes in the contract |
| `{FIRST_NEW_DECISION_NUMBER}` | The number of the first new decision to add — one after the last locked decision in the design decisions document |

---

## What This Prompt Does

The design documents were written before the code. This prompt reconciles them with what was actually built. It does not redesign anything — it records reality. Every update must be traceable to something observable in the codebase.

The output is a set of fully integrated updated documents, not bolt-ons. Each document is regenerated as a complete, clean file. Nothing is appended to the end.

---

## Inputs

Read every file in the following locations before doing anything else:

**Source files (the implemented codebase):**
{SOURCE_FILES}

**Test files:**
{TEST_FILES}

**Design documents (the plan):**
- `{project}_v{version}_masterplan.md`
- `{project}_v{version}_design_decisions.md`
- `{project}_v{version}_glossary.md`
- `{project}_v{version}_interface_contract.md`
- `{project}_v{version}_setup_notes.md`
- `prompts/01_design_methodology.md`

Read all of them before forming any conclusions. Do not update any document until all source files have been read.

---

## Inspection Pass

Before writing any updates, work through the following four inspection categories systematically. For each finding, note: which document it affects, what the discrepancy is, and whether it is a correction (the plan was wrong), a gap (the plan was silent), or a status update (the plan was right but the status language is stale).

Do not skip categories because they seem unlikely to have findings. Every category must be checked.

### Category 1 — Interface Drift

Compare every interface specification in `{project}_v{version}_interface_contract.md` against the actual implemented code.

Check each of the following explicitly:
{INTERFACE_CHECKS}

Flag any discrepancy, no matter how small. A field renamed from `document_id` to `doc_id` is a discrepancy.

### Category 2 — Decisions That Changed Under Implementation Pressure

Read through `{project}_v{version}_design_decisions.md`. For each decision, ask: does the implemented code actually reflect this decision, or was it quietly adjusted during the build?

Pay particular attention to:
{DECISION_CHECKS}

If any decision was modified, record the modification as a new decision addendum with the same format: what changed, why it changed, and what was rejected.

### Category 3 — Implementation Details That Became Decisions

The interface contract explicitly defers the following to implementation. Read the relevant source files and record what was actually decided for each:
{DEFERRED_IMPLEMENTATIONS}

Each of these should be added to `{project}_v{version}_design_decisions.md` as new decisions (Decision {FIRST_NEW_DECISION_NUMBER} onward) using the standard format: Decision, Reasoning, Rejected option. They are not new design choices — they are implementation choices that need to be recorded so they are not lost before the next version.

### Category 4 — New Known Issues

Read `requirements.txt` (or equivalent dependency file) and note the actual pinned versions of all dependencies. Compare against the install command in `{project}_v{version}_setup_notes.md`.

Then check: were there any dependency conflicts, version issues, import errors, or runtime surprises encountered during the build that are not currently recorded in the setup notes? If any test files contain comments about workarounds or unexpected behavior, those are candidates.

Add any new issues to `{project}_v{version}_setup_notes.md` in the existing Known Issues format.

---

## Output Documents

Generate fully integrated updated versions of all documents. Do not append to existing documents — regenerate each one clean.

### 1. `{project}_v{version}_design_decisions.md`

- Update the status header to reflect that implementation is complete.
- Update "Next phase" to "{NEXT_VERSION} design."
- Record any decisions that changed (Category 2 findings) as addenda to the relevant existing decisions, clearly marked as implementation-phase updates.
- Add all Category 3 findings as new numbered decisions (Decision {FIRST_NEW_DECISION_NUMBER} onward) using the standard format: Decision, Reasoning, Rejected option.
- Do not alter any existing decision text. Additions only.

### 2. `{project}_v{version}_interface_contract.md`

- Update the status header to reflect implementation complete.
- Correct any interface drift found in Category 1. For each correction, add a one-line note: `[Updated post-implementation: <what changed and why>]` directly below the corrected specification.
- Do not alter specifications that matched the implementation.

### 3. `{project}_v{version}_masterplan.md`

- Update all status headers to reflect implementation complete.
- Update the folder structure diagram if the actual folder structure differs from the plan.
- Update component descriptions if any component behaves differently than described.
- Do not redesign anything — only correct descriptions that no longer match the code.

### 4. `{project}_v{version}_setup_notes.md`

- Update the status header.
- Update the install command if the dependency file reveals the actual installed packages differ from the listed command.
- Add a pinned versions table drawn from the dependency file — this is the version set that is known to work.
- Add any new known issues found in Category 4.

### 5. `{project}_v{version}_glossary.md`

- Update any term definitions that no longer match the implementation. Cross-reference with the actual source files, not the design documents.
- Do not add new terms — that is the job of `04_rewrite_glossary.md`.

### 6. `prompts/01_design_methodology.md`

- Update the status line at the top of the document to reflect the current phase.
- Do not alter any other section.

---

## Constraints

- Every update must be traceable to something in the source files. Do not infer, assume, or invent.
- If a source file is ambiguous, flag the ambiguity explicitly rather than guessing.
- If a design document section has no corresponding discrepancy, reproduce it unchanged.
- Generate all documents in a single pass after completing the full inspection. Do not generate documents mid-inspection.
- Use the pending changes list method: maintain a running list of all findings as you inspect, then generate documents from the list at the end.
