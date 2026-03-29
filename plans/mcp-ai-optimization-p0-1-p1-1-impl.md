# Implementation Plan: MCP AI Optimization P0-1 (Action Descriptions) & P1-1 (Field Descriptions)

**Date:** 2026-03-18
**Status:** Draft
**Scope:** 2 features across ~35 files, 6 layers
**Estimated complexity:** MEDIUM

## Context

The FineCode MCP server currently uses a generic string `"Run {name} on a project or the whole workspace"` as the `Tool.description` for every action. Payload field schemas have no `"description"` key. This makes it difficult for AI assistants to understand what each tool does and what parameters mean.

This plan covers two features that propagate human-readable descriptions from the source-of-truth (action class docstrings and field attribute docstrings) all the way through ER -> WM -> MCP to the AI assistant.

## Work Objectives

1. **P0-1 — Action descriptions:** Each MCP `Tool` shows a meaningful description derived from the `Action` subclass docstring.
2. **P1-1 — Field descriptions:** Each JSON Schema property in the `Tool.inputSchema` includes a `"description"` extracted from attribute docstrings on `RunActionPayload` dataclass fields.

## Guardrails

### Must Have
- Backwards-compatible: existing clients that ignore `description` fields are unaffected.
- Silent fallback: if `__doc__` is `None` or `inspect.getsource()` fails, descriptions default to `""` — no errors.
- The `project` synthetic field in `mcp_server.py` keeps its existing hardcoded description.

### Must NOT Have
- No changes to the WM/ER protocol message format beyond adding optional fields.
- No new dependencies (AST parsing uses stdlib `ast` and `inspect`).
- No changes to how actions are registered, configured, or executed.

---

## Task Flow

```
Step 1: Action definitions (finecode_extension_api)
   |
Step 2: schema_utils.py (finecode_extension_runner) — AST field description extractor
   |
Step 3: ER services.py (finecode_extension_runner) — include action description in schema response
   |
Step 4: WM server + domain (finecode) — propagate description through actions/list
   |
Step 5: MCP server (finecode) — consume descriptions for Tool objects
   |
Step 6: Verification
```

---

## Step 1: Add class docstrings to Action subclasses and attribute docstrings to RunActionPayload fields

**Package:** `finecode_extension_api`
**Why:** These are the source-of-truth for all descriptions. Everything downstream reads from here.

### 1a. Add class docstrings to all Action subclasses (P0-1)

Add a one-line class docstring to each `Action` subclass. The docstring should describe what the action does from the perspective of an AI assistant invoking it as a tool.

**Files (30 Action subclasses):**

User-facing actions (prioritize clear, useful descriptions):
- `actions/lint.py` — `LintAction`: e.g. `"""Run linters on a project or specific files and report diagnostics."""`
- `actions/format.py` — `FormatAction`: e.g. `"""Auto-format source code in a project or specific files."""`
- `actions/run_tests.py` — `RunTestsAction`: e.g. `"""Execute tests and return structured pass/fail results."""`
- `actions/list_tests.py` — `ListTestsAction`: e.g. `"""Discover tests and return their hierarchical structure without running them."""`
- `actions/lint_files.py` — `LintFilesAction`
- `actions/format_files.py` — `FormatFilesAction`
- `actions/check_formatting.py` — (file is currently empty except for comment; skip if no class exists)
- `actions/build_artifact_action.py` — `BuildArtifactAction`
- `actions/dump_config.py` — `DumpConfigAction`
- `actions/clean_finecode_logs.py` — `CleanFinecodeLogsAction`

