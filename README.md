# Agent Skills

A collection of specialized skills for AI coding agents, designed to enhance code analysis and security workflows.

## Overview

This repository contains two types of agent skills:

1. **Scanning Agents** - Deep code analysis skills that go beyond traditional static analysis
2. **MCP Integration** - Skills that teach agents how to integrate with the ThinkingPatterns platform

## Scanning Agents

Located in `scanning-agents/`, these skills enable agents to perform comprehensive, context-aware code analysis. Unlike pattern-matching scanners, these skills guide agents to understand code flow, trace data, and identify issues that require semantic understanding.

### Available Scanning Agents

| Skill | Category | Purpose |
|-------|----------|---------|
| `security-deep-analysis.md` | Security | Contextual security analysis: auth flows, data propagation, trust boundaries, business logic flaws |
| `architecture-review.md` | Architecture | Structural health analysis: dependency direction, module boundaries, layering violations, API consistency |
| `reliability-review.md` | Reliability | Error handling, failure modes, observability, resilience patterns |
| `scalability-review.md` | Scalability | Performance bottlenecks, resource usage, concurrency issues, scalability patterns |

### How Scanning Agents Work

Each scanning agent provides:
- **Analysis Strategy**: Step-by-step methodology for exploring the codebase
- **What to Look For**: Specific patterns, anti-patterns, and vulnerability classes
- **Severity Classification**: Guidelines for rating findings (critical, high, medium, low, info)
- **Reporting**: SARIF-compatible rule ID prefixes and structured finding formats

### Using Scanning Agents

These skills are designed to be used with Claude Code or other AI coding assistants that support custom skills:

1. **As a Claude Code skill**: Place the skill file in `.claude/skills/<skill-name>/SKILL_.md`
2. **Direct prompt**: Copy the skill content into your conversation with an AI agent
3. **Via ThinkingPatterns**: Use the `run_agent` MCP tool to trigger these analyses programmatically

Example workflow:
```bash
# Install as a Claude Code skill
mkdir -p .claude/skills/security-deep-analysis
cp scanning-agents/security-deep-analysis.md .claude/skills/security-deep-analysis/SKILL_.md

# Run the analysis
claude-code
# Then in Claude Code:
/security-deep-analysis
```

## MCP Integration

Located in `mcp-integration/`, this skill teaches coding agents how to interact with the ThinkingPatterns platform.

### ThinkingPatterns MCP Skill

**File**: `thinkingpatterns.md`

This skill provides:
- Architecture overview of the ThinkingPatterns codebase
- Key conventions for DB layer, API routes, authentication, and schemas
- MCP server implementation patterns
- Ingestion pipeline flow
- Common development tasks

**Use this skill when**:
- Working on the ThinkingPatterns Next.js application
- Developing or modifying MCP tools
- Building integrations with the ThinkingPatterns API
- Contributing to the ingestion pipeline or agent system

### Installing the MCP Skill

```bash
# For development on the ThinkingPatterns codebase
mkdir -p .claude/skills/thinkingpatterns
cp mcp-integration/thinkingpatterns.md .claude/skills/thinkingpatterns/SKILL_.md
```

## Skill Format

All skills follow the frontmatter convention:

```markdown
---
name: Skill Name
description: Brief description of what the skill does
category: security|architecture|reliability|scalability
---

# Skill Name

[Detailed instructions for the agent...]
```

## Output Format

Scanning agents produce findings in SARIF-compatible format with:
- **Rule IDs**: Prefixed by category (e.g., `security/auth-bypass`, `architecture/circular-dependency`)
- **Severity levels**: `critical`, `high`, `medium`, `low`, `info`
- **Structured descriptions**: `shortDescription` (1-sentence title), `fullDescription` (detailed explanation)
- **Contextual messages**: Instance-specific details with file paths and line numbers

## Integration with ThinkingPatterns

These skills are designed to work seamlessly with the ThinkingPatterns platform:

1. **Run agents via MCP**: Use the `run_agent` tool to trigger scanning agents
2. **Ingest results**: Agents output SARIF, which can be ingested via `ingest_sarif` MCP tool
3. **Track findings**: Results appear in the ThinkingPatterns dashboard with full context
4. **Continuous monitoring**: Schedule agents to run on every commit or PR

## Contributing

To add a new scanning agent:

1. Create a new `.md` file in `scanning-agents/`
2. Include proper frontmatter with name, description, and category
3. Follow the structure: Analysis Strategy → What to Look For → Severity Classification → Reporting
4. Define clear rule ID prefixes for your findings
5. Provide concrete examples and patterns to search for

## License

[Add your license here]

## Related Projects

- [ThinkingPatterns](https://github.com/yourusername/thinkingpatterns) - AI-assisted security findings management platform
- [Claude Code](https://claude.ai/claude-code) - AI-powered CLI coding assistant
