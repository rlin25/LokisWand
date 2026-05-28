# Prompt: Generate Walkthrough Directory

**Purpose:** Generate a `walkthrough/` directory of annotated design rationale documents for every source file in the project.
**When to use:** After all subplans are implemented and the codebase is complete.
**Reusability:** Replace the source file list and reference document names for each new project.

---

## Before Running This Prompt

Fill in the following placeholders before handing this to Claude Code:

| Placeholder | Replace with |
|---|---|
| `{PROJECT_NAME}` | The name of the project (e.g. LokisWand) |
| `{VERSION}` | The current version (e.g. v1) |
| `{NEXT_VERSION}` | The next planned version (e.g. v2) |
| `{SOURCE_FILES}` | The full list of source files in the project |

---

## Prompt

Create a `walkthrough/` directory in the project root. For each source file in the project, create a corresponding markdown file explaining it to someone who understands the overall system design but is reading the implementation for the first time.

Each walkthrough file should cover:

1. **Purpose** — what this file does and why it exists as a separate component. Reference the specific design decisions that created it.
2. **Relationships** — what it imports from, what imports it, and what it deliberately does not touch.
3. **Decision rationale** — for non-obvious implementation choices, explain why the code is written that way rather than what it is doing. Reference design decision numbers where relevant.
4. **What it does not do** — responsibilities explicitly excluded from this file and where those responsibilities live instead.
5. **{NEXT_VERSION} touch points** — which parts of this file are intentionally minimal in {VERSION} and what will change in {NEXT_VERSION}.

Do not explain syntax. Do not describe what lines of code do — describe why they exist. If a section of code is self-explanatory, skip it.

Name each walkthrough file to match its source file: `walkthrough/module_name.md` for `path/to/module.py`, and so on.

Source files to cover:
{SOURCE_FILES}

Reference documents:
- `{project}_v{version}_interface_contract.md`
- `{project}_v{version}_design_decisions.md`
- `{project}_v{version}_masterplan.md`

---

## Notes

- Run this after all subplans are complete and the Phase 7 feedback loop is done — the walkthrough is only useful once the source files and design documents are fully reconciled.
- The goal is decision rationale, not documentation. If a walkthrough file reads like a comment block, it's too shallow.
- Update the source file list and reference document names when reusing on a new project.
