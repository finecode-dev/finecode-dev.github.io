# MCP Server Optimization for AI Assistants

Analysis of how AI assistants (Claude Code, etc.) interact with FineCode via the MCP server,
and what can be improved.

## Current Architecture

```
AI Assistant <-stdio/MCP-> MCP Server <-TCP/JSON-RPC-> WM Server <-JSON-RPC-> Extension Runners
```

The MCP server (`src/finecode/mcp_server.py`) is a thin proxy that dynamically discovers
actions from the WM, generates JSON Schema from payload dataclasses via
`finecode_extension_runner/src/finecode_extension_runner/schema_utils.py`, and exposes them
as MCP tools.

## What works well

- **Dynamic tool discovery** — new extensions/actions automatically appear as MCP tools
- **Typed schemas from dataclasses** — `schema_utils.py` extracts real JSON Schema from
  `RunActionPayload` fields
- **Batch mode** — omitting `project` runs across all projects
- **`devEnv: "ai"` tagging** — handlers can detect AI-triggered runs

---

## Issues (ordered by priority)

### P0-1: Tool descriptions are generic

Every action gets: `"Run {name} on a project or the whole workspace"`.
AI assistants cannot distinguish tools or decide which to use.

**Solution:** Class docstring on each `Action` subclass, read via `__doc__` in
`services.py`, passed through the existing schema dict (ER → WM → MCP), used as
`Tool.description` in `mcp_server.py`.

**Status:** DONE

---

### ~~P0-2: Results always requested as JSON only~~ — NOT AN ISSUE

JSON is actually better for AI assistants than text: it provides exact file paths, line
numbers, error codes for programmatic decisions. The `string` format (`to_text()`) is
designed for human terminal output. Current `resultFormats: ["json"]` is correct.

**Status:** CLOSED (not an issue)

---

### P1-1: Schema lacks field descriptions

`schema_utils.py` extracts types but not docstrings or field descriptions. AI assistants
misuse parameters when descriptions are absent (e.g., not knowing empty `file_paths` means
"use defaults").

**Solution:** Attribute docstrings (string literal after field assignment) on
`RunActionPayload` fields, extracted in `schema_utils.py` via `inspect.getsource()` +
`ast.parse()`, injected as `"description"` in each JSON Schema property.

**Status:** DONE

---

### P1-2: Missing workspace introspection tools

AI assistants need to understand the workspace before acting. Not exposed via MCP:
- `runners/list` — which runners exist/running (critical for diagnosing failures)
- `actions/list` per-project — what actions are available for each project
- `workspace/getProjectRawConfig` — what is configured

**Solution:** Add these as MCP tools.

**Status:** DONE

---

### P1-3: Internal actions exposed to AI

Actions like `lint_files`, `format_files`, `prepare_runner_envs`, `create_envs` are
LSP/internal. Exposing them alongside user-facing counterparts (`lint`, `format`) confuses
AI assistants.

**Solution:** Filter internal actions from the MCP tool list (e.g., via a flag on the
action class or a deny-list).

**Status:** TODO

---

### P2-1: Project parameter requires prior knowledge

AI must call `list_projects` first to discover valid paths. Including an enum of all project
paths in every tool schema would be too verbose (e.g., 26 paths × N tools).

**Solution:** Improve the `project` field description to reference `list_projects`:
`"Absolute path to the project directory. Use the list_projects tool to see available
projects. Omit to run on all projects in the workspace."`

**Status:** DONE

---

### P2-2: No MCP tool annotations

MCP supports `readOnlyHint`, `destructiveHint`, `openWorldHint`. Not set, so AI assistants
can't distinguish read-only (`lint`, `list_tests`) from mutating (`format`, `publish_artifact`)
from infrastructure (`prepare_runner_envs`) tools.

**Solution:** Add annotations based on action class metadata or a mapping.

**Status:** TODO

---

### P2-3: Error messages lack actionable guidance

When envs aren't prepared, errors propagate as raw JSON-RPC errors. The WM emits helpful
`server/userMessage` notifications ("Did you run prepare-envs?") but these go to LSP
clients, not MCP responses.

**Solution:** Catch common failure patterns in MCP `call_tool` and enrich error text with
next-step hints.

**Status:** TODO

---

### P3-1: dump_config not exposed as MCP tool

AI assistants have no way to understand the resolved (post-preset-merge) configuration
without reading pyproject.toml themselves.

**Solution:** Expose `dump_config` as an MCP tool.

**Status:** DONE

---

### P3-2: No progress visibility for long operations

MCP server uses blocking `actions/run`. Long operations (prepare_handler_envs, run_tests)
show no progress. WM has `actions/runWithPartialResults` but MCP stdio doesn't natively
support server-initiated streaming.

**Solution:** All action tool calls now use `actions/runWithPartialResults` (passing
`project=""` for the all-projects case, replacing `run_batch`). Each `actions/partialResult`
notification from the WM is forwarded to the MCP client as a `notifications/message` log
message via `session.send_log_message()`. Per-request isolation is handled by a
token-keyed `asyncio.Queue`; a concurrent `_forward` task drains it and is cancelled
when the final result arrives.

**Status:** DONE
