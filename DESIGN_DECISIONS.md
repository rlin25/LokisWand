# DESIGN_DECISIONS.md — LokisWand Design Decisions v1

**Status:** Implementation complete. Decisions 1–15 were locked before implementation began. Decisions 16–19 were recorded post-implementation as part of the Phase 7 feedback loop — implementation discoveries that needed to be documented before v2 design begins. Next phase: v2 design.

Every decision in this document was made deliberately: either before implementation began (Decisions 1–15, using the Socratic method) or surfaced during implementation and recorded during Phase 7 reconciliation (Decisions 16–19). Claude posed structured questions about each design area, presented two options with a preference and rationale for each, and challenged weak reasoning. No decision was locked until the reasoning was explicit and all alternatives were evaluated and documented with rejection rationale. This document is not a record of what was built — it is a record of what was decided and why. The rejected alternatives are as important as the chosen ones: they show that the decisions were deliberate, not accidental. For the inputs and outputs that each decision governs, see [INTERFACE_CONTRACT.md](INTERFACE_CONTRACT.md).

---

All decisions in this document are locked. Claude Code must not re-litigate any decision marked LOCKED during implementation. If an implementation constraint makes a locked decision unachievable, surface the conflict explicitly and halt rather than substituting an alternative silently.

---

## Decision 1 — Input method: pasted job description over URL fetch
**Status: LOCKED**

**Decision:** The user pastes the full job description text into the n8n form. No URL fetching is performed.

**Rationale:** Job listing pages are unreliable targets for automated fetching. JavaScript rendering, robots.txt restrictions, authentication walls, and ATS-specific markup make consistent extraction across job boards infeasible. The user has already read the listing to decide it is worth submitting — copying and pasting is a trivial additional step at that point. Pasted text arrives clean and requires no parsing heuristics.

**Rejected alternatives:**
- URL fetch via n8n HTTP request node — unreliable across job boards, fails silently on JavaScript-rendered pages
- Browser extension to extract listing content — adds external tooling dependency outside the defined stack

---

## Decision 2 — Dual instance persona framing: fit signals vs. risk signals
**Status: LOCKED**

**Decision:** Instance A is prompted to identify the strongest fit signals. Instance B is prompted to identify the strongest risk signals. The two instances are complementary, not adversarial.

**Rationale:** An adversarial optimist/skeptic framing produces manufactured disagreement — instances invent conflict on points where no genuine ambiguity exists, which makes the synthesis misleading rather than clarifying. Complementary framing gives each instance a constrained, concrete task that naturally produces variance without manufacturing conflict. The synthesis call then has cleaner inputs to resolve.

**Rejected alternatives:**
- Optimist vs. skeptic persona framing — produces artificial disagreement, degrades synthesis quality
- Axis differentiation (technical fit vs. cultural fit) — complementary rather than conflicting by design, produces little genuine variance for synthesis to resolve
- Single instance assessment — eliminates the multi-perspective evaluation that justifies the three-call pattern

---

## Decision 3 — Instance silence: deferral over forced output
**Status: LOCKED**

**Decision:** Both Instance A and Instance B are explicitly permitted to defer to the synthesis agent if no genuine signals exist. A deferral is treated as meaningful signal by the synthesis prompt, not as an error.

**Rationale:** Forcing instances to produce a fixed number of signals regardless of genuine evidence produces manufactured assessments. A strong candidate evaluated against a perfectly matched role will have no genuine risk signals — Instance B should say so rather than inventing three. The synthesis agent is better positioned to interpret a deferral as positive or negative evidence than to receive fabricated signals and treat them as real.

**Rejected alternatives:**
- Mandatory three-signal output — produces manufactured disagreement, degrades assessment reliability
- Error handling on empty output — treats absence of signals as a system failure rather than a meaningful result

---

## Decision 4 — Synthesis behavior: collapse false conflicts, resolve genuine ones
**Status: LOCKED**

**Decision:** The synthesis prompt explicitly instructs the third instance to distinguish genuine conflicts from artifactual ones, collapse the latter into confident statements, incorporate deferrals as meaningful evidence, and conclude with a fixed enum recommendation followed by a one paragraph justification.

