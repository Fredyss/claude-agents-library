# skill-name-kebab-case

One sentence: what this skill does and when a user would invoke it as `/skill-name`.

$ARGUMENTS

## What to do

Describe the steps Claude should follow when this skill is invoked. Be specific:

1. What to read or run first.
2. How to interpret `$ARGUMENTS` (what happens when empty, what formats are accepted).
3. What to produce.

## Output format

Use a fenced block to show the exact shape of the output:

```
### Section heading
<content>
```

## Boundaries

- Do NOT <thing this skill must never do>.
- If `$ARGUMENTS` is missing and context is ambiguous, ask rather than guess.
