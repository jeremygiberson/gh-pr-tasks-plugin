# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Claude Code plugin that provides skills for using GitHub pull request reviews to trigger work:
- List pull requests
- Provide reviews to pull requests
- Get review feedback
- Trigger fix sessions to checkout a PR branch, address feedback, and push

## Dependencies

- GitHub CLI (`gh`) must be installed and authenticated
- Requires the `gh-pr-review` extension for managing inline PR review comments (`gh pr-review`)
