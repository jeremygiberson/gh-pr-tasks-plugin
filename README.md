# gh-pr-tasks

A Claude Code plugin that provides skills for working with GitHub pull request reviews -- listing PRs, giving code reviews, and addressing reviewer feedback.

## Prerequisites

- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated (`gh auth login`)
- The `gh-pr-review` extension, installed at a pinned commit:

```bash
gh extension install agynio/gh-pr-review --pin 9c422aaa07cfc296cdc8e349ec490d9236b7bfdf
```

The pinned hash is tracked in `docs/security-review-log.md`. Do not update it without a security review.

## Installation

Add the plugin to your Claude Code project by adding to `.claude/settings.json`:

```json
{
  "plugins": [
    "/path/to/gh-pr-tasks-plugin"
  ]
}
```

## Skills

### pr-check-deps

Validates that the environment is ready: gh CLI installed, authenticated, and `gh-pr-review` extension present at the expected pinned version. Called automatically by the other skills; you typically don't invoke it directly.

### pr-list

Lists pull requests needing your attention. Shows two sections:
- **Your PRs with feedback** -- open PRs you authored that have unresolved review threads
- **PRs awaiting your review** -- open PRs where you are a requested reviewer

**Triggers:** "list PRs", "what PRs need work", "PR status", "any PRs waiting on me"

### pr-review

Full code review workflow for a pull request. Checks out the PR to a worktree, analyzes the diff and changed files, drafts inline review comments with a summary, presents everything for your approval, and submits.

- Never auto-submits; always waits for your approval
- Supports APPROVE, COMMENT, and REQUEST_CHANGES events
- Cleans up the worktree when done

**Triggers:** "review PR #123", "give a review on this PR", "check this PR for issues"

### pr-fix

Addresses review feedback on your own PR. Checks out the branch to a worktree, walks through each feedback item with you (skip or address), creates an implementation plan, executes the fixes, runs tests, and pushes after your approval. Skipped items get a reply comment; addressed items are resolved.

- User approval required at every phase (triage, plan, push)
- Runs tests before pushing
- Posts reply comments for skipped feedback and resolves addressed threads

**Triggers:** "fix PR feedback", "address PR #123 feedback", "work on PR review comments"

## Project Structure

```
.claude-plugin/
  plugin.json          # Plugin manifest
skills/
  pr-check-deps/       # Environment validation skill
  pr-list/             # PR listing skill
  pr-review/           # Code review skill
  pr-fix/              # Fix feedback skill
docs/
  security-review-log.md   # Pinned extension hash and review history
  security-reviews/        # Detailed security review reports
```

## License

MIT
