# Slack App Setup

This guide walks through creating the Slack app that powers Stages 3 (welcome DM) and 5 (chat bot).

Estimated time: **15 minutes**.

## Prerequisites

- Slack workspace admin access (or someone who can install apps on your behalf)
- Your n8n instance already running and accessible from the internet (Railway, n8n Cloud, etc.)
- The Anchor Crew Pipeline workflow imported into n8n (but not yet activated)

## Step 1 — Create the app

1. Go to [https://api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App** → **From scratch**
3. **App Name:** `Anchor Knowledge AI` (or whatever you want users to see)
4. **Workspace:** select your Anchor workspace
5. Click **Create App**

You'll land on the Basic Information page.

## Step 2 — Add OAuth scopes

Left sidebar → **OAuth & Permissions**. Scroll to **Scopes** → **Bot Token Scopes**. Click **Add an OAuth Scope** and add each of the following:

| Scope | Purpose |
|-------|---------|
| `users:read` | Look up team members by name |
| `chat:write` | Send messages as the bot |
| `im:write` | Open a DM channel with a user the bot hasn't messaged before |
| `im:history` | Read messages users send to the bot |

You do NOT need `users:read.email` — we use name-based lookup instead.

## Step 3 — Enable the Messages tab

Left sidebar → **App Home**. Scroll to **Show Tabs**.

1. Toggle **Messages Tab** to ON
2. Check the box: **"Allow users to send Slash commands and messages from the messages tab"** — this is the specific setting that removes the "Sending messages to this app has been turned off" banner

## Step 4 — Install the app

Left sidebar → **Install App** → **Install to Workspace** → **Allow**.

You'll be sent back to the OAuth page with a **Bot User OAuth Token** visible at the top. It starts with `xoxb-...`. **Copy this** — you'll paste it into n8n in the next step.

## Step 5 — Create the n8n credential

1. In n8n, left sidebar → **Credentials** → **Add Credential**
2. Search for **Slack** → select **Slack API** (NOT "Slack OAuth2 API")
3. **Access Token:** paste your `xoxb-...` token
4. **Name:** `Anchor Slack Bot`
5. Click **Save**

After saving, copy the credential ID from the browser URL (the long alphanumeric string after `/credentials/`). This is what you'll plug into the workflow JSON if you're doing find-and-replace.

## Step 6 — Activate the workflow

In n8n, open the Anchor Crew Pipeline workflow:

1. Make sure each Slack node has your `Anchor Slack Bot` credential selected
2. Toggle the workflow **Active** at the top right

The webhook for Stage 5 now listens at:

```
https://<your-n8n-domain>/webhook/slack-events
```

You'll need this URL in Step 7.

## Step 7 — Subscribe to Slack events

Back in your Slack app dashboard → **Event Subscriptions**.

1. Toggle **Enable Events** to ON
2. In **Request URL**, paste your webhook URL from Step 6
3. **Wait for the green ✅ Verified** to appear

If verification fails:
- Make sure the n8n workflow is actually **active** (not just saved)
- If your n8n is on Railway and may be sleeping from a cold start, hit the URL once in your browser to wake it, then retry
- Double-check the URL ends in `/webhook/slack-events` (not `/webhook-test/slack-events` — that's the test URL)

Once verified, scroll down to **Subscribe to bot events** → **Add Bot User Event** → add:

| Event | Why |
|-------|-----|
| `message.im` | Fires when a user DMs the bot |

Click **Save Changes** at the bottom of the page.

## Step 8 — Reinstall the app

After adding scopes and events, Slack shows a yellow banner: *"You changed your app's permissions or events. Please reinstall your app for these changes to take effect."*

Click **Reinstall to Workspace** → **Allow**.

This step is required. Skipping it leaves the app in a broken state where it appears to be configured but the new scopes/events don't actually work.

## Step 9 — Test

1. In Slack, find **Anchor Knowledge AI** under **Apps** in the left sidebar (or search for it)
2. Click it to open a DM
3. Verify the "Sending messages..." banner is gone — you should be able to type
4. Send "hello"
5. Within 5–10 seconds, the bot should reply

If nothing happens:
- Open n8n → **Executions** in the left sidebar → see if a new execution appeared when you sent the message
- If yes, click it to see which node failed
- If no, the Slack → n8n event delivery isn't working; re-verify the webhook URL and that `message.im` is subscribed

## Common gotchas

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| "missing_scope" errors in n8n executions | Forgot to reinstall after adding a scope | Reinstall the app to workspace |
| Bot doesn't see DMs even though trigger is active | `message.im` not subscribed, OR app not reinstalled | Subscribe + reinstall |
| `users_not_found` when WF3 looks up a hire | New hire hasn't accepted Slack invite yet | Stage 3 will retry automatically for 7 days |
| Bot replies to itself in a loop | The Filter node in Stage 5 isn't catching bot messages | Make sure the filter excludes `bot_id`, `app_id`, and `subtype: 'bot_message'` |
| "Sending messages turned off" banner persists | Messages tab is on but the checkbox below it isn't ticked | App Home → tick the "Allow users to send messages" checkbox |

## What you have now

After completing these steps:

- A working Slack bot named **Anchor Knowledge AI**
- The bot can send DMs (welcome message from Stage 3)
- The bot can receive DMs and respond intelligently (chat from Stage 5)
- All events route through your n8n workflow
