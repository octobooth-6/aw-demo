---
emoji: 🏷️
description: Triages new issues by labelling type and priority, detecting duplicates, asking clarifying questions, and assigning to the right team members.
on:
  issues:
    types: [opened]
  roles: all
permissions:
  contents: read
  issues: read
  pull-requests: read
  copilot-requests: write
tools:
  github:
    mode: gh-proxy
    toolsets: [default]
steps:
  - name: Fetch recent issues for duplicate detection
    run: |
      mkdir -p /tmp/gh-aw/triage
      REPO="${{ github.repository }}"
      ISSUE_NUMBER="${{ github.event.issue.number }}"

      # Fetch the triggering issue
      gh api "repos/$REPO/issues/$ISSUE_NUMBER" \
        --jq '{number, title, body, labels: [.labels[].name], user: .user.login}' \
        > /tmp/gh-aw/triage/issue.json

      # Fetch the 100 most recently updated open issues (excluding the current one) for duplicate search
      gh api "repos/$REPO/issues?state=open&per_page=100&sort=updated&direction=desc" \
        --jq --argjson n "$ISSUE_NUMBER" \
        '[.[] | select(.pull_request == null and .number != $n) | {number, title, body: (.body // "" | .[0:300]), labels: [.labels[].name], url: .html_url}]' \
        > /tmp/gh-aw/triage/recent_issues.json

      # Fetch repo collaborators for assignee suggestions (triage role and above)
      gh api "repos/$REPO/collaborators?permission=triage&per_page=30" \
        --jq '[.[] | {login, permissions}]' \
        > /tmp/gh-aw/triage/collaborators.json
    env:
      GH_TOKEN: ${{ github.token }}
safe-outputs:
  add-labels:
    allowed:
      - bug
      - enhancement
      - question
      - documentation
      - duplicate
      - needs-clarification
      - priority:critical
      - priority:high
      - priority:medium
      - priority:low
    max: 6
  add-comment:
    max: 1
    hide-older-comments: true
  update-issue:
    max: 1
  close-issue:
    state-reason: duplicate
    max: 1
network:
  allowed:
    - github
---

# Issue Triage

## Context

- **New issue:** read `/tmp/gh-aw/triage/issue.json`
- **Recent open issues:** read `/tmp/gh-aw/triage/recent_issues.json`
- **Available collaborators:** read `/tmp/gh-aw/triage/collaborators.json`

## Task

Triage the new issue by performing all of the following steps in order.

### 1. Duplicate detection

Compare the new issue's title and body against the recent open issues list. An issue is a duplicate when it describes the same problem or request as an existing open issue (allowing for paraphrase or minor wording differences).

- If a duplicate is found: post a comment linking to the existing issue, close this issue as a duplicate (`state-reason: duplicate`, set `duplicate_of` to the existing issue number), and add the `duplicate` label. Then stop — skip the remaining steps.

### 2. Classify type

Assign exactly one type label based on the issue content:

| Label | When to use |
|---|---|
| `bug` | Reports a defect, crash, or unexpected behaviour |
| `enhancement` | Requests a new feature or improvement |
| `question` | Asks how something works or seeks guidance |
| `documentation` | Relates to docs, README, or examples |

### 3. Classify priority

Assign exactly one priority label using these criteria:

| Label | Criteria |
|---|---|
| `priority:critical` | Data loss, security vulnerability, or complete outage |
| `priority:high` | Core functionality broken; no workaround |
| `priority:medium` | Significant impact with a viable workaround |
| `priority:low` | Minor inconvenience, cosmetic issue, or nice-to-have |

### 4. Clarifying questions

If the issue body is missing key information needed to reproduce a bug or understand the request (e.g. no steps to reproduce, no expected vs actual behaviour, no context for a feature request), post a single comment asking concise, specific questions. Add the `needs-clarification` label alongside the type and priority labels.

Skip this step when the issue is sufficiently detailed.

### 5. Assign to team member

From the collaborators list, assign the issue to the most appropriate person based on:

- The issue type and any area mentioned (e.g. frontend, backend, infra, docs)
- The collaborator's permissions (prefer `maintain` or `write`)
- If no suitable assignee can be determined, do not assign anyone

## Safe Outputs

- Use `add-labels` to apply type, priority, and any other relevant labels.
- Use `add-comment` when posting clarifying questions or a duplicate notice.
- Use `update-issue` (assignees only) to assign the issue to a team member.
- Use `close-issue` (with `state-reason: duplicate`) only when a clear duplicate is found.
- Call `noop` with a brief explanation if the issue is already fully labelled and assigned and no action is needed.
