# 🚀 Anchor Crew Pipeline

> An end-to-end HR onboarding & knowledge-transfer automation built on [n8n](https://n8n.io), powering [Anchor](https://getanchor.co)'s new-hire experience from form submission to AI-powered knowledge chat.

[![n8n](https://img.shields.io/badge/built%20with-n8n-EA4B71?logo=n8n&logoColor=white)](https://n8n.io)
[![OpenAI](https://img.shields.io/badge/AI-OpenAI%20GPT--4o--mini-412991?logo=openai&logoColor=white)](https://platform.openai.com)
[![Pinecone](https://img.shields.io/badge/vector%20db-Pinecone-000000)](https://pinecone.io)
[![Slack](https://img.shields.io/badge/messaging-Slack-4A154B?logo=slack&logoColor=white)](https://slack.com)

---

## What this does

The **Anchor Crew Pipeline** is a 5-stage automation that turns a Google Form submission into a fully onboarded, knowledge-equipped new hire — with zero manual coordination between HR, IT, and the new joiner.

When someone fills out the new-hire intake form, the pipeline:

1. Validates and de-duplicates the entry
2. Loops in HR for approval if a duplicate is detected
3. Emails IT (Dotun) with a one-click confirmation
4. Schedules a Slack welcome DM for the day after their start date
5. Hands them off to an AI Knowledge Bot trained on Anchor's curated reading list — ready to answer questions about Banking-as-a-Service, embedded finance, and the African fintech landscape

The whole system is built on a single [n8n](https://n8n.io) instance deployed on Railway and removes ~2 hours of manual coordination per new hire.

---

## Architecture at a glance

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   Google Form           Google Sheets         Gmail          Slack      │
│      │                       │                  │              │        │
│      ▼                       ▼                  ▼              ▼        │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │                                                              │       │
│  │              🚀  ANCHOR CREW PIPELINE  (n8n)                 │       │
│  │                                                              │       │
│  │   Stage 1 ──► Stage 2 ──► Stage 3 ──► Stage 4 ──► Stage 5    │       │
│  │   Intake     IT Setup    Welcome     Ingestion   Chat        │       │
│  │                                                              │       │
│  └──────────────────────────────────────────────────────────────┘       │
│                                  │                                      │
│                          ┌───────┴────────┐                             │
│                          ▼                ▼                             │
│                     ┌─────────┐      ┌─────────┐                        │
│                     │ OpenAI  │      │Pinecone │                        │
│                     │ GPT-4o  │      │  Vector │                        │
│                     │  Mini   │      │  Store  │                        │
│                     └─────────┘      └─────────┘                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## How a new hire flows through the pipeline

```
                       👤 New hire fills onboarding form
                                    │
                                    ▼
       ┌──────────────────────────────────────────────────────┐
       │  📝 STAGE 1 · INTAKE                                  │
       │  ─────────────────────                                │
       │  • Validates form fields                              │
       │  • Detects duplicate emails → HR approval flow        │
       │  • Generates EMP-ID (EMP-XXXXXXXX)                    │
       │  • Writes Employees + Payroll sheets                  │
       │  • Emails Dotun (IT) with action button               │
       └──────────────────────┬───────────────────────────────┘
                              │
                              ▼  (Dotun clicks "✅ Setup Complete")
       ┌──────────────────────────────────────────────────────┐
       │  ✅ STAGE 2 · IT SETUP CONFIRMATION                   │
       │  ─────────────────────                                │
       │  • Webhook receives Dotun's confirmation              │
       │  • Constructs company email: firstname@anchor.co      │
       │  • Schedules welcome for start_date + 1 day @ 9am     │
       │  • Updates status: "IT Setup Complete"                │
       │  • Notifies HR, shows Dotun a success page            │
       └──────────────────────┬───────────────────────────────┘
                              │
                              ▼  (Cron checks every 30 min)
       ┌──────────────────────────────────────────────────────┐
       │  👋 STAGE 3 · WELCOME DISPATCHER                      │
       │  ─────────────────────                                │
       │  • Finds hires whose welcome_due_at has passed        │
       │  • Looks up their Slack user by first_name match      │
       │  • Sends a friendly welcome DM from the bot           │
       │  • Retries 14× over 7 days if Slack invite pending    │
       │  • Marks "Welcomed by Bot" in sheet                   │
       └──────────────────────┬───────────────────────────────┘
                              │
                              ▼  (New hire replies in Slack)
       ┌──────────────────────────────────────────────────────┐
       │  🤖 STAGE 5 · KNOWLEDGE CHAT BOT                      │
       │  ─────────────────────                                │
       │  • Listens for DMs to the bot                         │
       │  • AI Agent (GPT-4o-mini) with strict scope:          │
       │    answers ONLY fintech / BaaS / Anchor questions     │
       │  • Retrieves from Pinecone (Anchor's reading list)    │
       │  • Offers a 5-question quiz on demand                 │
       │  • Per-user conversation memory (last 20 messages)    │
       └──────────────────────────────────────────────────────┘

       ▲ Independent ingestion path runs whenever the reading list updates:
       │
       ┌──────────────────────────────────────────────────────┐
       │  📚 STAGE 4 · KNOWLEDGE INGESTION                     │
       │  ─────────────────────                                │
       │  • Watches the Reading List Google Doc                │
       │  • Extracts URLs + titles from doc content            │
       │  • Scrapes each article (skips paywalled pages)       │
       │  • Embeds with OpenAI text-embedding-3-large          │
       │  • Upserts to Pinecone with deterministic IDs         │
       │    (re-runs overwrite, no duplicates)                 │
       └──────────────────────────────────────────────────────┘
```

---

## Tech stack

| Layer | Tool | Purpose |
|-------|------|---------|
| **Automation engine** | n8n (self-hosted on Railway) | All five stages run here |
| **Data store** | Google Sheets | Employees, Payroll, Audit Log |
| **Email** | Gmail (via OAuth) | HR notifications, IT requests |
| **Messaging** | Slack (Bot Token + Events API) | Welcome DM, knowledge chat |
| **Knowledge source** | Google Docs (Reading List) | Curated fintech articles |
| **Vector DB** | Pinecone (serverless, 3072-dim, cosine) | Semantic search over articles |
| **LLM** | OpenAI GPT-4o-mini | Chat + embeddings (text-embedding-3-large) |
| **Triggers** | Google Sheets, Google Drive, Webhooks, Cron | Each stage has its own trigger |

---

## Folder structure (this repo)

```
anchor-crew-pipeline/
├── README.md                          ← this file
├── workflows/
│   ├── anchor-crew-pipeline.json      ← the full n8n workflow (74 nodes)
│   └── README.md                      ← deployment instructions
├── docs/
│   ├── architecture.md                ← deeper dive into each stage
│   ├── setup-slack-app.md             ← step-by-step Slack app config
│   ├── setup-pinecone.md              ← Pinecone index creation
│   ├── setup-openai.md                ← OpenAI credential setup
│   └── runbooks/
│       ├── new-hire-stuck.md          ← debugging when a hire isn't progressing
│       ├── bot-not-responding.md      ← Slack chat troubleshooting
│       └── reading-list-update.md     ← how to add new articles
└── .github/
    └── workflows/
        └── lint-workflows.yml         ← (optional) CI to validate JSON
```

---

## Deployment

### Prerequisites

- An n8n instance (self-hosted or n8n Cloud)
- A Google Workspace account with admin access
- A Slack workspace with admin access
- OpenAI API key with available credit
- Pinecone account

### Step-by-step

1. **Set up your data** — create the Google Sheets (Form Responses, Employees, Payroll, Audit Log) with the schemas defined in [`docs/architecture.md`](docs/architecture.md)
2. **Set up the Slack app** — follow [`docs/setup-slack-app.md`](docs/setup-slack-app.md) to create the bot, add scopes, and subscribe to `message.im`
3. **Set up Pinecone** — create a serverless index with **dimension 3072**, metric `cosine`, namespace `anchor-knowledge-base`
4. **Create n8n credentials** — Google Sheets, Gmail, Google Drive, Google Docs, Slack, OpenAI, Pinecone (7 in total)
5. **Import the workflow** — load `workflows/anchor-crew-pipeline.json` into n8n
6. **Wire up the credentials** — open each node with a credential dropdown and select your saved credentials
7. **Activate** — toggle the workflow ON. The Slack Events webhook URL becomes live
8. **Verify Slack** — paste the production webhook URL into your Slack app's Event Subscriptions → wait for ✅ verification
9. **Seed Pinecone** — run Stage 4 manually once so the Knowledge Bot has content to retrieve
10. **Test end-to-end** — submit a test form entry with your own email; verify the Slack DM lands the day after the start date

Full step-by-step setup walkthrough: [`workflows/README.md`](workflows/README.md)

---

## Stage details

<details>
<summary><strong>📝 Stage 1 · Onboarding Intake</strong></summary>

**Trigger:** New row in the Onboarding Form Responses sheet

**Key logic:**
- Validates required fields (first_name, last_name, email, designation, start_date, etc.)
- Coerces inconsistent types (account number arrives as a number, gets stringified)
- Parses the M/D/YYYY date format Google Forms emits
- Checks the Employees sheet for duplicate emails
- If duplicate → emails HR with approve/reject links pointing to `/webhook/hr-decision`
- Generates an emp_id of format `EMP-XXXXXXXX` using vanilla `Math.random()` (the n8n sandbox blocks `crypto`)
- Writes to Employees + Payroll sheets in one execution
- Calculates urgency: < 7 days = ⚠️ urgent prefix, ≥ 7 days = normal
- Emails Dotun (IT) a clean HTML message with EMP-ID, expected company email, and a confirmation button

**Outputs:** A new row in Employees, a corresponding row in Payroll, an audit log entry, and an email in Dotun's inbox.
</details>

<details>
<summary><strong>✅ Stage 2 · IT Setup Confirmation Handler</strong></summary>

**Trigger:** POST to `/webhook/it-setup-confirmed?empId=…&name=…` (from Dotun's email button)

**Key logic:**
- Looks up the employee row by emp_id
- Constructs `company_email = first_name.toLowerCase() + '@anchor.co'`
- Calculates `welcome_due_at = start_date + 1 day at 09:00`
- Updates row: status = "IT Setup Complete", adds company_email + welcome_due_at + slack_user_id (empty for now)
- Sends Dotun a clean HTML success page in his browser
- Audit logs the confirmation

**Why a separate workflow:** Stage 1's email pauses the flow on the `Wait` node. Splitting Stage 2 makes the action independent of Stage 1's execution lifetime — Dotun can confirm hours or days later.
</details>

<details>
<summary><strong>👋 Stage 3 · Welcome Dispatcher</strong></summary>

**Trigger:** Cron every 30 minutes

**Key logic:**
- Reads all rows where `status === "IT Setup Complete"`
- Filters to those where `welcome_due_at <= now` AND `welcomed_at` is empty
- For each due hire:
  1. Calls `users.list` from Slack API
  2. Filters out bots, deleted users, restricted users
  3. Matches by `user.name === hire.first_name.toLowerCase()`
  4. Takes the matched user's Slack `id`
  5. Sends a welcome DM via `chat.postMessage`
  6. Marks `welcomed_at = now`, stores `slack_user_id`
- If lookup fails, increments `welcome_attempts` and tries again next cycle
- Gives up after 14 attempts (~7 days)

**Why scheduled, not event-driven:** Slack invites take time to accept. Some new hires don't accept until day 3. The retry pattern handles this without manual intervention.
</details>

<details>
<summary><strong>📚 Stage 4 · Knowledge Ingestion</strong></summary>

**Trigger:** Google Drive — fires when the Reading List Doc is updated

**Key logic:**
- Reads the doc as plain text (current n8n Google Docs node returns flat string)
- Regex extracts every URL
- For each URL, walks upward in the doc to find the nearest non-empty line as its title
- Normalizes URLs (strips UTM params, fragments, trailing slashes)
- Generates a deterministic `doc_id` from the URL hash — re-runs on the same URL overwrite the same Pinecone vector
- Scrapes each URL with 15s timeout, browser User-Agent
- Extracts article content via CSS selectors (`article`, `main`, `[role=main]`, fallback to body)
- Skips empty/paywalled pages (< 200 chars)
- Embeds with OpenAI `text-embedding-3-large` (3072 dim)
- Upserts to Pinecone namespace `anchor-knowledge-base`
- 2s delay between URLs to respect rate limits

**Why deterministic IDs:** Avoids duplicates when the reading list is edited. Add a new URL? New vector. Edit a URL? New vector (the hash changes). Re-run the workflow without changes? Same hashes, same overwrites — Pinecone storage stays clean.
</details>

<details>
<summary><strong>🤖 Stage 5 · Knowledge Chat Bot</strong></summary>

**Trigger:** Slack Events webhook (`/webhook/slack-events`) — Slack POSTs `message.im` events here

**Key logic:**
1. Handles Slack URL verification challenge (responds with `{challenge}`)
2. Returns 200 OK to Slack within 100ms (Slack's 3-second rule)
3. Filters out: bot's own messages, edits/deletes, non-DM channels, retries
4. **AI Agent** processes the message:
   - System prompt: strict scope (only fintech / BaaS / Anchor / quiz)
   - Memory: per-user, last 20 messages, keyed by `slack-{user_id}`
   - Tool: `knowledge_base` (wraps Pinecone query)
   - Translates Western fintech concepts to Nigerian/African context
   - Refuses off-topic questions politely
5. Posts the reply to the same DM channel, threaded

**Scope enforcement:** The system prompt is strict but not bulletproof against jailbreaking. For an internal HR tool with trusted users, this is sufficient. A future Layer-2 classifier could pre-filter messages for stronger guarantees.
</details>

---

## Things that took a while to figure out

A few hard-won lessons baked into this codebase:

- **n8n's Code node sandbox blocks `crypto`** — including `require('crypto')`. We use vanilla `Math.random()` hex for emp_id generation.
- **n8n's Google Docs node returns plain text, not the structured `body.content[]` tree.** Doc parsing uses regex on the flat string, pairing URLs with the nearest title line above.
- **Slack `users.lookupByEmail` requires a separate scope (`users:read.email`)** — we work around this by calling `users.list` with just `users:read` and filtering by name client-side.
- **`message.im` is NOT an option in n8n's Slack Trigger node** — we use a generic Webhook node and let Slack's Events API filter server-side.
- **The Slack node's `Block Kit` mode is fragile** — for an internal welcome DM, plain text with Slack markdown (`*bold*`, `_italic_`, `\n`) is far more reliable than nested block JSON expressions.
- **Pinecone dimension MUST match the embedding model output exactly** — `text-embedding-3-large` = 3072. Get this wrong and every upsert fails silently.
- **Slack's "messages tab" needs an explicit checkbox** — beyond just enabling the Messages tab, there's a separate "Allow users to send messages" checkbox that defaults to OFF. Easy to miss.
- **n8n Slack messages include "Automated with this n8n workflow" by default** — disable it per-node via `Options → Include Link to Workflow → OFF`.

---

## Roadmap

- [ ] Add a **Layer-2 classifier** before the AI Agent for hard scope guarantees
- [ ] Build a **manager dashboard** showing each new hire's onboarding progress (Streamlit + read-only Sheets API)
- [ ] Add **quiz score tracking** to the Audit Log so HR can see who's engaging with the bot
- [ ] Support **bulk re-ingestion** of the reading list (delete-and-rebuild button)
- [ ] Extend the bot to answer questions about **internal Anchor policies** (separate Pinecone namespace)
- [ ] Add **calendar invite generation** to Stage 2 — schedule the new hire's first-week meetings automatically

---

## Maintenance

### Adding a new article to the reading list

1. Open the Reading List Google Doc
2. Paste the article title, then the URL on the next line
3. Save. Stage 4 will auto-trigger within a few minutes and ingest it
4. The bot can answer questions about the new article within ~5 minutes

### Adding a new HR team member who should receive notifications

1. Open the n8n workflow → search for `roqeeb@getanchor.co` in node parameters
2. Add the new email to the recipient lists (Stage 1 HR notifications, Stage 2 HR success notification)

### Disabling the bot temporarily

In n8n, toggle the workflow OFF. Slack will retry events for ~3 attempts then drop them; the bot becomes silent. Re-activate to resume.

### Rotating credentials

When an API key is rotated (Slack, OpenAI, Pinecone, Google):
1. Update the credential in n8n's Credentials section (the workflow auto-uses the updated value)
2. No workflow re-import needed
3. For Slack token rotation specifically: reinstall the app in your workspace to issue a new `xoxb-` token, then paste it into the existing credential

---

## License

Internal Anchor tooling. Not licensed for external use without permission.

---

## Acknowledgments

- Built for [Anchor Software](https://getanchor.co) — Africa's leading Banking-as-a-Service provider
- Powered by [n8n](https://n8n.io) — fair-code workflow automation
- AI by [OpenAI](https://openai.com) · Vectors by [Pinecone](https://pinecone.io)
- Reading list curated by Anchor's leadership team

---

<p align="center">
  <i>Made with care for every new hire who deserves a smooth first week. 🚀</i>
</p>
