# PR Slack Notifications POC - Implementation Plan

## Overview

This is a Proof of Concept for PR Slack notifications using GitHub Actions + Slack Bot API.
The goal is to test the workflow from minimal (channel post) to complex (user mentions via email lookup).

## Prerequisites

- GitHub repo (this one)
- Personal Slack workspace with the same email as GitHub account
- Slack App with Bot Token (see Phase 1)

## Test User

- GitHub username: `ivanvgh`
- Same user will act as PR author and reviewer

---

## Phase 1: Slack App Setup

### 1.1 Create Slack App
- Go to https://api.slack.com/apps
- Create New App → From scratch
- Name: `PR Notifications Bot`
- Workspace: Select personal workspace

### 1.2 Add Bot Scopes
Navigate to OAuth & Permissions → Bot Token Scopes, add:
- `chat:write`
- `chat:write.public`
- `users:read.email`

### 1.3 Install to Workspace
- Click "Install to Workspace"
- Copy Bot User OAuth Token (`xoxb-...`)

### 1.4 Add GitHub Secret
- Repo → Settings → Secrets and variables → Actions
- New secret: `SLACK_BOT_TOKEN` = `xoxb-...`

### Validation
- [ ] Slack App created
- [ ] Scopes added
- [ ] App installed to workspace
- [ ] `SLACK_BOT_TOKEN` secret added to repo

---

## Phase 2: Minimal Notification (No Mentions)

### Goal
Post a message to a Slack channel when a PR is opened. No user lookup, just channel post.

### 2.1 Create Workflow
Create `.github/workflows/pr-ready-for-review.yml`:

```yaml
name: PR Ready for Review

on:
  pull_request:
    types: [opened, ready_for_review]

env:
  SLACK_CHANNEL: "#general"

jobs:
  notify:
    name: Notify Ready for Review
    runs-on: ubuntu-latest
    if: github.event.pull_request.draft == false

    steps:
      - name: Send Slack notification
        uses: slackapi/slack-github-action@v2.0.0
        with:
          method: chat.postMessage
          token: ${{ secrets.SLACK_BOT_TOKEN }}
          payload: |
            {
              "channel": "${{ env.SLACK_CHANNEL }}",
              "text": "🆕 New PR: #${{ github.event.pull_request.number }} - ${{ github.event.pull_request.title }} by ${{ github.event.pull_request.user.login }}"
            }
```

### 2.2 Test

1. Commit and push workflow to main
2. Create branch test-phase-2
3. Add a test file, commit, push
4. Open PR (non-draft)
5. Check Slack #general for message

### Validation

- [ ] Workflow runs successfully
- [ ] Message appears in Slack channel
- [ ] PR title and author shown

---

## Phase 3: Rich Message Format

### Goal

Improve message with blocks, buttons, and better formatting.

### 3.1 Update Workflow

Replace the payload in `.github/workflows/pr-ready-for-review.yml`:

```yaml
      - name: Send Slack notification
        uses: slackapi/slack-github-action@v2.0.0
        with:
          method: chat.postMessage
          token: ${{ secrets.SLACK_BOT_TOKEN }}
          payload: |
            {
              "channel": "${{ env.SLACK_CHANNEL }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "🆕 *New PR Ready for Review*\n<${{ github.event.pull_request.html_url }}|#${{ github.event.pull_request.number }} - ${{ github.event.pull_request.title }}>"
                  }
                },
                {
                  "type": "context",
                  "elements": [
                    {
                      "type": "mrkdwn",
                      "text": "👤 *Author:* @${{ github.event.pull_request.user.login }} | 🌿 *Branch:* `${{ github.event.pull_request.head.ref }}`"
                    }
                  ]
                },
                {
                  "type": "actions",
                  "elements": [
                    {
                      "type": "button",
                      "text": { "type": "plain_text", "text": "Review PR" },
                      "url": "${{ github.event.pull_request.html_url }}/files"
                    }
                  ]
                }
              ]
            }
```

### 3.2 Test

1. Create branch test-phase-3
2. Open PR
3. Verify rich formatting in Slack

