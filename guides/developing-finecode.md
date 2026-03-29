# Developing FineCode

This guide is for developers contributing to FineCode itself — the monorepo structure, conventions, and workflows used internally.

## Repository structure

The repo is a monorepo. Each package has its own `pyproject.toml`. The root directory is the workspace.

```text
finecode/                          # Main package (Workspace Manager)
finecode_extension_api/            # Public API for extension authors
finecode_extension_runner/         # Extension execution engine
finecode_jsonrpc/                  # JSON-RPC client/transport layer
finecode_httpclient/               # HTTP client for extensions
finecode_builtin_handlers/         # Built-in action handlers
extensions/                        # Extension packages (ruff, flake8, mypy, ...)
presets/                           # Preset packages (recommended, lint, format)
finecode_dev_common_preset/        # Preset used for developing FineCode itself
tests/                             # Test suite
```

## Setting up the development environment

```bash
# From the repo root, inside the dev_workspace venv:
python -m finecode prepare-envs

# Re-prepare a single environment (e.g. after changing its dependencies):
python -m finecode prepare-envs --env=dev_no_runtime

# Multiple envs at once:
python -m finecode prepare-envs --env=dev --env=dev_no_runtime

# Prepare only a specific project:
python -m finecode prepare-envs --project=finecode_extension_api

# Combine filters — one env in one project:
python -m finecode prepare-envs --project=finecode_extension_api --env=dev_no_runtime
```

## Running checks

```bash
python -m finecode run lint
python -m finecode run check_formatting
pytest tests/
```

## Logging strategy (development policy)

This section defines the logging policy contributors should follow when adding or changing logs in FineCode.

The policy below defines the approach for reducing noise while keeping deep diagnostics available.

### Goals

- keep logs useful in normal development and CI runs
- allow deep diagnostics only when needed
- make noisy areas controllable per module
- avoid logging sensitive data

### Level policy

- `ERROR`: operation failed and needs attention; include actionable context
- `WARNING`: recoverable problem, degraded behavior, or skipped step
- `INFO`: lifecycle milestones and key business events (start/stop, action run result)
- `DEBUG`: developer diagnostics for branch decisions and compact internal state
- `TRACE`: high-volume details (payload previews, loop-level details, per-item processing)

Rules:

- default global level must be `INFO`
- `TRACE` must be disabled by default
- `TRACE` should be opt-in for specific modules or short debugging sessions
- avoid `INFO` in tight loops; use `TRACE`/`DEBUG` instead

### Module-level overrides (target contract)

Use per-module log levels so diagnostics can be enabled surgically without turning on global trace.

Recommended config shape:

```toml
[tool.finecode.logging]
default_level = "INFO"
format = "json"

[tool.finecode.logging.module_levels]
"finecode.wm_server.services.run_service" = "TRACE"
"finecode.wm_server.runner.runner_manager" = "DEBUG"
```

Recommended env override pattern:

```bash
FINECODE_LOG_LEVEL=INFO
FINECODE_LOG_LEVEL_FINECODE_API_SERVER_SERVICES_RUN_SERVICE=TRACE
```

Recommended CLI override pattern:

```bash
python -m finecode run --log-level=TRACE lint
python -m finecode start-wm-server --log-level=DEBUG
```

Notes:

- `--log-level` is supported by all commands: `run`, `prepare-envs`, `dump-config`, `start-lsp`, `start-wm-server`, `start-mcp`
- `prepare-envs --env=<name>` limits environment preparation to the named env(s); the flag may be repeated
- `prepare-envs --project=<name>` limits to the named project(s); the flag may be repeated; can be combined with `--env`
- when a CLI command spawns a dedicated WM server subprocess, the log level is propagated automatically
- module overrides should take precedence over global level (not yet implemented)

### What to log

Log at boundaries where failures or latency matter:

- request start/end with identifiers (`request_id`, `run_id`, `project`, `action`)
- external process and RPC boundaries (spawn, send, receive, timeout, cancel)
- retries, fallbacks, and decision points
- final result summary (status, duration, item counts)

For high-volume objects:

- log previews and metadata instead of full payloads
- include sizes/counts (`len`, keys, return code) rather than full dumps
- use full payload logs only at `TRACE`

### Safety and performance guardrails

- never log secrets or tokens (API keys, auth headers, credentials, full env dumps)
- redact known sensitive keys (`token`, `password`, `secret`, `authorization`)
- prefer lazy/cheap log construction on hot paths
- guard expensive `TRACE` formatting with level checks

### Incident workflow

- keep production/dev default at `INFO`
- during incident analysis, enable `TRACE` only for affected modules
- ~~prefer time-bounded overrides (TTL) so verbose logging auto-reverts~~
- once resolved, remove temporary overrides and keep only useful `INFO`/`WARNING`

## Dependency lock files

