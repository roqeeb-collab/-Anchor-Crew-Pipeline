# Runbook: New Hire Stuck in Pipeline

Use this when a new hire's onboarding has stalled — they filled out the form but haven't received their welcome DM, or their status hasn't advanced.

## Step 1 — Find them in the Employees sheet

Open the Employees Google Sheet and find the row by `email_address` or `first_name`.

Look at the `status` column. The value tells you exactly where they're stuck.

## Step 2 — Diagnose based on status

### Status: empty / row doesn't exist

**What it means:** Stage 1 never ran successfully.

**Investigate:**
1. Check Form Responses sheet — does their submission appear?
2. If no → the form didn't submit (Google Forms issue, not pipeline)
3. If yes → check n8n executions for Stage 1 failures around the submission time
4. Common causes:
   - Duplicate email blocked entry (check Audit Log for `duplicate_email_detected`)
   - Missing required field — Stage 1 validation rejected the row
   - n8n was down or rate-limited

### Status: `Pending IT Setup`

**What it means:** Stage 1 ran, but Stage 2 hasn't been triggered. This means Dotun hasn't clicked the confirmation button yet.

**Investigate:**
1. Ask Dotun if he received the IT setup email
2. Search his inbox for the EMP-ID (e.g. `EMP-A3F92B71`)
3. If found → he just hasn't clicked the button. Nudge him
4. If not found → the email failed to send. Check Stage 1 execution log for the Gmail node error

### Status: `IT Setup Complete` but no `welcomed_at`

**What it means:** Stage 2 ran, Stage 3 cron is running every 30 minutes but hasn't been able to DM the hire yet.

**Investigate:**

Check three columns in the row:

1. **`welcome_due_at`** — has this timestamp passed?
   - If it's in the future, the hire isn't due yet. Wait
   - If it's in the past, Stage 3 should have tried. Continue
2. **`welcome_attempts`** — how many retries have happened?
   - If 0 → Stage 3 cron isn't running. Check workflow is active
   - If 1–13 → Slack lookup is failing repeatedly. See `last_welcome_error`
   - If 14+ → Stage 3 has given up. Manual intervention needed
3. **`last_welcome_error`** — what does the error say?
   - `no_slack_user_with_name_<firstname>` → the hire hasn't joined Slack yet, OR their Slack name doesn't match their first_name
   - Other errors → check the Slack API response in the n8n execution log

**Manual fix if Slack lookup keeps failing:**

If the hire is in Slack but with a different username (e.g. `tosinsobogun` instead of `tosin`):

1. In Slack, find the hire's User ID (right-click profile → Copy member ID)
2. Open the Employees row and manually set `slack_user_id` to their ID
3. Set `welcomed_at` to now and `welcome_attempts` to 0
4. (Optional) Send them a manual welcome DM since the bot won't

### Status: `Welcomed by Bot`

**What it means:** The bot already DM'd them successfully. They're not stuck — they're just not chatting.

**Investigate:**
1. Check Slack — did they receive the welcome DM?
2. If they replied to the bot but didn't get a response, see [bot-not-responding.md](bot-not-responding.md)

## Step 3 — Check the Audit Log

The Audit Log sheet has timestamps and details for every meaningful action. Filter by the hire's `emp_id` to see their full timeline.

Common patterns:
- Many `welcome_failed` entries → Slack lookup issue (see above)
- One `created_employee` but nothing after → Stage 1 succeeded but downstream is broken
- Empty audit log → Stage 1 itself never ran

## Step 4 — Last resort: nudge the pipeline manually

If diagnostics say everything looks fine but the hire still isn't moving:

| To re-trigger | Do this |
|---------------|---------|
| Stage 2 (IT confirmation) | In n8n, manually execute the workflow with the empId in the webhook query string |
| Stage 3 (welcome DM) | Set `welcome_due_at` to a timestamp in the past, then wait up to 30 minutes for the cron |
| Stage 5 (chat reply) | Have the hire send another message; it should fire a fresh execution |

## Escalation

If you've tried the above and the hire still isn't progressing, the most likely root causes are:

1. The n8n workflow isn't active — check the top-right toggle
2. A credential expired — check the Credentials section in n8n
3. An external service is down — Slack, Google Sheets, or Pinecone status pages

Worst case: drop a manual welcome DM, set their status to `Welcomed by Bot` in the sheet, and follow up on the root cause separately. The hire shouldn't suffer for our pipeline's issues.
