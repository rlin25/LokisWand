# Walkthrough: loki_instance_a_prompt.md

**Source file:** `prompts/project/loki_instance_a_prompt.md`
**Role:** Fit signals directive — the constrained task description for the first parallel Claude assessment call

---

## Purpose

Instance A's prompt exists as a separate file because it is a locked interface contract artifact, not a convenience string. The prompt directive specifies exactly what Instance A is allowed to do, what it must not do, and what it must return when it cannot do its job. These constraints are design decisions (specifically Decisions 2 and 3) expressed as Claude instructions. The prompt file makes those decisions visible and auditable rather than buried in an n8n node body.

---

## Relationships

**Consumed by:** `workflows/loki_core_pipeline.json` — embedded in the Instance A HTTP Request node body, alongside the candidate profile text and job description text

**Governed by:** Design Decisions 2 (complementary framing) and 3 (deferral over forced output)

**Does not reference:** Instance B output, synthesis prompt, or any Airtable schema — Instance A has no knowledge of what happens after it returns its output

---

## Decision rationale

### "A separate agent is handling risk signals. You do not need to cover both sides."
This instruction is Decision 2 applied directly. Without it, Claude would attempt a balanced assessment — identifying both fit and risk signals — which defeats the purpose of having two separate instances. The explicit statement of the division of labor constrains Instance A to its assigned task. Claude is not told why a separate agent exists (that is n8n orchestration context), only that it does and that Instance A is not responsible for the other side.

### Specificity requirement: "A fit signal is only genuine if it connects something concrete in the profile to something concrete in the listing"
This is the anti-hallucination constraint for fit signals. Without it, Instance A produces general background statements: "The candidate has experience in AI engineering, which is relevant to this role." That statement tells the synthesis agent nothing it could not infer from the job description title alone. The specificity requirement forces Instance A to cite exact profile elements (a specific project, a specific skill, a specific highlight tag) against exact listing requirements. Vague signals are not signals.

### Deferral instruction is verbatim and frozen
The deferral statement that Instance A returns when no genuine fit signals exist is specified exactly:

> No genuine fit signals identified. Deferring to synthesis. [One sentence explaining why...]

The synthesis prompt's Step 3 (Deferral Handling) interprets this exact format as evidence of weak fit. The synthesis prompt was written to recognize this language. If Instance A's deferral statement is paraphrased or reformatted, the synthesis agent might not recognize it as a deferral and might treat it as a signal instead — corrupting the reasoning chain.

The instruction "do not attempt to compensate for it by finding marginal signals" exists because Claude tends to hedge: if instructed that a deferral is acceptable but that it has negative downstream consequences, Claude might manufacture weak signals to avoid triggering those consequences. The explicit instruction removes that incentive.

---

## What it does not do

- **Does not evaluate risk signals** — by design (Decision 2). Any risk observation Instance A notices is not its output to produce.
- **Does not produce a recommendation** — that is the synthesis call's responsibility
- **Does not receive or reference Instance B's output** — the parallel calls are truly independent; each sees only the shared context (profile + job description), not the other's output
- **Does not know what happens to its output** — Instance A produces a numbered list and returns it. Synthesis, Airtable logging, and the parse step are downstream concerns invisible to this prompt.

---

## v2 touch points

- **Deferral language is frozen**: Any change to how synthesis interprets deferrals requires updating both instance prompts and the synthesis prompt atomically. The three prompts form a coupled system at the deferral boundary.
- **Specificity requirement**: v2 might strengthen this further by instructing Instance A to quote the specific profile field name it is citing (e.g., "from the LangGraph project's technical details field..."), which would make signals easier to verify and trace.
