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

## End-to-End Testing Agents

Two standalone agents verify a test case against an implementation. They are **not** part of the `/dev` pipeline and have no slash command — ask for them by name in a prompt and the main thread spawns the right one.

```
"run the static e2e check for PROJ-123"
"verify PROJ-123 against staging"
```

| | `e2e-static-agent` | `e2e-runtime-agent` |
|---|---|---|
| **Network** | none, ever | real HTTP to the URL you supply |
| **Tools** | Read, Grep, Glob | Read, Grep, Glob, Bash (curl), MCP DB |
| **Method** | trace route → middleware → handler → service → repo → serializer | execute your request, capture status/headers/body, compare |
| **Database** | reads schema and models statically | optional read-only `SELECT` via MCP |
| **Verdicts** | `MATCH` / `MISMATCH` / `NOT_FOUND` / `UNVERIFIABLE` | `PASS` / `FAIL` / `SKIPPED` / `UNVERIFIED` |

### How a run works

1. **You give the test case** — a PDF path, a ticket key, or criteria pasted inline.
2. **The main thread fetches it** — tickets come through the connected MCP server; PDFs are read directly. Nothing is fetched by the agent.
3. **It writes a normalized test case** to `.test-reports/YYYYMMDD-slug-testcase.md` — endpoint, method, expected request, rules, expected response.
4. **It spawns the agent** with that path, and the agent returns a report.

The ticket lookup happens in the main thread on purpose: `e2e-static-agent` has no network tools, so it cannot be the one calling Jira.

**Ticket platforms** need an MCP server in `.claude/mcp.json`. The template ships entries for Jira, Trello, ClickUp, and Asana — delete the ones you don't use, fill in the token entries you keep, or pass a PDF/inline criteria instead. See [Ticket Platforms](#ticket-platforms-for-e2e-test-case-intake).

### Static verification is not proof

`e2e-static-agent` reads code; it does not run it. Middleware ordering, framework serialization, DI wiring, and database triggers are invisible to it. Anything it cannot settle from the source is reported `UNVERIFIABLE` with a note on what would settle it — never silently upgraded to `MATCH` because the code looks right. Every `MATCH`/`MISMATCH` carries a `file:line`.

For proof about runtime behavior, use `e2e-runtime-agent`.

### Runtime safety

- **Read-only by default.** `GET` and `HEAD` only. `POST`/`PUT`/`PATCH`/`DELETE` require you to grant `ALLOW_WRITES` for that run — a curl containing a POST is not itself permission.
- **One target.** The agent stays on the base URL you name and never redirects off that host.
- **No credential leakage.** Tokens are substituted from environment variables and never echoed into a command, report, or file. A leaked token should be rotated.
- **DB checks are read-only** — `SELECT`/`SHOW`/`EXPLAIN` only, and they reflect state at query time, not at request time.

### Reports

Written to `.test-reports/` (git-ignored). Before writing, the agent asks whether you want the report sent to Slack or another platform as well; if not, it stays a local file. Captured responses are redacted — no tokens, cookies, or real personal data.

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
│       ├── finalizer-agent.md
│       ├── e2e-static-agent.md  # Standalone — code-path verification
│       └── e2e-runtime-agent.md # Standalone — live request verification
├── .plans/                      # Git-ignored — plans stay local
│   └── YYYYMMDD-slug.md
├── .test-reports/               # Git-ignored — E2E evidence stays local
│   ├── YYYYMMDD-slug-testcase.md
│   ├── YYYYMMDD-slug-static.md
│   └── YYYYMMDD-slug-runtime.md
├── .ai-logs/                    # Committed — AI development summaries
│   └── YYYYMMDD-slug.md
└── .gitignore
```

- **`.plans/`** — local only. The planner's output; the shared source of truth for implementation and review.
- **`.test-reports/`** — local only. Normalized test cases and verification reports.
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
2. Create `.plans/` and `.test-reports/` (local, git-ignored) and `.ai-logs/` with a `.gitkeep`.
3. Optionally copy `.claude/mcp.json.tmpl` → `.claude/mcp.json` and fill in credentials.

No other configuration is needed — `/dev` works immediately.

### MCP Database Connections

The template ships example PostgreSQL, MySQL, and Redis entries:

```bash
cp .claude/mcp.json.tmpl .claude/mcp.json
# Edit .claude/mcp.json with real credentials
```

`.claude/mcp.json` is git-ignored. Until you fill it in, no MCP server is connected — the planner's ticket lookups and the implementer's database access need it.

**Redis needs `uv` installed** — its server is the official Python `redis-mcp-server`, launched via `uvx`:

```bash
brew install uv
```

The connection string accepts the standard `redis://` and `rediss://` schemes:

```
redis://<user>:<password>@<host>:6379/0    # TLS: rediss://
```

Omit `<user>` and `<password>` for a local Redis with no auth. If you already use the npm package `@modelcontextprotocol/server-redis`, note that it is **deprecated** — this template deliberately uses Redis's own server instead.

### Ticket Platforms (for E2E test-case intake)

The template also carries entries for the four supported ticket platforms. **Delete the ones you don't use** — every entry spawns a process on startup, and credentials are real.

| Server | How it authenticates | Required values |
|---|---|---|
| `jira` | OAuth via `mcp-remote` to Atlassian's hosted endpoint — first run opens a browser | none in the file |
| `trello` | API key + token | `TRELLO_API_KEY`, `TRELLO_TOKEN`, `TRELLO_BOARD_ID` |
| `clickup` | Personal API token | `CLICKUP_API_KEY`, `CLICKUP_TEAM_ID` |
| `asana` | Personal access token | `ASANA_ACCESS_TOKEN` |

Jira needs no credential in the file: `mcp-remote` proxies to `https://mcp.atlassian.com/v1/sse` and completes an OAuth flow in the browser on first use. The other three take tokens in the `env` block — keep them in `.claude/mcp.json`, which is git-ignored, never in `.mcp.json.tmpl`.

**Licensing note:** the Jira endpoint is Atlassian's own; `mcp-server-trello` and `asana-mcp` are MIT. `@taazkareem/clickup-mcp-server` is **proprietary** — fine for local use, but check its terms before redistributing this template with that entry enabled.

**If a ticket lookup fails**, confirm the server starts at all before assuming the agent is at fault:

```bash
npx -y mcp-server-trello          # should print a startup line, not an error
```

A silent hang usually means a missing env var; the process waits rather than exiting.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `/dev` not listed | Command file not in `.claude/commands/` | Check the path, restart the session |
| Agent returns malformed report | Its Output Contract was edited | Keep the contract block in sync with the parser in `dev.md` |
| Loop runs past 3 rounds | Orchestrator didn't stop | Check the Stage 3 rules in `dev.md` |
| Finalizer commits unrelated files | Pre-existing local changes | The finalizer should refuse; stage explicitly |
| `MR_SETUP: missing` after finalize | No `gh` and no token | Set `GITHUB_TOKEN` or install `gh` — see [MR / PR creation](#mr--pr-creation) |
| MR API returns 401/403 | Token expired or missing scope | Rotate the PAT and check its scopes |
| E2E agent returns `NEEDS_INPUT` | Test case, curl, or base URL missing | Supply what the report asks for and re-run |
| E2E runtime refuses a request | Mutating verb without `ALLOW_WRITES` | Grant writes explicitly for that run, or test read paths only |
| Ticket lookup fails | No ticket MCP configured | Add a Jira/Trello/ClickUp/Asana server, or pass a PDF |

## License

Free to use and modify for any project.
