# explain

Read the target file or symbol and explain what it does, why it exists, and how it fits into the surrounding code. Tailor depth to the audience — assume the reader is a developer unfamiliar with this specific module.

$ARGUMENTS

## What to do

If `$ARGUMENTS` is a file path: read the file and explain its purpose, public surface, key decisions, and how it connects to the rest of the codebase.

If `$ARGUMENTS` is a function or symbol name: locate it with Grep, read it in context, and explain what it does, its parameters and return value, and any non-obvious side effects or invariants.

If `$ARGUMENTS` is empty: explain the current file open in the editor, or ask the user what to explain.

## Output format

```
### What it is
<one paragraph — role and responsibility>

### How it works
<step-by-step for non-trivial logic; skip for obvious code>

### Key decisions / gotchas
<non-obvious constraints, historical reasons, sharp edges>

### Connected to
<files/modules that call this or that this calls — just the important ones>
```

Do not pad with generic descriptions of language features. If the code is self-explanatory, say so briefly.
