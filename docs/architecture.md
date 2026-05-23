# Architecture

A deeper look at how the Anchor Crew Pipeline is structured.

## High-level shape

The pipeline is **one n8n workflow with five logical stages**, each anchored by its own trigger. Stages 1 and 2 chain together via email-and-webhook; Stages 3, 4, and 5 run independently of one another, coordinated through shared data (the Employees sheet and Pinecone).

```
                                 ┌─────────────────────┐
              ┌─────────────────►│   Employees Sheet   │◄──────────────────┐
              │                  └─────────────────────┘                   │
              │                           ▲                                │
              │                           │                                │
              │                  ┌────────┴────────┐                       │
              │                  │                 │                       │
        ┌─────┴────┐       ┌─────┴─────┐    ┌─────┴────┐           ┌──────┴───┐
        │ Stage 1  │  ──►  │  Stage 2  │    │ Stage 3  │           │ Stage 5  │
        │ Intake   │ email │ IT Confirm│    │ Welcome  │           │ Chat Bot │
        └──────────┘       └───────────┘    └──────────┘           └──────────┘
                                                                        ▲
                                                                        │
                                                                  ┌─────┴─────┐
                                                                  │ Pinecone  │
                                                                  └─────▲─────┘
                                                                        │
                                                                   ┌────┴────┐
                                                                   │ Stage 4 │
                                                                   │ Ingest  │
                                                                   └─────────┘
```

## Data schemas

All structured data lives in a single Google Spreadsheet with four tabs.

### Form Responses (read-only, populated by Google Form)

This is auto-managed by Google Forms. The columns reflect the form fields exactly as the form is configured.

### Employees

The system-of-record for every employee.

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| `emp_id` | string | Stage 1 generates | Format `EMP-XXXXXXXX` |
| `first_name` | string | Form | |
| `middle_name` | string | Form | Optional |
| `last_name` | string | Form | |
| `designation` | string | Form | Job title |
| `phone_number` | string | Form | |
| `email_address` | string | Form | Personal email, NOT company |
| `house_address` | string | Form | |
| `tshirt_size` | string | Form | For swag |
| `headshot_url` | string | Form | Optional, drive link |
| `start_date` | date | Form | M/D/YYYY |
| `status` | enum | Pipeline writes | See status lifecycle below |
| `created_at` | timestamp | Stage 1 | ISO 8601 |
| `last_modified` | timestamp | All stages | ISO 8601 |
| `company_email` | string | Stage 2 | `{firstname}@anchor.co` |
| `welcome_due_at` | timestamp | Stage 2 | `start_date + 1 day @ 9am` |
| `welcomed_at` | timestamp | Stage 3 | Set when Slack DM succeeds |
| `slack_user_id` | string | Stage 3 | `U07ABC123` |
| `welcome_attempts` | number | Stage 3 | Caps at 14 |
| `last_welcome_error` | string | Stage 3 | Most recent failure reason |

### Status lifecycle

```
                  ┌──────────────────┐
                  │ Pending IT Setup │   ← Stage 1 writes this
                  └────────┬─────────┘
                           │
                           ▼  (Dotun confirms)
                  ┌──────────────────┐
                  │ IT Setup Complete│   ← Stage 2 writes this
                  └────────┬─────────┘
                           │
                           ▼  (Stage 3 cron picks it up after welcome_due_at)
                  ┌──────────────────┐
                  │ Welcomed by Bot  │   ← Stage 3 writes this
                  └──────────────────┘
```

### Payroll

A parallel record created at the same time as Employees, intended for the finance team. Schema mirrors Employees but only the columns that finance needs (emp_id, name, designation, start_date, bank details from the form).

### Audit Log

Append-only log of every meaningful action.

| Column | Type |
|--------|------|
| `timestamp` | ISO 8601 |
| `actor` | string (e.g. `stage-1-intake`, `dotun`, `knowledge-bot`) |
| `action` | string (e.g. `created_employee`, `welcomed_new_hire`) |
| `emp_id` | string |
| `details` | string |

