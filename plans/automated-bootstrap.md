# Automated Bootstrap for dev_workspace

## Problem

The first `dev_workspace` environment at the workspace root must be created manually
before `prepare-envs` can run. This is a chicken-and-egg problem: FineCode needs to be
installed to run its own environment setup, but the environment setup is what installs
FineCode.

Current manual process (3 commands):

```bash
python -m venv .venvs/dev_workspace
source .venvs/dev_workspace/bin/activate   # Windows: .venvs\dev_workspace\Scripts\activate
python -m pip install --group="dev_workspace"
```

**Prerequisites:** Python 3.11+ and pip 25.1+ (for `--group` support).

**Goal:** Provide an automated alternative so users don't need to know these steps.

---

## Proposed Solution: Reuse Existing Handlers via Temporary Runner

### Core Idea

When `finecode bootstrap` is invoked (e.g. via `pipx run finecode bootstrap`), finecode
and all its handlers are already installed in the invoking Python environment (the pipx
ephemeral venv). Instead of reimplementing venv creation and dependency installation, the
bootstrap command starts the WM + Extension Runner infrastructure using **the current
Python process** (`sys.executable`) as a temporary `dev_workspace` runner, then runs the
same `create_envs` + `install_envs` actions that `prepare-envs` uses.

### How It Works

```
pipx run finecode bootstrap
    │
    ├─ Start WM server
    ├─ Discover project (add_dir, start_runners=False)
    ├─ Start dev_workspace runner using sys.executable (not .venvs/dev_workspace/bin/python)
    │   └─ All handlers available: fine_python_virtualenv, fine_python_pip, builtins
    ├─ Run create_envs (for dev_workspace env of workspace root)
    ├─ Run install_envs (for dev_workspace env of workspace root)
    └─ Shut down
```

After this, `.venvs/dev_workspace/` exists with finecode installed. The user can then
run `prepare-envs` normally.

### Why This Approach

- **No reimplementation** — uses the same `create_envs` / `install_envs` handlers that
  `prepare-envs` uses, including all config resolution, `include-group` handling,
  path-based deps, editable installs
- **Single source of truth** — if handler logic changes, bootstrap gets the changes for free
- **Tested code paths** — the handler code is already tested via `prepare-envs`
- **Consistent behavior** — the resulting `dev_workspace` venv is identical to what
  `prepare-envs` would produce

---

## Invocation Methods

### With pipx (recommended for users who have Python)

```bash
pipx run finecode bootstrap
```

`pipx run` creates a temporary venv, installs finecode from PyPI, runs bootstrap, cleans
up. The bootstrap command then creates the project-local `.venvs/dev_workspace/` with the
exact versions from `pyproject.toml`.

### With uv

```bash
uvx finecode bootstrap
```

Same mechanism, using `uv` instead. Also works when user has no Python — `uv` can install
Python itself.

### With finecode already installed somewhere

```bash
finecode bootstrap
# or
python -m finecode bootstrap
```

### No-Python path

```bash
# Install uv (single binary, no Python needed)
# Linux/macOS:
curl -LsSf https://astral.sh/uv/install.sh | sh
# Windows:
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# uv installs Python automatically if needed
uvx finecode bootstrap
```

This is documentation-only — no code changes needed.

### Version mismatch concern

The pipx/uvx-installed finecode version might differ from what `pyproject.toml` requests.
This is acceptable because:
- Bootstrap only creates the venv and installs deps — it doesn't run user handlers
- The installed deps come from `pyproject.toml`, not from the temp finecode
- After bootstrap, the project-local finecode (inside dev_workspace) takes over

---

## Implementation Plan

### Phase 1: Allow runner to use an arbitrary Python path

#### 1.1 Extend `get_python_cmd` to accept an override

**File:** `src/finecode/wm_server/runner/finecode_cmd.py`

Currently `get_python_cmd` always resolves to `.venvs/<env>/bin/python`. Add a mechanism
for the WM to override the Python path for a specific runner. Two possible approaches:

**Option A — Context-level override map:**

Add a `python_cmd_overrides: dict[tuple[Path, str], str]` to `WorkspaceContext` (keyed by
`(project_path, env_name)`). When set, `_start_extension_runner_process` uses the override
instead of calling `get_python_cmd`.

**Option B — Runner-level field:**

Add an optional `python_cmd_override: str | None` to `ExtensionRunnerInfo`. When set,
`_start_extension_runner_process` uses it directly.

