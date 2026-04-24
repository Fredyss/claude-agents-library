# Contributing

## Adding an agent

1. Copy [TEMPLATE.md](TEMPLATE.md) to `agents/<name>.md`.
2. Fill in frontmatter and body.
3. Add a row to the table in [README.md](README.md).
4. Open a PR.

## Quality bar

Your agent should pass **at least 4 of these 7** checks:

- [ ] **Frequency** — invoked weekly, not yearly
- [ ] **Delegation clarity** — description makes "when to call" obvious
- [ ] **Context isolation** — protects parent context from noise
- [ ] **Parallelizable** — can run alongside other agents
- [ ] **Deterministic output** — explicit output format
- [ ] **Narrow scope** — one job, not three
- [ ] **Tool minimalism** — ≤ 6 tools, read-only when possible

## Style

- **System prompt length**: 40–200 lines. Longer is usually bloat.
- **No persona fluff**: skip "You are a 10x engineer with 20 years of experience". Plain role statements perform better.
- **Write descriptions as triggers**: "Use PROACTIVELY after any edit to `.py` files" beats "Helps with Python".
- **Specify output format explicitly**: agents without it return inconsistent shapes.

## Review checklist

Before merging, verify:

- Frontmatter has `name`, `description`, and `tools`.
- Agent name is kebab-case and matches filename.
- Tools list is minimal — justify anything beyond read/search.
- System prompt has a clear `## Output format` section.
- Description starts with "Use..." and names concrete triggers.
