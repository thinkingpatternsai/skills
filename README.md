# Agent Skills

A collection of specialized skills for AI coding agents, designed to enhance code analysis and security workflows.

## Overview

This repository contains skills for AI coding agents.

### `npx skills`-discoverable (installed automatically)

```
skills/
  thinkingpatterns/SKILL.md        — MCP + CLI platform integration (CLI-preferred)
  thinkingpatterns-mcp/SKILL.md    — MCP + CLI platform integration (MCP-preferred)
```

### Scanning agents (not auto-installed)

```
scanning-agents/
  security-deep-analysis/SKILL.md — Contextual security analysis
  architecture-review/SKILL.md    — Structural health analysis
  reliability-review/SKILL.md     — Error handling & resilience
  scalability-review/SKILL.md     — Performance & scalability
```

## Installation

Install the ThinkingPatterns skill into your coding agent:

```bash
# CLI-preferred (default)
npx skills add thinkingpatternsai/skills -s thinkingpatterns -a claude-code -y

# MCP-preferred (for MCP-first setups)
npx skills add thinkingpatternsai/skills -s thinkingpatterns-mcp -a claude-code -y
```

Replace `claude-code` with your agent slug (`cursor`, `copilot`, `codex`, `antigravity`).

Only the `skills/` directory is discovered by `npx skills`. Scanning agents in `scanning-agents/` are used internally by the platform via the `run_agent` API.

## Skills

### ThinkingPatterns (two variants)

Teaches coding agents how to interact with the ThinkingPatterns platform via CLI (`tp` commands) and MCP tools — list/fix/triage findings, trigger scans, manage agents.

- **`thinkingpatterns`** — Prefers CLI when available (10-32x cheaper in tokens). Best for CLI-first setups.
- **`thinkingpatterns-mcp`** — Prefers MCP tools when available (structured protocol, rich schemas). Best for MCP-first setups.

### Scanning Agents

Deep code analysis skills that go beyond traditional static analysis. Each produces SARIF-compatible findings. These are **not** installed by `npx skills` — they are executed server-side via `run_agent`.

| Skill | Category | Purpose |
|-------|----------|---------|
| `security-deep-analysis` | Security | Auth flows, data propagation, trust boundaries, business logic flaws |
| `architecture-review` | Architecture | Dependency direction, module boundaries, layering violations, API consistency |
| `reliability-review` | Reliability | Error handling, failure modes, observability, resilience patterns |
| `scalability-review` | Scalability | Performance bottlenecks, resource usage, concurrency issues |

## Output Format

Scanning agents produce findings in SARIF-compatible format with:
- **Rule IDs**: Prefixed by category (e.g., `security/auth-bypass`, `architecture/circular-dependency`)
- **Severity levels**: `critical`, `high`, `medium`, `low`, `info`
- **Structured descriptions**: `shortDescription` (1-sentence title), `fullDescription` (detailed explanation)
- **Contextual messages**: Instance-specific details with file paths and line numbers

## Integration with ThinkingPatterns

1. **Run agents via MCP**: Use the `run_agent` tool to trigger scanning agents
2. **Ingest results**: Agents output SARIF, which can be ingested via `ingest_sarif` MCP tool
3. **Track findings**: Results appear in the ThinkingPatterns dashboard with full context
4. **Continuous monitoring**: Schedule agents to run on every commit or PR

## Contributing

To add a new scanning agent:

1. Create a new directory `skills/<skill-name>/`
2. Add a `SKILL.md` file with proper frontmatter (name, description, category)
3. Follow the structure: Analysis Strategy → What to Look For → Severity Classification → Reporting
4. Define clear rule ID prefixes for your findings
