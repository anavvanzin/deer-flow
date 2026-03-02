# 🦌 DeerFlow – Copilot Instructions

This guide is for **AI coding agents** (Copilot, Claude, etc.) that are stepping into the
`deer‑flow` repository.  Its purpose is to surface the minimal but essential knowledge
required to be immediately productive: how the system is organised, how developers
build and test, and the idiosyncratic patterns you must follow.

> ⚠️ This is a living document.  When you add or change behaviour in the codebase,
> update **both** this file and `backend/CLAUDE.md` (which is written for human
> developers).  The root `README.md` focuses on users; `CLAUDE.md` and this file
> focus on internal architecture and workflows.

---

## 1. Big‑Picture Architecture

The project is a **full‑stack super‑agent harness** built on LangGraph.

```
project root
├─ backend/            # Python LangGraph server + FastAPI gateway
├─ frontend/           # Next.js UI using LangGraph SDK
└─ skills/             # Agent skills (public + custom folders)
```

- **LangGraph Server** (port 2024) hosts the `lead_agent` defined in
  `backend/src/agents/lead_agent/agent.py`.  A single agent handles all threads.
- **Gateway API** (FastAPI, port 8001) exposes models, MCP config, skills, memory,
  uploads and artifacts.  See `backend/src/gateway/routers/*` for routes.
- **Frontend** (Next.js, port 3000) uses the LangGraph SDK and talks to the
  gateway via nginx (port 2026 in dev).  See `frontend/AGENTS.md` for details.
- **Nginx** proxies `/api/langgraph/*` → LangGraph, others → Gateway.

### Backend subsystems

- **Middleware chain** (strict order, defined in lead_agent): thread directories,
  sandbox acquisition, uploads, summarization, todo lists, titles, memory,
  image handling, subagent limits, clarification interrupts.  See
  `src/agents/lead_agent/agent.py` for the list and order.

- **Sandbox**: virtual paths (`/mnt/user-data/...`, `/mnt/skills`) map to
  `backend/.deer-flow/threads/{thread_id}`.  Providers include local filesystem
  and Docker-based `AioSandboxProvider`.  Tools (`bash`, `ls`, `read_file`, etc.)
  live in `src/sandbox/tools.py` and automatically translate paths.

- **Subagents**: invoked by the `task()` tool.  Execution happens in background
  thread pools (`src/subagents/executor.py`) with a concurrency limit of 3.
  Built‑ins are `general-purpose` and `bash`.  The middleware truncates excess
  requests to enforce the limit.

- **Tools**: loaded via `get_available_tools()`; they come from
  `config.yaml`, MCP servers (`mcp/`), built‑in helpers (`src/tools/builtins`),
  community providers (`src/community/*`), and the `task` subagent tool.

- **Memory**: asynchronous, LLM‑powered extraction (`src/agents/memory/*`).
  Configurable via `config.yaml` and stored in `backend/.deer-flow/memory.json`.
  Triggered by `MemoryMiddleware`; top facts are injected into the system prompt
  on every turn.

- **Skills**: Markdown files under `skills/public` or `skills/custom` with
  `SKILL.md` front‑matter.  Enabled state lives in `extensions_config.json`.
  Skills are mentioned verbatim in the system prompt and mounted in the sandbox
  under `/mnt/skills`.

- **MCP**: Model Context Protocol servers configured in `extensions_config.json`.
  Tools are lazily initialized and cached (mtime‑based invalidation).
  Supports `stdio`, `sse`, and `http` transports.

## 2. Developer Workflows and Commands

Use `make` targets from either the project root or `backend/` directory.

```bash
# project root (full stack)
make check        # prerequisites (Node 22+, pnpm, uv, nginx)
make install      # install frontend + backend deps
make dev          # start LangGraph+Gateway+Frontend+Nginx
make stop         # stop all services

# backend only (inside backend/)
make install      # pip install dependencies
make dev          # run LangGraph server on 2024
make gateway      # run FastAPI gateway on 8001
make test         # run pytest suite
make lint         # ruff linting
make format       # ruff formatting
```

Tests are **mandatory** for every feature/bug fix.  The existing suite lives in
`backend/tests/`; follow the `test_<module>.py` naming convention.  Run
`PYTHONPATH=. uv run pytest tests/test_*.py -v` to target a single file.

