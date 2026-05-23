# Runbook: Reading List Update Didn't Ingest

Use this when you've added or edited an article in the Reading List Google Doc, but the bot doesn't seem to know about it.

## Step 1 — Confirm Stage 4 actually triggered

Stage 4 fires when the Google Doc is modified. The trigger polls every minute by default, so changes take **up to a minute** to register.

In n8n, left sidebar → **Executions** → filter by the **Anchor Crew Pipeline** workflow. Look for a recent execution that started with the **Google Drive Trigger** node.

| What you see | What it means |
|--------------|---------------|
| Execution appeared within ~2 minutes of your edit | Trigger fired. Skip to Step 3 |
| No execution at all | Trigger didn't fire. Continue to Step 2 |

## Step 2 — Trigger didn't fire

### Most common cause: the wrong doc

The Google Drive Trigger watches a **specific doc by ID**, not all docs. If you edited a different doc by mistake, nothing would trigger.

Open the workflow → click the **Google Drive Trigger** node → look at the `File to Watch` field. The value should be the doc ID from your Reading List doc's URL:

```
https://docs.google.com/document/d/THIS_IS_THE_DOC_ID/edit
                                    ^^^^^^^^^^^^^^^^
```

If the IDs don't match, that's the problem.

### Other causes

- The workflow is **inactive**. Activate it
- The Google Drive credential expired. Re-authenticate in n8n Credentials
- Google's APIs are degraded — check [Google Workspace Status](https://www.google.com/appsstatus/dashboard/)

### Manual trigger

To force Stage 4 to run regardless:

1. In n8n, open the workflow
2. Click the **Google Drive Trigger** node
3. Click **Execute Node** at the top of the right-hand panel
4. The downstream nodes will fan out from there

## Step 3 — Trigger fired but URL wasn't extracted

Look at the execution and click the **Extract URLs from Doc** node.

| Output | What it means |
|--------|---------------|
| Your new URL appears in the list | Extraction worked. Continue to Step 4 |
| Output is empty (0 items) | The doc returned no content. Check the previous node's output |
| Your URL is missing from the list | The URL extraction regex didn't catch it |

### URL not being picked up

The extraction logic looks for `https://` or `http://` followed by non-whitespace characters. Things that break it:

- The URL was added as a **hyperlink** with display text only (the URL itself isn't visible in the doc text)
  - **Fix:** paste the URL as plain text on its own line, not just as a hyperlink on a title
- The URL has unusual characters that the regex stripped (quotes, angle brackets)
- The URL appears inside a table cell with weird formatting

### Recommended format

For each article in the Reading List, put it like this:

```
Title of the article goes here

https://example.com/full-url-to-the-article

Title of the next article

https://example.com/another-url
```

Plain text title on one line, blank line, URL on the next line, blank line, repeat. The extractor pairs each URL with the line above it as its title.

## Step 4 — URL extracted but scraping failed

In the execution, click the **Scrape URL** node and look at the output for your URL.

| Output | What it means |
|--------|---------------|
| Status 200 with content | Scrape worked. Continue to Step 5 |
| Status 403 or 401 | Site requires login or blocks bots |
| Status 404 | URL is dead or wrong |
| Status 429 or 503 | Site is rate-limiting |
| Timeout error | Site took too long to respond |

### Sites that won't scrape

Some publications consistently block automated scraping:
- The Wall Street Journal
- The Financial Times
- Most newspapers behind paywalls

If you want to include an article from one of these, your options are:
- Find a free mirror of the article
- Copy the article text into a Google Doc and link to that instead
- Manually add the content as a Pinecone vector via a separate one-off workflow

## Step 5 — Scraping succeeded but article was skipped

Click the **Prepare & Skip Empty** node and look at the output for your URL.

| Output field | Means |
|--------------|-------|
| `skip: true, reason: "Too short (X chars)"` | Page scraped but content under 200 chars. Probably hit a paywall or login wall |
| `skip: false` and you see `pageContent` populated | Article is ready for embedding. Continue to Step 6 |

If skipping is happening unexpectedly, the page might be rendering content via JavaScript (the scraper sees an empty shell). Same workarounds as Step 4.

## Step 6 — Embedding or Pinecone upsert failed

Click the **Pinecone — Upsert Vectors** node.

| Error | Cause | Fix |
|-------|-------|-----|
| `Vector dimension X does not match the dimension of the index 3072` | Embeddings node using wrong model | Set to `text-embedding-3-large` |
| `401 Unauthorized` | Pinecone credential invalid | Re-create credential in n8n |
| `Index not found` | Wrong index name | Check it matches your Pinecone dashboard |
| No error but bot still doesn't know about the article | Cache or wait time | Pinecone is consistent within ~30 seconds, just wait |

## Step 7 — Everything succeeded but bot still doesn't know

If Stage 4 completed cleanly but the bot says "I don't have information about that":

1. **Verify the vector is in Pinecone** — go to Pinecone dashboard → your index → check vector count went up
2. **Ask the bot a more specific question** — sometimes retrieval misses if the query is too vague
3. **Check the namespace** — Stage 4 writes to `anchor-knowledge-base`, Stage 5 reads from the same namespace. If they don't match, the bot can't find anything

## Quick test after adding a new article

Once Stage 4 finishes successfully, ask the bot a question that ONLY your new article would answer. For example, if you just added an article about a specific Nigerian fintech regulation, ask:

> "What does Anchor know about [specific topic from new article]?"

If the bot pulls relevant content, ingestion worked. If it says it doesn't know, retest the steps above.

## When to do a full re-ingestion

If the Pinecone index gets corrupted or inconsistent, you can wipe and rebuild:

1. In Pinecone dashboard → your index → namespace → **Delete all vectors**
2. In n8n, manually trigger Stage 4 from the **Google Drive Trigger** node
3. Wait 5–10 minutes for all articles to re-embed
4. Verify the vector count in Pinecone matches your expectation (~150–300 vectors for a typical reading list)
