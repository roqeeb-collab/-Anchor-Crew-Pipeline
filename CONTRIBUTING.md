# Contributing to the Anchor Crew Pipeline

This is an internal tool, but if you're an Anchor teammate planning to make changes — here's how to do it cleanly.

## Before you change anything

1. **Talk to the current owner.** This is a critical path for every new hire. A bug here means someone has a bad first week
2. **Back up the current workflow.** Download a copy of `anchor-crew-pipeline.json` from n8n before modifying anything
3. **Test in a staging n8n if possible.** If you only have one n8n instance, at least disable the workflow's auto-trigger while testing

## Making changes to the workflow

The recommended flow:

1. Export the current workflow from n8n as JSON
2. Make your changes in n8n's visual editor
3. Test by manually executing individual nodes
4. When satisfied, export the workflow again and replace `workflows/anchor-crew-pipeline.json`
5. Commit with a clear message: `feat(stage-5): add layer 2 classifier` or `fix(stage-3): handle compound usernames`
6. Open a PR

## What goes in a good commit

- **Stage prefix:** `stage-1`, `stage-2`, etc., or `cross-stage` for changes affecting multiple
- **Verb:** `feat`, `fix`, `docs`, `refactor`, `chore`
- **Example:** `fix(stage-3): handle compound Slack usernames like 'tosinsobogun'`

## Updating the reading list

The reading list lives in a separate Google Doc, not in this repo. To add an article:

1. Open the Reading List Google Doc
2. Add the title on one line, the URL on the next
3. Save — Stage 4 will auto-ingest within a few minutes
4. (Optional) Comment on the doc explaining why you added it, so future readers have context

You don't need to commit anything to this repo to update the reading list.

## Updating documentation

Docs live in `/docs`. Same Git flow as code:

1. Edit the markdown
2. Preview it on GitHub before pushing if you've used any tricky syntax
3. Open a PR

For runbooks specifically, please test the steps yourself before adding them — a runbook that gets the steps wrong is worse than no runbook.

## Updating credentials

**Don't commit credentials.** Ever. They live in n8n's encrypted credential store, not in this repo.

If you rotate a key:

1. Update it in n8n's Credentials section
2. The workflow auto-picks it up — no JSON change needed
3. If for some reason you change the credential's name or ID, you'll need to update the workflow node references — but normally you can just edit the existing credential in place

## Questions

Ask in #people-ops or DM the current pipeline owner.
