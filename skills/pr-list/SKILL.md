---
name: pr-list
description: Lists pull requests needing attention — your PRs with unresolved review feedback and PRs awaiting your review. Use when the user asks to list PRs, check PR status, see what needs work, or find PRs to review.
allowed-tools: Bash, Read
---

# PR List

Lists pull requests that need the user's attention in the current repository.

## When to Activate

- "list PRs", "what PRs need work", "show my PRs"
- "PR status", "any PRs waiting on me"
- "what should I review", "PRs to review"

## Prerequisites

First, invoke the `pr-check-deps` skill to validate the environment. If it reports a failure, stop and show the remediation instructions to the user.

## Workflow

### Step 1: Detect Repository

Determine the repository. If the user provided `-R owner/repo`, use that. Otherwise detect from git context:

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

If this fails, ask the user to specify the repository.

### Step 2: Get Your PRs With Feedback

Fetch open PRs authored by the current user:

```bash
gh pr list --author @me --state open --json number,title,url,updatedAt,reviewDecision -R <owner/repo>
```

For each PR returned, check for unresolved review threads:

```bash
gh pr-review threads list -R <owner/repo> <PR_NUMBER> --unresolved
```

Count the items in the returned JSON array to get the unresolved thread count.

### Step 3: Get PRs Awaiting Your Review

Fetch PRs where the current user is a requested reviewer:

```bash
gh pr list --search "review-requested:@me" --state open --json number,title,url,author,updatedAt -R <owner/repo>
```

### Step 4: Present Results

Display two tables:

**Your PRs with feedback:**

| PR | Title | Unresolved Threads | Updated |
|---|---|---|---|
| #123 | Add user auth | 3 | 2h ago |

If no PRs have unresolved threads, say "No PRs with unresolved feedback."

**PRs awaiting your review:**

| PR | Title | Author | Updated |
|---|---|---|---|
| #456 | Fix login bug | @teammate | 1d ago |

If no PRs are waiting, say "No PRs awaiting your review."

### Step 5: Offer Next Actions

After displaying results, suggest:
- "Use `pr-fix` to address feedback on one of your PRs"
- "Use `pr-review` to review a PR"

## Notes

- Always use `--state open` to filter to open PRs only
- The `gh pr-review threads list --unresolved` call may be slow for repos with many PRs; only check PRs that have `reviewDecision` of `CHANGES_REQUESTED` or `REVIEW_REQUIRED` to minimize API calls
- Format dates as relative time (e.g., "2h ago", "3d ago") for readability
