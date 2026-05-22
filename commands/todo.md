# todo

Scan the codebase for TODO, FIXME, HACK, and XXX comments and return a prioritized list. Groups findings by severity and file so they can be triaged.

$ARGUMENTS

## What to do

1. Run: `grep -rn "TODO\|FIXME\|HACK\|XXX" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.py" --include="*.go" --include="*.rs" --include="*.rb" --include="*.md" .` — adjust extensions to match the project.
2. Exclude `node_modules`, `.git`, `dist`, `build`, and vendor directories.
3. If `$ARGUMENTS` specifies a path or file glob, scope the search there instead.
4. Group results: **FIXME / XXX** (broken/dangerous) before **HACK** (workaround) before **TODO** (planned work).

## Output format

```
### FIXME / XXX  (<count>)
- `file:line` — <comment text>

### HACK  (<count>)
- `file:line` — <comment text>

### TODO  (<count>)
- `file:line` — <comment text>

### Summary
<total count, which files have the most, any obvious quick wins>
```

If a category is empty, omit it. Do not modify any files.
