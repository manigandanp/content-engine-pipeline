# Slack PR Notification Workflow

`.github/workflows/slack-pr-notify.yml` posts pull-request activity from this repository to a Slack channel using a Slack Incoming Webhook.

## Required Secret

Only one secret is required:

- `SLACK_WEBHOOK_URL` – the URL of a Slack Incoming Webhook.

`GITHUB_TOKEN` is provided automatically by GitHub Actions; you do not need to create it manually.

### How to create and store the Slack webhook

1. Create or open a Slack App:
   - Go to <https://api.slack.com/apps> and click **Create New App** (or choose an existing app).
   - Alternatively, use the legacy **Incoming WebHooks** app in your Slack workspace.
2. Enable Incoming Webhooks:
   - In the app settings, go to **Incoming Webhooks** and toggle the switch to **On**.
   - Click **Add New Webhook to Workspace**, choose the channel where notifications should appear, and click **Allow**.
3. Copy the webhook URL (it looks like `https://hooks.slack.com/services/T.../B.../...`).
4. Add it to this repository:
   - Open the repository on GitHub and go to **Settings → Secrets and variables → Actions**.
   - Click **New repository secret** (or add it as an organization secret if you want to share it across repos).
   - Name the secret exactly `SLACK_WEBHOOK_URL`, paste the copied URL, and click **Add secret**.

## Triggered Events

The workflow runs on the following PR-related activity:

- `pull_request`: `opened`, `closed`, `reopened`, `synchronize`, `edited`
- `pull_request_review`: `submitted`
- `issue_comment`: `created` — but only when the comment is on a pull request
- `pull_request_review_comment`: `created`

The job has an `if` guard that skips plain issue comments (comments on non-PR issues).

## Color Legend

Slack attachment colors indicate the activity type:

- **Green** (`good`) — PR opened, PR merged, or review approved.
- **Yellow** (`warning`) — PR updated with new commits (`synchronize`), PR edited, new comments, or review commented/dismissed.
- **Red** (`danger`) — PR closed without merge, or review changes requested.

## Mentioning Masa

When a PR is opened or updated with new commits, the notification appends the line:

> Masa please review this PR

## Automated Review

Pull requests in this repository also receive an automated review by Masa (Hermes Agent) and require human approval before they can be merged.