### Validation

- [ ] Message has clickable PR link
- [ ] "Review PR" button works
- [ ] Branch name displayed

---

## Phase 4: User Mentions via Email Lookup

### Goal

Look up Slack user ID by email and create real @mention that pings the user.

### Prerequisite

- GitHub email must be PUBLIC: GitHub → Settings → Public profile → Public email
- Slack account uses the same email

### 4.1 Update Workflow

Add email lookup step before notification:

```yaml
name: PR Ready for Review

on:
  pull_request:
    types: [opened, ready_for_review]

env:
  SLACK_CHANNEL: "#general"

jobs:
  notify:
    name: Notify Ready for Review
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read
    if: github.event.pull_request.draft == false

    steps:
      - name: Get Slack user ID from email
        id: slack-user
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
        run: |
          GITHUB_EMAIL=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/users/${{ github.event.pull_request.user.login }} | jq -r '.email // empty')

          echo "GitHub email: $GITHUB_EMAIL"

          if [ -n "$GITHUB_EMAIL" ]; then
            RESPONSE=$(curl -s -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
              "https://slack.com/api/users.lookupByEmail?email=$GITHUB_EMAIL")

            SLACK_ID=$(echo "$RESPONSE" | jq -r '.user.id // empty')

            if [ -n "$SLACK_ID" ] && [ "$SLACK_ID" != "null" ]; then
              echo "mention=<@$SLACK_ID>" >> $GITHUB_OUTPUT
              exit 0
            fi
          fi

          echo "mention=@${{ github.event.pull_request.user.login }}" >> $GITHUB_OUTPUT

      - name: Send Slack notification
        uses: slackapi/slack-github-action@v2.0.0
        with:
          method: chat.postMessage
          token: ${{ secrets.SLACK_BOT_TOKEN }}
          payload: |
            {
              "channel": "${{ env.SLACK_CHANNEL }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "🆕 *New PR Ready for Review*\n<${{ github.event.pull_request.html_url }}|#${{ github.event.pull_request.number }} - ${{ github.event.pull_request.title }}>"
                  }
                },
                {
                  "type": "context",
                  "elements": [
                    {
                      "type": "mrkdwn",
                      "text": "👤 *Author:* ${{ steps.slack-user.outputs.mention }} | 🌿 *Branch:* `${{ github.event.pull_request.head.ref }}`"
                    }
                  ]
                },
                {
                  "type": "actions",
                  "elements": [
                    {
                      "type": "button",
                      "text": { "type": "plain_text", "text": "Review PR" },
                      "url": "${{ github.event.pull_request.html_url }}/files"
                    }
                  ]
                }
              ]
            }
```

### 4.2 Test

1. Create branch test-phase-4
2. Open PR
3. Verify you get actually pinged in Slack (not just text)

### Validation

- [ ] Email lookup succeeds (check workflow logs)
- [ ] Slack mention is clickable and shows user profile
- [ ] User receives Slack notification ping

---

## Phase 5: Labels for Duplicate Prevention

### Goal

Add labels to prevent duplicate notifications.

### 5.1 Update Workflow

Add label checks and management:

