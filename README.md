<h1 align="center">codebase-intel</h1>

<p align="center">
  <strong>Your AI agent writes code. But does it know <em>why</em> your code exists?</strong>
</p>

<p align="center">
  <a href="https://github.com/MutharasuArchunan13/codebase-intel/stargazers"><img src="https://img.shields.io/github/stars/MutharasuArchunan13/codebase-intel?style=flat-square" alt="Stars"></a>
  <a href="https://opensource.org/licenses/MIT"><img src="https://img.shields.io/badge/License-MIT-green.svg?style=flat-square" alt="MIT"></a>
  <a href="https://www.python.org/"><img src="https://img.shields.io/badge/python-3.11+-blue.svg?style=flat-square" alt="Python 3.11+"></a>
  <a href="https://modelcontextprotocol.io/"><img src="https://img.shields.io/badge/MCP-compatible-purple.svg?style=flat-square" alt="MCP"></a>
  <a href="#19-languages"><img src="https://img.shields.io/badge/languages-19-orange.svg?style=flat-square" alt="19 Languages"></a>
  <a href="https://pypi.org/project/codebase-intel/"><img src="https://img.shields.io/pypi/v/codebase-intel?style=flat-square" alt="PyPI"></a>
</p>

---

AI coding agents can autocomplete, refactor, and generate code. But they still fail at the things that matter most in production:

- They don't know your team **decided** to use token bucket over sliding window — and why
- They don't know the compliance team **requires** rate limit headers on every response
- They don't know that changing `config.py` will **break** billing and analytics
- They generate code that **looks right** but violates your project's architectural patterns

**codebase-intel** fixes this. It's the context layer that sits between your codebase and any AI agent — providing not just *what* code exists, but *why* it exists, *what rules* it must follow, and *what breaks* if you change it.

---

## Before vs After

```
WITHOUT codebase-intel                    WITH codebase-intel
─────────────────────                     ───────────────────

Agent reads: every file in the dir        Agent reads: only what matters
Tokens used: 16,063                       Tokens used: 5,955
Knows why code exists: No                 Knows why: Yes (13 decisions)
Quality guardrails: None                  Guardrails: 4 contracts enforced
Drift awareness: None                     Drift: stale context detected
Impact analysis: None                     Impact: knows what else breaks

Result: faster but fragile                Result: faster AND correct
```

### Real benchmarks on production codebases

| Project | Files | Naive Tokens | codebase-intel | Reduction | Decisions | Contracts |
|---|---:|---:|---:|---:|---:|---:|
| **FastAPI monolith** (auth + frontend) | 359 | 16,063 | 5,955 | **63%** | 13 | 4 |
| **Microservice A** (AI processing) | 358 | 14,611 | 5,955 | **59%** | 0 | 7 |
| **Microservice B** (document generation) | 87 | 2,461 | 1,275 | **48%** | 0 | 6 |
| **Microservice C** (user management) | 153 | 5,904 | 1,476 | **75%** | 0 | 4 |

> Numbers from `codebase-intel benchmark` on real production repos. The token reduction comes from targeted graph traversal. The decisions and contracts are what no other tool provides.

---

## What makes this different

There are great tools for code graphs (code-review-graph is excellent — 6K+ stars). **We don't compete with them.** We solve what they don't:

| Capability | Graph-only tools | codebase-intel |
|---|---|---|
| Code graph + dependencies | Yes | Yes (19 languages) |
| **Decision Journal** — *why* code is the way it is | No | **Yes** |
| **Quality Contracts** — rules AI must follow | No | **Yes** |
| **AI Anti-pattern Detection** — catches hallucinated imports, over-abstraction | No | **Yes** |
| **Drift Detection** — stale context, context rot alerts | No | **Yes** |
| **Token Budgeting** — fits context to any agent's window | No | **Yes** |
| **Live Analytics** — prove efficiency over time | No | **Yes** |

### The missing layer

```
What exists today:          What codebase-intel adds:

Code → Graph → Agent        Code → Graph ──────────────────→ Agent
                                     ↓                         ↑
                              Decision Journal ──→ WHY ────────┤
                              Quality Contracts → RULES ───────┤
                              Drift Detector ──→ WARNINGS ─────┘
                              
                              "Here are the 3 files that matter,
                               the decision your team made 6 months ago,
                               and the 2 rules you must not violate."
```

---

## Quick Start

```bash
# Install globally (no venv needed)
uvx codebase-intel --help          # ephemeral, like npx
# or
pipx install codebase-intel        # persistent global install
# or
pip install codebase-intel         # traditional

# Initialize on your project
cd your-project
codebase-intel init

# See what it found
codebase-intel status

# Mine decisions from git history
codebase-intel mine --save

# Auto-detect code patterns and generate quality contracts
codebase-intel detect-patterns --save

# Run benchmarks (see the before/after)
codebase-intel benchmark

# View live dashboard
codebase-intel dashboard
```

### Connect to Claude Code / any MCP client

**Single project:**
```json
{
  "mcpServers": {
    "codebase-intel": {
      "command": "codebase-intel",
      "args": ["serve", "/path/to/your/project"]
    }
  }
}
```

