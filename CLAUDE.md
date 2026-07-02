# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this is

**Omnigent** is an open-source "meta-harness" for AI coding agents: a common
orchestration layer over Claude Code, Codex, Cursor, OpenCode, Hermes, Pi, and
custom YAML-defined agents. It lets you swap or combine harnesses without
rewriting, enforce policies/sandboxing, and collaborate on sessions from
terminal, browser, phone, or the native desktop app.

- Python **3.12+**, Apache 2.0, status **alpha** (`pyproject.toml` version
  `0.3.0.dev0`). Built by Databricks.
- Package manager: **uv** (not poetry/plain pip).
- CLI entry points: `omnigent` and `omni` (alias), both map to
  `omnigent.cli:main`.

## Repo layout

| Path | What's there |
| --- | --- |
| `omnigent/` | Main Python package — see Architecture below |
| `tests/` | pytest suite; mirrors `omnigent/` structure 1:1 per area |
| `ap-web/` | Frontend: Vite + React + TypeScript, Electron, iOS |
| `sdks/python-client`, `sdks/ui` | Companion packages `omnigent-client` and `omnigent-ui-sdk` |
| `docs/` | Specs: `AGENT_YAML_SPEC.md`, `POLICIES.md`, native-harness design docs |
| `designs/` | Architecture/feature proposals: `CLI_CONTRACT.md`, `SANDBOX_CREDENTIAL_PROXY.md`, etc. |
| `scripts/` | Dev/release helper scripts (`install_oss.sh`, `dump_openapi.py`, `update_versions.py`) |
| `dev/lint/` | Custom pre-commit lint checks |
| `deploy/` | Sandbox/deploy targets: modal, daytona, e2b, kubernetes, docker, fly, railway, render, databricks, cloudflare |
| `examples/` | Sample agents (`polly/`, `scribe/`, `debby/`) |
| `.claude/skills/` | Claude Code skills for developing *this* repo (see below) |
| `.github/` | CI workflows + `copilot-instructions.md` (review-bot test-coverage rules) |

## Architecture

### Executors and harnesses (`omnigent/inner/`)

A **harness** is a per-vendor integration that lets Omnigent drive a different
AI backend (Claude, Antigravity/Gemini, Copilot, Cursor, Codex, Goose, Kimi,
Qwen, Pi, OpenCode, Kiro, ...). The abstract base is `Executor` in
`omnigent/inner/executor.py`: `run_turn()` is an async generator yielding
`ExecutorEvent`s (`TextChunk`, `ReasoningChunk`, `ToolCallRequest`,
`ToolCallComplete`, `TurnComplete`, `CompactionComplete`, `TurnCancelled`,
`ExecutorError`), plus capability hooks (`supports_streaming`,
`handles_tools_internally`, `interrupt_session`, `close_session`, ...).
`MockExecutor` in the same file supports scripted tests without real API calls.

Each vendor gets a naming triad under `omnigent/inner/`:
`<vendor>_executor.py` (adapter logic) + `<vendor>_harness.py` (FastAPI
`create_app()` entrypoint), and often a `<vendor>_native_executor.py` /
`<vendor>_native_harness.py` pair for a second "native" track. Two
architectural tracks:

- **SDK/subprocess harnesses** — own the model lifecycle in-process or via
  CLI subprocess/ACP (e.g. `antigravity_executor.py`, `copilot_executor.py`,
  `cursor_executor.py`, `claude_sdk_executor.py`).
- **Native harnesses** — wrap the vendor's own TUI/server and mirror output
  into Omnigent (observe/relay only).

The shared glue is `ExecutorAdapter` in
`omnigent/runtime/harnesses/_executor_adapter.py`: lazy per-conversation
executor construction, translating requests into `Message`/`ExecutorConfig`,
translating `ExecutorEvent`s into SSE events, and cancellation propagation.
Harness selection happens in `omnigent/runtime/harnesses/_runner.py`; config
flows in via `HARNESS_<NAME>_*` env vars set before spawning.