```yaml
name: PR Ready for Review

on:
  pull_request:
    types: [opened, ready_for_review]

env:
  SLACK_CHANNEL: "#general"

jobs:
  notify:
    name: Notify Ready for Review
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read
    if: github.event.pull_request.draft == false

    steps:
      - name: Check if already notified
        id: check
        run: |
          LABELS='${{ toJson(github.event.pull_request.labels.*.name) }}'
          if echo "$LABELS" | grep -q "notified:ready_for_review"; then
            echo "skip=true" >> $GITHUB_OUTPUT
          else
            echo "skip=false" >> $GITHUB_OUTPUT
          fi

      - name: Get Slack user ID from email
        if: steps.check.outputs.skip == 'false'
        id: slack-user
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
        run: |
          GITHUB_EMAIL=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/users/${{ github.event.pull_request.user.login }} | jq -r '.email // empty')

          if [ -n "$GITHUB_EMAIL" ]; then
            RESPONSE=$(curl -s -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
              "https://slack.com/api/users.lookupByEmail?email=$GITHUB_EMAIL")
            SLACK_ID=$(echo "$RESPONSE" | jq -r '.user.id // empty')
            if [ -n "$SLACK_ID" ] && [ "$SLACK_ID" != "null" ]; then
              echo "mention=<@$SLACK_ID>" >> $GITHUB_OUTPUT
              exit 0
            fi
          fi
          echo "mention=@${{ github.event.pull_request.user.login }}" >> $GITHUB_OUTPUT

      - name: Add state label
        if: steps.check.outputs.skip == 'false'
        run: gh pr edit ${{ github.event.pull_request.number }} --add-label "state:ready_for_review"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Send Slack notification
        if: steps.check.outputs.skip == 'false'
        continue-on-error: true
        id: slack-notify
        uses: slackapi/slack-github-action@v2.0.0
        with:
          method: chat.postMessage
          token: ${{ secrets.SLACK_BOT_TOKEN }}
          payload: |
            {
              "channel": "${{ env.SLACK_CHANNEL }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "🆕 *New PR Ready for Review*\n<${{ github.event.pull_request.html_url }}|#${{ github.event.pull_request.number }} - ${{ github.event.pull_request.title }}>"
                  }
                },
                {
                  "type": "context",
                  "elements": [
                    {
                      "type": "mrkdwn",
                      "text": "👤 *Author:* ${{ steps.slack-user.outputs.mention }} | 🌿 *Branch:* `${{ github.event.pull_request.head.ref }}`"
                    }
                  ]
                },
                {
                  "type": "actions",
                  "elements": [
                    {
                      "type": "button",
                      "text": { "type": "plain_text", "text": "Review PR" },
                      "url": "${{ github.event.pull_request.html_url }}/files"
                    }
                  ]
                }
              ]
            }

      - name: Add notified label
        if: steps.check.outputs.skip == 'false' && steps.slack-notify.outcome == 'success'
        run: gh pr edit ${{ github.event.pull_request.number }} --add-label "notified:ready_for_review"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 5.2 Test

1. Create branch test-phase-5
2. Open PR → should notify + add labels
3. Close and reopen PR → should NOT notify again
4. Manually remove notified:ready_for_review label
5. Close and reopen → should notify again

### Validation

- [ ] Labels added on first notification
- [ ] No duplicate notification on reopen
- [ ] Removing label allows re-notification

---

## Phase 6: Draft Conversion Handling

### Goal

Remove labels when PR is converted to draft, allowing re-notification when ready again.

### 6.1 Create New Workflow

Create `.github/workflows/pr-converted-to-draft.yml`:

```yaml
name: PR Converted to Draft

on:
  pull_request:
    types: [converted_to_draft]

jobs:
  cleanup:
    name: Remove Ready State
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write

    steps:
      - name: Remove ready for review labels
        run: |
          gh pr edit ${{ github.event.pull_request.number }} \
            --remove-label "state:ready_for_review" \
            --remove-label "notified:ready_for_review" || true
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 6.2 Test

1. Create branch test-phase-6
2. Open PR as draft
3. Convert to ready → should notify + add labels
4. Convert back to draft → labels removed
5. Convert to ready again → should notify again

### Validation

- [ ] Labels removed on draft conversion
- [ ] Re-notification works after draft → ready → draft → ready

---

## Summary

| Phase | Feature                         | Complexity |
|-------|--------------------------------|------------|
| 1     | Slack App setup                | Setup only |
| 2     | Minimal channel post           | Low        |
| 3     | Rich message format            | Low        |
| 4     | User mentions via email        | Medium     |
| 5     | Labels for duplicate prevention| Medium     |
| 6     | Draft conversion handling      | Low        |

## Resources

- Slack GitHub Action docs: https://github.com/slackapi/slack-github-action
- Slack API method (Bot): https://github.com/slackapi/slack-github-action?tab=readme-ov-file#technique-2-slack-api-method
- Slack Block Kit Builder: https://app.slack.com/block-kit-builder
