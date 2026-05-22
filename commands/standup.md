# standup

Generate a concise standup update from recent git activity. Summarizes what was done, what's in progress, and surfaces any blockers mentioned in commit messages or TODOs added since yesterday.

$ARGUMENTS

## What to do

1. Run `git log --since="yesterday" --oneline --author="$(git config user.name)"` to get recent commits. If `$ARGUMENTS` contains a time range (e.g. "last 3 days"), adjust `--since` accordingly.
2. Run `git diff HEAD~$(git log --since="yesterday" --oneline | wc -l | tr -d ' ')...HEAD --stat` to see which files changed.
3. Grep for TODO/FIXME added in the diff: `git diff HEAD~1...HEAD | grep "^+.*TODO\|^+.*FIXME"`.
4. Draft the standup — do not mention file names unless they add meaning; focus on outcomes.

## Output format

```
**Yesterday**
- <what was completed, in plain English — no commit hashes>

**Today**
- <inferred from in-progress branches or latest commits if not yet merged>

**Blockers**
- <any blockers found — or "None">
```

Keep each bullet to one line. If there are no commits in the range, say so and suggest running `git log --oneline -10` to find the right range.