Infrastructure/CI actions:
- `actions/publish_artifact.py` — `PublishArtifactAction`
- `actions/publish_artifact_to_registry.py` — `PublishArtifactToRegistryAction`
- `actions/verify_artifact_published_to_registry.py` — `VerifyArtifactPublishedToRegistryAction`
- `actions/is_artifact_published_to_registry.py` — `IsArtifactPublishedToRegistryAction`
- `actions/get_src_artifact_version.py` — `GetSrcArtifactVersionAction`
- `actions/get_dist_artifact_version.py` — `GetDistArtifactVersionAction`
- `actions/get_src_artifact_registries.py` — `GetSrcArtifactRegistriesAction`
- `actions/get_src_artifact_language.py` — `GetSrcArtifactLanguageAction`
- `actions/list_src_artifact_files_by_lang.py` — `ListSrcArtifactFilesByLangAction`
- `actions/group_src_artifact_files_by_lang.py` — `GroupSrcArtifactFilesByLangAction`
- `actions/init_repository_provider.py` — `InitRepositoryProviderAction`
- `actions/install_deps_in_env.py` — `InstallDepsInEnvAction`
- `actions/prepare_runner_envs.py` — `PrepareRunnerEnvsAction`
- `actions/prepare_runner_env.py` — `PrepareRunnerEnvAction`
- `actions/prepare_handler_envs.py` — `PrepareHandlerEnvsAction`
- `actions/prepare_handler_env.py` — `PrepareHandlerEnvAction`
- `actions/create_envs.py` — `CreateEnvsAction`
- `actions/create_env.py` — `CreateEnvAction`
- `actions/lock_dependencies.py` — `LockDependenciesAction`
- `actions/lock_python_dependencies.py` — `LockPythonDependenciesAction`

**Pattern:**
```python
class LintAction(code_action.Action[LintRunPayload, LintRunContext, LintRunResult]):
    """Run linters on a project or specific files and report diagnostics."""

    PAYLOAD_TYPE = LintRunPayload
    ...
```

**Note on `run_tests.py` and `list_tests.py`:** These files already have **module** docstrings (design rationale). Those must be preserved. The new **class** docstring goes on the `Action` subclass, not the module.

### 1b. Add attribute docstrings to RunActionPayload fields (P1-1)

Convert existing inline comments on dataclass fields to **attribute docstrings** (a bare string literal on the line after the field assignment). This is the pattern `ast.parse()` can extract.

**Priority payload classes** (the ones most useful to AI assistants):

`LintRunPayload` (2 fields):
```python
@dataclasses.dataclass
class LintRunPayload(code_action.RunActionPayload):
    target: LintTarget = LintTarget.PROJECT
    """Whether to lint the whole project or specific files. One of 'project' or 'files'."""
    file_paths: list[Path] = dataclasses.field(default_factory=list)
    """Files to lint. Only used when target is 'files'. Empty list means lint the whole project."""
```

`FormatRunPayload` (3 fields):
```python
    save: bool = True
    """Whether to save formatted files to disk. Set to False for dry-run."""
    target: FormatTarget = FormatTarget.PROJECT
    """Whether to format the whole project or specific files."""
    file_paths: list[Path] = dataclasses.field(default_factory=list)
    """Files to format. Only used when target is 'files'."""
```

`RunTestsRunPayload` (3 fields):
```python
    file_paths: list[Path] = dataclasses.field(default_factory=list)
    """Test files or directories to run. Empty list means use the handler's configured defaults."""
    test_ids: list[str] = dataclasses.field(default_factory=list)
    """Runner-native test identifiers to restrict execution to. Format is handler-specific (e.g. pytest node IDs)."""
    markers: list[str] = dataclasses.field(default_factory=list)
    """Marker/tag names to filter the test suite (e.g. 'unit', 'integration', 'slow')."""
```

`ListTestsRunPayload` (1 field):
```python
    file_paths: list[Path] = dataclasses.field(default_factory=list)
    """Files or directories to search for tests. Empty list means use handler defaults."""
```

Apply the same pattern to all other `RunActionPayload` subclasses that have fields with inline comments. For payload classes with no fields (e.g. `CleanFinecodeLogsRunPayload: ...`), no change is needed.

**Acceptance criteria:**
- Every `Action` subclass has a class docstring (accessible via `ActionClass.__doc__`).
- Every `RunActionPayload` field that had an inline comment now has an attribute docstring instead (or in addition).
- Existing module docstrings on `run_tests.py` and `list_tests.py` are preserved.
- All existing tests still pass.

---

## Step 2: Extend `schema_utils.py` to extract field descriptions via AST

