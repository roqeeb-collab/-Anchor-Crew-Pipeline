# Workflows

This folder contains the n8n workflow that powers the Anchor Crew Pipeline.

## Files

| File | Purpose |
|------|---------|
| `anchor-crew-pipeline.json` | The full n8n workflow (all 5 stages merged into one file) |

## Importing

1. Open your n8n instance
2. Click **Workflows** in the left sidebar → **Add Workflow** → **Import from File**
3. Select `anchor-crew-pipeline.json`
4. The workflow loads with all 74 nodes and the 5 sticky note comment boxes

If you already have an active workflow using the same webhook paths (`/webhook/slack-events`, `/webhook/it-setup-confirmed`, `/webhook/hr-decision`), n8n will refuse to activate the new one. Deactivate the old workflow first.

## Credentials to set up

After importing, you'll need to attach **seven** credentials to the relevant nodes:

| Credential type | Used by | Scopes / Setup |
|-----------------|---------|----------------|
| Google Sheets OAuth2 | Stages 1, 2, 3 | `https://www.googleapis.com/auth/spreadsheets` |
| Gmail OAuth2 | Stages 1, 2 | `https://www.googleapis.com/auth/gmail.send` |
| Google Drive OAuth2 | Stage 4 | `https://www.googleapis.com/auth/drive.readonly` |
| Google Docs OAuth2 | Stage 4 | `https://www.googleapis.com/auth/documents.readonly` |
| Slack API (token-based) | Stages 3, 5 | See [`../docs/setup-slack-app.md`](../docs/setup-slack-app.md) |
| OpenAI API | Stages 4, 5 | See [`../docs/setup-openai.md`](../docs/setup-openai.md) |
| Pinecone API | Stages 4, 5 | See [`../docs/setup-pinecone.md`](../docs/setup-pinecone.md) |

## Placeholders inside the JSON

Before importing, you may want to search-and-replace these placeholder strings to match your environment:

| Placeholder | Replace with |
|-------------|--------------|
| `YOUR_SLACK_CREDENTIAL_ID` | Your Slack credential ID after creation |
| `YOUR_OPENAI_CREDENTIAL_ID` | Your OpenAI credential ID |
| `YOUR_PINECONE_CREDENTIAL_ID` | Your Pinecone credential ID |
| `YOUR_PINECONE_INDEX_NAME` | The Pinecone index name (e.g. `anchor-knowledge`) |
| `YOUR_READING_LIST_DOC_ID` | Google Doc ID of your reading list |
| `YOUR_DRIVE_CREDENTIAL_ID` | Google Drive credential ID |
| `YOUR_DOCS_CREDENTIAL_ID` | Google Docs credential ID |

Alternatively, leave the placeholders in place and select credentials from the dropdown in each node's UI after import. This avoids any risk of mismatched IDs.

## Activating

Once credentials are wired up:

1. Toggle the workflow **Active** at the top right
2. The Slack Events webhook URL becomes live at `https://your-n8n-domain.com/webhook/slack-events`
3. Submit that URL to your Slack app's Event Subscriptions (see [`../docs/setup-slack-app.md`](../docs/setup-slack-app.md))
4. Run Stage 4 manually once to seed Pinecone with your reading list

## Testing

The safest end-to-end test:

1. Add yourself as a fake new hire by filling the onboarding form with your own email and a start date of tomorrow
2. Confirm Stage 1 fired (check n8n executions, the Employees sheet, and your inbox for Dotun's email)
3. Click the confirmation button in the test Dotun email — Stage 2 should fire
4. Wait until tomorrow at 9am, or manually set `welcome_due_at` in the past — Stage 3 will DM you
5. Reply to the bot in Slack to test Stage 5

When done, delete the test row from the Employees and Payroll sheets.

## Troubleshooting

For common issues, see the runbooks:

- [Bot not responding](../docs/runbooks/bot-not-responding.md)
- [New hire stuck in pipeline](../docs/runbooks/new-hire-stuck.md)
- [Reading list update didn't ingest](../docs/runbooks/reading-list-update.md)