**Multiple projects (global mode):**
```bash
# Register repos once from anywhere
codebase-intel register ~/projects/user-service
codebase-intel register /opt/services/payment-api
codebase-intel register /var/repos/notification-service
```

```json
{
  "mcpServers": {
    "codebase-intel": {
      "command": "uvx",
      "args": ["codebase-intel", "serve", "--auto"]
    }
  }
}
```

In `--auto` mode, the MCP server automatically routes each request to the correct project based on file paths in the request. No manual switching — works like `npx @playwright/mcp`.

Now your agent automatically gets relevant context, decisions, and contracts before writing code.

---

## The Three Pillars

### 1. Decision Journal — "Why does this code exist?"

Every team makes hundreds of decisions that never get documented. *Why* did you choose Postgres over Mongo? *Why* is auth middleware structured that way? *Why* was the sliding window approach rejected?

codebase-intel captures these from git history automatically and links them to code:

```yaml
# .codebase-intel/decisions/DEC-042.yaml
id: DEC-042
title: "Use token bucket for rate limiting"
status: active
context: "Payment endpoint was getting hammered during flash sales"
decision: "Token bucket algorithm with per-user buckets, 100 req/min"
alternatives:
  - name: sliding_window
    rejection_reason: "Memory overhead too high at scale"
constraints:
  - description: "Must not add >2ms p99 latency"
    source: sla
    is_hard: true
code_anchors:
  - "src/middleware/rate_limiter.py:15-82"
```

**Without this**: your agent proposes sliding window (the exact approach you rejected 6 months ago).
**With this**: your agent sees the decision, follows it, and respects the SLA constraint.

### 2. Quality Contracts — "What rules must AI follow?"

Linters check syntax. Contracts enforce **your project's patterns**:

```yaml
# .codebase-intel/contracts/api-rules.yaml
rules:
  - id: no-raw-sql
    name: No raw SQL in API layer
    severity: error
    pattern: "execute\\(.*SELECT|INSERT|UPDATE"
    fix_suggestion: "Use the repository pattern"

  - id: async-everywhere
    name: All I/O must be async
    severity: error
    pattern: "requests\\.(get|post)"
    fix_suggestion: "Use httpx.AsyncClient"
```