**File:** `finecode_extension_runner/src/finecode_extension_runner/schema_utils.py`
**Why:** The schema extractor needs to parse attribute docstrings from source code and inject them as `"description"` in each JSON Schema property.

### Implementation

Add a helper function `_extract_field_descriptions(cls) -> dict[str, str]` that:

1. Calls `inspect.getsource(cls)` to get the class source.
2. Calls `ast.parse(source)` and finds the `ast.ClassDef` node.
3. Iterates over the class body statements pairwise: for each `ast.AnnAssign` (annotated assignment), checks if the **next** statement is `ast.Expr(value=ast.Constant(value=str))`.
4. If so, maps the field name (from `ast.AnnAssign.target.id`) to the string constant value.
5. Wraps the entire function body in `try/except Exception: return {}` for silent fallback.

Modify `extract_payload_schema()` to:

1. Call `_extract_field_descriptions(payload_cls)`.
2. For each field, if a description exists, add `"description": desc` to the property dict.

**New imports needed:** `ast`, `inspect`, `textwrap`

**Pseudocode for `_extract_field_descriptions`:**
```python
def _extract_field_descriptions(cls: type) -> dict[str, str]:
    try:
        source = inspect.getsource(cls)
        source = textwrap.dedent(source)
        tree = ast.parse(source)
    except Exception:
        return {}

    descriptions: dict[str, str] = {}
    # Find the ClassDef node
    class_def = next(
        (node for node in ast.walk(tree) if isinstance(node, ast.ClassDef)),
        None,
    )
    if class_def is None:
        return {}

    body = class_def.body
    for i, stmt in enumerate(body):
        if not isinstance(stmt, ast.AnnAssign):
            continue
        if not isinstance(stmt.target, ast.Name):
            continue
        field_name = stmt.target.id
        # Check if the next statement is a docstring
        if i + 1 < len(body):
            next_stmt = body[i + 1]
            if (
                isinstance(next_stmt, ast.Expr)
                and isinstance(next_stmt.value, ast.Constant)
                and isinstance(next_stmt.value.value, str)
            ):
                descriptions[field_name] = next_stmt.value.value.strip()
    return descriptions
```

**Modified `extract_payload_schema`:**
```python
def extract_payload_schema(payload_cls: type) -> dict:
    # ... existing code ...
    field_descriptions = _extract_field_descriptions(payload_cls)

    for field in dataclasses.fields(payload_cls):
        schema = _type_to_schema(hints.get(field.name, type(None)))
        desc = field_descriptions.get(field.name)
        if desc:
            schema["description"] = desc
        properties[field.name] = schema
        # ... existing required logic ...
```

**Acceptance criteria:**
- `extract_payload_schema(LintRunPayload)` returns properties where `file_paths` has a `"description"` key.
- If `inspect.getsource()` raises (e.g. built-in class, frozen module), the function returns `{}` and `extract_payload_schema` still works (just without descriptions).
- Descriptions from parent classes are **not** extracted (only the immediate class body). This matches `dataclasses.fields()` which returns inherited fields but the docstrings should be on the subclass where the field is defined.

---

## Step 3: Include action description in the ER's schema response

**File:** `finecode_extension_runner/src/finecode_extension_runner/services.py`
**Function:** `get_payload_schemas()`
**Why:** The ER already returns `{action_name: schema_dict}`. We need to also return the action class docstring so the WM can propagate it.

### Implementation

Modify the `get_payload_schemas()` function. After importing the action class and extracting the payload schema, also read `action_cls.__doc__`:

```python
def get_payload_schemas() -> dict[str, dict | None]:
    # ... existing code ...
    for action_name, action in global_state.runner_context.project.actions.items():
        try:
            action_cls = run_utils.import_module_member_by_source_str(action.source)
            payload_cls = getattr(action_cls, "PAYLOAD_TYPE", None)
            if payload_cls is None:
                result[action_name] = None
            else:
                schema = schema_utils.extract_payload_schema(payload_cls)
                # Attach action-level description from class docstring
                doc = getattr(action_cls, "__doc__", None)
                if doc:
                    schema["description"] = doc.strip()
                result[action_name] = schema
        except Exception as exception:
            logger.debug(...)
            result[action_name] = None
    return result
```

