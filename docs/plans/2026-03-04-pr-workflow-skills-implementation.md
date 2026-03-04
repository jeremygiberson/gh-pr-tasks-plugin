# PR Workflow Skills Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a Claude Code plugin with 4 skills for managing GitHub PR reviews — checking deps, listing PRs, giving reviews, and fixing review feedback.

**Architecture:** Plugin uses standard Claude Code plugin layout with `.claude-plugin/plugin.json` and `skills/` directory. Each skill is a `SKILL.md` with YAML frontmatter. Skills invoke `gh` CLI and the `gh-pr-review` extension for all GitHub operations. Workflow skills (pr-review, pr-fix) include inline worktree setup.

**Tech Stack:** Claude Code plugin system, GitHub CLI (`gh`), `gh-pr-review` extension, git worktrees.

**Design doc:** `docs/plans/2026-03-04-pr-workflow-skills-design.md`

---

### Task 1: Scaffold Plugin Structure

**Files:**
- Create: `.claude-plugin/plugin.json`
- Modify: `CLAUDE.md` (update to reflect actual extension name)

**Step 1: Create plugin manifest**

Create `.claude-plugin/plugin.json`:

```json
{
  "name": "gh-pr-tasks",
  "version": "0.1.0",
  "description": "Skills for working with GitHub PR reviews — list PRs, give reviews, and address review feedback.",
  "author": {
    "name": "Jeremy Giberson"
  },
  "license": "MIT",
  "keywords": ["github", "pull-request", "review", "workflow"]
}
```

Do NOT add `skills` field — the `skills/` directory is auto-discovered.

**Step 2: Fix CLAUDE.md extension name**

The current CLAUDE.md references `gh-pr-comments` but the actual extension is `gh-pr-review`. Update the dependency reference:

```markdown
- Requires the `gh-pr-review` extension for managing inline PR review comments (`gh pr-review`)
```

**Step 3: Verify structure**

Run: `ls -la .claude-plugin/`
Expected: `plugin.json` exists

**Step 4: Commit**

```bash
git add .claude-plugin/plugin.json CLAUDE.md
git commit -m "feat: scaffold plugin manifest and fix extension name"
```

---

### Task 2: Create `pr-check-deps` Skill

**Files:**
- Create: `skills/pr-check-deps/SKILL.md`

**Step 1: Create skill directory**

```bash
mkdir -p skills/pr-check-deps
```

**Step 2: Write SKILL.md**

Create `skills/pr-check-deps/SKILL.md` with this content:

````markdown
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
````

**Step 3: Verify file exists**

Run: `cat skills/pr-check-deps/SKILL.md | head -5`
Expected: frontmatter with `name: pr-check-deps`

**Step 4: Commit**

```bash
git add skills/pr-check-deps/SKILL.md
git commit -m "feat: add pr-check-deps skill for environment validation"
```

---

### Task 3: Create `pr-list` Skill

**Files:**
- Create: `skills/pr-list/SKILL.md`

**Step 1: Create skill directory**

```bash
mkdir -p skills/pr-list
```

**Step 2: Write SKILL.md**

Create `skills/pr-list/SKILL.md` with this content:

````markdown
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
````

**Step 3: Verify**

Run: `cat skills/pr-list/SKILL.md | head -5`
Expected: frontmatter with `name: pr-list`

**Step 4: Commit**

```bash
git add skills/pr-list/SKILL.md
git commit -m "feat: add pr-list skill for listing PRs needing attention"
```

---

### Task 4: Create `pr-review` Skill

**Files:**
- Create: `skills/pr-review/SKILL.md`
- Create: `skills/pr-review/gh-pr-review-reference.md` (CLI reference for progressive disclosure)

**Step 1: Create skill directory**

```bash
mkdir -p skills/pr-review
```

**Step 2: Write CLI reference file**

Create `skills/pr-review/gh-pr-review-reference.md`:

````markdown
# gh-pr-review CLI Reference

All commands output JSON. Review IDs start with `PRR_`, thread IDs with `PRRT_`.

## Start a pending review

```bash
gh pr-review review --start -R <owner/repo> <PR_NUMBER>
```

Output: `{ "id": "PRR_...", "state": "PENDING" }`

## Add inline comment to pending review

```bash
gh pr-review review --add-comment \
  --review-id <PRR_...> \
  --path <file> \
  --line <number> \
  --body "<text>" \
  -R <owner/repo> <PR_NUMBER>
```

Optional flags: `--side LEFT|RIGHT` (default RIGHT)

Output: `{ "id": "...", "path": "...", "line": 42, "is_outdated": false }`

## Submit pending review

```bash
gh pr-review review --submit \
  --review-id <PRR_...> \
  --event <APPROVE|COMMENT|REQUEST_CHANGES> \
  --body "<summary text>" \
  -R <owner/repo> <PR_NUMBER>
```

## View review

