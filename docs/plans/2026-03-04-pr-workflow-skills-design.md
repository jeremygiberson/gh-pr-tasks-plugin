# Design: PR Workflow Skills for gh-pr-tasks-plugin

**Date**: 2026-03-04
**Status**: Approved

## Overview

A Claude Code plugin providing 4 composable skills for working with GitHub pull request reviews. Covers both sides of the review cycle: giving reviews and addressing received feedback.

### Dependencies

- GitHub CLI (`gh`) — installed and authenticated
- `gh-pr-review` extension — installed at pinned commit from `docs/security-review-log.md`

### Design Decisions

- **4 workflow-oriented skills** chosen over fine-grained (11) or hybrid (5) approaches. Fewer skills = clearer mental model for users.
- **Worktree setup inlined** in workflow skills rather than extracted to a separate skill or depending on `superpowers:using-git-worktrees`. Claude Code has no plugin dependency mechanism, and the worktree logic is ~15-20 lines of shell instructions.
- **Repo detection is dynamic** — skills detect repo from git context or accept `-R owner/repo`.
- **Reviews always require user approval** before submission.
- **Fix workflow uses per-item triage** — user categorizes each feedback item (skip/address/address with guidance) before a unified plan is created.

---

## Skill 1: `pr-check-deps`

**Purpose**: Validates environment prerequisites. Called at the start of every workflow.

**Trigger**: Used internally by other skills (not user-invoked directly).

**Checks (in order)**:
1. `gh` CLI installed → if not: link to https://cli.github.com/
2. `gh auth status` succeeds → if not: instruct `gh auth login`
3. `gh-pr-review` extension installed → if not: provide pinned install command from `docs/security-review-log.md`
4. Installed version matches pinned hash → if not: warn and provide upgrade command

**Behavior**:
- All checks pass → brief success message, calling skill continues
- Any check fails → report issue with copy-paste remediation commands, stop workflow
- Reads pinned hash from `docs/security-review-log.md` (single source of truth)

---

## Skill 2: `pr-list`

**Purpose**: Lists PRs needing the user's attention.

**Trigger**: "list PRs", "what PRs need work", "show my PRs", "PR status"

**Output sections**:

1. **Your PRs with feedback** — PRs authored by `@me` with unresolved review threads:
   - `gh pr list --author @me --json number,title,url,reviewDecision,updatedAt`
   - For each PR: `gh pr-review threads list --unresolved` to count unresolved threads
   - Display: PR number, title, unresolved thread count, last updated

2. **PRs awaiting your review** — PRs where current user is a requested reviewer:
   - `gh pr list --search "review-requested:@me"`
   - Display: PR number, title, author, review status

**Behavior**:
- Runs `pr-check-deps` first
- Detects repo from git context (`gh repo view --json nameWithOwner`) or accepts `-R owner/repo`
- Output is a formatted summary table

---

## Skill 3: `pr-review`

**Purpose**: Full review-giving workflow for someone else's PR.

**Trigger**: "review PR #123", "review this PR", "give a review on PR"

**Flow**:

```
1. Run pr-check-deps validation
2. Resolve PR (number or URL) → get repo + PR metadata via gh
3. Setup worktree:
   - Ensure .worktrees/ exists and is gitignored
   - git worktree add .worktrees/<branch> <remote-branch>
   - Auto-detect and install dependencies (package.json, go.mod, etc.)
4. Fetch the PR diff (gh pr diff <number>)
5. Read relevant changed files in the worktree for full context
6. Draft review:
   - Analyze code for correctness, design, and clarity issues
   - Produce inline comments (file, line, body)
   - Produce overall summary + event recommendation (APPROVE/COMMENT/REQUEST_CHANGES)
7. Present draft to user:
   - Show each inline comment with surrounding code context
   - Show overall summary and recommended event
   - User can edit, remove, add comments, or change the event
8. On user approval:
   - gh pr-review review --start
   - gh pr-review review --add-comment (for each inline comment)
   - gh pr-review review --submit with chosen event + body
9. Clean up worktree (git worktree remove)
10. Report success with link to PR
```

**Constraints**:
- Never auto-submits — always presents full review for user approval
- Worktree keeps user's working directory undisturbed
- Cleans up worktree after completion

---

## Skill 4: `pr-fix`

**Purpose**: Address review feedback on the user's own PR.

**Trigger**: "fix PR feedback", "address PR #123 feedback", "work on PR review comments"

