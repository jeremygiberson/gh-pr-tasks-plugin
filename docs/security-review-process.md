# Security Review Process: agynio/gh-pr-review

Workflow for periodically reviewing the [agynio/gh-pr-review](https://github.com/agynio/gh-pr-review) extension for security concerns before updating our pinned install version.

## Key Concepts

- **Release tag**: The version tag (e.g., `v1.6.2`) used with `gh extension install --pin`. Tags are mutable — they can be moved to point to different commits.
- **Release commit hash**: The immutable commit SHA that the release tag pointed to at review time. We record this to detect if a tag is moved after review.
- **Reviewed commit**: The commit SHA we actually reviewed (may be ahead of the release if we reviewed `main`).

## When to Review

- Before first installation (completed 2026-03-04)
- When a new version is released that we want to adopt
- Periodically (recommended: monthly or before any significant project milestone)
- If a security advisory is published for the repo or its dependencies

## Step 1: Check for New Releases

Check the latest release and compare to our approved version:

```bash
# Our current approved release
grep "Approved Release" docs/security-review-log.md

# Latest release on GitHub
gh api "repos/agynio/gh-pr-review/releases/latest" --jq '{tag: .tag_name, created: .created_at}'
```

If the latest release matches our approved version, check that the tag hasn't been moved:

```bash
# Get the commit hash the tag currently points to
TAG_SHA=$(gh api "repos/agynio/gh-pr-review/git/ref/tags/<TAG>" --jq '.object.sha')
# Dereference annotated tag to get actual commit
COMMIT_SHA=$(gh api "repos/agynio/gh-pr-review/git/tags/$TAG_SHA" --jq '.object.sha' 2>/dev/null || echo "$TAG_SHA")
echo "$COMMIT_SHA"
```

Compare against the "Release Commit Hash" in `docs/security-review-log.md`. If they match, no review needed. If they differ, **the release tag was moved** — treat this as a new release requiring review.

## Step 2: Review the Diff Since Last Reviewed Commit

Get the diff between our last reviewed commit and the new release:

```bash
# Get the new release's commit hash (dereference annotated tags)
TAG_SHA=$(gh api "repos/agynio/gh-pr-review/git/ref/tags/<NEW_TAG>" --jq '.object.sha')
NEW_COMMIT=$(gh api "repos/agynio/gh-pr-review/git/tags/$TAG_SHA" --jq '.object.sha' 2>/dev/null || echo "$TAG_SHA")

# Replace OLD_HASH with the "Reviewed Commit" from security-review-log.md
gh api "repos/agynio/gh-pr-review/compare/OLD_HASH...$NEW_COMMIT" --jq '.commits | length'
gh api "repos/agynio/gh-pr-review/compare/OLD_HASH...$NEW_COMMIT" --jq '.files[] | "\(.status) \(.filename)"'
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

- Release tag and commit hash
- Reviewed commit hash (if reviewing `main` beyond the release)
- Summary of changes since last review
- Security findings (or confirmation that no concerns were found)
- Updated install command with new release tag (if approved)

## Step 6: Update the Review Log

Append a new row to the table in `docs/security-review-log.md`:

```markdown
| `<TAG>` | `<RELEASE_COMMIT_HASH>` | `<REVIEWED_COMMIT>` | YYYY-MM-DD | PASS/FAIL — summary | [YYYY-MM-DD-review.md](security-reviews/YYYY-MM-DD-review.md) |
```

If the review passes, also update the "Current Approved Install Command" section at the top of the log:
- Update the `--pin` value to the new release tag
- Update the "Approved Release", "Release Commit Hash", and "Reviewed Source Commit" fields

## Step 7: Communicate to Team

After updating the pinned version, notify the team to re-install:

```bash
gh extension remove pr-review
gh extension install agynio/gh-pr-review --pin <NEW_TAG>
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
