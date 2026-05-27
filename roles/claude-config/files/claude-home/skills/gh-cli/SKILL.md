---
name: gh-cli
description: Using the GitHub CLI (gh) for PRs, issues, runs, releases, and API calls. TRIGGER when the user wants to inspect or modify GitHub state — open/list/merge PRs, comment on issues, watch CI runs, fetch raw API data, manage repos, or hits gh CLI errors. Covers PR review vs comment distinction, run watching with proper exit codes, JSON output for parsing, and recovery from common auth / scope issues.
---

# gh-cli

`gh` covers ~80% of common GitHub workflows. Most failures come from confusing similar commands (e.g. PR comment vs review comment) or from forgetting `--json` for machine-readable output.

## Authentication Sanity Check

```bash
gh auth status                      # who am I + which scopes
gh auth refresh -s repo,workflow    # add scopes if needed
gh auth login                       # re-auth from scratch (browser flow)
```

If commands fail with `HTTP 401` / `requires scope`, run `gh auth refresh` first.

## Pull Requests

### Listing
```bash
gh pr list                                  # all open PRs in current repo
gh pr list --author "@me" --state all       # mine, any state
gh pr list --search "is:open is:draft"      # GitHub search syntax
gh pr list --json number,title,state --limit 50    # machine-readable
```

### Reading
```bash
gh pr view 123                              # body, files, conversation
gh pr view 123 --comments                   # include all comments
gh pr view 123 --json title,body,files,reviewDecision,statusCheckRollup
gh pr diff 123                              # the patch
gh pr checkout 123                          # checkout to local branch
```

### Creating
```bash
# From current branch — title and body required (don't rely on default)
gh pr create --title "fix: foo" --body "$(cat <<'EOF'
## Summary
- Did X
- Fixed Y

## Test plan
- [ ] manual smoke
- [ ] CI green
EOF
)"

# Draft PR
gh pr create --draft --title "..." --body "..."
```

### Comments vs Reviews — IMPORTANT distinction
GitHub has **three** kinds of "comments":
1. **Issue/PR conversation comment** (general discussion at bottom)
2. **Review comment** (with approval/changes requested)
3. **Inline review comment** (on a specific line of a specific file)

```bash
# (1) Conversation comment on a PR (= same as on an issue)
gh pr comment 123 --body "LGTM!"

# (2) Submit a review (with optional verdict)
gh pr review 123 --approve --body "ship it"
gh pr review 123 --request-changes --body "needs X"
gh pr review 123 --comment --body "non-blocking thoughts"

# (3) Inline comment on specific line — needs the API endpoint
gh api repos/OWNER/REPO/pulls/123/comments \
  -f body="this loop is O(n²)" \
  -f commit_id="$(gh pr view 123 --json headRefOid -q .headRefOid)" \
  -f path="src/foo.py" \
  -F line=42 -f side="RIGHT"
```

Common mistake: using `gh pr comment` when you mean to leave a review verdict, or using `gh api` when `gh pr review` would do.

### Merging
```bash
gh pr merge 123 --squash                    # most common
gh pr merge 123 --merge                     # merge commit
gh pr merge 123 --rebase                    # rebase merge
gh pr merge 123 --auto --squash             # auto-merge when checks pass
```

## CI Runs (Actions)

### Watching
```bash
# Block until run completes; exits non-zero on failure
gh run watch <run-id>                       # if you have the ID
gh run watch                                # picks latest run on current branch

# List recent runs
gh run list --limit 10
gh run list --workflow=deploy.yml --branch main --limit 5

# Show details + logs
gh run view <run-id>
gh run view <run-id> --log                  # full logs of failed steps
gh run view <run-id> --log-failed           # only failed step logs
```

For "wait for CI" workflows, **`gh run watch` is the right tool** — it polls + blocks + exits with the run's exit code. Don't loop `gh run list` mannually.

### Re-running
```bash
gh run rerun <run-id>                       # all failed jobs
gh run rerun <run-id> --failed              # only failed jobs
gh run rerun <run-id> --job <job-id>        # specific job
```

## Issues

```bash
gh issue list --state open --label "bug"
gh issue view 456                           # body + comments
gh issue view 456 --json state,assignees,labels,milestone
gh issue create --title "..." --body "..." --label "bug,priority-high"
gh issue close 456 --reason completed
gh issue comment 456 --body "fixed in #123"

# Search across repos
gh search issues --owner ORG "label:security state:open"
```

## Releases

```bash
gh release list --limit 10
gh release view v1.2.3
gh release create v1.2.4 --notes "..." ./dist/*.tar.gz   # creates + uploads assets
gh release download v1.2.3 -p '*.tar.gz'                 # fetch assets
```

## Direct API for Things gh Doesn't Wrap

```bash
# REST API — use --paginate for list endpoints
gh api repos/OWNER/REPO/branches --paginate --jq '.[].name'

# GraphQL API
gh api graphql -f query='{ viewer { login } }'

# POST/PATCH/PUT bodies
gh api repos/OWNER/REPO/issues/456 \
  --method PATCH \
  -f state=closed \
  -f state_reason=completed
```

Use `--jq <expr>` for inline jq filtering rather than piping through external jq.

## JSON Output for Scripts

Almost every gh command supports `--json <fields>` + `--jq <expr>`:
```bash
gh pr list --json number,title,headRefName --jq '.[] | "\(.number): \(.title) (\(.headRefName))"'
gh pr view 123 --json reviewDecision -q .reviewDecision
gh run view RUN --json conclusion,jobs --jq '.jobs[] | select(.conclusion=="failure") | .name'
```

Discover fields available: `gh pr view --help` shows `Available JSON fields:` near the bottom.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `gh: ... requires the "X" scope` | Missing OAuth scope | `gh auth refresh -s repo,workflow` (add the scope it asked for) |
| `gh: HTTP 401: Bad credentials` | Token expired | `gh auth login` (fresh login) |
| `no PR with branch X` | You're not in the repo, or branch never had a PR | `cd` to the repo, or pass `--repo OWNER/REPO` |
| `Could not resolve to a Repository` | wrong --repo arg | Check the org spelling; `gh repo view OWNER/REPO` to confirm access |
| `query parsing failed` (jq) | Forgot to wrap path expressions in quotes | Use single quotes around full jq expression |

## Anti-Patterns

- **Don't write a bash loop that polls `gh run list`** — use `gh run watch`.
- **Don't shell out to `curl https://api.github.com/...`** when `gh api` does it with auth.
- **Don't `gh pr create` without a body** — auto-generated default is rarely right.
- **Don't leave inline review comments via `gh pr comment`** — that goes to PR conversation, not the line. Use the API form above.
- **Don't `gh pr merge` without verifying checks** — pass `--auto` to defer to CI, or read `statusCheckRollup` first.