**Flow**:

```
Phase 1: Setup
  1. Run pr-check-deps validation
  2. Resolve PR (number or URL) → get repo + PR metadata
  3. Setup worktree:
     - Ensure .worktrees/ exists and is gitignored
     - git worktree add .worktrees/<branch> <remote-branch>
     - Auto-detect and install dependencies
     - Run baseline tests to confirm clean starting state

Phase 2: Triage
  4. Fetch all review feedback:
     - gh pr-review review view (unresolved, not outdated threads)
     - Group by reviewer, then by file
  5. For EACH feedback item, present to user with context and ask:
     a) Skip — user provides reason (will become a reply comment)
     b) Address — Claude proposes approach
     c) Address with guidance — user provides direction/suggestions
  6. Collect all triage decisions

Phase 3: Plan
  7. Draft skip-reply comments for all skipped items
  8. Create unified implementation plan for all "address" items:
     - Group changes by file
     - Identify dependencies between changes
     - Note any clarifying questions
  9. Present plan + skip comments to user for approval
     - User can revise, re-triage items, or ask questions
  10. Iterate until user approves

Phase 4: Implement
  11. Execute the implementation plan
  12. Run tests after implementation
  13. Present results to user:
      - Summary of changes made
      - Test results
      - Diff of all modifications

Phase 5: Finalize
  14. User gives final approval to push and comment
  15. On approval:
      - git push (from the worktree)
      - Post skip-reason replies to skipped threads via gh pr-review comments reply
      - Resolve addressed threads via gh pr-review threads resolve
  16. Clean up worktree
  17. Report success with link to PR
```

**Constraints**:
- Every phase has a user approval gate
- Skip comments are drafted but not posted until final approval
- Tests must pass before pushing (user can override)
- Addressed threads are resolved after push succeeds

---

## Shared Worktree Setup (Inlined in pr-review and pr-fix)

Both workflow skills include this setup sequence:

1. Check for `.worktrees/` directory; create if missing
2. Verify `.worktrees` is in `.gitignore`; add and commit if not
3. `git fetch origin <branch>`
4. `git worktree add .worktrees/<branch> origin/<branch>`
5. `cd` into worktree
6. Auto-detect project type and install dependencies:
   - `package.json` → `npm install`
   - `go.mod` → `go mod download`
   - `requirements.txt` → `pip install -r requirements.txt`
   - `pyproject.toml` → `poetry install` or `pip install -e .`
   - `Cargo.toml` → `cargo build`
7. (pr-fix only) Run baseline tests

Cleanup after workflow:
1. `cd` back to original directory
2. `git worktree remove .worktrees/<branch>`

---

## Plugin Structure

```
gh-pr-tasks-plugin/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── pr-check-deps/
│   │   └── SKILL.md
│   ├── pr-list/
│   │   └── SKILL.md
│   ├── pr-review/
│   │   └── SKILL.md
│   └── pr-fix/
│       └── SKILL.md
├── docs/
│   ├── security-review-log.md
│   ├── security-review-process.md
│   ├── security-reviews/
│   └── plans/
├── CLAUDE.md
└── README.md
```

---

## gh-pr-review CLI Reference

Commands available from the `gh-pr-review` extension (pinned at reviewed commit):

| Command | Purpose |
|---|---|
| `gh pr-review review --start -R owner/repo <PR>` | Start a pending review |
| `gh pr-review review --add-comment --review-id PRR_... --path <file> --line <n> --body "<text>" -R owner/repo <PR>` | Add inline comment to pending review |
| `gh pr-review review --submit --review-id PRR_... --event <EVENT> --body "<text>" -R owner/repo <PR>` | Submit pending review |
| `gh pr-review review view -R owner/repo --pr <n> [--unresolved] [--not_outdated] [--reviewer <login>]` | View review with threads |
| `gh pr-review comments reply <PR> -R owner/repo --thread-id PRRT_... --body "<text>"` | Reply to a review thread |
| `gh pr-review threads list -R owner/repo <PR> [--unresolved]` | List review threads |
| `gh pr-review threads resolve --thread-id PRRT_... -R owner/repo <PR>` | Resolve a thread |
| `gh pr-review threads unresolve --thread-id PRRT_... -R owner/repo <PR>` | Unresolve a thread |

All commands output JSON. Review IDs start with `PRR_`, thread IDs with `PRRT_`.
