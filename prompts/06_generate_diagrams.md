# Prompt: Generate Architecture Diagrams
**Phase:** Post-implementation documentation
**When to use:** After the feedback loop is complete and all design documents are reconciled. Run after `05_feedback_loop.md` and before or alongside `02_readme_generation.md`.
**Produces:** A Mermaid architecture diagram for the README and a Mermaid file flow diagram for the walkthrough directory.

---

## Before Running This Prompt

Fill in the following placeholders before handing this to Claude Code:

| Placeholder | Replace with |
|---|---|
| `{PROJECT_NAME}` | The name of the project (e.g. LokisWand) |
| `{VERSION}` | The current version (e.g. v1) |
| `{SOURCE_FILES}` | The full list of source files in the project, in logical execution order |

---

## Inputs

Read the following documents in full before generating either diagram:

- `{project}_v{version}_interface_contract.md` — authoritative source for component inputs, outputs, and data flow
- `{project}_v{version}_design_decisions.md` — authoritative source for why components are structured the way they are
- `{project}_v{version}_masterplan.md` — authoritative source for component responsibilities and build sequence

Do not invent relationships not present in the source documents. If a relationship is ambiguous, flag it explicitly rather than guessing.

---

## Diagram 1 — Architecture Diagram (for README.md)

### Purpose
A high-level system diagram showing the major components of {PROJECT_NAME}, their relationships, and the direction of data flow. This diagram appears in the README and is the first technical artifact a recruiter or hiring manager sees. It must be readable by both technical and nontechnical audiences.

### Requirements
- Use Mermaid `flowchart TD` syntax
- Show every major component as a node
- Show data flow between components as directed edges
- Label each edge with what is being passed (e.g. "job description text", "fit signals", "Airtable record")
- Group logically related components using Mermaid subgraphs where appropriate (e.g. "User Input", "Assessment Pipeline", "Output Layer")
- Do not show internal implementation details — nodes represent components, not functions or classes
- Do not show infrastructure (n8n, Airtable, Slack) as architecture components unless they are the component — show them as output destinations if relevant
- Keep the diagram shallow enough that a nontechnical reader can trace the flow from input to output in one read

### Output
Embed the diagram directly in `README.md` using a fenced Mermaid code block:

```mermaid
flowchart TD
    ...
```

Place it in the README after the "What {PROJECT_NAME} does" section and before the "Why the design matters" section, so it anchors the plain English description visually before the technical depth begins.

---

## Diagram 2 — File Flow Diagram (for walkthrough/README.md)

### Purpose
A diagram showing the order in which source files are described in the walkthrough directory and how they relate to each other in terms of dependency and execution flow. This diagram appears at the top of `walkthrough/README.md` and orients a developer reading the walkthrough for the first time.

### Requirements
- Use Mermaid `flowchart TD` syntax
- Show every source file covered by the walkthrough as a node
- Use the actual filename as the node label (e.g. `models/exception.py`)
- Show dependency relationships as directed edges — an edge from A to B means A is imported by or feeds into B
- Show execution order where dependency alone does not determine sequence
- Do not show external libraries or framework internals as nodes — only project source files
- If a file is a standalone utility with no dependency relationships, show it as an isolated node with a brief label annotation
- The diagram should match the order in which walkthrough files are listed in `walkthrough/README.md`

### Output
Embed the diagram at the top of `walkthrough/README.md` using a fenced Mermaid code block, before the file index:

```mermaid
flowchart TD
    ...
```

Follow the diagram immediately with a one-sentence explanation of how to read it — what the nodes represent and what the edges mean — for developers unfamiliar with dependency flow diagrams.

---

## Constraints

- Both diagrams must render correctly in standard Mermaid renderers (GitHub, VS Code Mermaid preview)
- Do not use Mermaid features that are not widely supported — avoid subgraph nesting beyond two levels, avoid custom themes, avoid HTML labels
- Node labels must be concise — three to five words maximum per node
- Edge labels must be concise — two to four words maximum per edge
- If a component relationship is too complex to represent cleanly in one diagram, split it into two diagrams rather than producing one unreadable diagram
- Every node in Diagram 1 must correspond to a component described in the interface contract
- Every node in Diagram 2 must correspond to a source file in `{SOURCE_FILES}`

---

## Notes

- Run this after `05_feedback_loop.md` so the source documents reflect the actual implementation, not the plan
- The architecture diagram is the first technical artifact most readers will see — prioritize clarity over completeness
- The file flow diagram is for developers, not recruiters — prioritize accuracy over simplicity
- If the project has no walkthrough directory yet, run `03_generate_walkthrough.md` first
