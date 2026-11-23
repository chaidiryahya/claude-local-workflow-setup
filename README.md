# Claude Development Workflow

A reusable, stack-agnostic development workflow powered by Claude Code agents. Works with any language — Go, JavaScript, Python, or anything else.

## How It Works

One orchestrator, four workers. You talk to the orchestrator; the orchestrator talks to the agents. **Subagents never talk to each other and never talk to you** — each one gets a scoped brief, does its job, and returns a structured report.

```
You ──▶ ORCHESTRATOR (/dev) ──▶ planner ──▶ report
                       │                     │
                       │  asks you if the plan needs review
                       │                     │
                       ├──▶ implementer ──▶ report
                       ├──▶ reviewer ─────▶ report ──▶ fix loop (max 3)
                       ├──▶ implementer ──▶ report (only if findings)
                       └──▶ finalizer ────▶ report ──▶ summary + branch + push
```

Because the orchestrator holds the loop, it also holds the decisions: it resolves what it can from context, and asks you only what it genuinely cannot answer (ambiguous requirements, plan approval, whether to push).

## Agents

| Agent | Model | Tools | Role |
|---|---|---|---|
| **Orchestrator** [`/dev`](.claude/commands/dev.md) | session model | all | Drives the pipeline, relays reports, owns human checkpoints and `CLAUDE.md` |
| **Planner** | Opus | Read, Write (`.plans/` only), Grep, Glob, web | Produces a phased plan or declares "no plan needed" |
| **Implementer** | Sonnet | Read, Write, Edit, Grep, Glob, Bash | Executes a phase, writes and runs unit tests |
| **Reviewer** | Opus | Read, Grep, Glob, Bash | Read-only quality gate; reports findings, never edits |
| **Finalizer** | Haiku | Read, Write, Grep, Glob, Bash | Summary → branch → commit → push → open MR/PR |

The reviewer has no edit tools on purpose. It compares and reports; the orchestrator sends the fix list back to the implementer.

### The Report Contract

Every agent ends with a parseable block, and the orchestrator is what reads it. For example:

```
STATUS: PLAN_READY
PLAN_PATH: .plans/20261123-user-auth.md
SIZE: L
PHASES: 5
OPEN QUESTIONS:
1. "Fast response" — <100ms p95 needs a cache; <500ms does not. Which?
```

`STATUS: BLOCKED` plus a `BLOCKER` field is how an agent escalates. The orchestrator either answers from context or brings the question to you.

### Planner in Detail

- Accepts PDFs, MCP ticket references, and plain prompts — all passed as paths by the orchestrator.
- **Size detection**: XS requests skip the plan entirely and go straight to the implementer.
- **Phasing**: S/M/L/XL with phase counts; every phase is ~S-sized and ends green.
- **Ambiguity**: the planner cannot ask you anything, so unresolved questions come back as a structured list you answer in one pass.

### Review Loop

The reviewer checks every planned task for presence, fidelity, completeness, tests, bugs, side effects, and `CLAUDE.md` drift. Findings are categorized (`MISMATCH`, `GAP`, `BUG`, `TEST_GAP`, `STRUCTURE`) and returned as a remediation list the implementer can act on without guessing. The orchestrator caps this at **3 rounds**, then stops and escalates to you.

### MR / PR creation

The finalizer opens the merge request after pushing, using whatever is available:

| Remote | Detection order |
|---|---|
| GitHub | `gh` CLI → `GITHUB_TOKEN` / `GH_TOKEN` via REST API → compare link |
| GitLab | `glab` CLI → `GITLAB_TOKEN` via REST API → manual |

If neither is set up, the finalizer still pushes and reports `MR_SETUP: missing`. The orchestrator then asks whether you want to set up tooling now, use a token, or just get the compare link — the pipeline does **not** fail at that point.

**Token setup (no CLI install needed):**

```bash
# ~/.zshrc — do not commit this
export GITHUB_TOKEN="ghp_..."     # classic PAT, `repo` scope
# or fine-grained PAT: Contents: read + Pull requests: read and write
```

Restart the shell so Claude Code inherits it. Tokens are detected by presence only — they are never echoed into a report, commit, or log. A leaked token should be rotated immediately.

**CLI alternative:** `brew install gh && gh auth login` stores the credential in the macOS keychain and also gets you `gh pr view`, `gh pr checks`, and `gh pr diff` later on.

## Directory Structure

```
project/
├── .claude/
│   ├── commands/
│   │   └── dev.md               # Orchestrator
│   └── agents/
│       ├── planner-agent.md
│       ├── implementer-agent.md
│       ├── reviewer-agent.md
│       └── finalizer-agent.md
├── .plans/                      # Git-ignored — plans stay local
│   └── YYYYMMDD-slug.md
├── .ai-logs/                    # Committed — AI development summaries
│   └── YYYYMMDD-slug.md
└── .gitignore
```

- **`.plans/`** — local only. The planner's output; the shared source of truth for implementation and review.
- **`.ai-logs/`** — committed. TL;DR summaries and lessons learned from each cycle.

## How to Use

### Run the pipeline

```
/dev add user authentication with JWT and refresh tokens
```

Or point it at a spec or a ticket:

```
/dev implement .specs/checkout-flow.pdf
/dev implement the changes from JIRA ticket PROJ-123
```

The orchestrator will:

1. Spawn the planner (or skip it, for a trivial request).
2. Bring you the plan outline and any open questions, then ask whether to hand it off.
3. Drive implement → review rounds until approved.
4. Ask before finalizing, since that pushes to your remote and opens the MR/PR.

### Running a single agent

The pipeline is the recommended path, but each agent can be invoked on its own when you only want one stage — asking for the planner by name, or naming the reviewer to check an existing branch. Note that a lone agent has no orchestrator: it cannot ask you anything, so it will return its report to the main conversation instead.

## Stack-Agnostic

The workflow doesn't care what language you're using. The planner produces language-agnostic plans. The implementer adapts to the conventions it finds. The reviewer checks the plan and runs whatever test command the project uses.

For stack-specific behavior, add instructions to your project's `CLAUDE.md`.

## Setup

1. Copy `.claude/commands/`, `.claude/agents/`, `.gitignore`, and `CLAUDE.md` into any project.
2. Create `.plans/` (local, git-ignored) and `.ai-logs/` with a `.gitkeep`.
3. Optionally copy `.claude/mcp.json.tmpl` → `.claude/mcp.json` and fill in credentials.

No other configuration is needed — `/dev` works immediately.

### MCP Database Connections

The template ships example PostgreSQL and MySQL entries:

```bash
cp .claude/mcp.json.tmpl .claude/mcp.json
# Edit .claude/mcp.json with real credentials
```

`.claude/mcp.json` is git-ignored. Until you fill it in, no MCP server is connected — the planner's ticket lookups and the implementer's database access need it.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `/dev` not listed | Command file not in `.claude/commands/` | Check the path, restart the session |
| Agent returns malformed report | Its Output Contract was edited | Keep the contract block in sync with the parser in `dev.md` |
| Loop runs past 3 rounds | Orchestrator didn't stop | Check the Stage 3 rules in `dev.md` |
| Finalizer commits unrelated files | Pre-existing local changes | The finalizer should refuse; stage explicitly |
| `MR_SETUP: missing` after finalize | No `gh` and no token | Set `GITHUB_TOKEN` or install `gh` — see [MR / PR creation](#mr--pr-creation) |
| MR API returns 401/403 | Token expired or missing scope | Rotate the PAT and check its scopes |

## License

Free to use and modify for any project.
