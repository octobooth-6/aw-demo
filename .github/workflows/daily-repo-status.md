---
emoji: 📊
description: Opens a daily GitHub issue summarising the last 24 hours of repository activity — commits, PRs, and issues.
on:
  schedule: daily
  workflow_dispatch:
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
  - name: Fetch repository activity
    run: |
      mkdir -p /tmp/gh-aw/data
      REPO="${{ github.repository }}"
      SINCE=$(date -u -d "@$(($(date -u +%s) - 86400))" '+%Y-%m-%dT%H:%M:%SZ')
      TODAY=$(date -u '+%Y-%m-%d')
      echo "$TODAY" > /tmp/gh-aw/data/today.txt

      # Commits
      gh api "repos/$REPO/commits?since=$SINCE&per_page=100" \
        --jq '[.[] | {sha: .sha[0:7], url: .html_url, message: (.commit.message | split("\n")[0]), author: (.author.login // .commit.author.name)}]' \
        > /tmp/gh-aw/data/commits.json

      # PRs opened in last 24 h
      gh api "repos/$REPO/pulls?state=all&sort=created&direction=desc&per_page=100" \
        --jq --arg s "$SINCE" '[.[] | select(.created_at >= $s) | {number, title, url: .html_url, author: .user.login, state, merged_at}]' \
        > /tmp/gh-aw/data/prs_opened.json

      # PRs merged in last 24 h
      gh api "repos/$REPO/pulls?state=closed&sort=updated&direction=desc&per_page=100" \
        --jq --arg s "$SINCE" '[.[] | select(.merged_at != null and .merged_at >= $s) | {number, title, url: .html_url, author: .user.login}]' \
        > /tmp/gh-aw/data/prs_merged.json

      # PRs closed (unmerged) in last 24 h
      gh api "repos/$REPO/pulls?state=closed&sort=updated&direction=desc&per_page=100" \
        --jq --arg s "$SINCE" '[.[] | select(.merged_at == null and .closed_at >= $s) | {number, title, url: .html_url, author: .user.login}]' \
        > /tmp/gh-aw/data/prs_closed.json

      # Issues opened and closed in last 24 h
      gh api "repos/$REPO/issues?state=all&sort=updated&direction=desc&per_page=100&since=$SINCE" \
        --jq --arg s "$SINCE" '[.[] | select(.pull_request == null) | {number, title, url: .html_url, author: .user.login, state, created_at, closed_at}]' \
        > /tmp/gh-aw/data/issues.json
    env:
      GH_TOKEN: ${{ github.token }}
safe-outputs:
  create-issue:
network:
  allowed:
    - github
---

# Daily Repo Status

## Task

Today's date is in `/tmp/gh-aw/data/today.txt`. Read all pre-fetched data from `/tmp/gh-aw/data/` and create one GitHub issue that summarises the last 24 hours of activity in this repository.

**Data files:**
- `commits.json` — commits pushed
- `prs_opened.json` — PRs opened
- `prs_merged.json` — PRs merged
- `prs_closed.json` — PRs closed without merging
- `issues.json` — all issues touched; filter `created_at >= 24 h ago` for opened, `state == "closed"` and `closed_at >= 24 h ago` for closed

**Issue title:** `📊 Daily Status — <TODAY>`

**Issue body format:**

```
## 📊 Daily Status — <TODAY>

> Last 24 hours · `<repo>`

---

### 🔀 Pull Requests

**Opened:** N | **Merged:** N | **Closed:** N

<bullet list: - [#N](url) Title (@author)>

---

### 🐛 Issues

**Opened:** N | **Closed:** N

<bullet list: - [#N](url) Title (@author)>

---

### 📝 Commits

**Total:** N

<bullet list: - [`sha`](url) Message — @author>

---

_Nothing to report in a section → write "Nothing to report."_
```

Add the label `daily-report` to the issue (create it with colour `#0075ca` if it does not exist).

## Safe Outputs

- Use `create-issue` to post the daily status report.
- Call `noop` with reason "no activity in the last 24 hours" if all data files contain empty arrays.