**Recommendation:** Option B — it's local to the runner and doesn't leak into the global
context. The override is only set during bootstrap.

#### 1.2 Modify `_start_extension_runner_process`

**File:** `src/finecode/wm_server/runner/runner_manager.py` (line ~80)

```python
async def _start_extension_runner_process(runner, ws_context, debug=False):
    if runner.python_cmd_override:
        python_cmd = runner.python_cmd_override
    else:
        python_cmd = finecode_cmd.get_python_cmd(
            runner.working_dir_path, runner.env_name
        )
    # ... rest unchanged
```

### Phase 2: `finecode bootstrap` command

#### 2.1 Add `bootstrap_cmd.py`

**File:** `src/finecode/cli_app/commands/bootstrap_cmd.py`

```python
async def bootstrap(
    workdir_path: Path,
    recreate: bool = False,
    log_level: str = "INFO",
) -> None:
```

The function follows the same pattern as `prepare_envs_cmd.prepare_envs` but only for
the workspace root's `dev_workspace`:

1. **Pre-check:** Read `pyproject.toml`, verify `[dependency-groups].dev_workspace` exists.
2. **Check existing venv:**
   - If `.venvs/dev_workspace/` exists and `--recreate` not set → print message, exit.
   - If exists and `--recreate` → remove it.
3. **Start WM server** (same as `prepare-envs`: `wm_lifecycle.start_own_server`).
4. **Connect ApiClient.**
5. **Discover project:** `client.add_dir(workdir_path, start_runners=False)`.
6. **Tell WM to use `sys.executable` for dev_workspace runner.** This requires a new
   API call or parameter — see Phase 1. Alternatively, the bootstrap command can pass
   the override via an env var or a new parameter on `add_dir` / `start_runners`.
7. **Start dev_workspace runner** (using `sys.executable`).
8. **Run `create_envs`** for `dev_workspace` env of workspace root — same as
   `prepare_envs_cmd` step 3a.
9. **Run `install_envs`** for `dev_workspace` env of workspace root — same as
   `prepare_envs_cmd` step 3b.
10. **Shut down** WM server and runners.
11. **Print success message** with next-step instructions.

#### 2.2 How to pass the python_cmd_override to the WM

The bootstrap command runs as a client talking to the WM server via JSON-RPC. The override
needs to reach `_start_extension_runner_process` inside the WM. Options:

**Option A — New API method `set_runner_python_override(project, env, python_path)`:**

A dedicated WM API method that sets the override before starting runners. Clean but
adds a purpose-specific API.

**Option B — Parameter on `start_runners`:**

Extend `start_runners` to accept `python_overrides: dict[str, str]` (env_name → python
path). The WM stores this in context and uses it when starting runners.

**Option C — Environment variable:**

Set `FINECODE_BOOTSTRAP_PYTHON` env var before starting the WM server subprocess. The WM
reads it in `_start_extension_runner_process` for the `dev_workspace` env. Simple but
implicit.

**Recommendation:** Option B — it's explicit, doesn't pollute the API with single-use
methods, and the WM server subprocess inherits no hidden state.

#### 2.3 Register CLI subcommand

**File:** `src/finecode/cli.py`

```python
@cli.command()
@click.option("--recreate", is_flag=True, default=False,
              help="Delete and recreate dev_workspace if it already exists")
@click.option("--log-level", "log_level", default="INFO",
              type=click.Choice(["TRACE", "DEBUG", "INFO", "WARNING", "ERROR"],
              case_sensitive=False), show_default=True)
def bootstrap(recreate: bool, log_level: str) -> None:
    """Create the dev_workspace environment for the workspace root."""
    from finecode.cli_app.commands import bootstrap_cmd

    logger_utils.init_logger(log_name="cli", log_level=log_level, stdout=True)
    asyncio.run(
        bootstrap_cmd.bootstrap(
            workdir_path=pathlib.Path(os.getcwd()),
            recreate=recreate,
            log_level=log_level,
        )
    )
```

#### 2.4 Console script entry point

**File:** `pyproject.toml`

Ensure the `finecode` console script is registered:

```toml
[project.scripts]
finecode = "finecode.cli:cli"
```

This is what makes `pipx run finecode bootstrap` and `uvx finecode bootstrap` work.

### Phase 3: Shared logic between bootstrap and prepare-envs

The core of steps 3a-3b in `prepare_envs_cmd._run` (lines 159-210) and the bootstrap
command do the same thing: run `create_envs` + `install_envs` for a `dev_workspace` env.
Extract this into a shared helper:

