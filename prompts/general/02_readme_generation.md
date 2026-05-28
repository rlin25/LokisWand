# Prompt: README and Documentation Generation
**Phase:** Post-implementation documentation
**Produced by:** Rubber duck specification pass → Socratic pass → locked decisions
**Produces:** README.md, DESIGN.md, INTERFACE_CONTRACT.md, DESIGN_DECISIONS.md, prompts/README.md

---

## Before Running This Prompt

Fill in the following placeholders before handing this to Claude Code:

| Placeholder | Replace with |
|---|---|
| `{PROJECT_NAME}` | The name of the project (e.g. LokisWand) |
| `{PROJECT_DESCRIPTION}` | One sentence describing what the project does |
| `{TARGET_AUDIENCE}` | Who the documentation is for (e.g. technical and nontechnical recruiters for a junior AI engineer role) |
| `{TARGET_ROLE}` | The specific role or role type being targeted |
| `{KEY_SKILLS}` | The skills the target role emphasizes, drawn from the job listing |
| `{VERSION}` | The current version (e.g. v1) |
| `{DESIGN_ADVANCE}` | What specifically this version advances over the previous one, or what makes this version significant |

---

## Inputs

The following project files must be available and cross-referenced when generating all documents:

- `{project}_v{version}_interface_contract.md`
- `{project}_v{version}_design_decisions.md`
- `{project}_v{version}_masterplan.md`
- `{project}_v{version}_glossary.md`
- `{project}_v{version}_setup_notes.md`
- `prompts/01_design_methodology.md`

Do not invent details not present in the source documents. Flag any gap where the source documents are insufficient to complete a section.

---

## Context and Audience

You are generating documentation artifacts for a portfolio project called {PROJECT_NAME}, targeting {TARGET_AUDIENCE} for {TARGET_ROLE}. The role emphasizes {KEY_SKILLS}.

The core argument this documentation must make is this: **in the age of AI, code is disposable — design principles are not.** This project was built design-first. The interface contract and locked design decisions all preceded the code. That methodology is the primary signal this documentation should convey, above the technical stack.

The development methodology involved structured human-AI collaboration throughout the design phase — Claude was used as a Socratic design partner to stress-test decisions, surface gaps, and formalize specifications. All architectural judgment and design choices belonged to the developer. This should be stated transparently and framed as a demonstration of AI fluency, not apologized for.

{DESIGN_ADVANCE}

---

## Document 1 — README.md

**Audience:** both technical and nontechnical readers. Nontechnical readers should get full value from the first third and be able to stop. Technical readers should be pulled deeper.

**Structure:**

### 1. Opening statement
Direct and opinionated, not a generic project description. Lead with the design-first philosophy and what it means for how this project was built. This is the developer's voice and point of view, not a neutral description. Do not open with "{PROJECT_NAME} is a [technology] that..." or any equivalent generic opener.

### 2. What {PROJECT_NAME} does
Plain English, no jargon. What problem it solves, what the user input is, what the system produces. Two to three short paragraphs maximum.

### 3. Why the design matters
Briefly surface the key architectural decisions as deliberate choices, not implementation details. Connect each to the problem domain context. Reference the specific design advance this version represents.

### 4. How it was built
Describe the development methodology: domain learning, Socratic design, interface contract, masterplan, implementation, feedback loop. Frame the AI collaboration explicitly and offensively — Claude as Socratic partner, all design judgment belonging to the developer. Note that the design artifacts are version-stable while the code is intentionally disposable.

### 5. Companion documents
Link all companion documents with one-line descriptions framing who should read each and why:

- **DESIGN.md** — architecture and system design. Start here if you want to understand how the system thinks.
- **INTERFACE_CONTRACT.md** — every component's inputs, outputs, and data types. The stable contract the code was built against.
- **DESIGN_DECISIONS.md** — every major decision with its reasoning and rejected alternatives documented. The clearest signal of how design tradeoffs were evaluated.

