# Plan: Git Pre-commit Hook Support

## Overview

Add pre-commit hook functionality to FineCode: install a git hook that runs configured code quality checks on staged files before commit.

Two new actions:
- **`install_git_hooks`** — generates a Python-based `.git/hooks/pre-commit` script
- **`precommit`** — orchestrator invoked by the hook; gets staged files, dispatches to sub-actions

## Action Flow

```
git commit
  └─ .git/hooks/pre-commit (Python script)
       └─ python -m finecode run --dev-env=precommit precommit
            └─ PrecommitHandler:
                 1. git diff --cached --name-only --diff-filter=ACMR
                 2. group_src_artifact_files_by_lang (filter to known languages)
                 3. for each configured sub-action (check_formatting, lint):
                      run action with target=FILES, file_paths=staged_files
                 4. aggregate results → exit 0 or 1
```

`DevEnv.PRECOMMIT = "precommit"` already exists in `code_action.py`.

## Registration

| Component | Where |
|---|---|
| `install_git_hooks` action + handler | `base_config.toml` (always available, like `dump_config`) |
| `precommit` action + handler | `fine_python_precommit` preset only |

## Action Definitions

### `precommit_action.py` — `actions/code_quality/`

```python
@dataclasses.dataclass
class PrecommitRunPayload(code_action.RunActionPayload):
    actions: list[str] = dataclasses.field(default_factory=list)
    """Actions to run. Empty list uses handler config defaults (e.g. ['check_formatting', 'lint'])."""
    file_paths: list[Path] = dataclasses.field(default_factory=list)
    """Explicit file list. Empty means auto-detect staged files from git."""

@dataclasses.dataclass
class PrecommitRunResult(code_action.RunActionResult):
    action_results: dict[str, code_action.RunActionResult]

    def update(self, other: PrecommitRunResult) -> None:
        self.action_results.update(other.action_results)

    @property
    def return_code(self) -> code_action.RunReturnCode:
        if any(r.return_code != code_action.RunReturnCode.SUCCESS
               for r in self.action_results.values()):
            return code_action.RunReturnCode.ERROR
        return code_action.RunReturnCode.SUCCESS

class PrecommitRunContext(code_action.RunActionContext[PrecommitRunPayload]): ...

class PrecommitAction(code_action.Action[PrecommitRunPayload, PrecommitRunContext, PrecommitRunResult]):
    """Run configured code quality checks on git-staged files before commit."""
    PAYLOAD_TYPE = PrecommitRunPayload
    RUN_CONTEXT_TYPE = PrecommitRunContext
    RESULT_TYPE = PrecommitRunResult
```

### `install_git_hooks_action.py` — `actions/system/`

```python
@dataclasses.dataclass
class InstallGitHooksRunPayload(code_action.RunActionPayload):
    hook_types: list[str] = dataclasses.field(default_factory=lambda: ["pre-commit"])
    """Git hook types to install."""
    force: bool = False
    """Overwrite existing hook files."""

@dataclasses.dataclass
class InstallGitHooksRunResult(code_action.RunActionResult):
    installed_hooks: list[str]
    """Hook types that were successfully installed."""
    skipped_hooks: list[str]
    """Hook types skipped because a hook already exists (and force=False)."""

class InstallGitHooksRunContext(code_action.RunActionContext[InstallGitHooksRunPayload]): ...

class InstallGitHooksAction(code_action.Action[InstallGitHooksRunPayload, InstallGitHooksRunContext, InstallGitHooksRunResult]):
    """Install git hooks that run FineCode checks before commit."""
    PAYLOAD_TYPE = InstallGitHooksRunPayload
    RUN_CONTEXT_TYPE = InstallGitHooksRunContext
    RESULT_TYPE = InstallGitHooksRunResult
```

## Handler Implementations

### `PrecommitHandler` — `finecode_builtin_handlers/`

```python
@dataclasses.dataclass
class PrecommitHandlerConfig(code_action.ActionHandlerConfig):
    actions: list[str] = dataclasses.field(default_factory=lambda: ["check_formatting", "lint"])

class PrecommitHandler(code_action.ActionHandler[PrecommitAction, PrecommitHandlerConfig]):
    def __init__(self, action_runner, logger, file_editor): ...

    async def run(self, payload, run_context) -> PrecommitRunResult:
        # 1. Get staged files (or use payload.file_paths if provided)
        staged_files = payload.file_paths or await self._get_staged_files()
        if not staged_files:
            return PrecommitRunResult(action_results={})

        # 2. Which actions to run (payload overrides config)
        actions_to_run = payload.actions or self.config.actions

        # 3. Run sequentially — order matters (format before lint)
        action_results = {}
        for action_name in actions_to_run:
            sub_payload = self._build_sub_payload(action_name, staged_files)
            result = await self.action_runner.run_action(...)
            action_results[action_name] = result

        return PrecommitRunResult(action_results=action_results)

    async def _get_staged_files(self) -> list[Path]:
        # git diff --cached --name-only --diff-filter=ACMR
        # Filter: Added, Copied, Modified, Renamed (excludes Deleted)
        ...

    def _build_sub_payload(self, action_name, staged_files):
        # Maps action names to their payload types:
        # - check_formatting / format → FormatRunPayload(target=FILES, file_paths=..., save=True/False)
        # - lint → LintRunPayload(target=FILES, file_paths=...)
        ...
```

