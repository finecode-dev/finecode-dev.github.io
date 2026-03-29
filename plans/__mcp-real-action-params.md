# Plan: Real action parameters in MCP client

## Goal

Replace hardcoded `project` + `file_paths` parameters in MCP tools with the actual
payload fields defined by each action's `RunActionPayload` subclass.

## Design decisions

- **Schema source**: Extension Runner (`dev_workspace` env), not WM — the ER runs in
  the venv where action packages are installed. `dev_workspace` is always started first
  and has all preset/action packages installed for preset resolution.
- **On-demand only**: `actions/list` stays unchanged (lightweight). Schemas are fetched
  via a separate `actions/getPayloadSchemas` endpoint, called only when needed (e.g.
  MCP `list_tools`).
- **Cache in WM**: `WorkspaceContext` holds a per-project schema cache. Invalidated
  whenever `update_runner_config` is called for the project (single invalidation point).
- **Graceful degradation**: If an action class cannot be imported in `dev_workspace`,
  `null` is returned for that action's schema. MCP tool falls back to `{project: str}`.
- **MCP library**: Replace `fastmcp` with the official `mcp` SDK (`mcp.server.Server`).
  `list_tools()` is a per-request handler returning `list[Tool]` with explicit
  `inputSchema` dicts — no Pydantic introspection or function signature tricks needed.

---

## Progress

- [x] Step 1 — Schema extraction utility (ER)
- [x] Step 2 — New ER command `actions/getPayloadSchemas`
- [x] Step 3 — WM client function in `runner_client.py`
- [x] Step 4 — Schema cache in `WorkspaceContext`
- [x] Step 5 — New WM endpoint `actions/getPayloadSchemas`
- [x] Step 6 — New `ApiClient` method
- [x] Step 7 — `actions/list` stays unchanged (no work needed)
- [x] Step 8 — Replace FastMCP with `mcp` SDK in `mcp_server.py`
- [x] Step 9 — Documentation

---

## Steps

### Step 1 — Schema extraction utility (ER)

**New file**: `finecode_extension_runner/src/finecode_extension_runner/schema_utils.py`

Function `extract_payload_schema(payload_cls: type) -> dict` using
`dataclasses.fields()` + `typing.get_type_hints()`.

Returns a JSON Schema fragment:
```json
{
  "properties": {
    "file_paths": {"type": "array", "items": {"type": "string"}},
    "target":     {"type": "string", "enum": ["project", "files"]},
    "save":       {"type": "boolean"}
  },
  "required": ["save"]
}
```

Type mapping:
- `bool` → `boolean`
- `str`, `pathlib.Path` → `string`
- `int` → `integer`
- `list[T]` → `array` with resolved item schema
- `StrEnum` subclass → `string` + `enum` values
- Fields where both `field.default` and `field.default_factory` are `MISSING` → required

---

### Step 2 — New ER command `actions/getPayloadSchemas`

**`finecode_extension_runner/src/finecode_extension_runner/services.py`**

Add imports at the top alongside existing ones:
```python
from finecode_extension_runner import run_utils, schema_utils
```

New function `get_payload_schemas() -> dict[str, dict | None]`:

- Return `{}` immediately if `global_state.runner_context is None`
- Iterates `global_state.runner_context.project.actions.items()`
- For each action: calls `run_utils.import_module_member_by_source_str(action.source)`
  (same helper already used in `create_action_exec_info`) to get the action class
- Reads `action_cls.PAYLOAD_TYPE` via `getattr(action_cls, "PAYLOAD_TYPE", None)`,
  calls `schema_utils.extract_payload_schema`
- On any exception: stores `None` for that action (log at DEBUG level)

**`finecode_extension_runner/src/finecode_extension_runner/lsp_server.py`**

Add handler function (before `merge_results_cmd`):
```python
async def get_payload_schemas_cmd(ls: lsp_server.LanguageServer):
    logger.trace("Get payload schemas")
    return services.get_payload_schemas()
```
Note: `ls` is required as the first argument by the pygls command handler protocol,
even though it is not used here.

Register in `create_lsp_server()` alongside existing commands:
```python
server.command("actions/getPayloadSchemas")(get_payload_schemas_cmd)
```

---

### Step 3 — WM client function in `runner_client.py`

**`src/finecode/wm_server/runner/runner_client.py`**

New function:
```python
async def get_payload_schemas(runner: ExtensionRunnerInfo) -> dict[str, dict | None]:
```
Sends `actions/getPayloadSchemas` via the same `ExecuteCommandParams` pattern used by
all other ER commands.