**Built-in AI guardrails** catch the patterns AI agents mess up most:
- Hallucinated imports (modules that don't exist)
- Over-abstraction (base classes with one subclass)
- Unnecessary error handling for impossible conditions
- Comments that restate code instead of explaining why
- Features that weren't requested (YAGNI violations)

### 3. Drift Detection — "Is our context still valid?"

Context rots. Decisions get outdated. Code anchors point to deleted files. codebase-intel detects this:

```bash
$ codebase-intel drift

╭──────────────── Drift Report ────────────────╮
│ Overall: MEDIUM                               │
│ 3 items need attention                        │
╰───────────────────────────────────────────────╯

- [MEDIUM] Decision DEC-012 anchored to deleted file
- [MEDIUM] Decision DEC-008 is past its review date
- [LOW] 2 files changed since last graph index
```

---

## 19 Languages

Full tree-sitter parsing via [tree-sitter-language-pack](https://github.com/nicolo-ribaudo/tree-sitter-language-pack):

| Category | Languages |
|---|---|
| **Web** | JavaScript, TypeScript, TSX |
| **Backend** | Python, Java, Go, Ruby, PHP, Elixir |
| **Systems** | Rust, C, C++ |
| **Mobile** | Swift, Kotlin, Dart |
| **Other** | C#, Scala, Lua, Haskell |

---

## CLI Commands

```bash
# Core
codebase-intel init [path]              # Initialize — build graph, create configs
codebase-intel analyze [--incremental]  # Rebuild or update the code graph
codebase-intel mine [--save]            # Mine git history for decision candidates
codebase-intel detect-patterns [--save] # Auto-detect code patterns → quality contracts
codebase-intel drift                    # Run drift detection
codebase-intel benchmark                # Measure token efficiency (before/after)
codebase-intel dashboard                # Live efficiency tracking over time
codebase-intel serve [path]             # Start MCP server (single project)
codebase-intel status                   # Component health check
codebase-intel intent [--verify]        # Track and verify delivery goals

# Global workspace (multi-project)
codebase-intel register <path>          # Register a project globally
codebase-intel unregister <id>          # Remove from global registry
codebase-intel projects                 # List all registered projects
codebase-intel serve --auto             # Start MCP server for ALL registered projects

# Cross-repo (microservices)
codebase-intel crossrepo <paths...>     # Scan repos for cross-service dependencies
codebase-intel crossrepo --all          # Scan all registered projects
```

## MCP Tools (12 tools)

| Tool | What it does |
|---|---|
| `get_context` | **The main tool.** Assembles relevant files + decisions + contracts within a token budget. |
| `query_graph` | Query dependencies, dependents, or run impact analysis. |
| `get_decisions` | Get architectural decisions relevant to specific files. |
| `get_contracts` | Get quality contracts for files you're editing. |
| `check_drift` | Verify context freshness before trusting old decisions. |
| `impact_analysis` | "What breaks if I change this file?" |
| `get_status` | Health check — graph stats, decision count, contract count. |
| `record_feedback` | Record if AI output was accepted/rejected — powers the learning loop. |
| `get_efficiency_report` | Live token savings, acceptance rate, before/after proof. |
| `set_intent` | Capture what the user wants with machine-verifiable acceptance criteria. |
| `check_intent` | Verify if acceptance criteria are actually met before marking done. |
| `list_intents` | Show all tracked intents with completion status. |

---

## Community Contract Packs

Pre-built quality rules for popular frameworks:

| Pack | Rules | Covers |
|---|---|---|
| **fastapi.yaml** | 10 | Layered architecture, Pydantic schemas, async, Depends(), secrets |
| **react-typescript.yaml** | 11 | Functional components, no `any`, custom hooks, lazy loading |
| **nodejs-express.yaml** | 12 | Error handling, helmet, rate limiting, structured logging |

```bash
cp community-contracts/fastapi.yaml .codebase-intel/contracts/
```

---

## New in v0.2.0

### Global Workspace — One server, all your projects

No more running a separate MCP server per project. Register your repos once, serve them all:

```bash
codebase-intel register ~/projects/user-service
codebase-intel register /opt/services/payment-api
codebase-intel serve --auto
```

The server auto-routes each request to the correct project based on file paths. LRU cache keeps the 5 most recently used projects loaded — older ones are evicted and reloaded on demand.

### Intent Tracking — "Did you actually build what was requested?"

AI agents say "done" when the code compiles. But did they actually deliver what was asked? Intent tracking captures goals with **machine-verifiable** acceptance criteria:

```bash
# Agent sets intent at task start (via MCP tool set_intent)
# Agent works on the task...
# Before marking done, agent calls check_intent
# System runs automated checks: file_exists, function_exists, grep_match, test_passes...
# Returns: 18/21 criteria met — 3 gaps identified
```

11 verification types: `file_exists`, `file_contains`, `function_exists`, `wired`, `cli_works`, `mcp_tool_exists`, `grep_match`, `grep_no_match`, `test_passes`, `custom`, `manual`.

### Cross-Repo Awareness — Microservice dependency mapping

Scans 14 web frameworks across 10 languages to find exposed endpoints and outbound HTTP calls. Maps which service depends on which endpoint:

```bash
codebase-intel crossrepo --all --impact user-service

# Output:
# CRITICAL: 3 services depend on /api/v1/users/{id}
# → payment-service calls this endpoint [CRITICAL] at src/clients/user.py:42
# → notification-service calls this endpoint at lib/api/users.ts:18
```

Supported: FastAPI, Express, Flask, Django, Spring Boot, Gin, Actix, Rails, Phoenix, Laravel, ASP.NET, Ktor, Vapor, Echo.

### Feedback Loop — Learn from acceptance/rejection

Records whether AI-generated code was accepted, modified, or rejected. Over time, identifies which context patterns lead to better output and which rejection reasons are most common.

## Architecture

```
AI Agent (any) ──→ MCP Server (12 tools)
                        │
                  Workspace Manager ←── Global Registry (~/.codebase-intel/)
                  (multi-project routing, LRU cache)
                        │
                  Context Orchestrator
                  (token budgeting, priority, conflicts)
                   ╱          │          ╲
          Code Graph    Decision     Quality
          (19 langs)    Journal      Contracts
          SQLite+WAL    YAML files   YAML+builtins
                   ╲          │          ╱
                    Drift Detector
                    (staleness, rot, orphans)
                          │
               ┌──────────┼──────────┐
         Analytics    Feedback    Intent
         Tracker      Loop       Tracker
               └──────────┼──────────┘
                          │
                   Cross-Repo Scanner
                   (14 frameworks, 10 languages)
```

---

## The Philosophy

**We don't make AI agents smarter. We make them informed.**

An agent with 1M tokens of context is like a developer with access to every file in the company — overwhelming and unfocused. An agent with codebase-intel is like a developer who just had a 5-minute conversation with the senior engineer: *"Here's what you need to know, here's why we did it this way, and here are the three things you absolutely cannot break."*

**We don't compete with graph tools. We complete them.**

Code graphs answer "what depends on what." That's necessary but not sufficient. codebase-intel answers the harder questions: *Why was this decision made? What constraints apply? What will the compliance team flag? What did we already try and reject?*

**We don't hide the truth. We prove it.**

Run `codebase-intel benchmark` on your project. See the numbers. Run `codebase-intel dashboard` over time. Watch the improvement. Every claim is backed by reproducible, project-specific data.

---

## Contributing

Areas with the most impact:

1. **Contract packs** — Share quality rules for your framework
2. **Language extraction** — Improve parsing for specific languages
3. **Decision mining** — Better git history analysis
4. **Benchmarks** — Run against your repos, share results

```bash
git clone https://github.com/MutharasuArchunan13/codebase-intel.git
cd codebase-intel
pip install -e ".[dev]"
pytest
```

## License

MIT
