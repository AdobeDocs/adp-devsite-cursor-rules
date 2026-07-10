Upgrade one or more general content repos from the v2 lint workflow to the v3 `adp-devsite-workflow` pipeline, using `adp-devsite-scripts/update-bot`. $ARGUMENTS

## Context

- Repos are split into two groups: most **general content repos** still run the legacy v2 setup, where `lint.yml` is embedded directly in the content repo. **Commerce team** content repos have already migrated to v3.
- The target v3 architecture is defined in [adp-devsite-workflow/validate-pr.puml](https://github.com/AdobeDocs/adp-devsite-workflow/blob/main/validate-pr.puml): a content repo's `pr-validation.yml` calls the shared `validate-pr-v3.yml` (which calls `lint-v3.yml`), and its `pr-comment.yml` calls the shared `pr-comment-v3.yml` on `workflow_run` completion — replacing the old self-contained `lint.yml` / `post-lint-comment.yml`.
- The reference migration is [AdobeDocs/dev-docs-template PR #60](https://github.com/AdobeDocs/dev-docs-template/pull/60/files):
  - **Delete** `.github/workflows/lint.yml`
  - **Delete** `.github/workflows/post-lint-comment.yml`
  - **Add** `.github/workflows/pr-validation.yml` (calls `AdobeDocs/adp-devsite-workflow/.github/workflows/validate-pr-v3.yml@main`)
  - **Add** `.github/workflows/pr-comment.yml` (calls `AdobeDocs/adp-devsite-workflow/.github/workflows/pr-comment-v3.yml@main`)
- This command drives `adp-devsite-scripts/update-bot`, which pushes the above changes as a branch + PR to every repo listed in its `repos.json`, fetching the `add` files from `AdobeDocs/dev-docs-template@main`.
- The repos to upgrade are **not** auto-discovered — they must be supplied in `$ARGUMENTS` (GitHub URLs or `owner/repo` shorthand, one or more, space/comma/newline separated). If `$ARGUMENTS` is empty, ask the user for the repo link(s) before doing anything else.

## Instructions

Work inside `/Users/yuxuanj/git/ADP/adp-devsite-scripts/update-bot`. Follow every step in order.

### Step 0 — Parse target repos

Parse `$ARGUMENTS` into a list of `{ "owner": "...", "repo": "..." }` pairs (strip `https://github.com/`, trailing `.git`, trailing slashes, `/pull/...` suffixes, etc). If nothing parses, stop and ask the user for valid repo links — do not guess or invent repos.

### Step 1 — Check the GitHub token

The bot loads `.env` from its own directory (`update-bot/.env`, via `dotenv/config`, since `npm start` runs with that directory as cwd) — a token elsewhere (shell env, the top-level `ADP/.env`) will **not** be picked up automatically.

1. Check whether `update-bot/.env` exists and contains a `GITHUB_TOKEN` that is not the `.env.example` placeholder (`ghp_your_personal_access_token_here`) and not empty.
2. If missing or a placeholder:
   - Check if a usable `GITHUB_TOKEN` exists elsewhere (e.g. top-level `/Users/yuxuanj/git/ADP/.env`, or `$GITHUB_TOKEN` in the shell).
   - If found, tell the user you found one and ask before copying it into `update-bot/.env` (writing credentials to a new file is worth a quick confirmation).
   - If nothing usable is found anywhere, stop and tell the user to create `update-bot/.env` with a token that has **repo** (and **workflow**) scope, per [update-bot/README.md](../../adp-devsite-scripts/update-bot/README.md) — do not proceed further.
3. Never print the token value itself in any output.

### Step 2 — Update `file-mappings.json`

**Prerequisite:** the bot fetches `add` files from `dev-docs-template@main` (hardcoded `TEMPLATE_REF` in [src/index.js](../../adp-devsite-scripts/update-bot/src/index.js)), and `getRawContent` throws on a non-200 response. `pr-validation.yml` / `pr-comment.yml` currently only exist on the `v3-workflow-default` branch behind [PR #60](https://github.com/AdobeDocs/dev-docs-template/pull/60) (not yet merged) — running the bot before that PR merges into `main` will 404 and crash the whole run before any repo is processed. Check this first:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://raw.githubusercontent.com/AdobeDocs/dev-docs-template/main/.github/workflows/pr-validation.yml
```

If this is not `200`, stop and tell the user PR #60 needs to merge first — do not proceed to Step 3.

Overwrite `/Users/yuxuanj/git/ADP/adp-devsite-scripts/update-bot/file-mappings.json` with exactly:

```json
[
  { "action": "delete", "path": ".github/workflows/lint.yml" },
  { "action": "delete", "path": ".github/workflows/post-lint-comment.yml" },
  { "action": "add", "path": ".github/workflows/pr-validation.yml" },
  { "action": "add", "path": ".github/workflows/pr-comment.yml" }
]
```

Deletes are skipped harmlessly (with a warning) on repos where a file doesn't exist, so this is safe to run even if a target repo's v2 setup differs slightly.

If any target repo is under `AdobeDocsPrivate` (private), re-check the current contents of `dev-docs-template` for `-private` variants of `pr-validation.yml` / `pr-comment.yml` before assuming no `destPath` is needed — the README's private/public convention only documents this for deploy/stage/build workflows, not these two, so don't assume silently.

### Step 3 — Update `repos.json`

Overwrite `/Users/yuxuanj/git/ADP/adp-devsite-scripts/update-bot/repos.json` with the repo list parsed in Step 0, in the documented format:

```json
[
  { "owner": "AdobeDocs", "repo": "example-repo" }
]
```

### Step 4 — Confirm before pushing

`npm start` will push a commit and open a real PR on GitHub against every listed repo. Show the user:
- The final `repos.json` contents (which repos will be touched)
- The final `file-mappings.json` contents (what will change)

and get explicit confirmation before running the bot — this is a shared/external effect, not a local, reversible one.

### Step 5 — Run the bot

```bash
cd /Users/yuxuanj/git/ADP/adp-devsite-scripts/update-bot
[ -d node_modules ] || npm install
npm start
```

### Step 6 — Report results

Summarize the bot's printed output: which repos got a PR created, which had an existing PR updated, which were skipped (already up to date), and any warnings or failures — with PR links. Note that each resulting PR should look like [dev-docs-template PR #60](https://github.com/AdobeDocs/dev-docs-template/pull/60/files) (lint.yml + post-lint-comment.yml removed, pr-validation.yml + pr-comment.yml added). If a PR's diff doesn't match that shape, flag it instead of declaring success.