---

### Step 4 — Schema cache in `WorkspaceContext`

**`src/finecode/wm_server/context.py`**

Add to `WorkspaceContext`:
```python
# project_path → {action_name: schema | None}
# cleared on every update_runner_config for the project
ws_action_schemas: dict[Path, dict[str, dict | None]]
```

**`src/finecode/wm_server/runner/runner_manager.py`**

`update_runner_config()` does not currently have `ws_context` in scope. Add it as a
new parameter:
```python
async def update_runner_config(
    runner: ExtensionRunnerInfo,
    project: CollectedProject,
    handlers_to_initialize: dict[str, list[str]] | None,
    ws_context: context.WorkspaceContext,           # ← new
) -> None:
```

After `runner_client.update_config(...)` succeeds, add:
```python
ws_context.ws_action_schemas.pop(project.dir_path, None)
```

Update both call sites (both have `ws_context` available):

1. `start_runner()` — the existing call near the end of the function.
2. `start_runners_with_presets()` — the call after preset resolution.

Both calls become:

```python
await update_runner_config(..., ws_context=ws_context)
```

This is the single invalidation point — it covers both initial config load and
preset-resolution updates.

---

### Step 5 — New WM endpoint `actions/getPayloadSchemas`

**`src/finecode/wm_server/wm_server.py`**

New handler `_handle_get_payload_schemas(params, ws_context)`:

```
params: { project: str, action_names: list[str] }
returns: { schemas: { action_name: schema | null } }
```

Logic:

1. Resolve the project with `_find_project_by_path(ws_context, params["project"])`
   (same helper used by `_handle_run_action`). Raise `ValueError` if not found or not
   a `CollectedProject`.
2. Check `ws_context.ws_action_schemas.get(project.dir_path, {})` for cached entries.
3. If any requested action names are missing from cache, populate the cache:

   **a.** Query `dev_workspace` runner (fast path — covers all `finecode_extension_api`
   actions in one call). Look it up via
   `ws_context.ws_projects_extension_runners[project.dir_path].get("dev_workspace")`.
   If running, call `await runner_client.get_payload_schemas(runner)` and store every
   returned entry in `ws_context.ws_action_schemas[project.dir_path]`.

   **b.** For actions still `None` after the `dev_workspace` call (action class not
   importable there — e.g. preset-specific or third-party actions): read
   `project.actions[action_name].handlers` to find the handler envs, then for each env
   that has a `RUNNING` runner call `await runner_client.get_payload_schemas(runner)`.
   Take the first non-`None` result and update the cache entry. Stop trying envs once a
   schema is found.

4. Return `{"schemas": {name: cache[name] for name in action_names}}`.

Register in the handler map at the bottom of the file:

```python
"actions/getPayloadSchemas": _handle_get_payload_schemas,
```

---

### Step 6 — New `ApiClient` method

**`src/finecode/wm_client.py`**

New method on `ApiClient`, following the same pattern as `list_actions`:

```python
async def get_payload_schemas(
    self, project: str, action_names: list[str]
) -> dict[str, dict | None]:
    """Return payload schemas for the given actions in a project.

    Delegates to the WM ``actions/getPayloadSchemas`` endpoint.
    The WM caches results and invalidates them on config changes,
    so repeated calls are cheap.

    Args:
        project: Absolute path to the project directory.
        action_names: List of action names to fetch schemas for.

    Returns:
        Mapping of action name → JSON Schema fragment, or ``None``
        for actions whose class could not be imported by the ER.
    """
    result = await self.request(
        "actions/getPayloadSchemas",
        {"project": project, "action_names": action_names},
    )
    if not isinstance(result, dict) or "schemas" not in result:
        raise ApiResponseError(
            "actions/getPayloadSchemas",
            f"missing 'schemas' field, got {result!r}",
        )
    return result["schemas"]
```

---

### Step 7 — `actions/list` stays unchanged

No changes to `domain.Action`, `_handle_list_actions`, or the list response format.

---

### Step 8 — Replace FastMCP with `mcp` SDK in `mcp_server.py`

**`src/finecode/mcp_server.py`** — full rewrite. Key imports:

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
```

**Structure:**

```python
_wm_client = ApiClient()
server = Server("FineCode")

@server.list_tools()
async def list_tools() -> list[Tool]: ...

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]: ...