**Adding a new harness**: read
`.claude/skills/harness-integration-guide/SKILL.md` first — it has the full
checklist (tool bridging, policy verdicts ALLOW/ASK/DENY, interrupt,
resume/fork, compaction, cost tracking, required tests).

### Onboarding / auth (`omnigent/onboarding/`)

Each vendor SDK harness gets its own `<vendor>_auth.py` (e.g. `cursor_auth.py`,
`copilot_auth.py`, `antigravity_auth.py`) following the same template — the
module docstrings literally say "Mirrors `omnigent.onboarding.<other_vendor>_auth`":

- A dedicated config block in `~/.omnigent/config.yaml` (e.g. `cursor:`) —
  separate from the shared `auth:` gateway block, so a vendor key can't leak
  into claude-sdk/codex/pi/openai-agents.
- Secrets stored via `omnigent/onboarding/secrets.py` (OS keychain, fallback
  `0600` JSON file), referenced as `keychain:<name>` or `env:VAR`, resolved
  through `resolve_secret()`.
- A soft prefix-check function (e.g. `looks_like_cursor_api_key`) that never
  hard-blocks a pasted key.
- `resolve_*()` functions that never raise — they return `None` on failure so
  callers fall back to ambient env vars.

### CLI / REPL

`omnigent/cli.py` is the click-based entry point (setup, server lifecycle,
sandbox commands, harness aliasing, config). Key subcommands: `omnigent run
<agent.yaml>`, `omnigent claude/codex/cursor/opencode/hermes/pi` (harness
launchers), `omnigent server start/stop/status`, `omnigent host`. Server
lifecycle lives in `omnigent/host/local_server.py`. The interactive terminal
chat is `omnigent/repl/` (`_repl.py` is the core loop).

### Server

FastAPI app at `omnigent/server/app.py`, routers under
`omnigent/server/routes/` (sessions, comments, policy registry, MCP servers,
etc.). API documented in `omnigent/server/API.md`. Errors use a centralized
`OmnigentError`/`ErrorCode` system (`omnigent/errors.py`).

### Runtime

`omnigent/runtime/` is the execution engine — how an agent turn actually runs
(LLM calls, tool calls, skills, compaction). See `omnigent/runtime/README.md`.

### Other docs worth knowing about

- `docs/AGENT_YAML_SPEC.md` — the agent YAML authoring spec
- `docs/POLICIES.md` — policy system (pause for approval, spend caps, tool limits)
- `designs/CLI_CONTRACT.md` — CLI output/branding contract
- `RELEASING.md` — versioning/release process

## Development workflow

**Supported OS: macOS or Linux only.** Native Windows doesn't work for
development (some test deps are POSIX-only, some modules call `os.getuid()`
at import time, pre-commit assumes `.venv/bin/`). On Windows, use WSL2
(Ubuntu) and clone into the Linux filesystem — this matches CI.

Prerequisites: `uv`, `tmux`, `bubblewrap` (`bwrap`, Linux only — macOS uses
built-in `seatbelt`), Node.js 22 LTS + `npm` for `ap-web/`.

```bash
git clone https://github.com/omnigent-ai/omnigent.git
cd omnigent

uv python install
uv venv --python "$(cat .python-version)"
uv sync --extra all --extra dev
source .venv/bin/activate    # or prefix commands with `uv run`
```

Common checks:

```bash
uv run pytest                      # Python tests (e2e/live/integration skipped by default)
uv run ruff check . && uv run ruff format --check .
uv run pre-commit run --all-files
```

Frontend (`ap-web/`):

```bash
cd ap-web && npm install && npm run lint && npm run build
```

Running locally end-to-end (three terminals):

```bash
# Terminal 1: local server on :6767
omnigent server

# Terminal 2: register your machine as a host
omnigent host --server http://localhost:6767

# Terminal 3: frontend dev server
cd ap-web && npm run dev
```

Open the Vite URL (usually `http://localhost:5173/`). Host registration is
what lets the web UI browse your filesystem and start new sessions on your
machine.

## Testing conventions

