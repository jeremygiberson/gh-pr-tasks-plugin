# Security Review Process: agynio/gh-pr-review

Workflow for periodically reviewing the [agynio/gh-pr-review](https://github.com/agynio/gh-pr-review) extension for security concerns before updating our pinned install hash.

## When to Review

- Before first installation (completed 2026-03-04)
- When a new version is released that we want to adopt
- Periodically (recommended: monthly or before any significant project milestone)
- If a security advisory is published for the repo or its dependencies

## Step 1: Check for Updates

Compare our pinned hash against the current `main` HEAD:

```bash
# Our current pinned hash
grep "pin" docs/security-review-log.md | head -1

# Current main HEAD
gh api "repos/agynio/gh-pr-review/commits/main" --jq '.sha'
```

If the hashes match, no review is needed.

## Step 2: Review the Diff Since Last Reviewed Commit

Get the diff between our last reviewed commit and current `main`:

```bash
# Replace OLD_HASH with the hash from security-review-log.md
gh api "repos/agynio/gh-pr-review/compare/OLD_HASH...main" --jq '.commits | length'
gh api "repos/agynio/gh-pr-review/compare/OLD_HASH...main" --jq '.files[] | "\(.status) \(.filename)"'
```

If only documentation, tests, or CI config changed, the review can be expedited.

## Step 3: Fetch and Review Changed Files

For each changed source file, fetch and review it:

```bash
gh api "repos/agynio/gh-pr-review/contents/PATH?ref=main" -q .content | base64 -d
```

Or fetch the full diff:

```bash
gh api "repos/agynio/gh-pr-review/compare/OLD_HASH...main" --jq '.files[] | select(.filename | test("\\.(go|mod|sum|yml)$")) | {filename, status, patch}'
```

## Step 4: Perform Security Analysis

Check each changed file against these criteria:

### Critical (must pass)
- [ ] No new external command execution patterns beyond `exec.Command("gh", args...)`
- [ ] No new `net/http` imports or direct HTTP clients
- [ ] No hardcoded credentials, tokens, or suspicious URLs
- [ ] No obfuscated or encoded strings
- [ ] No new hidden subcommands or undocumented functionality
- [ ] No data sent to services other than GitHub's API
- [ ] GraphQL queries still use parameterized variables (no string interpolation)

### Important (note in findings)
- [ ] New dependencies in `go.mod` — are they well-known and necessary?
- [ ] CI/CD workflow changes — do they introduce new supply chain risks?
- [ ] Changes to `internal/ghcli/ghcli.go` — this is the single point of external command execution; scrutinize heavily
- [ ] Changes to `internal/resolver/resolver.go` — this handles host/URL parsing

### Architecture Notes for Reviewers

These facts about the codebase speed up subsequent reviews:

1. **Single execution point**: All external commands go through `internal/ghcli/ghcli.go` via `exec.Command("gh", args...)`. If this file hasn't changed, the command execution surface is unchanged. This is the most critical file to review.

2. **No direct networking**: The extension has zero `net/http` usage. All networking is delegated to the `gh` CLI. Any new `import "net/http"` is an immediate red flag.

3. **Layered architecture**: `cmd/` -> `internal/*/service.go` -> `internal/ghcli/ghcli.go`. User input enters at `cmd/`, flows through service layers, and exits only through `ghcli.go`.

4. **GraphQL via stdin**: GraphQL queries and variables are JSON-marshaled and piped to `gh api graphql --input -`. Variables are never interpolated into query strings. Any change to this pattern warrants scrutiny.

5. **Host handling**: `GH_HOST` env var and `--hostname` flag flow through `resolver.sanitizeHost()` which strips schemes, ports, and paths. This only affects which GitHub instance is targeted (supports GitHub Enterprise).

6. **Dependencies are minimal**: Only `cobra` and `testify` (plus their transitive deps). Any new direct dependency in `go.mod` should be investigated.

## Step 5: Write Review Document

Create `docs/security-reviews/YYYY-MM-DD-review.md` following the format of existing reviews. Include:

- Reviewed commit hash
- Summary of changes since last review
- Security findings (or confirmation that no concerns were found)
- Updated install command with new hash (if approved)

## Step 6: Update the Review Log

Append a new row to the table in `docs/security-review-log.md`:

```markdown
| `NEW_COMMIT_HASH` | YYYY-MM-DD | PASS/FAIL — summary | [YYYY-MM-DD-review.md](security-reviews/YYYY-MM-DD-review.md) |
```

If the review passes, also update the "Current Approved Install Command" section at the top of the log with the new hash.

## Step 7: Communicate to Team

After updating the pinned hash, notify the team to re-install:

```bash
gh extension remove gh-pr-review
gh extension install agynio/gh-pr-review --pin NEW_COMMIT_HASH
```

## Expedited Review (Diff-Only)

When changes are small and limited to non-critical files (docs, tests, CI), an expedited review is acceptable:

1. Confirm no changes to `internal/ghcli/ghcli.go`
2. Confirm no new dependencies in `go.mod`
3. Confirm no new `net/http` or `os/exec` imports in any file
4. Spot-check any new `.go` files
5. Document as expedited in the review detail file

## Failing a Review

If a review finds security concerns:

1. Document the findings in the review detail file
2. Log as FAIL in `security-review-log.md`
3. **Do not update the pinned hash**
4. Evaluate whether the concern affects our currently pinned version
5. If the current pinned version is also affected, assess whether to uninstall
