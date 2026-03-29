# E2E Tests: Full-Stack Lifecycle & Ghost Process Prevention

## Problem

In real-world usage, closing the IDE often leaves ghost processes (at least WM server, often Extension Runners too). The existing e2e tests cover individual component lifecycles but miss the **full-stack scenario** where LSP → WM → ER are all running together.

## Root Cause

`wm_lifecycle.ensure_running()` spawns WM as a **detached subprocess** (`Popen` with no process group sharing). If anything breaks in the graceful shutdown chain, WM and its ERs become ghosts.

## Gap Analysis

| Scenario | Covered? |
|----------|----------|
| LSP alone starts/stops on SIGINT | Yes |
| LSP exits on client disconnect (no `initialized`) | Yes |
| MCP + dedicated WM: SIGINT kills both | Yes |
| MCP + dedicated WM: SIGKILL kills both | Yes |
| WM alone: SIGINT + port file cleanup | Yes |
| WM alone: auto-stop after disconnect | Yes |
| WM → ER cleanup on shutdown | Yes |
| **LSP sends `initialized` → WM spawned via `ensure_running` → client disconnects → WM + ERs die** | **No** |
| **LSP sends `initialized` → WM spawned → LSP SIGKILL → WM + ERs eventually die** | **No** |
| **LSP sends `initialized` → WM spawned → LSP exits normally → WM auto-stop timeout fires → ERs cleaned** | **No** |
| **Two LSP clients share one WM → first disconnects → WM stays → second disconnects → WM auto-stops** | **No** |

## Proposed Tests

### Test 1: `test_full_stack_lsp_client_disconnect_cleans_up_all_processes`

Simulates IDE open → IDE close (connection dropped without LSP shutdown):

1. Start LSP server (TCP)
2. Connect as an LSP client, send `initialize` + `initialized` (triggers WM spawn)
3. Wait for WM to appear (poll discovery file or psutil)
4. Wait for ER child of WM to appear (psutil)
5. Record all PIDs: LSP, WM, ER(s)
6. Drop the TCP connection (simulate IDE crash / close without shutdown)
7. Assert LSP process exits (pygls EOF detection)
8. Assert WM process exits within disconnect_timeout + buffer
9. Assert ALL ER PIDs are dead
10. Assert discovery port file is removed

### Test 2: `test_full_stack_lsp_sigkill_wm_auto_stops`

IDE is force-killed (OOM, kill -9). **Highest priority** — most likely cause of ghost processes.

1. Start LSP server, connect, send `initialized` (WM spawned)
2. Wait for WM + ERs
3. Record WM + ER PIDs
4. SIGKILL the LSP process (simulates OOM kill)
5. WM detects client disconnect → auto-stop timer fires
6. Assert WM exits within disconnect_timeout + buffer
7. Assert ERs are dead

Key: since `ensure_running()` spawns WM detached, SIGKILL on LSP does NOT kill WM. WM must self-terminate via auto-stop.

### Test 3: `test_full_stack_lsp_clean_shutdown_sequence`

IDE sends proper LSP shutdown/exit:

1. Start LSP, connect, send `initialize` + `initialized`
2. Wait for WM + ERs
3. Send LSP `shutdown` request + `exit` notification (clean protocol)
4. Assert LSP exits cleanly
5. Assert WM auto-stops after disconnect timeout
6. Assert ERs are dead

### Test 4: `test_shared_wm_survives_first_client_disconnect`

Multiple clients sharing one WM:

1. Start WM with short disconnect timeout (3s)
2. Connect client A, call `workspace/addDir` (ERs start)
3. Connect client B
4. Disconnect client A
5. Sleep 5s (longer than disconnect timeout)
6. Assert WM is still alive (client B keeps it alive)
7. Disconnect client B
8. Assert WM auto-stops within timeout + buffer
9. Assert ERs are dead

## Infrastructure Needs

### Minimal LSP client helpers (add to `tests/e2e/conftest.py`)

```python
def send_lsp_initialize(sock, workspace_dir, req_id=1):
    """Send LSP initialize request with workspace folders."""
    ...

def send_lsp_initialized(sock):
    """Send LSP initialized notification (triggers WM connection)."""
    ...

def read_lsp_response(sock, timeout=30):
    """Read Content-Length framed LSP response."""
    ...

def wait_for_child_process(parent_pid, cmdline_match, timeout=30):
    """Poll psutil until a child process matching cmdline appears."""
    ...
```

### WM disconnect timeout control

The LSP's `_on_initialized` calls `ensure_running()` which starts WM with default args (30s timeout). For fast tests, need a way to pass `--disconnect-timeout 2` through. Options:
- Environment variable `FINECODE_WM_DISCONNECT_TIMEOUT` read by `ensure_running()`
- Parameter on `ensure_running()` forwarded to the subprocess

### Fixture: `workspace_dir_with_er`

Already exists in `tests/e2e/er/test_lifecycle.py`. Should be moved to shared `conftest.py` for reuse by full-stack tests.

## Implementation Priority

1. **Test 2** (SIGKILL) — most likely cause of ghost processes
2. **Test 1** (client disconnect) — normal "close IDE" path
3. **Test 4** (shared WM) — multi-client lifecycle
4. **Test 3** (clean shutdown) — least likely to have bugs

## File location

New tests go in `tests/e2e/full_stack/test_lifecycle.py`.
