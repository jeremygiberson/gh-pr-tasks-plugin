# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Claude Code plugin (`gh-pr-tasks`) that provides 4 skills for GitHub pull request workflows:

- **pr-check-deps** -- Validates that gh CLI is installed, authenticated, and the `gh-pr-review` extension is present at the pinned version. Called internally by the other skills at the start of their workflows.
- **pr-list** -- Lists PRs needing attention: the user's PRs with unresolved review feedback, and PRs awaiting the user's review. Triggered by phrases like "list PRs", "PR status", "what should I review".
- **pr-review** -- Full review-giving workflow: checks out the PR to a worktree, analyzes the diff, drafts inline comments and summary, presents for user approval, then submits via `gh pr-review`. Triggered by "review PR #123", "give feedback on this PR".
- **pr-fix** -- Full fix-feedback workflow: checks out the PR branch to a worktree, triages each feedback item with the user (skip or address), creates an implementation plan, executes fixes, runs tests, pushes, and posts reply comments. Triggered by "fix PR feedback", "address review comments".

## Dependencies

- GitHub CLI (`gh`) must be installed and authenticated
- Requires the `gh-pr-review` extension for managing PR review comments and threads (`gh pr-review`)
- The extension is installed at a pinned commit hash tracked in `docs/security-review-log.md`

## Key Directories

- `.claude-plugin/plugin.json` -- Plugin manifest
- `skills/` -- Each skill has a directory with a `SKILL.md` defining its behavior
- `docs/security-review-log.md` -- Current approved install command and pinned hash for the gh-pr-review extension
- `docs/security-reviews/` -- Detailed security review reports for each hash
