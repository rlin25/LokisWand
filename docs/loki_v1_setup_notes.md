# LokisWand — Setup Notes v1

**Status:** Implementation complete. All phases passed. Phase 7 feedback loop complete.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| n8n cloud account | Cloud required for reliable scheduled workflow execution. Self-hosted instances may miss scheduled nudge windows. |
| Airtable account | Free tier supports up to 1,000 records — sufficient for personal job search use. |
| Anthropic API key | Obtain from console.anthropic.com. Used for all three Claude calls per submission. |
| Slack workspace | Personal workspace with an incoming webhook URL configured. |

---

## Environment Variables

The following values must be configured before importing the workflows:

| Variable | Where | Value |
|---|---|---|
| `SLACK_WEBHOOK_URL` | n8n Settings → Variables | Your Slack incoming webhook URL |
| Airtable Personal Access Token | n8n Credentials | Personal access token from Airtable developer settings |
| Anthropic API Key | n8n Credentials | API key from console.anthropic.com |

Sensitive values are stored in `.env` for local reference:

```
ANTHROPIC_API_KEY=...
AIRTABLE_BASE_ID=...
AIRTABLE_TABLE_ID=...
AIRTABLE_PERSONAL_ACCESS_TOKEN=...
SLACK_WEBHOOK_URL=...
```

The Slack webhook URL must also be added as an n8n Variable (`$vars.SLACK_WEBHOOK_URL`) in n8n Settings → Variables. The workflow JSON references `$vars.SLACK_WEBHOOK_URL` — if this variable is not set in n8n, all Slack notifications will silently fail.

---

## Airtable Setup

1. Create a base named **LokisWand** with one table: **Applications**
2. Add the following fields exactly (field names are case-sensitive):

| Field name | Type |
|---|---|
| Company | Single line text |
| Role title | Single line text |
| Location | Single line text |
| Employment type | Single line text |
| Date submitted | Date (include time) |
| Fit signals | Long text |
| Risk signals | Long text |
| Synthesis | Long text |
| Recommendation | Single select: Strong Fit, Moderate Fit, Weak Fit, Do Not Apply |
| Status | Single select: Researching, Applied, Following Up, Interviewing, Offer, Rejected, Skipped |
| Follow-up sent day 7 | Checkbox |
| Follow-up sent day 14 | Checkbox |
| Profile version | Single line text |

3. Create a Kanban view grouped by the **Status** field.
4. Add the Airtable Personal Access Token and Base ID to n8n credentials and the `.env` file.

---

## Workflow Import

1. Import `workflows/loki_core_pipeline.json` into n8n
2. Import `workflows/loki_followup_nudge.json` into n8n
3. After import, update all credential references in both workflows to match your configured credentials
4. Verify that `$vars.SLACK_WEBHOOK_URL` is set in n8n Settings → Variables
5. Activate both workflows

---

## Profile Setup

1. Open `profile/loki_conversion_prompt.md`
2. Paste the conversion prompt and your master resume into a Claude conversation (Claude.ai or API)
3. Review the generated output — verify everything in the Review Required section
4. Correct any inaccuracies in hard constraints, goals, and known gaps
5. Save the output as `profile/loki_candidate_profile.md`
6. Open the core pipeline workflow in n8n
7. Find the **Load Profile Document** code node
8. Replace the existing profile text in the node's JavaScript with the content of your `loki_candidate_profile.md`
9. Save and activate the workflow

Note: Every time the candidate profile is updated (new version), step 8 must be repeated in the n8n workflow. The profile text is embedded directly in the Code node, not loaded from an external file. See Decision 17.

---

## Known Issues

### n8n Airtable v2 batchUpdate bug
**Symptom:** 422 INVALID_RECORDS error when attempting to update Airtable records using the native n8n Airtable node.
**Cause:** The n8n Airtable node v2 routes all update operations through batchUpdate. "Columns to match on: id" treats "id" as a user-created column name, not the internal Airtable record ID. The search for a user-created "id" column returns nothing, causing a malformed request.
**Resolution:** The follow-up nudge workflow uses HTTP Request PATCH nodes with the Airtable REST API directly. This is already implemented in `loki_followup_nudge.json`. If the n8n Airtable node is ever substituted back in for update operations, this bug will recur. See Decision 16.

### n8n HTTP Request node expression syntax
**Symptom:** JSON body expressions in HTTP Request nodes produce `=[undefined]` or fail silently.
**Cause:** HTTP Request node JSON body fields set to Expression mode use `{{ expression }}` syntax (no leading `=`). The `=` prefix applies only to toggle-mode parameter fields. Using `={{ expression }}` in an HTTP Request JSON body field causes the expression to fail.
**Resolution:** All expressions in HTTP Request node JSON body fields use `{{ }}` without the `=` prefix. This is already correctly configured in the exported workflow JSONs. If expressions are edited manually in n8n, use `{{ }}` in HTTP Request JSON body fields and `={{ }}` in all other expression fields. See Decision 19.

### Profile update requires workflow edit
**Symptom:** Assessments use outdated candidate information after a profile update.
**Cause:** The candidate profile is embedded in the Load Profile Document code node, not loaded from an external file.
**Resolution:** After updating `profile/loki_candidate_profile.md`, open the core pipeline workflow in n8n, find the Load Profile Document code node, and replace the embedded profile text. Increment the version field in the new profile (e.g., v1.0 → v1.1) to ensure Airtable records reflect which profile version was used. See Decision 17.

---

## External Dependencies

| Service | Version / Model | Notes |
|---|---|---|
| n8n | Cloud (managed) | No version pinning for cloud |
| Claude API model | claude-opus-4-7 | Hardcoded in HTTP Request nodes; update all three nodes if switching models |
| Airtable API | v0 endpoint | Used in HTTP Request PATCH nodes for checkbox updates |
| Slack Incoming Webhooks | Current | Webhook URL must be set as `SLACK_WEBHOOK_URL` n8n variable |