## Why the workflow is one file, not five

You'll notice the workflow JSON has **74 nodes in a single file**, not five separate workflows. This is deliberate after some iteration:

**Pros of one file:**
- One workflow to back up, version, share
- Easier to see end-to-end flow when debugging a stuck hire
- One credential set to maintain

**Cons (which we accept):**
- The canvas in n8n is visually busy
- A misconfiguration in one stage could in theory affect others

To mitigate the cons:
- Each stage has a **color-coded sticky note** explaining what it does
- Stages are spatially separated on the canvas (each cluster is at its own Y-coordinate)
- Errors in one stage's branch don't propagate to others (n8n's per-node error handling)

## Trigger summary

| Stage | Trigger type | Activates when |
|-------|-------------|----------------|
| 1 | Google Sheets Trigger | New row in Form Responses sheet |
| 2 | Webhook | Dotun's email button is clicked |
| 3 | Schedule Trigger | Every 30 minutes |
| 4 | Google Drive Trigger | Reading List Doc is modified |
| 5 | Webhook | Slack POSTs a `message.im` event |

## Pinecone index design

| Setting | Value |
|---------|-------|
| Index name | `anchor-knowledge` |
| Dimension | 3072 |
| Metric | cosine |
| Capacity mode | Serverless |
| Cloud | AWS |
| Region | us-east-1 |
| Namespace | `anchor-knowledge-base` |

The dimension is locked to **3072** because we use OpenAI's `text-embedding-3-large` model. If you switch embedding models, you must recreate the index with a matching dimension and re-run Stage 4 to re-embed everything.

## How the AI Agent reasons

When a new hire DMs the bot in Stage 5, the AI Agent receives:

1. **The user's message** (e.g., "what is embedded finance?")
2. **The system prompt** (strict scope, Anchor framing, Nigerian/African context)
3. **The last 20 messages** of conversation memory for that user
4. **A tool definition** for `knowledge_base` that wraps a Pinecone query

The agent then decides:
- Is this in scope? If not, refuse politely
- Do I need fresh context? If yes, call `knowledge_base("embedded finance")`
- Pinecone returns the 5 most relevant chunks from the reading list
- The Tool Summarizer Model condenses them
- The agent uses the summary + memory + system prompt to compose the final reply

For greetings or simple follow-ups, the agent skips the tool call and replies directly from memory.

## Error handling philosophy

The pipeline is designed to **fail forward** — if any non-critical step fails, the rest of the pipeline keeps going and logs the failure rather than blocking the hire entirely.

| Failure mode | Behavior |
|--------------|----------|
| Slack lookup fails (hire hasn't accepted invite) | Stage 3 retries every 30 min, caps at 14 attempts |
| Article scraping returns empty (paywall) | Stage 4 logs and skips that URL, continues with the next |
| Pinecone upsert fails | Stage 4 logs error, continues to the next URL |
| AI Agent times out | Stage 5 falls back to a generic "give me a moment, try again" reply |
| Dotun doesn't click the button for days | Stage 2 doesn't fire, hire stays in `Pending IT Setup`. HR sees this in the sheet and follows up manually |

What does NOT fail forward:
- A duplicate email in Stage 1 blocks the new row from being created until HR approves
- A missing required field in Stage 1 bounces the entry back to the form submitter

## Security considerations

- **No PII in logs.** Audit Log entries reference emp_id, not personal data
- **Slack token has minimal scopes** — `users:read`, `chat:write`, `im:write`, `im:history` only
- **OpenAI gets no personal data** — only the user's question and the reading list content, neither of which contains PII
- **Pinecone data is anonymous** — vector embeddings of public articles, not user-specific
- **The Slack Events webhook is unauthenticated** by default — Slack signing secret verification is a future hardening step
