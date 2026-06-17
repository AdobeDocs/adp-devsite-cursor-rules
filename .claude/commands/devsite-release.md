Generate a Release PR description for the AdobeDocs/adp-devsite repository by comparing the `stage` branch against `main`. $ARGUMENTS

## Instructions

You are preparing a release for https://github.com/AdobeDocs/adp-devsite. Follow every step below in order.

### Step 1 — Find the commit range

Run these shell commands via Bash:

```bash
# Get the latest common ancestor (last release point)
gh api repos/AdobeDocs/adp-devsite/compare/main...stage \
  --jq '{behind_by: .behind_by, ahead_by: .ahead_by, base_commit: .merge_base_commit.sha}'
```

Then get all commits on `stage` that are not yet on `main`:

```bash
gh api repos/AdobeDocs/adp-devsite/compare/main...stage \
  --jq '[.commits[] | {sha: .sha, message: .commit.message, author: .commit.author.name, date: .commit.author.date}]'
```

### Step 2 — Collect all PRs in the range

For each commit SHA found in Step 1, find the associated merged PR:

```bash
gh api repos/AdobeDocs/adp-devsite/commits/{SHA}/pulls \
  --jq '[.[] | {number: .number, title: .title, body: .body, user: .user.login, html_url: .html_url, base: .base.ref, head: .head.ref, merged_at: .merged_at}]'
```

Deduplicate by PR number (a PR may appear for multiple commits). Only include PRs whose `base.ref` is `stage` (i.e., PRs merged into stage, not PRs merged into other branches). Sort by `merged_at` ascending.

### Step 3 — Retrieve approvers for each PR

For each unique PR number:

```bash
gh api repos/AdobeDocs/adp-devsite/pulls/{PR_NUMBER}/reviews \
  --jq '[.[] | select(.state == "APPROVED") | {user: .user.login, submitted_at: .submitted_at}]'
```

### Step 4 — Extract Jira ticket numbers

Search for Jira ticket IDs matching the pattern `DEVSITE-{digits}` (case-insensitive) in:
- The PR title
- The PR body
- The head branch name (e.g. `devsite-2195`, `DEVSITE-1996`, `feat/devsite-1269-something` all contain valid ticket IDs)

When extracting from a branch name, apply a case-insensitive match and normalize the result to uppercase (e.g. `devsite-2195` → `DEVSITE-2195`).

Collect all unique ticket IDs found for each PR (deduplicated, uppercased).

### Step 5 — Write a short description for each PR

Read the PR title and body. Write a 1–2 sentence neutral description of **what the change does** (not why it was filed). Keep it concise and non-technical enough for a release note audience.

### Step 6 — Compose the Release PR Description

Output the result in the following format. Do not output anything else before the formatted result.

---

## Release — `stage` → `main`

> **Commits ahead:** {ahead_by}  
> **Merge base:** `{base_commit_short_sha}`  
> **Generated:** {today's date}

### Changes included

| # | PR | Title | Jira | Author | Approved by |
|---|-----|-------|------|--------|-------------|
{one row per PR, see format below}

**Row format:**  
`| {i} | [#{number}](https://github.com/AdobeDocs/adp-devsite/pull/{number}) | {title} | {jira tickets comma-separated, or —} | @{author} | @{approvers comma-separated, or —} |`

---

### Descriptions

For each PR (in the same order as the table), output:

#### [{i}] #{number} — {title}

**Jira:** {ticket links or "—"}  
**Author:** @{author}  
**Approved by:** @{approvers or "—"}  
**Merged:** {merged_at date, YYYY-MM-DD}

{1–2 sentence description of what the change does}

---

### Notes

- If any PR has no associated Jira ticket, flag it with a `⚠️ No Jira ticket` note.
- If any PR has no approver, flag it with a `⚠️ No approver on record` note.
- If the `gh` API returns an empty list of commits (branches are in sync), output: "No changes — `stage` and `main` are already in sync."
- If authentication is needed, instruct the user to run `gh auth login` first.
