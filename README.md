# claude-agents-library

A curated library of Claude Code subagents and skills. Agents are focused assistants Claude spawns automatically; skills are slash commands the user invokes directly.

---

## Agents

Agents live in `agents/`. Copy any `.md` file into:

- `~/.claude/agents/` — available in every project
- `<project>/.claude/agents/` — scoped to that project

Claude Code picks it up automatically on next launch.

| Agent | Purpose |
|---|---|
| [code-reviewer](agents/code-reviewer.md) | Reviews diffs for bugs, style, and regressions |
| [security-auditor](agents/security-auditor.md) | Scans changes for OWASP issues and leaked secrets |
| [python-specialist](agents/python-specialist.md) | Deep Python expertise — idioms, perf, framework-aware |
| [js-ts-specialist](agents/js-ts-specialist.md) | JavaScript/TypeScript across all frameworks (detects from project) |
| [web3-specialist](agents/web3-specialist.md) | Solidity, smart-contract security, ethers/viem |
| [codebase-explorer](agents/codebase-explorer.md) | Answers "where/how" questions without bloating main context |
| [fullstack-developer](agents/fullstack-developer.md) | Builds complete features spanning database, API, and frontend |
| [system-design-expert](agents/system-design-expert.md) | Designs and reviews scalable backend systems; covers databases, scaling, APIs, and security |
| [solidity-specialist](agents/solidity-specialist.md) | Writes, reviews, audits, and tests Solidity contracts and EVM dApps end-to-end |

---

## Skills

Skills live in `skills/`. Copy any `.md` file into:

- `~/.claude/commands/` — available in every project
- `<project>/.claude/commands/` — scoped to that project

Invoke with `/skill-name [arguments]` in any Claude Code session. Skills that are directories (e.g. `system-design/`) must be copied as a whole folder, not just the `SKILL.md` file inside.

| Skill | Command | Purpose |
|---|---|---|
| [explain](skills/explain.md) | `/explain <file-or-symbol>` | Explains what a file or function does and why |
| [standup](skills/standup.md) | `/standup [time-range]` | Generates a standup update from recent git activity |
| [todo](skills/todo.md) | `/todo [path]` | Lists and prioritizes TODO/FIXME/HACK comments |
| [system-design](skills/system-design/SKILL.md) | `/system-design [prompt]` | Designs scalable systems end-to-end with tradeoff reasoning and reference material |
| [solidity-dapp](skills/solidity-dapp/SKILL.md) | `/solidity-dapp [prompt]` | Writes, reviews, and secures Solidity contracts and web3 frontends with reference material |

---

## Authoring your own

- **New agent**: copy [TEMPLATE.md](TEMPLATE.md) → `agents/<name>.md`
- **New skill**: copy [skills/SKILL-TEMPLATE.md](skills/SKILL-TEMPLATE.md) → `skills/<name>.md`

See [CONTRIBUTING.md](CONTRIBUTING.md) for quality guidelines and review checklist.

## Principles

1. **One job per agent/skill** — composable beats monolithic.
2. **Specific `description`** — for agents, this decides whether they get invoked at all.
3. **Minimal tool surface** — least privilege; read-only when possible.
4. **Deterministic output** — structured report the parent can consume.