The response shape changes from:
```json
{"action_name": {"properties": {...}, "required": [...]}}
```
to:
```json
{"action_name": {"properties": {...}, "required": [...], "description": "Run linters on ..."}}
```

This is additive and backwards-compatible — existing consumers that do not read `"description"` are unaffected.

**Acceptance criteria:**
- The ER's `actions/getPayloadSchemas` response includes a `"description"` key in each action's schema dict (when the action class has a docstring).
- The `"description"` is the stripped class docstring, not the module docstring.
- Actions without a class docstring have no `"description"` key (or it is absent).

---

## Step 4: Propagate action description through the WM layer

**Files:**
- `src/finecode/wm_server/wm_server.py` — `_handle_list_actions()` and `_handle_get_payload_schemas()`
- `src/finecode/wm_client.py` — no changes needed (already returns raw dicts)

### 4a. `_handle_list_actions` — add `description` field (optional approach)

**Decision:** The `actions/list` response currently does NOT include a description. There are two options:

**Option A (Simpler, chosen):** Do NOT change `actions/list`. The MCP server already fetches schemas via `get_payload_schemas`, which will now include the `"description"` key. The MCP server reads the description from the schema response.

**Option B (Alternative):** Add `description` to the `actions/list` response. This would require either storing it on the WM domain `Action` object or fetching it from the ER at list time. More invasive, no clear benefit since the MCP server already has the schema.

**With Option A, no changes are needed in `_handle_list_actions` or `_handle_get_payload_schemas`.** The WM already transparently proxies the schema dict from the ER, so the new `"description"` key flows through automatically via the existing cache mechanism in `_handle_get_payload_schemas`.

### 4b. Verify transparent passthrough

The existing `_handle_get_payload_schemas` code does:
```python
cache.update(schemas)  # stores whatever the ER returns
# ...
return {"schemas": {name: cache.get(name) for name in action_names}}
```

This already passes through any extra keys (like `"description"`) without modification. No code change needed.

**Acceptance criteria:**
- The WM's `actions/getPayloadSchemas` response includes `"description"` in each schema dict when the ER provides it.
- The WM's schema cache stores and returns the description.
- No changes to the `actions/list` response format.

---

## Step 5: Consume descriptions in the MCP server

**File:** `src/finecode/mcp_server.py`
**Function:** `list_tools()`
**Why:** This is where the generic description string must be replaced with the real action description.

### Implementation

Modify the tool-building loop. The `schema` dict returned from `get_payload_schemas` now contains an optional `"description"` key at the top level (alongside `"properties"` and `"required"`).

```python
for action in project_actions:
    name = action["name"]
    schema: dict | None = schemas.get(name)

    # Extract action-level description from schema (P0-1)
    action_description = (
        schema.pop("description", None) if schema else None
    ) or f"Run {name} on a project or the whole workspace"

    input_schema: dict = {
        "type": "object",
        "properties": {
            "project": {
                "type": "string",
                "description": "Absolute path to the project directory. Omit to run on all projects in the workspace.",
            },
            **(schema["properties"] if schema else {}),
        },
        "required": schema.get("required", []) if schema else [],
    }
    tools.append(
        Tool(
            name=name,
            description=action_description,  # was: f"Run {name} on ..."
            inputSchema=input_schema,
        )
    )
```

**Key detail:** Use `schema.pop("description")` rather than `schema.get("description")` to avoid leaking the `"description"` key into `inputSchema` (it is not a valid JSON Schema keyword at the top level of `inputSchema` in MCP context — it belongs on the `Tool` object). Alternatively, simply do not spread the top-level schema keys and only use `schema["properties"]` and `schema["required"]` (which is already the case).

Since the current code only reads `schema["properties"]` and `schema.get("required")`, the `"description"` key in the schema dict is naturally excluded from `input_schema`. So `.pop()` is not strictly necessary, but is cleaner. Either approach works.