**File:** `src/finecode/cli_app/commands/_bootstrap_utils.py` (or inline in both)

```python
async def create_and_install_dev_workspace(
    client: ApiClient,
    project_path: str,
    envs: list[dict],
    dev_env: str = "cli",
) -> None:
    """Run create_envs + install_envs for dev_workspace environments."""
    options = {
        "resultFormats": ["string"],
        "trigger": "user",
        "devEnv": dev_env,
    }
    # create_envs
    result = await client.run_action(
        action="create_envs", project=project_path,
        params={"envs": envs}, options=options,
    )
    # ... error handling ...

    # install_envs
    result = await client.run_action(
        action="install_envs", project=project_path,
        params={"envs": envs}, options=options,
    )
    # ... error handling ...
```

Then `prepare_envs_cmd` and `bootstrap_cmd` both call this helper.

### Phase 4: Documentation

#### 4.1 Update getting-started.md

Replace the manual 3-step process:

```markdown
## 1. Bootstrap dev_workspace

### Quick start (recommended)

If you have `pipx` (bundled with Python 3.13+):

    pipx run finecode bootstrap

Or with `uv`:

    uvx finecode bootstrap

### Manual alternative

    python -m venv .venvs/dev_workspace
    .venvs/dev_workspace/bin/pip install --group="dev_workspace"
```

#### 4.2 Update preparing-environments.md

Add the bootstrap command under "Workspace root bootstrap (manual, one-time)" — make
it "Workspace root bootstrap" with automated as primary and manual as fallback.

#### 4.3 Add "No Python" guidance

Short section in getting-started.md pointing to `uv` for users without Python.

### Phase 5: Testing

#### 5.1 Unit tests

- `python_cmd_override` is used when set on runner
- `get_python_cmd` still works normally when no override

#### 5.2 Integration test

- Bootstrap with a minimal `pyproject.toml`
- Verify `.venvs/dev_workspace/` exists and contains expected packages
- Verify `finecode prepare-envs` works afterward

---

## Edge Cases and Considerations

### venv module availability

When the runner uses `sys.executable` from a pipx venv, `virtualenv` (not `venv`) is used
for creating the target env — this is the existing `VirtualenvCreateEnvHandler` behavior.
`virtualenv` is a finecode dependency, so it's always available. No `python3-venv` package
needed.

### Windows compatibility

- `sys.executable` works cross-platform
- Runner startup already handles Windows in `get_python_cmd` (though currently only checks
  `bin/python` — this is a pre-existing issue, not bootstrap-specific, see finecode_cmd.py)
- The `finecode_cmd.get_python_cmd` Windows path (`Scripts/python.exe`) should be fixed
  as a prerequisite or in the same PR

### Idempotency

- Default: if `.venvs/dev_workspace/` exists, print message and exit 0
- With `--recreate`: delete and recreate from scratch

### What if the temporary finecode lacks a handler?

If a future version adds a new handler needed for bootstrap but the pipx-installed version
is older, bootstrap would fail with a clear error (action/handler not found). The user can
pin the version: `pipx run finecode==0.4.0 bootstrap`.

### Relationship to `prepare-envs`

`bootstrap` is a subset of `prepare-envs`:

| | `bootstrap` | `prepare-envs` |
|---|---|---|
| Creates workspace root dev_workspace | Yes | No (assumes it exists) |
| Creates subproject dev_workspaces | No | Yes |
| Creates all other envs | No | Yes |
| Installs all deps | No | Yes |
| Starts runners | Temporarily | Yes |

After `bootstrap`, the user runs `prepare-envs` for the rest. Alternatively, a future
enhancement could make `prepare-envs` call `bootstrap` internally when it detects the
workspace root's `dev_workspace` is missing.

---

## Summary of User Experience After Implementation

| User has | Command | Steps |
|---|---|---|
| Python 3.13+ (pipx bundled) | `pipx run finecode bootstrap` | 1 |
| Python + pipx | `pipx run finecode bootstrap` | 1 |
| Python + uv | `uvx finecode bootstrap` | 1 |
| Python only | `pip install finecode && finecode bootstrap` | 2 |
| Nothing | Install uv → `uvx finecode bootstrap` | 2 |
| Prefers manual | `python -m venv ... && pip install --group=...` | 3 (unchanged) |

After bootstrap:

```bash
finecode prepare-envs   # or: .venvs/dev_workspace/bin/finecode prepare-envs
finecode run lint        # etc.
```
