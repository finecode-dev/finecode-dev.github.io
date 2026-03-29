# Extension Runner Memory Leak Investigation

Diagnosed during analysis of `finecode_httpclient` `dev_no_runtime` ER process growing from
~1.4 GB to 2.2 GB+ with no errors logged. Lint runs trigger every 2-3 seconds via IDE SYSTEM
trigger. Root cause not yet pinpointed — tracemalloc profiling added to confirm which allocation
site dominates.

---

## Issues (ordered by suspected impact)

### P1 — MultiQueueIterator creates unawaited asyncio tasks on every iteration

**File:** `finecode_extension_runner/src/finecode_extension_runner/impls/file_editor.py`

On every `__anext__` call, N+2 new `asyncio.Task` objects are created (`queue.get()` per queue,
`queues_changed_event.wait()`, `shutdown_event.wait()`). When `queues_changed_event` fires (a
file is opened or closed while iterating), the loop hits `continue` and creates another batch of
tasks — while the previous batch is cancelled but **never awaited**. Cancelled-but-unawaited
tasks accumulate in the event loop indefinitely.

`subscribe_to_changes_of_opened_files` is called from the long-running
`send_changed_files_to_lsp_client` task and from `LspService._run_event_loop`, both of which run
for the entire lifetime of the ER process.

**Fix:**

```python
for task in pending:
    task.cancel()
    try:
        await task
    except (asyncio.CancelledError, Exception):
        pass
```

---

### P2 — `_finecode_async_tasks` list is never pruned

**File:** `finecode_extension_runner/src/finecode_extension_runner/lsp_server.py`, line 479

Every call to `update_config` appends a new task to `ls._finecode_async_tasks`. Completed tasks
are never removed, so the list (and the Task objects it holds) grows forever with each config
reload.

**Fix:** Prune completed tasks before appending:

```python
ls._finecode_async_tasks = [t for t in ls._finecode_async_tasks if not t.done()]
ls._finecode_async_tasks.append(send_changed_files_task)
```

---

### P3 — `LspService.check_file` race condition on `_diagnostics[uri]`

**File:** `finecode_extension_api/src/finecode_extension_api/contrib/lsp_service.py`, line ~190

Concurrent calls for the same URI overwrite `self._diagnostics[uri]` with a new
`threading.Event`. The old event is never set, so the thread that created it blocks for the full
30-second timeout. Currently ruled out as the primary memory cause (thread count stable at 6),
but still a correctness bug that causes latency spikes.

**Fix:** Reuse the existing event if a waiter is already pending, or use a dict of lists and
notify all waiters.

---

### P4 — `InMemoryCache` stale entries on file version change

**File:** `finecode_extension_runner/src/finecode_extension_runner/impls/inmemory_cache.py`

`cache_by_file` stores lint results keyed by `(file_path, key)`. When a file changes, the old
entry with the previous version key is never evicted; a new entry is added alongside it. The
cache grows proportionally to the number of unique `(file, version)` pairs seen over the lifetime
of the process. There is already a `TODO: clear file cache when file changes` comment in the
code.

**Fix:** On file change, delete all cache entries for that `file_path` before inserting the new
result.

---

### P5 — `PartialResultSender` cancels `_wait_and_send` task without awaiting it

**File:** `finecode_extension_runner/src/finecode_extension_runner/partial_result_sender.py`, line ~35

`self.scheduled_task.cancel()` is called and then `self.scheduled_task = None` is set immediately,
without `await`ing the cancelled task. The cancelled task remains alive in the event loop until
the next iteration of the event loop, which can cause ordering issues and leaves dangling task
references.

**Fix:**

```python
self.scheduled_task.cancel()
try:
    await self.scheduled_task
except asyncio.CancelledError:
    pass
self.scheduled_task = None
```

---

### P6 — High lint trigger frequency amplifies all per-run accumulation

IDE SYSTEM trigger fires lint every 2-3 seconds. Any per-run object that is not promptly
collected (tasks, closures, pydantic model instances created in `run_action_raw`) gets multiplied
by hundreds of runs per minute.

Consider debouncing SYSTEM-triggered lint runs (e.g. coalesce triggers within a 5-10 second
window) and/or caching pydantic model classes created by `pydantic_dataclass()` in
`run_action.py` and `merge_results.py` instead of recreating them on every invocation.

---

## Profiling status

`tracemalloc` instrumentation added to
`finecode_extension_runner/src/finecode_extension_runner/lsp_server.py`:

- `tracemalloc.start(10)` at module import time
- `_log_memory_stats()` coroutine logs top-15 allocation sites every 120 s, plus
  `gc.garbage` count and total asyncio task count
- Started via `asyncio.create_task(_log_memory_stats())` inside `update_config()`

**Next step:** restart the `dev_no_runtime` ER, wait ~2 minutes, and collect the
`=== TRACEMALLOC TOP 15 ===` block from the runner log to confirm which allocation site
dominates.
