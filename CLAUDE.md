# CLAUDE.md

This file provides a fast structural overview of the project for Claude Code agents. Agents should read this first before scanning the full codebase.

## Project Overview

- **Project**: claude-workflow-setup
- **Purpose**: Reusable, stack-agnostic development workflow powered by Claude Code agents
- **Stack**: Configuration-only (YAML/JSON/Markdown frontmatter). No runtime code.
- **MCP**: Postgres + MySQL connections available via `.claude/mcp.json`. Template at `.claude/mcp.json.tmpl`.

## Directory Structure

```
claude-workflow-setup/
├── .ai-logs/                    # Committed — AI dev summaries + lessons learned
├── .claude/
│   ├── agents/
│   │   ├── planner-agent.md     # Opus — comprehensive dev planning
│   │   ├── implementer-agent.md # Sonnet — plan execution + unit tests
│   │   ├── reviewer-agent.md    # Opus — quality gate, plan-vs-implementation
│   │   └── finalizer-agent.md   # Haiku — summary, branch, commit, push
│   ├── settings.json            # Project-level Claude settings
│   ├── settings.local.json      # Local overrides (git-ignored)
│   ├── mcp.json                 # MCP server configs (git-ignored — credentials)
│   └── mcp.json.tmpl            # MCP config template (committed)
├── .plans/                      # Git-ignored — development plans
├── .gitignore
├── CLAUDE.md                    # This file — fast project index for agents
└── README.md                    # Human-facing workflow documentation
```

## Key Conventions

- **Agent files**: Markdown with YAML frontmatter (`name`, `description`, `model`)
- **Model selection**: Opus for planning/review, Sonnet for implementation, Haiku for finalization
- **Plan format**: `.plans/YYYYMMDD-slug.md` — git-ignored, local only
- **Summary format**: `.ai-logs/YYYYMMDD-slug.md` — committed to repo
- **Branch naming**: `<prefix>/<short-description>` (feat/, fix/, docs/, chore/, refactor/, test/, perf/)
- **MCP setup**: Copy `.claude/mcp.json.tmpl` → `.claude/mcp.json`, fill in credentials. `mcp.json` is git-ignored.

## Tech Stack

- **Language**: None (configuration project)
- **Tooling**: Claude Code CLI with custom agent definitions
- **Version control**: Git

## Last Updated

- **Date**: 2026-08-02
- **By**: Manual setup
