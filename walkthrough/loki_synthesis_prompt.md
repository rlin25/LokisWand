# Walkthrough: loki_synthesis_prompt.md

**Source file:** `prompts/project/loki_synthesis_prompt.md`
**Role:** Synthesis directive — the prompt for the third Claude call that produces the final recommendation

---

## Purpose

The synthesis prompt is the most structurally complex of the three prompt directives because it has the most analytical work to do. It receives two assessment outputs that may conflict, agree, partially overlap, or include deferral statements. It must process all of these cases explicitly and conclude with a specific, actionable recommendation. The prompt structure is not style — every instruction corresponds to a failure mode that would occur if that instruction were absent.

---

## Relationships

**Consumed by:** `workflows/loki_core_pipeline.json` — embedded in the Run Synthesis HTTP Request node body, alongside Instance A's output and Instance B's output

**Governed by:** Design Decisions 3 (deferral interpretation) and 4 (synthesis as analytical work, fixed enum recommendation)

**Depends on:** Instance A output format (specifically the deferral language); Instance B output format (specifically the deferral language and the in-progress/unaddressed labels). The synthesis prompt was written with these formats in mind.

**Produces:** A structured four-section output parsed by the Parse Synthesis Output code node in the workflow

---

## Decision rationale

### Mandatory four-step structure with explicit completeness requirements
The synthesis prompt defines four steps: resolve genuine conflicts, collapse false conflicts, incorporate deferrals, and produce a recommendation. Each step includes the instruction "If no X exist, state that explicitly." This prevents Claude from silently skipping steps. Without the explicit completion requirement, Claude sometimes omits a step when there is nothing to report — "no genuine conflicts" becomes a missing section, which breaks the Parse Synthesis Output regex.

The four steps correspond to Decision 4: synthesis must do real work, not summarize. Each step has a specific analytical task that cannot be collapsed into the others. Steps 1 and 2 are about the relationship between the two assessment outputs. Step 3 is about the meaning of absent outputs (deferrals). Step 4 is the conclusion. Conflating them would produce a hybrid that addresses none cleanly.

### "You are not summarizing. You are doing analytical work."
This is the most load-bearing instruction in the prompt. Without it, Claude's default behavior is to produce a synthesis that is essentially a labeled summary of both assessments: "Instance A identified these fit signals. Instance B identified these risk signals. Overall, the candidate appears to be a moderate fit." That output adds no value — a user can read the assessments directly. The instruction reframes the synthesis agent's role: it is resolving, collapsing, and concluding, not recapping.

### Fixed enum: no variations, no qualifications appended
The recommendation must be exactly one of: Strong Fit, Moderate Fit, Weak Fit, Do Not Apply. The prompt explicitly prohibits "variations" and "qualifications appended to the label." Without this constraint, Claude produces outputs like "Moderate Fit (with caveats)" or "Moderate-to-Strong Fit" — which cannot be parsed by the workflow's regex and cannot be stored in the Airtable single select field. The constraint is necessary for the downstream parse step to function.

The choice of four enum values (not three, not five) is Decision 4: the enum forces a concrete, actionable output. A three-value enum (Fit, Neutral, No Fit) loses precision. A five-value enum is harder to act on. Four values map cleanly to the actual decision space: apply with confidence, apply with awareness of gaps, apply with low expectations, don't apply.

### Deferral interpretation is explicit and directional
The synthesis prompt states the meaning of each deferral type:
- Deferral from fit agent (Instance A) = weak fit evidence
- Deferral from risk agent (Instance B) = strong fit evidence

This is Decision 3 mechanized into the prompt. The asymmetry is intentional: the same behavior (deferral) means different things depending on which agent produced it, and the synthesis agent must know this to reason correctly. Without this explicit table, Claude either treats all deferrals the same or interprets them contextually — which introduces variance in exactly the place where determinism matters most.

### Output format with four labeled sections
The four output sections (Conflict Resolution, Collapsed Signals, Deferral Handling, Recommendation) are the targets of the Parse Synthesis Output regex in the workflow. The section labels in the prompt and the parse patterns are tightly coupled. Any change to section labels here requires an atomic update to the parse node.

---

## What it does not do

- **Does not re-evaluate the profile or job description** — synthesis receives only the two assessment outputs as inputs, not the raw source material. This is by design: the instance agents have already extracted the relevant signals; synthesis works with those extracted signals, not the originals.
- **Does not handle cases where both assessments are missing** — if both Instance A and Instance B fail (not deferral — actual API failure), the workflow routes to an error branch before synthesis runs. The synthesis prompt does not need to handle that case.
- **Does not justify the recommendation enum value** — it chooses one and justifies it in prose. The enum value is not explained or expanded within the label itself.

---

## v2 touch points

- **Parse coupling is the highest-risk dependency**: The section labels (Conflict Resolution, Collapsed Signals, Deferral Handling, Recommendation) are hardcoded in the Parse Synthesis Output code node's regex. Any rename of a section here breaks the parse. Both must be updated atomically.
- **Recommendation enum coupling**: The four enum values must match the Airtable single select options exactly. A rename in either location requires an atomic update to both.
- **Deferral coupling**: The deferral language that Instance A and Instance B produce is the input that triggers Step 3 reasoning. If the deferral phrasing changes in either instance prompt, the synthesis prompt may need to be updated to recognize the new format.
- **No citation back to source**: v2 might instruct the synthesis agent to cite specific signals from the assessment outputs when justifying the recommendation, making the justification traceable to specific evidence rather than a prose conclusion.
