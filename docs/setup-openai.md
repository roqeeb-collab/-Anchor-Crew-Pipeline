# OpenAI Setup

This guide creates the OpenAI credential used by Stages 4 (embeddings) and 5 (chat agent).

Estimated time: **3 minutes**.

## Step 1 — Create an OpenAI account

If you don't already have one:

1. Go to [https://platform.openai.com/signup](https://platform.openai.com/signup)
2. Sign up with your work email
3. Verify your email and phone number

## Step 2 — Add billing

OpenAI's API requires a payment method on file, even for low-volume use.

1. Go to [https://platform.openai.com/settings/organization/billing/overview](https://platform.openai.com/settings/organization/billing/overview)
2. Click **Add payment method**
3. Add a credit card and set a usage limit (recommended: cap monthly usage so unexpected spikes don't surprise you)

## Step 3 — Generate an API key

1. Go to [https://platform.openai.com/api-keys](https://platform.openai.com/api-keys)
2. Click **Create new secret key**
3. **Name:** `Anchor Crew Pipeline` (or any identifier you'll remember)
4. **Permissions:** All (you can restrict later if needed)
5. Click **Create secret key**
6. **Copy the key immediately** — OpenAI will not show it again
7. It starts with `sk-...` or `sk-proj-...` followed by a long string

## Step 4 — Create the n8n credential

1. In n8n, left sidebar → **Credentials** → **Add Credential**
2. Search for **OpenAI** → select **OpenAI API**
3. **API Key:** paste your `sk-...` key
4. **Name:** `Anchor OpenAI`
5. Click **Save**
6. Copy the credential ID from the browser URL

## Step 5 — Wire up the workflow

The OpenAI credential is used in multiple places. After importing the workflow, set the credential on each:

| Node | Stage | Purpose |
|------|-------|---------|
| OpenAI Embeddings (Ingestion) | 4 | Embeds reading list articles |
| OpenAI Chat Model | 5 | Agent's main brain |
| Tool Summarizer Model | 5 | Condenses retrieved Pinecone chunks |
| OpenAI Embeddings (Retrieval) | 5 | Embeds user questions for Pinecone search |

All four should use the same `Anchor OpenAI` credential.

## Models used

| Model | Where | Why |
|-------|-------|-----|
| `text-embedding-3-large` | Stages 4 and 5 | 3072 dimensions, high quality, matches Pinecone index |
| `gpt-4o-mini` | Stage 5 (agent + summarizer) | Cheap, fast, sufficient quality for an internal knowledge bot |

If you want to upgrade the chat model later (e.g. to `gpt-4o` or `gpt-4-turbo`), change the model name in the **OpenAI Chat Model** node. The embedding model must stay as `text-embedding-3-large` unless you also recreate the Pinecone index with a new dimension.

## Verifying it works

1. Make sure your OpenAI account has at least a few dollars of credit available
2. Trigger Stage 4 manually from n8n
3. Watch the **OpenAI Embeddings** node — it should output an array of 3072 numbers per chunk
4. If the node outputs `[]` (empty array), the model name is wrong or the credential is invalid

## Common gotchas

| Error | Cause | Fix |
|-------|-------|-----|
| `Incorrect API key provided` | Wrong key or whitespace in the credential | Re-copy and re-paste the key |
| `You exceeded your current quota` | No billing set up, OR usage limit hit | Add payment method, raise the cap |
| `Rate limit reached` | Too many requests too fast | Stage 4 has a 2-second delay; if you hit limits anyway, increase the delay |
| Embedding output is `[]` empty array | Model field empty, OR credential disconnected | Verify the model is set to `text-embedding-3-large` |
| Agent never replies in Slack | Chat model name typo or wrong credential | Test the OpenAI Chat Model node in isolation |

## Security notes

- The OpenAI key has access to your billed account — treat it like a password
- Don't commit it to git. n8n stores it encrypted server-side
- Rotate it if exposed (regenerate in OpenAI dashboard → update the n8n credential)
- OpenAI sees your reading list content and user questions, but no PII unless users send PII to the bot themselves
