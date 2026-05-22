# Contributing

## Adding an agent

1. Copy [TEMPLATE.md](TEMPLATE.md) to `agents/<name>.md`.
2. Fill in frontmatter and body.
3. Add a row to the agents table in [README.md](README.md).
4. Open a PR.

## Adding a skill

1. Copy [skills/SKILL-TEMPLATE.md](skills/SKILL-TEMPLATE.md) to `skills/<name>.md`.
2. Fill in the body — no frontmatter needed for skills.
3. Add a row to the skills table in [README.md](README.md).
4. Open a PR.

---

## Quality bar — agents

Your agent should pass **at least 4 of these 7** checks:

- [ ] **Frequency** — invoked weekly, not yearly
- [ ] **Delegation clarity** — description makes "when to call" obvious
- [ ] **Context isolation** — protects parent context from noise
- [ ] **Parallelizable** — can run alongside other agents
- [ ] **Deterministic output** — explicit output format
- [ ] **Narrow scope** — one job, not three
- [ ] **Tool minimalism** — ≤ 6 tools, read-only when possible

## Quality bar — skills

Your skill should pass **at least 3 of these 5** checks:

- [ ] **User-initiated** — something the user consciously wants to trigger, not a background task
- [ ] **Arguments documented** — explains what happens when `$ARGUMENTS` is empty or malformed
- [ ] **Deterministic output** — explicit output format with section headings
- [ ] **Narrow scope** — one job; does not duplicate a built-in Claude Code skill
- [ ] **Safe by default** — read-only unless the user explicitly asked for edits

---

## Style (both)

- **No persona fluff**: skip "You are a 10x engineer with 20 years of experience". Plain role statements perform better.
- **Write descriptions as triggers** (agents): "Use PROACTIVELY after any edit to `.py` files" beats "Helps with Python".
- **Specify output format explicitly**: agents and skills without it return inconsistent shapes.
- **System prompt / body length**: 30–150 lines. Longer is usually bloat.

## Review checklist — agents

- Frontmatter has `name`, `description`, and `tools`.
- Agent name is kebab-case and matches filename.
- Tools list is minimal — justify anything beyond read/search.
- System prompt has a clear `## Output format` section.
- Description starts with "Use..." and names concrete triggers.

## Review checklist — skills

- Filename is kebab-case (becomes the slash command name).
- `$ARGUMENTS` is present and its handling is documented.
- Skill has a clear output format section.
- Skill does not silently modify files unless that is its explicit purpose.