```bash
gh pr-review review view -R <owner/repo> --pr <number> \
  [--unresolved] [--not_outdated] [--reviewer <login>] \
  [--states APPROVED,CHANGES_REQUESTED,COMMENTED,DISMISSED] \
  [--tail <n>] [--include-comment-node-id]
```

## Reply to a thread

```bash
gh pr-review comments reply <PR_NUMBER> -R <owner/repo> \
  --thread-id <PRRT_...> \
  --body "<text>"
```

## List threads

```bash
gh pr-review threads list -R <owner/repo> <PR_NUMBER> \
  [--unresolved] [--mine]
```

## Resolve / unresolve thread

```bash
gh pr-review threads resolve --thread-id <PRRT_...> -R <owner/repo> <PR_NUMBER>
gh pr-review threads unresolve --thread-id <PRRT_...> -R <owner/repo> <PR_NUMBER>
```
````

**Step 3: Write SKILL.md**

Create `skills/pr-review/SKILL.md`:

````markdown
---
name: pr-review
description: Performs a full code review on a GitHub pull request — checks out the PR to a worktree, analyzes the diff, drafts inline review comments, presents for user approval, and submits. Use when the user asks to review a PR, give feedback on a PR, or examine a PR for issues.
allowed-tools: Bash, Read, Glob, Grep, Edit, Write, Agent, AskUserQuestion
---

# PR Review

Full review-giving workflow for a GitHub pull request. Checks out the PR, reads the code, drafts inline review comments, and submits after user approval.

## When to Activate

- "review PR #123", "review this PR"
- "give a review on PR", "look at PR #456"
- "check this PR for issues"

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
gh pr view <PR_NUMBER> -R <owner/repo> --json number,title,headRefName,baseRefName,url,author,body,additions,deletions,changedFiles
```

Report to the user: "Reviewing PR #N: <title> by @author (<additions>+/<deletions>-, <changedFiles> files)"

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

### Phase 2: Analyze

**Step 3: Read the PR diff**

```bash
gh pr diff <PR_NUMBER> -R <owner/repo>
```

**Step 4: Read changed files for context**

For each file in the diff, read the full file in the worktree to understand the surrounding context. Use the Read tool on files in `.worktrees/<branch>/`.

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

**Step 9: Clean up**

```bash
cd <original_directory>
git worktree remove .worktrees/<headRefName>
```

Report: "Review submitted on PR #N: <link>"

## Constraints

- **Never auto-submit.** Always present the complete review for user approval before submitting.
- **Worktree isolation.** Always work in the worktree, never modify the user's working directory.
- **Clean up.** Always remove the worktree when done, even if the review is cancelled.
- **Constructive feedback.** Review comments should be specific, actionable, and explain the reasoning.
````

**Step 4: Verify**

Run: `ls skills/pr-review/`
Expected: `SKILL.md` and `gh-pr-review-reference.md`

**Step 5: Commit**

```bash
git add skills/pr-review/
git commit -m "feat: add pr-review skill for giving PR reviews"
```

---

### Task 5: Create `pr-fix` Skill

**Files:**
- Create: `skills/pr-fix/SKILL.md`
- Create: `skills/pr-fix/gh-pr-review-reference.md` (symlink or copy from pr-review)

**Step 1: Create skill directory**

```bash
mkdir -p skills/pr-fix
```

**Step 2: Copy CLI reference**

```bash
cp skills/pr-review/gh-pr-review-reference.md skills/pr-fix/gh-pr-review-reference.md
```

**Step 3: Write SKILL.md**

Create `skills/pr-fix/SKILL.md`:

````markdown
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
````

**Step 4: Verify**

Run: `ls skills/pr-fix/`
Expected: `SKILL.md` and `gh-pr-review-reference.md`

**Step 5: Commit**

```bash
git add skills/pr-fix/
git commit -m "feat: add pr-fix skill for addressing PR review feedback"
```

---

### Task 6: Validate Plugin and Test Locally

**Step 1: Validate plugin structure**

Run: `/plugin-development:validate`

Fix any issues reported.

**Step 2: Test local installation**

Run: `/plugin-development:test-local`

Verify all 4 skills appear in the skill list.

**Step 3: Smoke test**

In a repo with open PRs, verify:
- `pr-check-deps` reports missing extension (since we haven't installed it yet)
- After installing the extension, `pr-check-deps` passes
- `pr-list` shows PRs

**Step 4: Commit any fixes**

```bash
git add -A
git commit -m "fix: address plugin validation issues"
```

---

### Task 7: Update README and CLAUDE.md

**Files:**
- Modify: `README.md`
- Modify: `CLAUDE.md`

**Step 1: Update README.md**

Add usage examples for each skill, installation instructions, and prerequisites.

**Step 2: Update CLAUDE.md**

Ensure CLAUDE.md accurately reflects:
- The 4 skills and their purposes
- The `gh-pr-review` extension name (not `gh-pr-comments`)
- The security review process for the extension

**Step 3: Commit**

```bash
git add README.md CLAUDE.md
git commit -m "docs: update README and CLAUDE.md with skill documentation"
```
