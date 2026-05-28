# Walkthrough: loki_followup_nudge.json

**Source file:** `workflows/loki_followup_nudge.json`
**Role:** Secondary n8n workflow — daily scheduled nudge delivery for follow-up tracking

---

## Purpose

The follow-up nudge workflow runs independently of the core pipeline. It fires daily at 9 AM, reads Airtable for overdue follow-up reminders, sends Slack messages, and updates the corresponding checkboxes. It exists as a separate workflow because its trigger is time-based, not user-initiated — coupling it to the core pipeline would require a different architecture entirely.

---

## Relationships

**Consumes:**
- Airtable records created by the core pipeline (reads Status, Date submitted, Follow-up sent day 7, Follow-up sent day 14, Company, Role title)
- Slack webhook URL (referenced via n8n variable `SLACK_WEBHOOK_URL`)
- Airtable credentials (referenced via n8n credential manager)

**Produces:**
- Slack messages for each qualifying record
- Airtable checkbox updates (Follow-up sent day 7 or Follow-up sent day 14 set to true)

**Does not touch:**
- Any Airtable field other than the two follow-up checkboxes
- The core pipeline workflow or its execution
- Any record not in Applied status

---

## Decision rationale

### Two parallel branches from a single daily trigger
Both the day 7 and day 14 query paths start from the same Daily Schedule trigger node and run in parallel. This is simpler than a single branch that queries twice sequentially — each query is independent of the other's result, and running them in parallel reduces total execution time. A single sequential path would introduce a race condition risk where the day 14 query result could be affected by the day 7 update running first, if any records qualify for both (which is not possible by design, but the parallel structure eliminates even the dependency).

### Range query (7–21 days) rather than exact date match
This is Decision 12 applied directly. An exact date match permanently misses a nudge on any day the workflow fails to execute. The range query (7 or more days ago, capped at 21) recovers missed nudges on the next successful run. The cap at 21 days prevents the query from accumulating very old records indefinitely. The checkbox set to true immediately after each nudge sent prevents the range query from sending the same nudge twice — the range expands the recovery window; the checkbox provides the idempotency guarantee.

### HTTP Request PATCH for Airtable updates instead of native Airtable node
The n8n Airtable v2 node's update operation uses batchUpdate internally. When "Columns to match on: id" is configured, the node treats "id" as a user-created column to search by — not as the internal Airtable record ID. Since no user-created column named "id" exists, the search returns nothing and the batchUpdate request is malformed, producing a 422 INVALID_RECORDS error. The fix is to bypass the native node entirely and use HTTP Request PATCH to call the Airtable REST API directly with the properly formatted `{ records: [{ id: <record_id>, fields: { ... } }] }` payload. This is a workaround for an n8n implementation bug, not a design preference.

### Prepare Day 7/14 Message code nodes
Each branch has a Code node that runs before the Slack HTTP Request. This node loops over the batch of qualifying records from the Airtable query, constructs a formatted Slack message for each, and adds the result as a `slackBody` field on the item. The Slack HTTP Request then uses `$json.slackBody` directly.

This structure is required because the Slack message needs to interpolate multiple record fields into a formatted string, which is significantly cleaner in JavaScript than in n8n expressions. It also avoids the need to write complex expressions in the Slack node's JSON body field.

### Node order: Prepare Message → Slack → Update Airtable
The Update Airtable HTTP Request node comes after the Slack node, not before. The Slack node uses `$json.slackBody` from the Code node, which is available while the Code node is the most recent node in the chain. After the Slack HTTP Request executes, `$json` becomes the Slack API response — so the Update node uses a cross-node reference (`$('Prepare Day 7 Message').item.json.id`) to retrieve the original Airtable record ID. This ordering and the cross-node reference are tightly coupled — reordering the nodes would break the reference.

### IF check after each query (Day 7/14 Records Found?)
Each branch includes an IF node that checks whether the Airtable query returned any records before attempting to prepare messages or send Slack notifications. Without this check, an empty query result would cause the Code node to produce no output and the Slack/Update nodes to execute with undefined data. The IF node routes empty results to a terminal path (no-op) while routing matched records to the message preparation path.

---

## What it does not do

- **Does not create Airtable records** — only reads and updates existing ones created by the core pipeline
- **Does not retry on Slack failure** — Slack nodes have `continueOnFail: true`, meaning a Slack failure does not prevent the Airtable checkbox from being updated; the nudge is considered sent even if Slack delivery fails
- **Does not alert on its own failure** — a workflow-level failure (not a node-level failure) would result in a missed nudge cycle with no notification. This is acceptable at v1 scale: the range query recovers missed nudges on the next run.
- **Does not send nudges beyond 21 days** — records older than 21 days fall outside the query window by design

---

## v2 touch points

- **Batching**: v1 sends one Slack message per qualifying record. If multiple records qualify on the same day, this generates multiple separate Slack messages. v2 could group all day 7 records into a single Slack message and all day 14 records into another, reducing notification noise.
- **Failure alerting**: v1 has no workflow-level failure notification. v2 could add an n8n error workflow that sends a Slack alert when the nudge workflow itself fails to execute.
- **The 21-day cap** is an implicit design choice not documented in the interface contract. v2 should make this explicit and potentially configurable.
