# CLAUDE.md

This file provides a fast structural overview of the project for Claude Code agents. Agents should read this first before scanning the full codebase.

## Project Overview

- **Project**: claude-workflow-setup
- **Purpose**: Reusable, stack-agnostic development workflow powered by Claude Code agents
- **Stack**: Configuration-only (YAML/JSON/Markdown frontmatter). No runtime code.
- **MCP**: Template at `.claude/mcp.json.tmpl` with Postgres, MySQL, Redis, and four ticket platforms (Jira, Trello, ClickUp, Asana). No servers are connected until you copy it to `.claude/mcp.json` and fill in real credentials. Jira uses Atlassian's hosted OAuth endpoint; Redis uses the official Python server via `uvx` (requires `uv`); the rest take tokens in `env`.

## Architecture: Orchestrator + Report-Back Workers

There is exactly one user-facing participant: the **orchestrator** (`/dev`). It spawns each subagent, receives a structured report, and decides the next step. Subagents never talk to each other and never talk to the user.

```
User ──▶ /dev (orchestrator, main thread) ──▶ planner-agent  ──report──┐
                                          ◀───────────────────────────┘
                                          ──▶ implementer-agent ──report──▶ review loop
                                          ──▶ reviewer-agent  ──report──▶ (max 3 rounds)
                                          ──▶ finalizer-agent ──report──▶ user
```

- Every subagent ends with a mandatory **Output Contract** block (`STATUS`/`VERDICT`, path, and detail fields) that the orchestrator parses.
- Handoffs are **paths + short reports**, never pasted plan bodies.
- Only the orchestrator writes `CLAUDE.md`; agents propose edits in their report instead.

## Testing Agents (outside the /dev pipeline)

Two standalone agents verify a test case against an implementation. They are **not** part of the `/dev` flow and have **no slash command** — they are spawned when the user's prompt asks for end-to-end verification. As with every subagent, they report back and cannot ask the user; the caller does intake.

| Agent | Network | Purpose |
|---|---|---|
| `e2e-static-agent` | **none, ever** | Traces the code path and compares request contract, business logic, and response shape against the test case. Read-only; no `Bash`. |
| `e2e-runtime-agent` | live API | Executes the real request and compares the live response. Read-only verbs by default; optional MCP database checks. |

**Intake belongs to the caller, not the agents.** The main thread fetches the ticket (via MCP) or reads the PDF, normalizes it to `.test-reports/YYYYMMDD-slug-testcase.md`, and passes that path to the agent. This is what keeps `e2e-static-agent` provably network-free.

- **Static verdicts**: `MATCH`, `MISMATCH`, `NOT_FOUND`, `UNVERIFIABLE`. Anything decided at runtime outside the visible source is `UNVERIFIABLE`, never an optimistic `MATCH`.
- **Runtime verdicts**: `PASS`, `FAIL`, `SKIPPED`, `UNVERIFIED`.
- **Write safety**: `e2e-runtime-agent` sends only `GET`/`HEAD` unless the brief carries `ALLOW_WRITES: yes` for that run. A curl alone is not permission.
- **Reports** go to `.test-reports/` (git-ignored). Ask the user first whether they want the report sent to Slack or another platform; default to the local file only.

## Directory Structure

```
claude-workflow-setup/
├── .ai-logs/                    # Committed — AI dev summaries + lessons learned
│   └── .gitkeep
├── .claude/
│   ├── commands/
│   │   └── dev.md               # Orchestrator — the only user-facing entry point
│   ├── agents/
│   │   ├── planner-agent.md     # Opus — dev planning, read-only except .plans/
│   │   ├── implementer-agent.md # Sonnet — plan execution + unit tests
│   │   ├── reviewer-agent.md    # Opus — read-only quality gate, no code edits
│   │   ├── finalizer-agent.md   # Haiku — summary, branch, commit, push
│   │   ├── e2e-static-agent.md  # Opus — code-path verification, zero network
│   │   └── e2e-runtime-agent.md # Opus — live request verification, read-only default
│   ├── settings.json            # Project-level Claude settings
│   ├── settings.local.json      # Local overrides (git-ignored)
│   ├── mcp.json                 # MCP server configs (git-ignored — credentials)
│   └── mcp.json.tmpl            # MCP config template (committed)
├── .plans/                      # Git-ignored — development plans
├── .test-reports/               # Git-ignored — E2E verification evidence
├── .gitignore
├── CLAUDE.md                    # This file — fast project index for agents
└── README.md                    # Human-facing workflow documentation
```

## Key Conventions

- **Agent files**: Markdown with YAML frontmatter (`name`, `description`, `model`, `tools`). `tools` is scoped per role — the reviewer is deliberately read-only.
- **Model selection**: Opus for planning/review, Sonnet for implementation, Haiku for finalization
- **Entry point**: `/dev <request>`. Do not invoke agents ad hoc to run a full cycle — that loses the review loop and the human checkpoints.
- **MR/PR creation**: the finalizer detects `gh` → `GITHUB_TOKEN`/`GH_TOKEN` → compare link (GitHub), or `glab` → `GITLAB_TOKEN` (GitLab). Missing setup is reported as `MR_SETUP: missing`, not a failure; the orchestrator offers setup help.
- **Secrets**: tokens are detected by presence only (`${VAR:+set}`), never echoed. `settings.local.json` and `mcp.json` are git-ignored.
- **Output Contract**: every agent ends with a parseable report block. Changing an agent's fields means changing the orchestrator's parsing in `.claude/commands/dev.md`.
- **Plan format**: `.plans/YYYYMMDD-slug.md` — git-ignored, local only
- **Summary format**: `.ai-logs/YYYYMMDD-slug.md` — committed to repo, same slug as the plan
- **Test case / report format**: `.test-reports/YYYYMMDD-slug-{testcase,static,runtime}.md` — git-ignored, local only
- **Branch naming**: `<prefix>/<short-description>` (feat/, fix/, docs/, chore/, refactor/, test/, perf/)
- **MCP setup**: Copy `.claude/mcp.json.tmpl` → `.claude/mcp.json`, fill in credentials. `mcp.json` is git-ignored.

## Tech Stack

- **Language**: None (configuration project)
- **Tooling**: Claude Code CLI with custom agent definitions and a slash command
- **Version control**: Git

## Last Updated

- **Date**: 2026-11-24
- **By**: Added standalone E2E verification agents
