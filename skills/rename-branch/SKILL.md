---
name: rename-branch
description: Renames the current git branch to match an expected naming convention based on a ticket or PR number. Useful when a coding agent orchestrator creates worktrees with random branch names. Use when the user says "rename branch", "fix branch name", or needs to align a random branch name with a PR or ticket.
allowed-tools: Bash, Read, AskUserQuestion
---

# Rename Branch

Renames the current git branch to match the expected branch name for a PR or ticket. Useful when working in a worktree created by an orchestrator that assigns random branch names.

## When to Activate

- "rename branch", "fix my branch name"
- "align branch to PR #123"
- "rename this branch to match the ticket"

## Workflow

**Step 1: Determine the target branch name**

Try to determine the correct branch name from context, in this priority order:

1. **PR number provided or in context:** If the user provides a PR number (or one is obvious from conversation context), fetch the PR's head branch:

   ```bash
   gh repo view --json nameWithOwner -q .nameWithOwner
   gh pr view <PR_NUMBER> -R <owner/repo> --json headRefName -q .headRefName
   ```

   Use the `headRefName` as the target branch name.

2. **Shortcut ticket number from context:** Scan the conversation for a Shortcut ticket reference — any of these forms:
   - `sc-12345` or `SC-12345`
   - A Shortcut URL (e.g., `https://app.shortcut.com/*/story/12345/...`)
   - Phrases like "ticket 12345", "story 12345"

   If found, use the ticket number to construct the branch name. The convention is that the branch name should contain the Shortcut ticket number (e.g., `sc-12345/description`). Use `AskUserQuestion` to confirm: "I found ticket sc-NNNNN in context. What should the full branch name be? (e.g., `sc-NNNNN/short-description`)"

3. **No context available:** Use `AskUserQuestion` to ask: "What should the branch be named? You can provide a PR number (e.g., `#123`), a Shortcut ticket number (e.g., `sc-12345`), or a full branch name."

   - If the user provides a PR number, fetch `headRefName` as in step 1.
   - If the user provides a ticket number, ask for the full branch name including that number.
   - If the user provides a branch name directly, use it as-is.

**Step 2: Check current state**

```bash
git rev-parse --abbrev-ref HEAD
```

If the current branch already matches the target name, report that no rename is needed and stop.

**Step 3: Rename the branch**

```bash
git branch -m <current_branch> <target_branch>
```

**Step 4: Set upstream tracking**

If the target branch exists on the remote, set the upstream:

```bash
git fetch origin <target_branch> 2>/dev/null && git branch --set-upstream-to=origin/<target_branch> <target_branch>
```

If the remote branch doesn't exist, inform the user that the branch was renamed locally and they'll need to push when ready.

Report: "Branch renamed from `<old_name>` to `<target_branch>`."
