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
> The `gh-pr-review` extension is not installed. Install it with the pinned version:
> ```
> [insert command from security-review-log.md]
> ```

### Check 4: Pinned version matches

Read `docs/security-review-log.md` and extract the pinned commit hash from the install command. Then check the installed version:

```bash
gh extension list | grep "pr-review"
```

The output includes the installed version/commit. If it does not match the pinned hash, warn:
> The installed `gh-pr-review` version does not match the reviewed pin. Currently pinned: `[hash]`. To update:
> ```
> gh extension remove pr-review
> [insert install command from security-review-log.md]
> ```

If the version check is inconclusive (gh extension list may not show the full hash), note the discrepancy but do not block the workflow.

### Success

If all checks pass, report briefly:
> Dependencies verified: gh CLI authenticated, gh-pr-review extension installed.

Then allow the calling workflow to continue.
