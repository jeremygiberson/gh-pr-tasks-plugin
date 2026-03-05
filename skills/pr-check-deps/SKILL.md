---
name: pr-check-deps
description: Validates GitHub CLI environment for PR workflow skills. Use when any pr-* skill needs to verify gh CLI is installed, authenticated, and the gh-pr-review extension is present at the expected version.
allowed-tools: Bash, Read
---

# PR Check Dependencies

Validates that the environment has all prerequisites for the PR workflow skills.

## When to Activate

This skill is called internally by other pr-* skills at the start of their workflows. It is not typically invoked directly by the user.

## Checks

Run these checks in order. Stop at the first failure and report the issue with remediation instructions.

### Check 1: gh CLI installed

```bash
gh --version
```

If this fails, report:
> GitHub CLI (`gh`) is not installed. Install it from https://cli.github.com/ and then run `gh auth login` to authenticate.

### Check 2: gh CLI authenticated

```bash
gh auth status
```

If this fails (non-zero exit), report:
> GitHub CLI is not authenticated. Run `gh auth login` to authenticate.

### Check 3: gh-pr-review extension installed

```bash
gh extension list | grep "pr-review"
```

If no output, the extension is not installed. Read `docs/security-review-log.md` to find the current approved install command (it's in the "Current Approved Install Command" section near the top of the file). Report:
> The `gh-pr-review` extension is not installed. Install it with the reviewed version:
> ```
> [insert command from security-review-log.md]
> ```

### Check 4: Installed version matches approved release

Read `docs/security-review-log.md` and extract the **Approved Release** tag (e.g., `v1.6.2`). Check `gh extension list` output to see if the installed version matches.

```bash
gh extension list | grep "pr-review"
```

If the installed version does not match the approved release tag, warn:
> The installed `gh-pr-review` version does not match the approved release. Approved: `[tag]`. To update:
> ```
> gh extension remove pr-review
> [insert install command from security-review-log.md]
> ```

### Check 5: Release tag integrity

Read `docs/security-review-log.md` and extract the **Release Commit Hash** (the commit SHA the release tag pointed to at review time). Then verify the tag still points to the same commit:

```bash
TAG_SHA=$(gh api "repos/agynio/gh-pr-review/git/ref/tags/<TAG>" --jq '.object.sha')
COMMIT_SHA=$(gh api "repos/agynio/gh-pr-review/git/tags/$TAG_SHA" --jq '.object.sha' 2>/dev/null || echo "$TAG_SHA")
echo "$COMMIT_SHA"
```

Compare `$COMMIT_SHA` against the Release Commit Hash from `docs/security-review-log.md`.

- If they **match**: the release is intact, proceed normally.
- If they **differ**: the release tag has been moved since our security review. This is a warning — the installed binary may contain code we haven't reviewed. Report:
  > **WARNING**: The `gh-pr-review` release tag `[tag]` now points to a different commit than when it was reviewed.
  > - Reviewed commit: `[hash from log]`
  > - Current commit: `[current hash]`
  >
  > This could mean the release was updated after our security review. A new security review should be performed before continuing. See `docs/security-review-process.md`.

  Do **not** block the workflow for this warning, but make it clearly visible to the user.

- If the API call fails (e.g., no network), note the failure but do not block the workflow.

### Success

If all checks pass, report briefly:
> Dependencies verified: gh CLI authenticated, gh-pr-review extension installed.

Then allow the calling workflow to continue.