**Field descriptions (P1-1):** No additional code needed in `mcp_server.py`. The `"description"` keys on individual properties inside `schema["properties"]` are already spread into `input_schema["properties"]` via `**(schema["properties"] if schema else {})`. The MCP SDK and AI assistants will read them automatically from the JSON Schema property objects.

**Acceptance criteria:**
- `Tool.description` shows the action's class docstring (e.g. "Run linters on a project or specific files and report diagnostics.") instead of the generic string.
- `Tool.inputSchema.properties.file_paths.description` shows the field's attribute docstring.
- The `project` synthetic field retains its existing hardcoded description.
- If an action has no docstring, the fallback generic string is used.

---

## Step 6: Verification

### 6a. Manual smoke test

1. Start the MCP server: `finecode start-mcp --workdir .`
2. Use an MCP inspector or `claude` CLI to list tools.
3. Verify that each tool has a meaningful description (not the generic one).
4. Verify that payload fields (e.g. `file_paths`, `target`, `markers`) have `"description"` in the schema.

### 6b. Unit test for `_extract_field_descriptions`

Add a test in the extension runner test suite that:
- Defines a test dataclass with attribute docstrings.
- Calls `_extract_field_descriptions()` and asserts the mapping is correct.
- Tests edge cases: no docstring, empty docstring, multiline docstring, field with no annotation.

### 6c. Unit test for `extract_payload_schema` with descriptions

- Calls `extract_payload_schema(LintRunPayload)` (or a test payload class).
- Asserts that `properties["file_paths"]["description"]` is a non-empty string.
- Asserts that `properties["target"]["description"]` is a non-empty string.

### 6d. Integration test (optional, if E2E test infrastructure exists)

- Start a WM server, connect an API client.
- Call `get_payload_schemas()` and verify `"description"` keys are present.

**Acceptance criteria:**
- All existing tests pass.
- New unit tests for the AST extractor pass.
- Manual smoke test confirms descriptions appear in MCP tool listing.

---

## File Change Summary

| Layer | File | Change Type | P0-1 | P1-1 |
|-------|------|-------------|------|------|
| Action API | `finecode_extension_api/.../actions/*.py` (30 files) | Add class docstrings | Yes | -- |
| Action API | `finecode_extension_api/.../actions/*.py` (~15 payload classes with fields) | Add attribute docstrings | -- | Yes |
| ER Schema | `finecode_extension_runner/.../schema_utils.py` | Add `_extract_field_descriptions()`, modify `extract_payload_schema()` | -- | Yes |
| ER Services | `finecode_extension_runner/.../services.py` | Read `action_cls.__doc__`, add to schema | Yes | -- |
| WM Server | `src/finecode/wm_server/wm_server.py` | No changes (transparent passthrough) | -- | -- |
| WM Client | `src/finecode/wm_client.py` | No changes | -- | -- |
| MCP Server | `src/finecode/mcp_server.py` | Use schema `description` for `Tool.description` | Yes | -- |
| Tests | New test file for schema_utils | Unit tests | -- | Yes |

## Success Criteria

1. Every MCP `Tool` for a FineCode action displays a human-readable description derived from the `Action` class docstring.
2. Every MCP `Tool.inputSchema` property that has an attribute docstring displays it as a `"description"` in the JSON Schema.
3. The fallback for missing docstrings is the existing generic string (action level) or no description (field level) -- never an error.
4. No breaking changes to any existing API, protocol, or behavior.
5. Unit tests cover the AST field description extractor including edge cases.

## Open Questions

- **Inherited fields:** `dataclasses.fields()` returns inherited fields. The AST extractor only reads the immediate class body. For subclasses that inherit fields without redefining them (e.g. `LockPythonDependenciesRunPayload` extends `LockDependenciesRunPayload`), descriptions from the parent will not appear. This is acceptable for now -- the parent class can have its own attribute docstrings, and the extractor can be enhanced later to walk the MRO if needed.
- **Cache invalidation:** The WM caches schemas in `ws_action_schemas`. If action docstrings change, the cache will be stale until the WM server restarts. This is the existing behavior and is acceptable.