### `InstallGitHooksHandler` — `finecode_builtin_handlers/`

```python
class InstallGitHooksHandler(code_action.ActionHandler[InstallGitHooksAction, ...]):
    async def run(self, payload, run_context) -> InstallGitHooksRunResult:
        # 1. Find .git directory via `git rev-parse --git-dir`
        # 2. For each hook_type in payload.hook_types:
        #    - Check if hook file exists; skip if exists and not force
        #    - Write Python hook script (see below)
        #    - chmod +x on POSIX (no-op on Windows, git handles it)
        ...
```

## Generated Hook Script (Python, cross-platform)

```python
#!/usr/bin/env python3
"""Git pre-commit hook installed by FineCode.

To uninstall: delete this file.
To reinstall: python -m finecode run install_git_hooks
"""
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-m", "finecode", "run", "--dev-env=precommit", "precommit"],
    cwd=subprocess.check_output(
        ["git", "rev-parse", "--show-toplevel"], text=True
    ).strip(),
)
sys.exit(result.returncode)
```

Uses `sys.executable` to ensure the hook runs with the same Python that installed it.

## Preset: `fine_python_precommit`

New package at `presets/fine_python_precommit/fine_python_precommit/preset.toml`:

```toml
[tool.finecode.action.precommit]
source = "finecode_extension_api.actions.code_quality.precommit_action.PrecommitAction"

[[tool.finecode.action.precommit.handlers]]
name = "precommit"
source = "finecode_builtin_handlers.PrecommitHandler"
env = "dev_workspace"
dependencies = ["finecode_builtin_handlers~=0.2.0a0"]

[[tool.finecode.action_handler]]
source = "finecode_builtin_handlers.PrecommitHandler"
config.actions = ["check_formatting", "lint"]
```

## `base_config.toml` additions (install_git_hooks only)

```toml
[tool.finecode.action.install_git_hooks]
source = "finecode_extension_api.actions.system.install_git_hooks_action.InstallGitHooksAction"

[[tool.finecode.action.install_git_hooks.handlers]]
name = "install_git_hooks"
source = "finecode_builtin_handlers.InstallGitHooksHandler"
env = "dev_workspace"
dependencies = ["finecode_builtin_handlers~=0.2.0a0"]
```

## User Configuration

### Enable precommit

```toml
# pyproject.toml
[tool.finecode]
presets = [
    { source = "fine_python_recommended" },
    { source = "fine_python_precommit" },
]
```

### Switch to auto-fix mode

```toml
[[tool.finecode.action_handler]]
source = "finecode_builtin_handlers.PrecommitHandler"
config.actions = ["format", "lint"]
```

## File Structure

```
New files:
  finecode_extension_api/src/.../actions/code_quality/precommit_action.py
  finecode_extension_api/src/.../actions/system/install_git_hooks_action.py
  finecode_builtin_handlers/src/.../precommit.py
  finecode_builtin_handlers/src/.../install_git_hooks.py
  presets/fine_python_precommit/fine_python_precommit/preset.toml
  presets/fine_python_precommit/pyproject.toml

Modified files:
  finecode_extension_api/src/.../actions/__init__.py               (add exports)
  finecode_extension_api/src/.../actions/code_quality/__init__.py   (add export)
  finecode_extension_api/src/.../actions/system/__init__.py         (add export)
  finecode_builtin_handlers/src/finecode_builtin_handlers/__init__.py (add exports)
  src/finecode/base_config.toml   (add install_git_hooks action + handler)
```

## Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| CLI entry point | `python -m finecode run precommit` (reuse `run`) | `DevEnv.PRECOMMIT` already exists; no new CLI commands needed |
| Default mode | Check-only (`check_formatting` + `lint`) | Configurable; auto-fix is opt-in |
| Hook script language | Python | Cross-platform (FineCode works on Windows) |
| Staged file detection | Handler calls `git diff --cached` internally | Keeps hook script trivial |
| Sub-action execution | Sequential, not parallel | Order matters (format before lint) |
| Existing hook conflict | `force` parameter; skip + warn by default | Safe default |
| `precommit` registration | Preset only (not `base_config.toml`) | Action doesn't exist until user opts in |
| `install_git_hooks` registration | `base_config.toml` | System action, always available |

## V1 Limitations (documented)

- Runs against **working tree files**, not the git index — partially staged files are checked in full working-tree state. Stash/unstash deferred to v2.
- Only `pre-commit` hook type supported.
- No automatic re-staging after auto-fix (user must `git add` if using `format` mode).
- Monorepo per-project routing deferred to v2.

## V2 Candidates

- Stash/unstash of unstaged changes for index-only checking
- Auto-format + re-stage mode
- `uninstall_git_hooks` action
- Other hook types (`pre-push`, `commit-msg`)
- Monorepo file-to-project routing
- `fine_python_recommended` integration (add `fine_python_precommit` to its preset list)
