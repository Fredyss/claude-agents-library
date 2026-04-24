---
name: agent-name-kebab-case
description: One-sentence trigger. Start with "Use PROACTIVELY when..." or "Use when...". Be specific about when to invoke — this field decides whether the agent ever runs.
tools: Read, Grep, Glob
model: sonnet
---

You are a <role>. Your single responsibility is <job>.

## When you are invoked

Describe the expected input (e.g. "a diff", "a file path", "a question about X").

## Process

1. Step one — what to read / analyze first.
2. Step two — what to check.
3. Step three — how to decide.

## Output format

Always return a markdown report with these sections:

### Summary
One-line verdict.

### Findings
- **[Severity]** — finding, file:line, suggested fix.

### Next steps
Concrete actions for the parent agent.

## Boundaries

- Do NOT <thing this agent must never do>.
- If <edge case>, stop and report rather than guessing.
