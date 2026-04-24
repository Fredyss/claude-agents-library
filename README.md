# claude-agents-library

A curated library of Claude Code subagents. Each agent is a focused, single-purpose assistant stored as a markdown file with YAML frontmatter.

## Install

Copy any `.md` file from `agents/` into one of:

- `~/.claude/agents/` — available in every project
- `<project>/.claude/agents/` — scoped to that project

Claude Code picks it up automatically on next launch.

## Available agents

| Agent | Purpose |
|---|---|
| [code-reviewer](agents/code-reviewer.md) | Reviews diffs for bugs, style, and regressions |
| [security-auditor](agents/security-auditor.md) | Scans changes for OWASP issues and leaked secrets |
| [python-specialist](agents/python-specialist.md) | Deep Python expertise — idioms, perf, framework-aware |
| [js-ts-specialist](agents/js-ts-specialist.md) | JavaScript/TypeScript across all frameworks (detects from project) |
| [web3-specialist](agents/web3-specialist.md) | Solidity, smart-contract security, ethers/viem |
| [codebase-explorer](agents/codebase-explorer.md) | Answers "where/how" questions without bloating main context |
| [fullstack-developer](agents/fullstack-developer.md) | Builds complete features spanning database, API, and frontend as a cohesive unit |

## Authoring your own

See [TEMPLATE.md](TEMPLATE.md) and [CONTRIBUTING.md](CONTRIBUTING.md).

## Principles

1. **One job per agent** — composable beats monolithic.
2. **Specific `description`** — it decides whether the agent gets invoked at all.
3. **Minimal tool surface** — least privilege; read-only when possible.
4. **Deterministic output** — structured report the parent can consume.