Regression tests for sandbox/provisioner behaviour are
`tests/test_docker_sandbox_mode_detection.py` and
`tests/test_provisioner_kubeconfig.py` and run in CI.

### Configuration

Copy `config.example.yaml` → `config.yaml` in the **project root** (preferred).
Values beginning with `$` are treated as environment variables.  The loader
resolves config paths in this precedence order:

1. Explicit `--config_path` argument
2. `DEER_FLOW_CONFIG_PATH` env var
3. `config.yaml` in current directory (usually `backend/`)
4. `config.yaml` in parent directory (project root) ✅

`extensions_config.json` follows the same priority rules and controls MCP servers
and skill states; the gateway API can update it at runtime.

### Running the app in Docker

Prefer `make docker-init` and `make docker-start` from the project root.  The
provisioner service only starts when `sandbox.use` is set to
`src.community.aio_sandbox:AioSandboxProvider` with a `provisioner_url`.

## 3. Conventions & Patterns

- **TDD enforcement**: no code without tests.  Use `sys.modules` mock in
  `tests/conftest.py` if needed to break import cycles (see example there).

- **Atomic file writes**: memory updates use temp files + rename.  Emulate this
  pattern if you write any data that must survive crashes.

- **Path translation helpers**: `replace_virtual_path()` and
  `replace_virtual_paths_in_command()` are used widely; replicate them when
  dealing with sandbox paths.

- **Configurable classes**: many components are referenced via Python import
  strings (e.g. `sandbox.use: src.sandbox.local:LocalSandboxProvider`).  Use
  `resolve_class()` and `resolve_variable()` in reflection code.

- **Middleware order matters**: tests often explicitly instantiate the chain
  with `agent._middleware`.  If you insert a new middleware, update
  `src/agents/lead_agent/agent.py` and add tests verifying order/side effects.

- **Prompt templates** live in `src/agents/lead_agent/prompts/`.  When updating,
  ensure new tags such as `<skills>` or `<memory>` are filled by the agent.

- **Skill loading** uses front‑matter YAML, parsed by `yaml.safe_load()`.  The
  allowed-tools list limits which sandbox commands a skill can invoke.

- **Subagent task calls**: the JSON payload must include `description`,
  `prompt`, and `max_turns`.  Excess calls beyond the concurrency limit are
  silently trimmed; the middleware logs the drop.

- **Memory injection**: the system prompt contains `<memory>`; the middleware
  populates it with the top N facts based on `memory.max_injection_tokens`.

- **Clarification requests** use `ask_clarification()` and are handled by
  `ClarificationMiddleware` which raises `Command(goto=END)`.  Tests often
  assert that a tool call message is converted to an interruption.

- **Artifcacts & uploads** are tied to `thread_id`.  Directories:
  `backend/.deer-flow/threads/{thread_id}/user-data/{workspace,uploads,outputs}`
  are created by `ThreadDataMiddleware`.

## 4. Cross‑Component Communication

- Frontend uses the LangGraph SDK and expects the gateway to be available at
  `${BASE_URL}/api`.  CORS and URL prefixes are handled by nginx config
  (`docker/nginx/nginx.local.conf`).

- Gateway updates to `extensions_config.json` trigger the LangGraph server to
  reload tools; the mechanism is mtime-based.  Do not import the file in
  long-lived modules; always call `get_cached_mcp_tools()` or
  `get_cached_skills()` which handle invalidation.

- Subagents communicate status via SSE events implemented in
  `src/subagents/executor.py` (see `_send_event()`).  Use the same event names
  when adding new behaviours.

- Memory updater runs in a background thread; it uses the shared configuration
  object (`config.configurable`).  Avoid global state outside of `config` and
  `thread_state`.

## 5. Useful References & Search Patterns

Search the repo for patterns when needed:

```bash
grep -R "<skills>" -n backend/src
grep -R "task(\"" -n backend/src
grep -R "memory\.configurable" -n backend/src
```

The `docs/` folder contains detailed write‑ups on individual subsystems.  If
you're updating code related to a doc, update the doc too.

---

Feel free to ask for clarification if any part of the system is opaque.  The
more context you can infer from code and tests, the better the suggestions you
provide will be.  Happy coding! 🛠️
