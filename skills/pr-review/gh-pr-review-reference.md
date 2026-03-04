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
