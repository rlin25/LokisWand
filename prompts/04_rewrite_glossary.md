# Prompt: Rewrite Glossary

**Purpose:** Rewrite the design-phase glossary as a post-implementation reference organized for two audiences: recruiters reading the repo, and the developer returning to the codebase.
**When to use:** After all subplans are implemented and the codebase is complete.
**Reusability:** Adapt the three-part structure, domain terms, and source file list for each new project.

---

## Before Running This Prompt

Fill in the following placeholders before handing this to Claude Code:

| Placeholder | Replace with |
|---|---|
| `{PROJECT_NAME}` | The name of the project (e.g. LokisWand) |
| `{VERSION}` | The current version (e.g. v1) |
| `{DOMAIN}` | The problem domain (e.g. job search automation, trade exception triage) |
| `{DOMAIN_TERMS}` | Key domain terms a non-technical reader would encounter (e.g. settlement, trade exception, counterparty) |
| `{SYSTEM_CONCEPTS}` | High-level system concepts a non-technical reader needs to understand what the system does |
| `{ARCHITECTURE_PATTERNS}` | The core architectural patterns used (e.g. state machine, execution paths, retrieval pipeline) |
| `{SOURCE_FILES}` | The full list of source files in the project, in the order they should be covered |

---

## Prompt

Rewrite `{project}_v{version}_glossary.md` from scratch. The original was written during the design phase and does not reflect the implemented codebase. The new glossary serves two audiences: recruiters (technical and non-technical) reading the repository, and the developer returning to the codebase after time away.

Organize the glossary into three parts:

---

### Part 1 — Domain and System Overview

Define all {DOMAIN} terms and high-level system concepts. This section should be readable by a non-technical recruiter with no prior knowledge of {DOMAIN}. Assume the reader has already read the README.

Include:
- Domain terms: {DOMAIN_TERMS}
- High-level system concepts: {SYSTEM_CONCEPTS}
- Any acronyms used in the codebase or documentation

Do not reference specific files, functions, or implementation details in this section.

---

### Part 2 — Architecture and Execution

Define the system's logic layer. This section bridges the domain overview and the implementation reference — a recruiter reading the architecture diagram or the README's flow description will encounter these terms before opening any source file.

Include:
- {ARCHITECTURE_PATTERNS} and when each fires
- Every major processing step and what it produces
- Key data structures passed between components
- Scoring, thresholds, or decision logic
- Any special behavioral patterns (e.g. deferral, escalation, fast-exit)

---

### Part 3 — Implementation Reference

File-by-file breakdown of implementation-specific terms. Organize by source file. For terms that span multiple files, define them at their source file and note where else they appear.

Cover these files in order:
{SOURCE_FILES}

For each file, define only the terms that are non-obvious to a developer reading it for the first time. Skip terms that are self-explanatory from the code.

---

## Constraints

- Do not carry over definitions from the original glossary without verifying them against the implementation.
- Do not define terms that are fully explained in the README — reference the README instead.
- Write Part 1 for a non-technical reader. Write Parts 2 and 3 for a technical reader.
- Output the full glossary as a single `{project}_v{version}_glossary.md` file.

Reference documents:
- `{project}_v{version}_masterplan.md`
- `{project}_v{version}_design_decisions.md`
- `{project}_v{version}_interface_contract.md`
