# Slack PR Notification Workflow

This directory contains the `slack-pr-notify.yml` GitHub Actions workflow, which posts notifications about pull request activity to a Slack channel via an incoming webhook.

## Required Secret

- `SLACK_WEBHOOK_URL` – A Slack incoming webhook URL (looks like `https://hooks.slack.com/services/T.../B.../...`).

Manny will set this up manually.

## How to Create the Slack Webhook

1. In Slack, go to <https://api.slack.com/apps> and create a new app (or choose an existing one) for the workspace.
2. Under **Settings > Incoming Webhooks**, toggle the switch **ON**.
3. Click **Add New Webhook to Workspace**, select the target channel, and click **Allow**.
4. Copy the webhook URL.
5. In the GitHub repository, go to **Settings > Secrets and variables > Actions > New repository secret**.
6. Name the secret exactly `SLACK_WEBHOOK_URL`, paste the copied URL, and click **Add secret**.

## Triggered Events

The workflow runs on the following PR activity:

- Pull request: `opened`, `closed`, `reopened`, `synchronize`, `edited`, `labeled`, `unlabeled`, `review_requested`, `review_request_removed`
- PR comments (`issue_comment` on a PR only): `created`, `edited`, `deleted`
- Pull request reviews: `submitted`, `edited`, `dismissed`
- Pull request review comments: `created`, `edited`, `deleted`

## Color Legend

The Slack attachment color indicates the activity type:

- **Green (`#36a64f`)** – PR opened, or review approved.
- **Yellow (`#daa032`)** – New commits, edited, labeled/unlabeled, review requested/removed, reopened, comments, or review commented.
- **Red (`#ff0000`)** – PR closed, review changes requested/dismissed, or review edited/dismissed.
