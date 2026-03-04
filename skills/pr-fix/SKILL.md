---
name: pr-fix
description: Addresses review feedback on your GitHub pull request — checks out the PR branch to a worktree, triages each feedback item with you, creates an implementation plan, executes fixes, and pushes. Use when the user wants to fix PR feedback, address review comments, or work on PR changes requested by reviewers.
allowed-tools: Bash, Read, Glob, Grep, Edit, Write, Agent, AskUserQuestion, EnterPlanMode, TaskCreate, TaskUpdate, TaskList
---

# PR Fix

Addresses review feedback on the user's own PR. Triages each feedback item with the user, creates a unified plan, implements fixes, and pushes the update.

## When to Activate

- "fix PR feedback", "address PR #123 feedback"
- "work on PR review comments"
- "fix the changes requested on my PR"

## Quick Links

- [gh-pr-review CLI reference](./gh-pr-review-reference.md)

## Prerequisites

First, invoke the `pr-check-deps` skill to validate the environment. If it reports a failure, stop and show the remediation instructions.

## Workflow

### Phase 1: Setup

**Step 1: Resolve the PR**

The user provides a PR number, URL, or you can ask them. Determine the repo:

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

Fetch PR metadata:

```bash
gh pr view <PR_NUMBER> -R <owner/repo> --json number,title,headRefName,baseRefName,url,author,body
```

Report: "Fixing feedback on PR #N: <title>"

**Step 2: Setup worktree**

```bash
# Ensure worktree directory exists and is gitignored
mkdir -p .worktrees
git check-ignore -q .worktrees 2>/dev/null || echo ".worktrees/" >> .gitignore

# Fetch and create worktree
git fetch origin <headRefName>
git worktree add .worktrees/<headRefName> origin/<headRefName>
```

Auto-detect and install dependencies in the worktree:
- If `package.json` exists: `cd .worktrees/<branch> && npm install`
- If `go.mod` exists: `cd .worktrees/<branch> && go mod download`
- If `requirements.txt` exists: `cd .worktrees/<branch> && pip install -r requirements.txt`
- If `Cargo.toml` exists: `cd .worktrees/<branch> && cargo build`

**Step 3: Run baseline tests**

Detect and run the project's test suite in the worktree. If tests fail, inform the user:
> Baseline tests are failing before any changes. Proceed anyway?

### Phase 2: Triage

**Step 4: Fetch review feedback**

```bash
gh pr-review review view -R <owner/repo> --pr <PR_NUMBER> --unresolved --not_outdated
```

Parse the JSON response. Group feedback items by reviewer, then by file. Each feedback item is a review thread with:
- `thread_id` (PRRT_...)
- `path` (file)
- `line`
- `body` (the reviewer's comment)
- `thread_comments` (any follow-up replies)

**Step 5: Triage each feedback item**

For EACH feedback item, present it to the user with the relevant code context. Read the file at the specified line in the worktree to show surrounding code.

Use `AskUserQuestion` for each item with these options:

- **Skip** — "Won't address this. I'll provide a reason."
  - Prompt the user for a reason. This will become a reply comment on the thread.
- **Address (Claude proposes approach)** — "Fix this. Claude will suggest how."
  - After the user selects this, analyze the feedback and the code to propose a brief approach.
- **Address (with my guidance)** — "Fix this with my direction."
  - Prompt the user for their approach, suggestions, or constraints.

Collect all triage decisions before moving to the next phase.

**Step 6: Summarize triage**

Present a summary of all decisions:
- N items to skip (with reasons)
- N items to address
- Total files affected

Ask: "Does this look right, or do you want to re-triage any items?"

### Phase 3: Plan

**Step 7: Draft skip-reply comments**

For each skipped item, draft a polite reply comment explaining why it's not being addressed. Example tone:
> "Thanks for the feedback. We're not addressing this in this PR because [user's reason]. [Optional: brief elaboration if appropriate]."

**Step 8: Create implementation plan**

For all "address" items, create a unified implementation plan:
- Group changes by file
- Identify dependencies between changes (e.g., a type change affecting multiple files)
- For each change, describe:
  - What to modify
  - The approach (from Claude's analysis or user's guidance)
  - Which tests need updating or adding

**Step 9: Present plan for approval**

Present to the user:
1. The skip-reply comments (what will be posted)
2. The implementation plan (what will be changed)

Ask for approval. The user may:
- Approve the plan
- Revise specific items
- Re-triage items (move from skip to address or vice versa)
- Ask clarifying questions

Iterate until the user approves.

### Phase 4: Implement

**Step 10: Execute the plan**

Working in the worktree (`.worktrees/<branch>/`), implement all planned changes. Use the Edit tool to modify files. Commit logical groups of changes as you go.

**Step 11: Run tests**

Run the full test suite in the worktree. Report results to the user.

If tests fail:
- Analyze failures and attempt to fix
- Re-run tests
- If still failing, present the failures to the user and ask how to proceed

**Step 12: Present results**

Show the user:
1. Summary of all changes made (files modified, what changed)
2. Test results (pass/fail)
3. The full diff:

```bash
cd .worktrees/<branch> && git diff origin/<headRefName>
```

### Phase 5: Finalize

**Step 13: Get final approval**

Ask the user: "Ready to push these changes and post reply comments to the PR?"

The user must explicitly approve before proceeding.

**Step 14: Push and comment**

On approval:

```bash
# Push from the worktree
cd .worktrees/<branch> && git push origin <headRefName>
```

Post skip-reason replies (see [CLI reference](./gh-pr-review-reference.md)):

```bash
gh pr-review comments reply <PR_NUMBER> -R <owner/repo> \
  --thread-id <PRRT_...> \
  --body "<skip reason comment>"
```

Resolve addressed threads:

```bash
gh pr-review threads resolve --thread-id <PRRT_...> -R <owner/repo> <PR_NUMBER>
```

**Step 15: Clean up**

```bash
cd <original_directory>
git worktree remove .worktrees/<headRefName>
```

Report: "PR #N updated and pushed. Replied to N skipped threads, resolved N addressed threads. <link>"

## Constraints

- **User approval at every phase.** Never skip a user approval gate.
- **Triage before planning.** Do not start planning until all items are triaged.
- **Plan before implementing.** Do not start coding until the plan is approved.
- **Tests before pushing.** Tests must pass before pushing (user can override if informed).
- **Push before commenting.** Push code first, then post comments and resolve threads — ensures the code backing the "resolved" threads is actually in the PR.
- **Worktree isolation.** All work happens in the worktree. Never modify the user's main working directory.
- **Clean up.** Always remove the worktree when done, even if the workflow is cancelled.
