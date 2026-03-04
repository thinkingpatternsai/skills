---
name: thinkingpatterns
description: "Security findings management for AI-assisted development. Use when running analysis agents, uploading scan results, or fixing findings in a project."
---

# ThinkingPatterns Security Platform

You have access to ThinkingPatterns MCP tools for managing security findings. Use them to help the user understand and improve their project's security posture. See the [tool reference](../../docs/tools.md) for full parameter details.

## Available Tools

| Tool | Purpose | When to use |
|------|---------|-------------|
| `list_findings` | Query findings by project, severity, status, module path | Starting point for any security review |
| `get_finding` | Full detail on a specific finding (code location, description) | After identifying a finding to investigate |
| `triage_finding` | Mark a finding as complete after fixing | After applying a fix |
| `bulk_triage` | Complete multiple findings with the same action | When one fix resolves multiple findings |
| `add_comment` | Annotate a finding with context | Documenting what was changed and why |
| `trigger_scan` | Start a security scan via GitHub Actions | After code changes or on request |
| `get_run_status` | Check if a scan is running/completed | After triggering a scan |
| `list_runs` | View scan history for a project | Checking recent scan activity |
| `list_projects` | List all projects in the org | Discovery, context switching |
| `get_metrics` | MTTR, fix rate, open count, rework ratio | Project health overview |
| `get_top_risk_modules` | Highest-risk file paths by severity | Prioritizing remediation work |
| `ingest_sarif` | Upload local SARIF scan results | After running a local scanner |
| `run_agent` | Fetch an agent's workflow and execute it against a project | Running a quality analysis agent |
| `list_agents` | List quality analysis agents in your org | Browse/manage agent definitions |
| `get_agent` | Full agent detail (SKILL.md + MCP configs) | Inspect an agent's workflow |
| `create_agent` | Create a new agent from SKILL.md + MCP configs | Authoring new agents |

## Common Workflows

### Security review of current state
1. `list_projects` to find the relevant project
2. `get_metrics` for the project health overview (MTTR, fix rate, open count)
3. `list_findings` with `severity: "critical"` or `severity: "high"` to see the most urgent issues
4. `get_finding` for each critical finding to understand the details
5. Present a summary to the user with recommendations

### Fix findings
1. `list_findings` filtered by status `"open"` to see items that need attention
2. For each finding, `get_finding` to review details
3. Read the code at `code_location_file`:`code_location_line_start`, understand the issue
4. Apply the minimal fix that resolves the root cause
5. `add_comment` explaining what was changed and why
6. `triage_finding` with `action: "complete"` to close the finding
7. Use `bulk_triage` with `action: "complete"` when one fix resolves multiple findings

### Post-code-change security check
1. `list_findings` with `module_path` set to the changed file/directory path
2. Review any findings in the modified area
3. Fix and complete findings as appropriate

### Trigger and monitor a scan
1. `trigger_scan` with the project ID
2. `get_run_status` to check progress (poll if needed)
3. Once completed, `list_findings` with `status: "open"` to see new findings

### Run an agent analysis
1. `list_agents` to discover available agents (filter by `category` if needed)
2. `run_agent` with the `agent_id` and `project_id` — this returns the agent's full workflow instructions
3. Follow the returned instructions step by step, using available MCP tools
4. When done, call `ingest_sarif` with the project ID and SARIF results (the instructions will remind you)

### Manage agents
1. `list_agents` to see all quality analysis agents in your org
2. `get_agent` to inspect a specific agent's SKILL.md workflow and MCP configuration
3. `create_agent` with `skill_md` content and optional `mcp_configs` to register a new agent
4. Agents are workflow definitions (SKILL.md + MCP configs) that always output SARIF

## Important Notes

- The `project_id` is a UUID. Use `list_projects` first if you don't know it
- If a `.thinkingpatterns.json` file exists in the repo root, it contains the `project_id`
- Findings have detailed `code_location_file` and `code_location_line_start` fields — use these to navigate to the affected code
- Metrics include MTTR (mean time to resolution in days), fix_rate_7d (percentage), and rework_ratio (findings that were reopened)
- Triage decisions (accept, dismiss, defer, reopen) are handled by humans — focus on scanning and fixing
