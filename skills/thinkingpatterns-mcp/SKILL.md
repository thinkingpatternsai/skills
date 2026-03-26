---
name: thinkingpatterns-mcp
description: "Security findings management for AI-assisted development. Use when running analysis agents, uploading scan results, or fixing findings in a project."
---

# ThinkingPatterns Security Platform

You have access to ThinkingPatterns tools for managing security findings. Tools are available via **MCP** (if configured) and **CLI** (`tp` commands). Both interfaces cover the same core operations — use whichever is available, preferring MCP tools when both are present for structured reliability.

## Setup check

Before starting, determine which interface is available:
- Check if MCP tools are available by attempting `list_projects`
- If MCP is not available, run `tp --version` — if it succeeds, CLI is installed
- Run `tp whoami` — if it succeeds, CLI is authenticated
- If not authenticated, run `tp login` to save credentials
- If neither works, install via: `npx skills add thinkingpatternsai/skills -s thinkingpatterns-mcp` (installs this skill for your agent)
- If `tp` is not installed but Node.js is available, use `npx @thinkingpatterns/cli@latest` as a prefix. For best performance, install globally: `npm install -g @thinkingpatterns/cli`

## Interface selection

| Situation | Use | Why |
|-----------|-----|-----|
| MCP tools are configured | **MCP** | Structured protocol, rich schemas, dynamic discovery |
| Only CLI (`tp`) is available | **CLI** | Full coverage via shell commands |
| First time exploring the platform | **MCP** | Dynamic discovery of available tools |
| Bulk operations (10+ findings) | **CLI** | Loop in shell, no per-call LLM overhead |
| CI/CD pipeline or script | **CLI** | Deterministic, no LLM required |

**Rule of thumb:** If MCP tools are available (try `list_projects`), prefer MCP. Fall back to CLI when MCP is unavailable.

## Available operations

| Operation | MCP tool | CLI command |
|-----------|----------|-------------|
| List findings | `list_findings` | `tp findings list --project <id> [--severity critical] [--status open] [--verbose]` |
| Get finding detail | `get_finding` | `tp findings get <finding-id>` |
| Triage a finding | `triage_finding` | `tp findings triage <finding-id> --action accept [--reason "..."]` |
| Bulk triage | `bulk_triage` | `tp findings bulk-triage --ids <id1,id2,...> --action dismiss --reason "..."` |
| Add comment | `add_comment` | `tp findings comment <finding-id> --body "..."` |
| List projects | `list_projects` | `tp projects list` |
| Get metrics | `get_metrics` | `tp metrics --project <id>` |
| Top risk modules | `get_top_risk_modules` | `tp risk-modules --project <id> [--limit 10]` |
| List runs | `list_runs` | `tp runs list --project <id>` |
| Ingest SARIF | `ingest_sarif` | `tp ingest sarif --project <id> --file results.sarif [--branch <name>]` |
| List agents | `list_agents` | `tp agents list` |
| Get agent | `get_agent` | `tp agents get <agent-id>` |
| Create agent | `create_agent` | `tp agents create --name "Name" --slug name --skill-md ./SKILL.md` |
| Run agent | `run_agent` | `tp agents run <agent-id> --project <id>` |
| Check auth | — | `tp whoami` |

## Common workflows

### Security review of current state

**Via MCP:**
1. `list_projects` to find the relevant project
2. `get_metrics` for the project health overview (MTTR, fix rate, open count)
3. `list_findings` with `severity: "critical"` then `severity: "high"`
4. `get_finding` for each critical finding to understand details
5. Present a summary to the user with recommendations

**Via CLI:**
```bash
tp projects list
tp metrics --project <id>
tp findings list --project <id> --severity critical --status open
tp findings list --project <id> --severity high --status open
tp findings get <finding-id>   # for each critical finding
```

### Fix findings

1. List open findings: `list_findings` or `tp findings list --project <id> --status open` (returns summaries)
2. For each finding, get full details: `get_finding` or `tp findings get <id>` (includes description, remediation_text, etc.)
3. Read the code at `code_location_file`:`code_location_line_start`, understand the issue
4. Apply the minimal fix that resolves the root cause
5. Comment what was changed: `add_comment` or `tp findings comment <id> --body "..."`
6. Close the finding: `triage_finding` or `tp findings triage <id> --action complete`
7. For one fix resolving multiple findings, use bulk triage:
   - MCP: `bulk_triage` with `action: "complete"`
   - CLI: `tp findings bulk-triage --ids <id1,id2,id3> --action complete --reason "Fixed in commit abc123"`

### Post-code-change security check

1. List findings in the changed area:
   - MCP: `list_findings` with `module_path: "src/lib/auth/"`
   - CLI: `tp findings list --project <id> --module-path src/lib/auth/`
2. Review any findings in the modified area
3. Fix and complete findings as appropriate

### Bulk operations (CLI can be more efficient)

For operations touching many findings at once, CLI can be more efficient:

```bash
# Dismiss all info-level findings in test files
tp findings list --project <id> --severity info --module-path test/ \
  | jq -r '.data[].id' \
  | xargs -I{} tp findings triage {} --action dismiss --reason "Test file, informational only"

# Export critical findings as JSON for reporting
tp findings list --project <id> --severity critical > critical-findings.json
```

### Run an agent analysis

1. Discover agents: `list_agents` or `tp agents list`
2. Run the agent: `run_agent` or `tp agents run <agent-id> --project <id>`
3. Follow the returned SKILL.md workflow step by step, using available tools
4. When done, ingest results: `ingest_sarif` (include `branch` parameter) or `tp ingest sarif --project <id> --file results.sarif --branch <branch>`

### Manage agents

1. List agents: `list_agents` or `tp agents list`
2. Inspect an agent: `get_agent` or `tp agents get <agent-id>`
3. Create a new agent: `create_agent` or `tp agents create --name "My Agent" --slug my-agent --skill-md ./SKILL.md`
4. Agents are workflow definitions (SKILL.md + MCP configs) that always output SARIF

## Important notes

- The `project_id` is a UUID. Use `list_projects` or `tp projects list` first if you don't know it
- If a `.thinkingpatterns.json` file exists in the repo root, it contains the `project_id` — CLI reads it automatically so `--project` can be omitted
- Findings have `code_location_file` and `code_location_line_start` fields — use these to navigate to affected code
- Metrics include MTTR (mean time to resolution in days), fix_rate_7d (percentage), and rework_ratio (findings reopened)
- Triage decisions (accept, dismiss, defer, reopen) are handled by humans — focus on scanning and fixing
- `tp findings list` returns summary fields by default (id, title, severity, status, category, file, line, first_seen_at). Use `--verbose` for all fields. `tp findings get <id>` always returns full detail
- CLI outputs JSON by default, making it easy to pipe to `jq` or other tools
- Run `tp login` to authenticate once — credentials are saved to `~/.thinkingpatterns/credentials.json`
- CLI commands also accept `--api-key` flag or `TP_API_KEY` env var as overrides (useful for CI/CD)
- Always pass `--branch` (CLI) or the `branch` parameter (MCP) when ingesting findings — without it, findings cannot be filtered by branch in the UI