def start(workdir, port_file=None) -> None:
    # start/connect WM server (same logic as current mcp_server.py start())
    # then run:
    async def _run():
        await _wm_client.connect("127.0.0.1", port)
        await _wm_client.add_dir(workdir)
        async with stdio_server() as (read_stream, write_stream):
            await server.run(read_stream, write_stream, server.create_initialization_options())
    asyncio.run(_run())
```

**`list_tools()` implementation:**

1. Call `await _wm_client.list_actions()` to get all actions (flat list with `name` and
   `project` fields per entry).
2. Group entries by `project` path to batch schema requests per project.
3. For each project, call `await _wm_client.get_payload_schemas(project, action_names)`.
4. Build one `Tool` per action:

```python
Tool(
    name=action["name"],
    description=f"Run {action['name']} on a project",
    inputSchema={
        "type": "object",
        "properties": {
            "project": {
                "type": "string",
                "description": "Absolute path to the project directory",
            },
            **(schema["properties"] if schema else {}),
        },
        "required": ["project"] + (schema.get("required", []) if schema else []),
    },
)
```

Also include a static `list_projects` tool with `inputSchema={"type":"object","properties":{}}`.

**`call_tool()` implementation:**

```python
if name == "list_projects":
    result = await _wm_client.list_projects()
    return [TextContent(type="text", text=json.dumps({"projects": result}))]

project = arguments.pop("project")   # remove routing key before passing as params
result = await _wm_client.run_action(
    name, project,
    params=arguments or None,
    options={"resultFormats": ["json", "string"], "trigger": "user", "devEnv": "ai"},
)
return [TextContent(type="text", text=json.dumps(result))]
```

**`pyproject.toml`**: swap `fastmcp` → `mcp`.

---

### Step 9 — Documentation

All external API changes must be documented before the step is considered complete.

**`docs/wm-protocol.md`** — add new entry under the `actions/` section:

```markdown
#### `actions/getPayloadSchemas`

- Type: request
- Clients: MCP
- Status: implemented

Params:
  { "project": "/abs/path/to/project", "action_names": ["lint", "format"] }

Result:
  {
    "schemas": {
      "lint":   { "properties": { "file_paths": {...}, "target": {...} }, "required": [] },
      "format": { "properties": { "save": {...}, "target": {...}, "file_paths": {...} }, "required": [] }
    }
  }

Schemas are null for actions whose class cannot be imported in the dev_workspace runner.
Schemas are cached per project in the WM and invalidated whenever runner config is updated.
```

**`docs/wm-er-protocol.md`** — add new entry under WM→ER commands:

```markdown
- `actions/getPayloadSchemas`
  - Arguments: none
  - Result: { action_name: JSON Schema fragment | null }
  - Returns a payload schema for every action currently known to the runner.
    Each schema has `properties` (field name → JSON Schema type object) and
    `required` (list of field names without defaults).
    null means the action class could not be imported.
```

**Code-level docstrings** (these are internal but must be present):

- `schema_utils.extract_payload_schema` — document the type mapping table and
  the `required` field rule (both `default` and `default_factory` are `MISSING`)
- `runner_client.get_payload_schemas` — note that it targets `dev_workspace` runner
- `ApiClient.get_payload_schemas` — document params, return type, and cache behaviour

---

## Files changed

| File | Change |
|---|---|
| `finecode_extension_runner/src/finecode_extension_runner/schema_utils.py` | **new** — payload schema extraction utility |
| `finecode_extension_runner/src/finecode_extension_runner/services.py` | new `get_payload_schemas()` function |
| `finecode_extension_runner/src/finecode_extension_runner/lsp_server.py` | register `actions/getPayloadSchemas` command |
| `src/finecode/wm_server/context.py` | add `ws_action_schemas` cache field |
| `src/finecode/wm_server/runner/runner_client.py` | new `get_payload_schemas()` ER client call |
| `src/finecode/wm_server/runner/runner_manager.py` | invalidate schema cache in `update_runner_config` |
| `src/finecode/wm_server/wm_server.py` | new `_handle_get_payload_schemas` endpoint |
| `src/finecode/wm_client.py` | new `get_payload_schemas()` WM client method |
| `src/finecode/mcp_server.py` | rewrite using `mcp` SDK |
| `pyproject.toml` | swap `fastmcp` → `mcp` |
| `docs/wm-protocol.md` | document `actions/getPayloadSchemas` WM endpoint |
| `docs/wm-er-protocol.md` | document `actions/getPayloadSchemas` ER command |
