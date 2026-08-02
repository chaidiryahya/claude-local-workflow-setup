# Claude Development Workflow

A reusable, stack-agnostic development workflow powered by Claude Code agents. Designed to work with any language — Go, JavaScript, Python, or anything else.

## Overview

Four specialized agents form a complete development pipeline:

```
User Request
    │
    ▼
┌─────────────────────────┐
│  PLANNER (Opus)         │  Comprehensive planning from PDFs, MCP tickets, or prompts.
│  Size detect → Plan     │  Asks clarifying questions. Breaks large work into phases.
│  → .plans/YYYYMMDD.md   │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  IMPLEMENTER (Sonnet)   │  Executes the plan phase by phase.
│  Code + unit tests      │  No planning — follows the plan exactly.
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐     ┌─────────────────────┐
│  REVIEWER (Opus)        │────▶│ Remediation list    │
│  Plan vs implementation │     │ → back to IMPLEMENTER│
│  Gaps / bugs / tests    │     └─────────────────────┘
└──────────┬──────────────┘     (max 3 rounds)
           │ APPROVED
           ▼
┌─────────────────────────┐
│  FINALIZER (Haiku)      │  Summary → .ai-logs/
│  Branch → commit → push │  Ready for PR/MR
└─────────────────────────┘
```

## Agents

| Agent | Model | File | Role |
|---|---|---|---|
| **Planner** | Opus | [planner-agent.md](.claude/agents/planner-agent.md) | Analyze requirements, ask clarifying questions, produce phased plan |
| **Implementer** | Sonnet | [implementer-agent.md](.claude/agents/implementer-agent.md) | Execute plan, write code, write unit tests |
| **Reviewer** | Opus | [reviewer-agent.md](.claude/agents/reviewer-agent.md) | Compare plan vs implementation, find gaps/bugs, re-spawn implementer |
| **Finalizer** | Haiku | [finalizer-agent.md](.claude/agents/finalizer-agent.md) | Write TL;DR summary, create branch, commit, push |

### Planner in Detail

- Accepts: PDFs (technical docs), MCP tickets (Jira/ClickUp/Trello), direct prompts
- **Size detection**: XS tasks skip the plan entirely → straight to implementer
- **Decomposition**: S/M/L/XL sizing with concrete phase counts
- **Human checkpoints**: asks whether to review the plan before handing off
- Output: structured plan written to `.plans/YYYYMMDD-slug.md`

### Reviewer Loop

The reviewer compares the implementation against the plan. If issues are found, it produces a concrete **remediation list** and re-spawns the implementer. This loops until clean (max 3 rounds, then escalates).

## Directory Structure

```
project/
├── .claude/
│   └── agents/
│       ├── planner-agent.md
│       ├── implementer-agent.md
│       ├── reviewer-agent.md
│       └── finalizer-agent.md
├── .plans/              # Git-ignored — plans stay local
│   └── YYYYMMDD-slug.md
├── .ai-logs/            # Committed — AI development summaries
│   └── YYYYMMDD-slug.md
└── .gitignore
```

- **`.plans/`** — local only, never committed. Contains the planner's output.
- **`.ai-logs/`** — committed to the repo. Contains TL;DR summaries and lessons learned from each development cycle.

## How to Use

### 1. Start with the Planner

Describe what you want to build, or point the planner at a PDF or MCP ticket:

```
/invoke planner-agent "Add user authentication with JWT and refresh tokens"
```

Or reference a ticket:
```
/invoke planner-agent "Implement the changes from JIRA ticket PROJ-123"
```

### 2. Review the Plan (Optional)

The planner will write the plan to `.plans/` and ask if you want to review. Take your time — this is the right moment to catch misunderstandings.

### 3. Hand Off to the Implementer

Once the plan is approved, the implementer executes it phase by phase, writing code and unit tests.

### 4. Reviewer Quality Gate

The reviewer checks everything against the plan. If something is off, it tells the implementer exactly what to fix. This repeats until clean.

### 5. Finalize

The finalizer writes the summary to `.ai-logs/`, creates a conventional-commit branch, and pushes to remote. Create your PR/MR from there.

## Stack-Agnostic

This workflow doesn't care what language you're using. The planner produces language-agnostic plans. The implementer adapts to whatever project conventions it finds. The reviewer checks against the plan, not language-specific rules.

For stack-specific behavior, add instructions to your project's `CLAUDE.md`.

## Setup

1. Copy `.claude/agents/`, `.gitignore`, `.plans/`, `.ai-logs/`, and `CLAUDE.md` into any project.
2. Copy `.claude/mcp.json.tmpl` → `.claude/mcp.json` and fill in your database credentials (if using MCP).
3. The agents will work immediately — no additional configuration needed.

### MCP Database Connections

PostgreSQL and MySQL MCP servers are pre-configured in the template:

```bash
cp .claude/mcp.json.tmpl .claude/mcp.json
# Edit .claude/mcp.json with your real credentials
```

`mcp.json` is git-ignored — it never gets committed.

## License

This workflow setup is free to use and modify for any project.
