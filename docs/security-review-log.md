# Security Review Log: agynio/gh-pr-review

Tracks security reviews of the [agynio/gh-pr-review](https://github.com/agynio/gh-pr-review) GitHub CLI extension before updating our pinned install version.

## Current Approved Install Command

```bash
gh extension install agynio/gh-pr-review --pin v1.6.2
```

**Approved Release**: `v1.6.2`
**Release Commit Hash**: `955b448eb7f44b7c89b35006cc54de5db7ee4ebf`
**Reviewed Source Commit**: `9c422aaa07cfc296cdc8e349ec490d9236b7bfdf` (main HEAD at review time, includes release + 3 additional commits)

## How to Verify Release Integrity

Check that the release tag still points to the same commit hash we reviewed:

```bash
# Get the commit hash the tag points to (dereference annotated tags)
TAG_SHA=$(gh api "repos/agynio/gh-pr-review/git/ref/tags/v1.6.2" --jq '.object.sha')
# If annotated tag, dereference to commit
COMMIT_SHA=$(gh api "repos/agynio/gh-pr-review/git/tags/$TAG_SHA" --jq '.object.sha' 2>/dev/null || echo "$TAG_SHA")
echo "$COMMIT_SHA"
# Expected: 955b448eb7f44b7c89b35006cc54de5db7ee4ebf
```

If the hash differs, the release tag has been moved — do not install until a new security review is performed.

## Review History

| Release | Release Commit Hash | Reviewed Commit | Review Date | Result | Detail |
| --- | --- | --- | --- | --- | --- |
| `v1.6.2` | `955b448eb7f44b7c89b35006cc54de5db7ee4ebf` | `9c422aaa07cfc296cdc8e349ec490d9236b7bfdf` | 2026-03-04 | PASS — Source clean, minor CI/CD supply chain hygiene concerns | [2026-03-04-review.md](security-reviews/2026-03-04-review.md) |