A change that alters behaviour under `omnigent/` should ship with a test; a
bug fix should add a test that fails before the fix. Pure refactors, renames,
type-only changes, dependency bumps, and edits with no observable behaviour
change don't need one. Prefer the smallest test that covers the change — a
fast, focused **unit test** in the matching area suite is the default. Reach
for `tests/integration/` only when behaviour genuinely spans components, and
`tests/e2e/` only for full-stack flows a unit test can't capture.

Most backend areas mirror their source directory under `tests/`:

| Area changed (`omnigent/…`) | Test suite (`tests/…`) |
| --- | --- |
| `server/` | `server/` |
| `runner/` | `runner/` |
| `runtime/` | `runtime/` |
| `tools/` | `tools/` |
| `inner/` | `inner/` |
| `llms/` | `llms/` |
| `db/` | `db/` (a schema migration especially warrants one) |
| `policies/` | `policies/` |
| `repl/` | `repl/` |
| `entities/` | `entities/` |
| `stores/` | `stores/` |
| `host/` | `host/` |
| `spec/` | `spec/` |

- `tests/integration/` — behaviour spanning several components (e.g. server +
  runtime).
- `tests/e2e/` — full-stack flows driven against a live LLM. Slow and
  gateway-bound. **A PR that adds new user-facing functionality must include
  at least one e2e happy-path test** here — enforced by
  `.github/copilot-instructions.md` review rules.
- pytest defaults **skip** `tests/e2e`, `tests/e2e_ui`, `tests/e2e_live`,
  `tests/integration` — run them explicitly to opt in.

### Frontend (`ap-web/`)

- Add/update a colocated Vitest test (`*.test.ts`/`*.test.tsx` next to the
  changed file), run with `npm test`.
- A change to user-facing UI behaviour also needs a Playwright test under
  `tests/e2e_ui/` — mechanically enforced by the `E2E UI Required` CI check
  (`.github/workflows/e2e-ui-required.yml`).
- Styling/copy-only changes and behaviour-neutral refactors are exempt.

## Code conventions

- `from __future__ import annotations` at the top of nearly every file.
- mypy `disallow_any_explicit = true` — an explicit `Any` needs a
  `# type: ignore[explicit-any]` comment explaining why.
- Sphinx/reST-style docstrings (`:param:`, `:returns:`, `:raises:`), often
  explaining *why*, and cross-referencing sibling vendor implementations
  ("mirrors X").
- No bare `except Exception` without a `# noqa: BLE001` justification;
  resolver/lookup functions return `None`/`False` softly rather than raising
  — exceptions are reserved for programmer errors.
- Executors are `async def run_turn(...) -> AsyncIterator[ExecutorEvent]`;
  blocking vendor SDK iterators are bridged via `iterate_blocking_stream`
  (background thread + `asyncio.Queue`).
- `logger = logging.getLogger(__name__)` at module scope — no `print`.
- Ruff config: line-length 99, rule sets
  `E,F,I,UP,ARG,BLE,B,SIM,RET,C4,PIE,RUF`.
- Naming: `_`-prefixed modules are private (`_repl.py`, `_executor_adapter.py`,
  `_runner.py`); env-var/config-key constants are centralized near the top of
  a file as a single grep target.

## Pull requests

- Branch from `main`, keep changes focused.
- Sign off commits with `git commit -s` (Developer Certificate of Origin).
- Include tests or docs when relevant (see Testing conventions above).
- Don't include secrets, internal URLs, customer data, or private
  configuration in issues, tests, examples, or logs.

## Repo-specific Claude Code skills (`.claude/skills/`)

- `harness-integration-guide` — reference for building a new harness
  integration (SDK/subprocess vs. native tracks).
- `antigravity-sdk-e2e-dev`, `copilot-sdk-e2e-dev`, `cursor-sdk-e2e-dev` — spin
  up a live local Omnigent server and exercise that vendor's SDK harness
  end-to-end.
- `cli-setup-verify` — drives the real `omnigent` binary through a PTY in an
  isolated sandbox to verify CLI onboarding/setup UX.