FineCode uses [pylock.toml](https://packaging.python.org/en/latest/specifications/pylock-toml/) lock files for reproducible dependency installation.

### Why lock files

Without lock files, `prepare-envs` resolves dependency versions from the ranges declared in `pyproject.toml` at install time. This means two developers (or CI runs) can end up with different versions depending on when they ran the command. Lock files pin exact versions for reproducible environments.

### Lock files are environment-specific

Each FineCode environment (`dev_workspace`, `dev_no_runtime`, `runtime`, etc.) has its own set of dependencies, so each needs its own lock file:

```text
pylock.<env_name>.toml
```

For example:

```text
myproject/
  pyproject.toml
  pylock.dev_workspace.toml
  pylock.dev_no_runtime.toml
  pylock.runtime.toml
```

### Lock files are platform- and Python version-specific

A lock file records the exact dependency resolution for one platform and one Python version. The same `pyproject.toml` can resolve differently on Linux vs macOS, or Python 3.12 vs 3.13.

If the project targets a single platform and Python version, one lock file per env is enough. For multiple targets, encode platform and version into the file name (the `<name>` segment in `pylock.<name>.toml` must not contain dots):

```text
myproject/
  pyproject.toml
  locks/
    pylock.dev_workspace-linux-py312.toml
    pylock.dev_workspace-linux-py313.toml
    pylock.dev_workspace-macos-py312.toml
    pylock.dev_no_runtime-linux-py312.toml
    ...
```

### Generating lock files

Use the `lock_dependencies` action:

```bash
python -m finecode run lock_dependencies \
    --src_artifact_def_path=pyproject.toml \
    --output_path=pylock.dev_workspace.toml
```

For the Python ecosystem, the `PipLockDependenciesHandler` runs `pip lock` under the hood.

### Installing from lock files

There are two lock-file handlers depending on the pipeline you use:

- **`PrepareEnvInstallDepsFromLockHandler`** — used in the per-environment `prepare_env` pipeline (the default). Reads `pylock.<env_name>.toml` and passes pinned versions to `install_deps_in_env` for that single env.
- **`PrepareEnvsInstallDepsFromLockHandler`** — legacy multi-env variant that handles all environments in one handler. Use only if you are running a custom `prepare_envs` pipeline that does not go through `PrepareEnvsDispatchHandler`.

Both look for `pylock.<env_name>.toml` next to the project's `pyproject.toml`. If a lock file is not found for an env, it is skipped with a warning.

### Lock files in CI

Lock files should be committed to the repository. CI should install from them, not regenerate them:

```bash
# CI installs from existing lock files — reproducible
python -m finecode prepare-envs
```

To update lock files, run `lock_dependencies` locally or in a scheduled CI job and commit the result. For multi-platform projects, use a CI matrix to generate lock files on each target platform.


## JSON-RPC key naming convention

All JSON-RPC channels in FineCode use **camelCase** for message keys:

| Channel | Convention | Reason |
|---|---|---|
| WM server ↔ any client (internal TCP) | **camelCase** | Standard for JSON-based protocols; language-agnostic (clients may be written in Go, TypeScript, Rust, etc.) |
| LSP command handlers → IDE | **camelCase** | Same convention; no conversion needed |
| ER ↔ WM (pygls custom commands) | **camelCase** | Consistent with WM protocol |

### Rule: write keys explicitly, no auto-conversion

Handler return dicts must use camelCase keys **written explicitly**. There is no automatic snake_case → camelCase conversion in the WM server. Auto-conversion is fragile — it was the root cause of the `return_code` bug in `_handle_run_action` where only the inner value was wrapped in `_NoConvert` but the outer keys were still silently converted.

```python
# correct — keys written as camelCase explicitly
return {"returnCode": result.return_code, "resultByFormat": result.result_by_format}

# wrong — snake_case keys in a JSON response
return {"return_code": result.return_code, "result_by_format": result.result_by_format}
```

Python **internal** data structures (dataclass fields, local variables, function parameters) stay snake_case per Python convention. Only the dict keys that cross a JSON-RPC boundary are camelCase.

### What this means per layer

**WM server handlers** (`wm_server.py`): return dicts with camelCase keys directly. No `_NoConvert` wrapper, no `_convert_to_camel_case` call.

**`wm_client.py`**: accesses response keys in camelCase.

**Python CLI clients** (`prepare_envs_cmd.py`, `run_cmd.py`): access camelCase keys from responses.

**LSP command handlers** (`lsp_server/endpoints/`): pass WM responses through to the IDE as-is — no conversion needed since the WM already produces camelCase.

**ER response dicts** (`finecode_extension_runner`): use camelCase keys (`returnCode`, `resultByFormat`, `status`).

## Referencing ADRs in source code

When code implements a non-obvious constraint or design choice, add a comment referencing the relevant ADR. This prevents future contributors from accidentally "fixing" something that was intentionally designed that way.

```python
# Single shared IO thread services all active ERs — see docs/adr/0003-*.md
_io_thread = threading.Thread(target=_service_loop, daemon=True)
```

**When to add an ADR reference:**

- The implementation looks like it could be simplified but cannot be
- There is a temptation to refactor in a way that would violate the decision
- The constraint is not derivable from the code itself

**When not to add one:**

- The code is self-explanatory
- The ADR covers a broad design area — reference it only at the specific site that enforces the decision, not everywhere related code appears

ADR references differ from user-doc references: user docs explain the *API surface* for consumers; ADRs explain *why a constraint exists* for contributors.

## Code Style

### Typing

- type the code
-- use complete types, no holes in generics like `list` instead of `list[int]`

### Imports

- keep imports at the top of the module
- keep imports at the root level of module
-- there are 2 exceptions:
    - you need to avoid circle dependency (usually it means there is a problem in code structure)
    - you want to avoid loading the module on startup (e.g. don't import all CLI command handlers if only one is needed for current CLI call)
  
### Exports

- explicitly export public module members using `__all__`
-- it may not contain dynamic elements, only literal strings
