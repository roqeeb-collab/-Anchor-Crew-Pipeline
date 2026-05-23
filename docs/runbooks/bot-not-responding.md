# Runbook: Bot Not Responding in Slack

Use this when the Knowledge Bot doesn't reply to DMs.

## Step 1 — Is the workflow active?

In n8n, go to the workflow and check the toggle at the top right. If it says **Inactive**, that's the problem. Toggle it ON.

## Step 2 — Does Slack see the message?

In n8n, left sidebar → **Executions**. Send the bot a test DM (e.g. "hi") and watch the executions list.

| What you see | What it means |
|--------------|---------------|
| A new execution appears within 2 seconds | Slack is reaching n8n. Skip to Step 4 |
| Nothing appears, even after a minute | Slack isn't reaching n8n. Continue to Step 3 |

## Step 3 — Slack→n8n delivery is broken

This means Slack's Event Subscriptions URL isn't pointing at your n8n correctly, or the URL is unreachable.

### Check Slack's Event Subscriptions

1. Go to [https://api.slack.com/apps](https://api.slack.com/apps) → your app → **Event Subscriptions**
2. Verify:
   - Toggle is **ON**
   - Request URL ends in `/webhook/slack-events`
   - Status shows ✅ **Verified**
   - Bot Events include `message.im`

### If "Verified" but still no executions

- The URL might have changed (e.g. you redeployed n8n with a new domain). Re-paste and re-verify
- Try unsubscribing and re-subscribing to `message.im`
- Reinstall the app to workspace (Install App → Reinstall)

### If URL verification fails

- Make sure the n8n workflow is **active** so the webhook listens
- If n8n is on Railway, wake it up by hitting the URL in a browser first
- Check Railway logs for crashes around verification time

## Step 4 — n8n received the event but didn't reply

If executions are appearing but the bot still isn't replying, click the execution and find which node failed.

### Common failure points

**"Filter & Prep Message" returns empty**

The filter dropped the event. Look at the input data — it might be:
- A bot's own message (intentionally filtered)
- An edit/delete (`subtype` is set)
- Not a DM (`channel_type !== 'im'`)

If your filter is too aggressive, loosen it.

**"AI Agent" times out or errors**

Check the OpenAI Chat Model node:
- Credential valid?
- Billing in good standing?
- Model name correct (`gpt-4o-mini`)?

Sometimes OpenAI returns an error like "rate limit reached" — wait a minute and try again.

**"Knowledge Base Tool" → "Pinecone Vector Store" error**

This usually means:
- Pinecone credential invalid
- Index name wrong or doesn't exist
- Index dimension doesn't match the embeddings model (must be 3072)
- The namespace `anchor-knowledge-base` is empty (run Stage 4 first)

**"Slack Reply" fails**

Most likely:
- Slack credential expired
- `chat:write` scope was revoked
- The channel ID expression failed to resolve

Look at the Slack Reply node's error message — it usually tells you exactly which.

## Step 5 — n8n replied but message didn't reach Slack

Rare but possible. Check:

1. The Slack Reply node's output — does it show `ok: true`?
2. Open the DM in Slack — is the message there but you missed it (maybe an empty bubble due to a malformed expression)?
3. Look at the rendered message — if it's empty, the `text` expression resolved to an empty string. Inspect what `{{ $json.output }}` actually contained

## Step 6 — Bot replies but answers are wrong/empty

If the bot replies but says things like "I don't have information about that" for every question:

- The Pinecone index is empty. Run Stage 4 to ingest the reading list
- The embedding model in Stage 5 doesn't match the one used in Stage 4 (both must be `text-embedding-3-large`)
- The namespace in Stage 5 doesn't match Stage 4 (both must be `anchor-knowledge-base`)

## Quick health check command

Send the bot exactly: **"What is BaaS?"**

| Bot's response | What it tells you |
|----------------|-------------------|
| Detailed answer mentioning Anchor, BaaS, embedded finance | Everything works ✅ |
| "I don't have information about that" | Pinecone is empty or wrong namespace |
| Polite refusal to answer | The strict scope prompt is too strict, or the question is being classified as off-topic |
| Nothing at all | Slack→n8n or n8n→Slack broken (start at Step 1) |
| Error message | Read the message, it usually tells you what's wrong |

## Escalation

If steps 1–6 don't pinpoint the issue, the most useful information for debugging is:

1. The exact message you sent
2. The execution ID from n8n (shows on the execution detail page)
3. A screenshot of the failed node
4. The Slack app's Event Subscriptions status page

Drop those in a Slack message to whoever maintains the pipeline.
