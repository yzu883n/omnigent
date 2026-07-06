# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## Project overview

Omnigent is the open-source **meta-harness for AI coding agents** — a common
orchestration layer over multiple agent backends (Claude Code, Codex, Cursor,
GitHub Copilot, Antigravity/Gemini, OpenCode, Hermes, Pi, and custom
`openai-agents`-based agents). It unifies sessions across terminal/browser/
phone/desktop, enforces declarative policies, and runs agents locally or in
cloud sandboxes.

- Status: **alpha**. License: Apache 2.0. Requires **Python 3.12+**.
- Package name `omnigent`, console scripts `omnigent` and `omni` are
  identical aliases (`omnigent.cli:main`).
- Full product description, harness list, credential/sandbox/deploy options:
  see `README.md`. Do not re-derive these from source — read the README.

## Repository layout

| Path | What it is |
| --- | --- |
| `omnigent/` | Main Python package (server, runner, host, runtime, harnesses, CLI — see below) |
| `ap-web/` | React + Vite + TS frontend (the web UI) |
| `tests/` | Test suite; mirrors `omnigent/` layout 1:1, plus `e2e/`, `e2e_ui/`, `integration/` |
| `sdks/python-client`, `sdks/ui` | Path-dependency SDK packages, versioned in lockstep with the main package |
| `docs/` | Design docs — `AGENT_YAML_SPEC.md`, `POLICIES.md`, harness design notes |
| `scripts/` | Repo maintenance scripts (OpenAPI dump, installer, version bump, lockfile normalizer) |
| `deploy/` | Per-target deploy configs (Docker, Fly, Render, Railway, k8s, Modal, etc.) |
| `examples/` | Example agent bundles (`polly`, `debby`) |
| `dev/lint/` | Custom pre-commit lint scripts |
| `.claude/skills/` | Claude Code skills already authored for this repo (see below) |

## Core architecture

**Process topology**: **server** (`omnigent/server/`, FastAPI, `create_app()`)
↔ **host** (`omnigent/host/`, a daemon that connects to a server over
WebSocket and spawns runners on demand) ↔ **runner** (`omnigent/runner/`, one
process per session, tool dispatch/routing/cost tracking) ↔ **runtime**
(`omnigent/runtime/workflow.py`, the actual agent loop: load agent → build
prompt → call LLM → execute tools → repeat, durably checkpointed).

**Harness pattern** — every backend integration is either:
- an **SDK/subprocess harness**: headless, drives an SDK or CLI per turn
  (e.g. antigravity, copilot, cursor, claude-sdk, codex, pi, hermes, goose,
  qwen, openai-agents), or
- a **native harness**: injects text into a resident TUI/tmux pane and
  mirrors its transcript back (`*-native` variants).

Each SDK harness is a pair of files:
- `omnigent/inner/<name>_executor.py` — implements the `Executor` ABC from
  `omnigent/inner/executor.py` (one abstract method:
  `run_turn(...) -> AsyncIterator[ExecutorEvent]`).
- `omnigent/inner/<name>_harness.py` — a thin `create_app() -> FastAPI` wrap,
  configured entirely from `HARNESS_<NAME>_*` env vars, that hands the
  executor to the shared `ExecutorAdapter`.

Harnesses are registered by name in `_HARNESS_MODULES`
(`omnigent/runtime/harnesses/__init__.py`) — a factory registry so optional
SDK dependencies aren't imported unless that harness is used.
`omnigent/harness_aliases.py` maps user-facing shorthands (`agy` →
`antigravity`) to canonical ids and classifies which harnesses are native.
**When adding a new harness, load the `.claude/skills/harness-integration-guide`
skill** — it has the full capability matrix and checklist.

**Agents are YAML files** (`name`, `prompt`, `executor.harness`, `tools:`),
run via `omnigent run path/to/agent.yaml`. Full schema:
`docs/AGENT_YAML_SPEC.md`.

**Policies** are YAML-declared guardrails (`ask_on_os_tools`,
`max_tool_calls_per_session`, `cost_budget`, etc.) stacking at
server/agent/session level. Full catalog: `docs/POLICIES.md`.

**Config/env vars**: `OMNIGENT_CONFIG_HOME` (default
`~/.omnigent/config.yaml`) and `OMNIGENT_DATA_DIR` (default `~/.omnigent`),
defined in `omnigent/cli.py`. Harnesses whose backend isn't reachable through
the shared gateway (antigravity, copilot, cursor) each get their own
top-level config block (`antigravity:`, `copilot:`, `cursor:`) instead of the
generic `auth:`/gateway block, deliberately, to avoid credential
cross-contamination — preserve this separation when touching auth code.

## Development setup

