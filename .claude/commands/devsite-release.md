Generate a Release PR description for the AdobeDocs/adp-devsite repository by comparing the `stage` branch against `main`. $ARGUMENTS

## Instructions

You are preparing a release for https://github.com/AdobeDocs/adp-devsite. Follow every step below in order.

### Step 0 — Authenticate with GitHub

Before making any `gh` API calls, ensure the CLI is authenticated. Use this priority order:

1. **Check if `gh` is already authenticated:**
```bash
gh auth status
```
If already logged in, proceed to Step 1.

2. **Check for a `GITHUB_TOKEN` in the environment or `.env` file at the working directory root:**
```bash
# Check shell environment first
echo $GITHUB_TOKEN

# If empty, read from .env in the working directory root
if [ -f .env ]; then
  export GITHUB_TOKEN=$(grep -E '^GITHUB_TOKEN=' .env | cut -d= -f2-)
fi
```

3. **If a token was found, log in with it:**
```bash
gh auth login --with-token <<< "$GITHUB_TOKEN"
```

4. **If no token is available anywhere**, instruct the user to either:
   - Run `gh auth login` interactively, or
   - Add `GITHUB_TOKEN=<token>` to a `.env` file in the working directory root.

Do not proceed to Step 1 until `gh auth status` confirms a successful login.

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

### Step 6 — Log the release to dev-docs-reference

Using the branch name derived from the release date (same convention as the release file, e.g. `Jun-17-release`), do the following against the `AdobeDocs/dev-docs-reference` repository:

**6a — Create the branch from main (or reuse if it already exists):**

```bash
# Get the current SHA of main
MAIN_SHA=$(gh api repos/AdobeDocs/dev-docs-reference/git/ref/heads/main --jq '.object.sha')

# Try to create the branch; if it already exists (HTTP 422), that's fine — continue
gh api repos/AdobeDocs/dev-docs-reference/git/refs \
  --method POST \
  --field ref="refs/heads/{branch-name}" \
  --field sha="$MAIN_SHA" 2>&1 | grep -qiE '"Reference already exists"' \
  && echo "Branch already exists — will update existing branch" \
  || echo "Branch created"
```

If the branch already exists, do **not** reset it to main — fetch its current file content in 6b and overwrite just the file in 6d.

**6b — Fetch the current content of the changelog page:**

```bash
gh api repos/AdobeDocs/dev-docs-reference/contents/src/pages/eds-release/index.md \
  --jq '{sha: .sha, content: .content}'
```

Decode the base64 `content` field to get the current file text.

**6c — Prepend a new release section:**

After any YAML front matter (lines between the opening and closing `---`), insert the following block at the top of the file body:

```
## {M}/{D}/{YY} EDS Release:

{For each PR, one bullet using this format:
`- **{Feat|Fix}:** {short phrase describing what the change does}{optional inline Jira link(s)}`

Rules:
- Derive type from the PR title prefix: `feat` → **Feat**, `fix` → **Fix**, anything else (chore, refactor, docs, etc.) → **Fix**.
- Keep the description short — a phrase, not a full sentence (match the tone of existing entries in the file).
- If the PR has one or more Jira tickets, append them inline after the description, space-separated: ` [DEVSITE-XXXX](https://jira.corp.adobe.com/browse/DEVSITE-XXXX)`. Multiple tickets go on the same line.
- If the PR has no Jira ticket, omit the link entirely — do not add a placeholder.
- Do NOT include PR numbers, PR links, or author names in the bullets.
}

```

Keep all existing content below unchanged.

**6d — Commit the updated file to the branch:**

Base64-encode the new file content, then:

```bash
gh api repos/AdobeDocs/dev-docs-reference/contents/src/pages/eds-release/index.md \
  --method PUT \
  --field message="chore: add {M}/{D}/{YY} release notes" \
  --field content="{base64-encoded-new-content}" \
  --field sha="{file-sha-from-6b}" \
  --field branch="{branch-name}"
```

**6e — Open a PR to main (or reuse if one already exists):**

```bash
# Check for an existing open PR from this branch
EXISTING_PR=$(gh api repos/AdobeDocs/dev-docs-reference/pulls \
  -f state=open \
  --jq "[.[] | select(.head.ref == \"{branch-name}\")] | first | .html_url // empty")

if [ -n "$EXISTING_PR" ]; then
  echo "PR already exists: $EXISTING_PR"
  PR_URL="$EXISTING_PR"
else
  PR_URL=$(gh api repos/AdobeDocs/dev-docs-reference/pulls \
    --method POST \
    --field title="{M}/{D}/{YY} EDS Release" \
    --field body="Adds release notes for the {M}/{D}/{YY} adp-devsite EDS release." \
    --field head="{branch-name}" \
    --field base="main" \
    --jq '.html_url')
  echo "PR created: $PR_URL"
fi
```

Save `$PR_URL` — it will be appended to the adp-devsite release description in the next step.

### Step 7 — Compose the Release PR Description

Write the result to a markdown file at the root of the working directory. Name the file using the current date formatted as `Mon-D-release.md` (e.g. `Jun-17-release.md`, `Sep-1-release.md`). Then output the same content to the user.

Use the following format:

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

---

**dev-docs-reference PR:** {PR URL from Step 6e}
