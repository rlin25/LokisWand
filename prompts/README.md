# prompts/

This folder contains the prompts and methodology documents that governed how LokisWand was designed, built, and documented. It exists because in AI-assisted development, the prompts and methodology that produce the code are as significant as the code itself.

In conventional software development, the artifacts that matter are the code and the architecture documents. In AI-assisted development, a third category matters equally: the structured instructions that shaped how an AI system was reasoned about, specified, and built. Those instructions encode the thinking. The code is the output of the thinking — but the prompts are the thinking itself. If the prompts and methodology are discarded, the project leaves no record of how decisions were made, which means it leaves no record of what was decided deliberately versus what emerged by accident.

The prompts in this folder are not prompt engineering curiosities. They are the operational specification for how a developer should work with Claude across a design-first project — from initial domain learning through implementation through post-implementation reconciliation and documentation. Each prompt corresponds to a specific phase of work with a specific method and specific output.

---

## Folder structure

```
prompts/
├── README.md               — this file
├── general/                — reusable methodology prompts applicable to any project
│   ├── 01_design_methodology.md
│   ├── 02_readme_generation.md
│   ├── 03_generate_walkthrough.md
│   ├── 04_rewrite_glossary.md
│   ├── 05_feedback_loop.md
│   └── 06_generate_diagrams.md
└── project/                — LokisWand-specific Claude prompt directives used in the live pipeline
    ├── loki_instance_a_prompt.md
    ├── loki_instance_b_prompt.md
    └── loki_synthesis_prompt.md
```

---

## general/ — Methodology prompts

These prompts define the human-AI collaboration workflow used throughout LokisWand's design and documentation phases. They are reusable: the design methodology they encode applies to any AI-assisted project, not just this one.

| File | Phase | Produced |
|---|---|---|
| `general/01_design_methodology.md` | Design phase (Phases 1–3) | Behavioral instructions and methods governing human-AI collaboration throughout design — Socratic method, rubber duck specification, draft and challenge, and all six behavioral instructions |
| `general/02_readme_generation.md` | Post-implementation (Phase 7+) | README.md, DESIGN.md, INTERFACE_CONTRACT.md, DESIGN_DECISIONS.md, prompts/README.md |
| `general/03_generate_walkthrough.md` | Post-implementation (Phase 7+) | `walkthrough/` directory of annotated design rationale for every source file |
| `general/04_rewrite_glossary.md` | Post-implementation (Phase 7+) | Polished glossary written for two audiences: recruiter and returning developer |
| `general/05_feedback_loop.md` | Phase 7 | Updated design documents reconciled against the implemented codebase |
| `general/06_generate_diagrams.md` | Post-implementation (Phase 7+) | Mermaid architecture diagram for README.md and file flow diagram for walkthrough/README.md |

---

## project/ — Pipeline prompt directives

These are the Claude prompt directives used by the live LokisWand pipeline. They are not methodology prompts — they are operational inputs to the n8n workflow. They are stored here so the full reasoning chain is visible: from the interface contract that specified what each directive must do, to the locked design decisions that constrained how each instance should behave, to the directives themselves.

| File | Role in pipeline |
|---|---|
| `project/loki_instance_a_prompt.md` | Fit signals directive — passed to Instance A in the dual assessment step |
| `project/loki_instance_b_prompt.md` | Risk signals directive — passed to Instance B in the dual assessment step |
| `project/loki_synthesis_prompt.md` | Synthesis directive — passed to the synthesis call that produces the final recommendation |

The content of these directives is specified in full in [INTERFACE_CONTRACT.md](../INTERFACE_CONTRACT.md). The design decisions that constrain their behavior are documented in [DESIGN_DECISIONS.md](../DESIGN_DECISIONS.md) — particularly Decisions 2, 3, and 4.
