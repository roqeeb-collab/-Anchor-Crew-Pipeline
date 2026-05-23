# Pinecone Setup

This guide creates the vector database that backs the Knowledge Bot's retrieval.

Estimated time: **5 minutes**.

## Step 1 — Create a Pinecone account

1. Go to [https://www.pinecone.io](https://www.pinecone.io)
2. Click **Sign Up** (top right)
3. Verify your email
4. Pick the **Starter** tier when prompted (sufficient for our needs)

## Step 2 — Create the index

From the Pinecone dashboard, click **Create Index** (or **+ New Index**).

Fill in these exact values:

| Field | Value |
|-------|-------|
| **Index name** | `anchor-knowledge` |
| **Dimensions** | `3072` |
| **Metric** | `cosine` |
| **Capacity mode** | `Serverless` |
| **Cloud provider** | `AWS` |
| **Region** | `us-east-1` |

Click **Create Index**. Wait ~30 seconds for status to flip from "Initializing" → "Ready".

### Why these specific values

- **Dimension 3072** must match the output of OpenAI's `text-embedding-3-large` model exactly. A mismatch causes every upsert to fail.
- **Cosine metric** is the standard for text similarity (vs. dot-product or euclidean).
- **Serverless** is pay-per-use with no fixed cost. Pod-based costs money even when idle.

## Step 3 — Get your API key

1. Left sidebar → **API Keys**
2. You'll see a default key already created. Click the **eye icon** to reveal it
3. Click the **copy icon** to copy
4. The key looks like `pcsk_...` followed by a long string
5. Save this somewhere secure — you'll paste it into n8n next

## Step 4 — Create the n8n credential

1. In n8n, left sidebar → **Credentials** → **Add Credential**
2. Search for **Pinecone** → select **Pinecone API**
3. **API Key:** paste your `pcsk_...` key
4. **Name:** `Anchor Pinecone`
5. Click **Save**
6. Copy the credential ID from the browser URL (the alphanumeric string after `/credentials/`)

## Step 5 — Wire up the workflow

In your imported Anchor Crew Pipeline workflow:

1. Click the **Pinecone — Upsert Vectors** node (Stage 4)
2. Set the credential to `Anchor Pinecone`
3. Set the index to `anchor-knowledge`
4. Repeat for the **Pinecone Vector Store** node (Stage 5)

## Step 6 — Verify

1. In Pinecone dashboard, click your `anchor-knowledge` index
2. The **Stats** tab should show:
   - Total vectors: 0 (until Stage 4 runs)
   - Dimension: 3072
   - Namespaces: (empty until Stage 4 runs)

You're done. Run Stage 4 manually once and the index will populate with vectors from your reading list.

## Common gotchas

| Error | Cause | Fix |
|-------|-------|-----|
| `Vector dimension X does not match the dimension of the index 3072` | Embeddings model produces different size than 3072 | Verify the OpenAI Embeddings node uses `text-embedding-3-large` |
| `Index not found` | Wrong index name in the workflow | Index names are case-sensitive |
| `401 Unauthorized` | API key wrong or revoked | Regenerate key in Pinecone, update n8n credential |
| No vectors appearing after Stage 4 runs | Embeddings node isn't connected, OR Stage 4 has 0 valid URLs | Check the Stage 4 execution log to see which step is failing |

## Storage notes

A typical reading list of 15–20 fintech articles produces roughly **150–300 vectors** in Pinecone (each article is chunked into ~10 chunks before embedding). At 3072 dimensions per vector × 4 bytes per float, that's about **2–4 MB of storage**, well within the free tier.

If your reading list grows to thousands of articles, you may eventually exceed the free tier's 2GB cap. At that point, you can either:
- Upgrade to a paid Pinecone plan
- Or move to a self-hosted vector store like Qdrant or Weaviate