### 6. Tech stack
Clean table, no commentary needed.

### 7. Setup and demo
Instructions to get the system running and the key interactions that tell the full demo story.

**Tone:** direct, confident, opinionated. This README opens with a position.

---

## Document 2 — DESIGN.md

**Audience:** technical readers who want architectural depth.

**Structure:**

### 1. System overview
Use the plain English paragraph from `{project}_v{version}_masterplan.md` verbatim as the opening.

### 2. The components
Each component described with its responsibility, its boundaries, and what it explicitly does not do. Draw from `{project}_v{version}_interface_contract.md`.

### 3. Key architectural patterns
Describe the core architectural patterns the system uses — state machines, execution paths, retrieval patterns, orchestration patterns — with the reasoning for why each exists as a distinct pattern rather than inline logic.

### 4. Key design decisions
Surface the most interview-worthy decisions from `{project}_v{version}_design_decisions.md` with their reasoning and rejected alternatives. Prioritize decisions that:
- Reflect deliberate tradeoffs between competing approaches
- Are non-obvious — where a reasonable developer might have made a different choice
- Directly map to skills emphasized in the target role

### 5. What this version deliberately excludes
The explicit out-of-scope list from the design decisions document, framed as evidence of scope discipline rather than incompleteness.

---

## Document 3 — INTERFACE_CONTRACT.md

Use `{project}_v{version}_interface_contract.md` as the source. Reproduce it with the following additions only:

- A one-paragraph introduction at the top explaining what an interface contract is, why it was written before the code, and how to use it as a reader.
- A one-line note at the top of each component section indicating which design decisions govern it.

Do not alter any existing specifications. The contract is locked.

---

## Document 4 — DESIGN_DECISIONS.md

Use `{project}_v{version}_design_decisions.md` as the source. Reproduce it with the following addition only:

- A one-paragraph introduction explaining the Socratic design methodology, why rejected alternatives are documented, and how this document relates to the interface contract.

Do not rewrite or alter any existing decisions. They are locked.

---

## Document 5 — prompts/README.md

Generate a one-page index for the `prompts/` folder explaining:
- What this folder is and why it exists
- The core argument: in AI-assisted development, the prompts and methodology that produce the code are as significant as the code itself
- How to read the folder — each prompt file listed with a one-line description of what phase it belongs to and what it produced

Frame this around the design-first philosophy established throughout the project. The prompts folder is evidence that the thinking happened before the code.

**Prompt file index to document:**

| File | Phase | Produced |
|---|---|---|
| `01_design_methodology.md` | Design phase (Phases 1–3) | Behavioral instructions and methods governing human-AI collaboration throughout design |
| `02_readme_generation.md` | Post-implementation (Phase 7+) | README.md, DESIGN.md, INTERFACE_CONTRACT.md, DESIGN_DECISIONS.md, prompts/README.md |
| `03_generate_walkthrough.md` | Post-implementation (Phase 7+) | `walkthrough/` directory of annotated design rationale for every source file |
| `04_rewrite_glossary.md` | Post-implementation (Phase 7+) | Polished glossary written for two audiences: recruiter and returning developer |
| `05_feedback_loop.md` | Phase 7 | Updated design documents reconciled against the implemented codebase |

---

## Output Checklist

Before finishing, verify:

- [ ] README opens with a position statement, not a system description
- [ ] README serves nontechnical readers in the first third without requiring technical knowledge
- [ ] AI collaboration is framed transparently and offensively in the README
- [ ] README surfaces the specific design advance this version represents
- [ ] DESIGN.md key decisions are drawn from this version's design decisions document
- [ ] INTERFACE_CONTRACT.md specifications are unaltered
- [ ] DESIGN_DECISIONS.md decisions are unaltered
- [ ] All companion documents are linked from the README with one-line descriptions
- [ ] prompts/README.md indexes all five prompt files with phase and output noted
- [ ] No details invented that are not present in the source documents
- [ ] Any gaps where source documents are insufficient are flagged explicitly
