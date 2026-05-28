# Walkthrough: loki_instance_b_prompt.md

**Source file:** `prompts/project/loki_instance_b_prompt.md`
**Role:** Risk signals directive — the constrained task description for the second parallel Claude assessment call

---

## Purpose

Instance B's prompt mirrors Instance A's structure but adds one constraint that Instance A does not have: the in-progress vs. unaddressed gap distinction. This distinction is not a stylistic choice — it is what makes Instance B's output meaningful rather than misleading. A risk signal that does not distinguish between a gap the candidate is closing and a gap they have never addressed produces the same output for two very different situations.

---

## Relationships

**Consumed by:** `workflows/loki_core_pipeline.json` — embedded in the Instance B HTTP Request node body, alongside the candidate profile text and job description text

**Governed by:** Design Decisions 2 (complementary framing) and 3 (deferral over forced output); the gap distinction draws from the Known Gaps section of the profile document (Decision 5)

**Does not reference:** Instance A output — the two assessment calls are fully independent

---

## Decision rationale

### In-progress vs. unaddressed gap distinction, required on every signal
This requirement directly consumes the Known Gaps section of the candidate profile. Instance B is instructed to check the profile for whether a gap is acknowledged and being actively closed before labeling it "unaddressed." This makes the Known Gaps section load-bearing — its accuracy directly determines whether Instance B's gap labels are correct.

The distinction matters at the synthesis stage: a gap labeled unaddressed is stronger negative evidence than one labeled in-progress. If Instance B does not distinguish, the synthesis agent cannot make this differentiation — and an in-progress gap might be weighted the same as a genuinely unaddressed one, producing a more negative recommendation than the situation warrants.

### "Do not flag items the candidate clearly has — check the profile carefully before identifying a gap"
Claude sometimes flags skills as gaps without checking the profile's skills list carefully. This instruction exists because the error occurs at the reading step, not the reasoning step — Instance B sometimes generates a "missing X" risk signal for a skill that is plainly listed in the profile. The explicit instruction to check before flagging reduces this class of false positives.

### Specificity requirement: "A risk signal is only genuine if it identifies a concrete gap between what the listing requires and what the candidate profile demonstrates"
The same anti-hallucination constraint as Instance A, but framed for risk signals. Without this constraint, Instance B produces general risk statements like "the candidate may lack experience in fast-paced environments" — which is not a gap between specific listing requirements and specific profile evidence. The specificity requirement forces Instance B to cite the exact listing requirement that the candidate's profile does not address.

### Deferral language matches Instance A's structure but carries opposite meaning
The deferral statement for Instance B is:

> No genuine risk signals identified. Deferring to synthesis. [One sentence explaining why...]

The synthesis prompt interprets a deferral from Instance B as evidence of strong fit — the candidate's profile produced no meaningful gaps against this role. This is the opposite interpretation from a deferral from Instance A. Both deferral statements use the same format, but the upstream identity (fit agent vs. risk agent) determines the interpretation. The synthesis prompt handles this correctly because it knows which agent produced which output.

---

## What it does not do

- **Does not evaluate fit signals** — by design (Decision 2)
- **Does not distinguish more than two gap types** — in-progress and unaddressed are sufficient for synthesis to interpret; finer-grained gap taxonomy is not specified in v1
- **Does not receive Instance A's output** — fully independent; both instances see only the shared context
- **Does not recommend** — that is synthesis's role

---

## v2 touch points

- **Known Gaps dependency**: Instance B's gap labeling depends entirely on the Known Gaps section of the candidate profile being accurate and up to date. As gaps close, the profile must be updated — otherwise Instance B will continue labeling closed gaps as "in-progress" indefinitely.
- **Deferral coupling**: Same as Instance A — any change to deferral interpretation at the synthesis level requires updating both instance prompts atomically.
- **False positive rate**: The "check the profile carefully" instruction helps but does not eliminate false positives on skills Claude misreads. v2 might address this with a more structured checking instruction (e.g., "before flagging any skill gap, quote the specific skill from the listing and confirm it is absent from the Skills section of the profile").