Supported dev OS: **macOS or Linux only** (Windows devs: use WSL2/Ubuntu,
cloned into the Linux filesystem — not Git Bash; native Windows dev is
unsupported due to POSIX-only test deps).

Prereqs: `uv`, `tmux`, `bubblewrap` (Linux sandboxing only), Node.js 22 LTS+
/ npm (for `ap-web/` and vendor CLIs).

```bash
uv python install
uv venv --python "$(cat .python-version)"
uv sync --extra all --extra dev
source .venv/bin/activate
```

Running the full stack locally (3 terminals):

```bash
omnigent server                                   # FastAPI server, :6767
omnigent host --server http://localhost:6767      # host daemon
cd ap-web && npm run dev                          # Vite dev server, :5173
```

## Testing

Framework: **pytest** (`asyncio_mode = "auto"`, `pytest-xdist`,
`pytest-timeout`, `pytest-rerunfailures`).

```bash
uv run pytest    # default run — e2e/e2e_ui/e2e_live/integration are excluded (opt-in only)
```

Most backend areas mirror their source directory 1:1 under `tests/`:
`server/`, `runner/`, `runtime/`, `tools/`, `inner/`, `llms/`, `db/`,
`policies/`, `repl/`, `entities/`, `stores/`, `host/`, `spec/`.

Test policy (see `CONTRIBUTING.md` / `.github/copilot-instructions.md` for
the canonical version):
- A behavior change under `omnigent/` needs a test in the matching area
  suite (table above). Prefer a fast unit test; reach for
  `tests/integration/` or `tests/e2e/` only when the change genuinely spans
  components or full-stack flows.
- Bug fixes need a test that fails before the fix.
- Pure refactors, renames, type-only changes, dependency bumps, and
  comment/logging-only edits are exempt.
- A PR introducing new user-facing functionality must include at least one
  e2e happy-path test under `tests/e2e/`.
- `ap-web/` behavior changes need a colocated Vitest test (`*.test.ts(x)`);
  user-facing UI changes additionally need a Playwright test under
  `tests/e2e_ui/` (mechanically enforced by CI's `E2E UI Required` check).

**Read `tests/integration/AGENTS.md` and `tests/e2e/AGENTS.md` before running
those suites** — they document required credentials, markers
(`--integration`, `--harness`, `--model`), and mock-LLM mode. Both mandate
running e2e/integration suites **in the background**, never blocking the
terminal on them.

## Linting, formatting, type-checking

**ruff** is the only linter/formatter (no black/flake8/isort — ruff's `I`
rules handle import sorting). **mypy** runs in strict mode
(`disallow_any_explicit = true`).

```bash
uv run ruff check . && uv run ruff format --check .
uv run pre-commit run --all-files
```

For `ap-web/`: `npm run lint`, `npm run type-check`, `npm run build`.

## Git / PR conventions

- Commit style observed in history: `type(scope): summary (#PR)` (e.g.
  `fix(hermes-native): confirm first-message delivery ... (#1457)`), squash-merged
  through GitHub PRs. Scopes typically match a component/area name.
- **Commits require DCO sign-off**: `git commit -s`.
- Branch from `main`; keep PRs focused and include tests/docs per the policy
  above.

## Claude Code skills already in this repo

`.claude/skills/` already contains harness-specific dev workflows — **load
these instead of re-deriving the same steps**:

| Skill | When to load |
| --- | --- |
| `antigravity-sdk-e2e-dev` | Developing/debugging the Antigravity (Gemini) SDK harness (`omnigent/inner/antigravity_executor.py`, `antigravity_harness.py`, `omnigent/onboarding/antigravity_auth.py`) |
| `copilot-sdk-e2e-dev` | Developing/debugging the GitHub Copilot SDK harness (`omnigent/inner/copilot_executor.py`, `copilot_harness.py`, `omnigent/onboarding/copilot_auth.py`) |
| `cursor-sdk-e2e-dev` | Developing/debugging the Cursor SDK harness (`omnigent/inner/cursor_executor.py`, `cursor_harness.py`, `cursor_auth.py`) |
| `cli-setup-verify` | Any CLI/onboarding/REPL/picker UX change (`omnigent/cli.py`, `omnigent/onboarding/*`, `omnigent/repl/*`, `scripts/install_oss.sh`) |
| `harness-integration-guide` | Adding a brand-new harness integration |

## Key reference docs

- `README.md` — product description, quick start, install methods
- `CONTRIBUTING.md` — canonical dev-workflow doc; treat as source of truth
  for anything not covered above
- `docs/AGENT_YAML_SPEC.md` — agent YAML schema
- `docs/POLICIES.md` — full policy catalog
- `RELEASING.md` — release process and versioning steps
- `SECURITY.md` — vulnerability reporting policy
