---
name: pr-review
description: Performs a full code review on a GitHub pull request — analyzes the diff, drafts inline review comments, presents for user approval, and submits. Use when the user asks to review a PR, give feedback on a PR, or examine a PR for issues.
allowed-tools: Bash, Read, Glob, Grep, AskUserQuestion
---

# PR Review

Full review-giving workflow for a GitHub pull request. Reads the code, drafts inline review comments, and submits after user approval.

## When to Activate

- "review PR #123", "review this PR"
- "give a review on PR", "look at PR #456"
- "check this PR for issues"

## Quick Links

- [gh-pr-review CLI reference](./gh-pr-review-reference.md)

## Prerequisites

First, invoke the `pr-check-deps` skill to validate the environment. If it reports a failure, stop and show the remediation instructions.

## Workspace Assumptions

This skill assumes the user has already set up their workspace (e.g., checked out the PR branch, created a worktree, etc.). All work happens in the current working directory. The user is responsible for workspace setup and cleanup.

If the current branch does not match the PR's `headRefName`, warn the user and ask if they'd like to continue anyway.

## Workflow

### Phase 1: Setup

**Step 1: Resolve the PR**

The user provides a PR number, URL, or you can ask them. Determine the repo:

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

Fetch PR metadata:

```bash
gh pr view <PR_NUMBER> -R <owner/repo> --json number,title,headRefName,baseRefName,url,author,body,additions,deletions,changedFiles
```

Report to the user: "Reviewing PR #N: <title> by @author (<additions>+/<deletions>-, <changedFiles> files)"

**Step 2: Verify branch**

Check that the current branch matches the PR branch:

```bash
git rev-parse --abbrev-ref HEAD
```

**If the current branch matches `<headRefName>`:** Fetch to ensure we have the latest:

```bash
git fetch origin <headRefName>
git pull origin <headRefName>
```

**If the current branch does NOT match `<headRefName>`:** Warn the user:
> "Current branch `<current_branch>` does not match the PR branch `<headRefName>`. You may want to use the `rename-branch` skill to align your branch name, or check out the correct branch."

Use `AskUserQuestion` to ask: "Continue working in the current directory anyway, or stop so you can set up your workspace?"

If the user chooses to stop, end the workflow.

### Phase 2: Analyze

**Step 3: Read the PR diff**

```bash
gh pr diff <PR_NUMBER> -R <owner/repo>
```

**Step 4: Read changed files for context**

For each file in the diff, read the full file to understand the surrounding context.

Focus on understanding:
- What the code does and why
- Correctness: logic errors, edge cases, off-by-one errors
- Design: naming, structure, separation of concerns
- Clarity: readability, comments where needed
- Testing: are changes tested?
- Security: injection, auth issues, data exposure

### Phase 3: Draft Review

**Step 5: Draft inline comments**

For each issue found, prepare an inline comment with:
- `path`: file path relative to repo root
- `line`: the line number in the diff (use the NEW file line numbers)
- `body`: the review comment text — be specific, constructive, and explain why

**Step 6: Draft overall summary**

Write a review summary covering:
- What the PR does (1-2 sentences)
- Overall assessment
- Key concerns if any

Choose a recommended event:
- `APPROVE` — code is good, no blocking issues
- `COMMENT` — observations/questions but not blocking
- `REQUEST_CHANGES` — issues that should be fixed before merge

### Phase 4: Present to User

**Step 7: Show the draft review**

Present each inline comment with the surrounding code context so the user can evaluate. Then show the overall summary and recommended event.

Use `AskUserQuestion` to let the user:
- Approve and submit as-is
- Edit specific comments (remove, modify, or add new ones)
- Change the event (APPROVE/COMMENT/REQUEST_CHANGES)
- Cancel the review entirely

Iterate until the user approves.

### Phase 5: Submit

**Step 8: Submit the review**

See [gh-pr-review CLI reference](./gh-pr-review-reference.md) for exact commands.

```bash
# Start pending review
gh pr-review review --start -R <owner/repo> <PR_NUMBER>
# Returns: { "id": "PRR_..." }

# Add each inline comment
gh pr-review review --add-comment \
  --review-id <PRR_...> \
  --path <file> \
  --line <line> \
  --body "<comment>" \
  -R <owner/repo> <PR_NUMBER>

# Submit with event and summary
gh pr-review review --submit \
  --review-id <PRR_...> \
  --event <EVENT> \
  --body "<summary>" \
  -R <owner/repo> <PR_NUMBER>
```

Report: "Review submitted on PR #N: <link>"

## Constraints

- **Never auto-submit.** Always present the complete review for user approval before submitting.
- **Work in current directory.** All work happens in the current working directory. Never modify files outside of it.
- **Constructive feedback.** Review comments should be specific, actionable, and explain the reasoning.