**Rationale:** A synthesis call that simply averages two assessments adds no analytical value. The synthesis must do real work: identifying where the two instances actually disagree versus where they appear to disagree but are saying the same thing. The fixed enum recommendation (Strong Fit, Moderate Fit, Weak Fit, Do Not Apply) forces a concrete output that is actionable rather than hedged.

**Rejected alternatives:**
- Averaging or summarizing both assessments — adds no analytical value over reading both assessments directly
- Freeform synthesis without structured output — produces hedged, non-actionable recommendations
- Omitting the synthesis step — leaves the user to reconcile two assessments manually, which is the problem the tool exists to solve

---

## Decision 5 — Profile document format: structured markdown with Review Required section
**Status: LOCKED**

**Decision:** The profile document is a markdown file following a defined schema. All inferred fields that require user verification are surfaced in a clearly labeled Review Required section at the top of the document. The remainder of the document contains fields Claude is confident about.

**Rationale:** Markdown is the optimal format for LLM input fidelity — it is clean, universally parseable, and free of rendering dependencies. The Review Required section solves the visual scanning problem during one-time setup without requiring color coding (not universally supported in markdown) or emoji flags (unprofessional). Grouping inferred fields structurally is as scannable as color coding and more portable.

**Rejected alternatives:**
- HTML file with color-coded inferred fields — introduces rendering dependency, reduces LLM input fidelity
- Emoji flags on inferred fields — unprofessional, inconsistent with the standard of the surrounding tooling
- Raw resume passed directly to LLM — uncontrolled input produces variable assessment quality; the structured profile document exists specifically to eliminate this variance

---

## Decision 6 — Profile document generation: Claude drafts, user reviews
**Status: LOCKED**

**Decision:** The profile document is generated by Claude from the master resume via a one-time conversion prompt. Claude drafts all fields including inferred ones. The user reviews only the Review Required section before saving. Hard constraints, goals, and known gaps always appear in Review Required regardless of Claude's confidence.

**Rationale:** Manual schema entry is prohibitively tedious and introduces transcription errors. Full automation without review risks propagating inference errors into every downstream assessment. Claude-drafted with targeted human review captures the benefits of both: minimal user effort on the recurring path, accuracy on the fields that matter most.

**Rejected alternatives:**
- Fully manual profile entry — too tedious, inconsistent with the minimum viable user input design principle
- Fully automated with no review — inference errors on hard constraints corrupt every downstream assessment silently
- Separate automated conversion workflow in n8n — adds complexity for an infrequent operation; the conversion prompt is sufficient for v1

---

## Decision 7 — Skill proficiency: presence only, no proficiency tags
**Status: LOCKED**

**Decision:** Skills are listed as flat lists within categories. No proficiency level is attached to individual skills.

**Rationale:** Proficiency tags are tedious to populate accurately and introduce a maintenance burden that degrades over time. Inaccurate proficiency tags produce confidently wrong assessments — worse than no tags. Claude infers relative skill strength contextually from the experience and highlights sections, which carry richer signal than a self-reported proficiency label.

**Rejected alternatives:**
- Explicit proficiency levels per skill — maintenance burden, risk of inaccuracy, false precision in assessment outputs

---

## Decision 8 — Experience entry structure: dual field (achievements + technical details)
**Status: LOCKED**

**Decision:** Each experience entry carries two fields: achievement-focused bullet points and a separate technical detail field listing the actual technologies, tools, and methods used in that role.

**Rationale:** Achievement bullets are written for human readers and emphasize impact over specifics. "Reduced processing time by 40%" provides no signal about what was built or what stack was used. The technical detail field gives Instance A concrete evidence for fit signals and Instance B concrete evidence for gap signals without requiring inference from vague achievement language.

**Rejected alternatives:**
- Achievement bullets only — forces Claude to infer technical depth from impact statements, introduces variance at a high-signal input point
- Technical details only — loses the achievement context that explains why the work mattered

---

## Decision 9 — Highlights tagging: explicit relevance tags per highlight
**Status: LOCKED**

**Decision:** Each highlight and accolade carries explicit tags indicating what role types, skills, or competencies it is most relevant to.

**Rationale:** Highlights are the highest-signal assets in the profile document. Without tags, Claude must infer relevance from descriptions — inference varies between calls, introducing variance at exactly the point where consistency matters most. Tags give Instance A a direct path to fit signals and give Instance B clarity on which highlights do not transfer to a specific role.

**Rejected alternatives:**
- Flat list with descriptions only — forces inference on highest-signal assets, produces inconsistent relevance attribution across calls

---

## Decision 10 — Tracker destination: Airtable
**Status: LOCKED**

**Decision:** All pipeline outputs are logged to Airtable.

**Rationale:** Airtable's native kanban view satisfies the pipeline visualization requirement out of the box. Its date fields and filter logic make follow-up trigger queries straightforward from n8n. It is a business tooling standard directly relevant to the target role at Alloy, adding portfolio signal beyond the technical implementation.

**Rejected alternatives:**
- Google Sheets — satisfies logging requirement but requires workarounds for kanban visualization and follow-up trigger logic; formula-based status columns are brittle

---

## Decision 11 — Follow-up nudge delivery: Slack
**Status: LOCKED**

**Decision:** Follow-up nudges are delivered as messages to a personal Slack workspace.

**Rationale:** Email notifications are high-volume and low-salience — job search follow-up reminders will be lost in inbox noise. A dedicated Slack workspace keeps job search notifications visually separated and immediately actionable. Slack integration is directly relevant to Alloy's internal tooling context and adds portfolio signal.

**Rejected alternatives:**
- Email via Gmail or SMTP — high noise environment, low salience for time-sensitive follow-up nudges

---

## Decision 12 — Follow-up nudge timing: range query over exact date match
**Status: LOCKED**

**Decision:** The follow-up nudge workflow queries for records where date submitted is 7 or more days ago (capped at 21 days) and the corresponding follow-up checkbox is false, rather than matching exactly 7 or 14 days.

**Rationale:** An exact date match permanently misses nudges if the workflow fails to run on the target day. A range query catches missed runs on the next successful execution without sending duplicate nudges, since the checkbox is set to true immediately after each nudge is sent.

**Rejected alternatives:**
- Exact date match — permanently drops nudges on workflow failure days with no recovery path

---

## Decision 13 — Duplicate detection: check before write
**Status: LOCKED**

**Decision:** Before writing a new Airtable record, n8n queries for an existing record matching the same company name and role title. If a match exists, the pipeline halts and notifies the user via Slack rather than creating a duplicate record.

**Rationale:** Nothing in the input flow prevents a user from submitting the same job description twice. Silent duplicate creation corrupts the tracker and produces redundant follow-up nudges. A pre-write check is a small implementation addition that prevents a meaningful data quality problem.

**Rejected alternatives:**
- No duplicate detection — allows silent tracker corruption on repeat submissions

---

## Decision 14 — Metadata extraction: Claude over regex parsing
**Status: LOCKED**

**Decision:** Company name, role title, location, and employment type are extracted from the pasted job description by Claude rather than by regex or string parsing heuristics.

**Rationale:** Job description formatting varies significantly across companies and job boards. Regex patterns that work for one format fail silently on another. Claude handles format variance gracefully and returns null explicitly when a field cannot be determined, enabling clean failure handling downstream.

**Rejected alternatives:**
- Regex or string parsing — brittle across format variance, fails silently rather than returning explicit null

---

## Decision 15 — Profile document versioning: explicit version field logged per assessment
**Status: LOCKED**

**Decision:** The profile document schema includes a version field (e.g. `v1.0`, `v1.1`). This version field is logged as a field in every Airtable record at the time of assessment.

**Rationale:** Without versioning, there is no way to determine whether a given Airtable assessment reflects the current profile document or a stale one. A candidate assessed as Moderate Fit against an older profile may be Strong Fit against an updated one. Logging the profile version per record makes assessments auditable and flags which records may warrant reassessment after a profile update.

**Rejected alternatives:**
- No versioning — assessments become uninterpretable after any profile document update, tracker reliability degrades silently over time

---

## Known Constraints
- Three Claude API calls per submission — costs and rate limits apply at high volume; acceptable for personal use at v1 scale
- Profile document must be manually re-updated in the Load Profile Document code node if the master resume changes (see Decision 17)
- Airtable free tier caps at 1,000 records — sufficient for personal job search use
- n8n self-hosted or cloud instance required; cloud instance recommended for follow-up nudge scheduling reliability

---

## Implementation-Phase Decisions (Phase 7 additions)

The following decisions were not anticipated during the design phase. They were made during implementation and recorded during the Phase 7 feedback loop.

---

## Decision 16 — Airtable checkbox updates: HTTP Request PATCH over native Airtable node
**Status: LOCKED**

**Decision:** The follow-up nudge workflow uses HTTP Request PATCH nodes to update the Follow-up sent day 7 and Follow-up sent day 14 checkbox fields, rather than the native n8n Airtable node.

**Rationale:** The n8n Airtable node v2 routes all update operations through batchUpdate internally. When configured with "Columns to match on: id," the node treats "id" as a user-created column name to search by — not as the internal Airtable record ID. Since no user-created column named "id" exists in the Applications table, the search returns nothing and the batchUpdate request is malformed, producing a 422 INVALID_RECORDS error. The Airtable REST API accepts the properly formatted `{ records: [{ id: <record_id>, fields: { ... } }] }` PATCH payload directly with no ambiguity. HTTP Request bypasses the n8n node's internal routing.

**Rejected alternatives:**
- Native n8n Airtable node update operation — produces 422 INVALID_RECORDS due to batchUpdate bug described above

---

## Decision 17 — Profile document delivery: embedded Code node string over static file reference
**Status: LOCKED**

**Decision:** The candidate profile text is embedded as a hardcoded string in a Code node (Load Profile Document) in the core pipeline workflow, rather than uploaded to n8n as a static file reference as specified in the masterplan.

**Rationale:** n8n's static file reference configuration required non-obvious setup during implementation (file storage configuration, path resolution behavior). Embedding the profile text directly in a Code node is simpler and eliminates a file management dependency. The tradeoff is that updating the profile requires editing the workflow directly, but profile updates are infrequent at v1 scale. The v2 remediation path is to externalize the profile to an n8n variable or properly configured file reference.

**Rejected alternatives:**
- n8n static file reference — configuration complexity discovered during implementation; risk of runtime path resolution errors

---

## Decision 18 — Slack message construction: JavaScript Code node over n8n expressions
**Status: LOCKED**

**Decision:** Slack message bodies in the follow-up nudge workflow are constructed in JavaScript Code nodes (Prepare Day 7 Message, Prepare Day 14 Message) rather than inline n8n expression syntax in the HTTP Request node body field.

**Rationale:** Constructing multi-field formatted strings in n8n expression syntax is error-prone and produces hard-to-debug failures when fields are undefined or null. JavaScript string templates in a Code node are readable, debuggable, and allow iterating over multiple records in a single pass. The Code node also provides a natural place to add the slackBody field to each item before the HTTP Request node consumes it.

**Rejected alternatives:**
- Inline n8n expressions in HTTP Request body — produces opaque "[undefined]" errors on missing fields, harder to debug and maintain for multi-field interpolation

---

## Decision 19 — n8n expression syntax for HTTP Request JSON body fields: `{{ }}` over `={{ }}`
**Status: LOCKED**

**Decision:** In n8n HTTP Request nodes, JSON body fields set to Expression mode use `{{ expression }}` syntax without a leading `=`. All other n8n parameter fields that toggle between Fixed and Expression mode use `={{ expression }}` with the leading `=`.

**Rationale:** The `=` prefix is a toggle-mode field syntax. HTTP Request node JSON body fields in Expression mode are not toggle-mode fields — they are always expression fields. The leading `=` in this context causes the expression to fail silently, producing the literal string "=undefined" or similar. This distinction is underdocumented in n8n. The pattern was confirmed through testing: `{{ $json.slackBody }}` works; `={{ $json.slackBody }}` fails.

**Rejected alternatives:**
- `={{ }}` syntax in HTTP Request JSON body fields — causes expression failure; produces literal `=[undefined]` output
