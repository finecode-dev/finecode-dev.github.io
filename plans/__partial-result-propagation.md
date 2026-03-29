# Plan: Explicit partial result propagation through action chains

- **Date:** 2026-03-24
- **Status:** planned
- **Related ADR:** 0009-explicit-partial-result-token-propagation.md

## Problem

Any action can be invoked with a `partial_result_token` — whether partial results are
actually delivered depends on the action's implementation, not on its payload type. But
when `lint` (or `format`) is the entry point, partial results are **never sent** to the
client, even though the subaction `lint_files` has full machinery for them.

### Root cause

The framework only supports **framework-level streaming** (iterable payloads decomposed
into per-item coroutines). There is no **handler-level streaming** — no way for a handler
to produce partial results from its own `run()` method. Specifically:

1. **`partial_result_token` is lost at the `IActionRunner.run_action()` boundary.**
   `LintHandler.run()` delegates to `lint_files` via `action_runner.run_action()`, but
   `IActionRunner` has no mechanism to stream per-part results back to the caller. The
   inner `run_action()` call always gets `partial_result_token=None` →
   `send_partial_results=False`.

2. **`RunActionContext` doesn't carry the token or a send callback.** Even if a handler
   *wanted* to send partial results, it has no API to do so.

### Flow trace (current, broken)

```text
run_action_raw(partial_result_token=42)
  → run_action(lint, token=42)          # send_partial_results=True
    → LintRunPayload is NOT AsyncIterable → non-iterable branch
    → LintHandler.run(payload, run_context)
      → action_runner.run_action(lint_files, payload, meta)   # NO token param!
        → ActionRunner._run_action_func(action, payload, meta)  # 3 args only
          → run_action(lint_files, token=None)                # send_partial_results=False!
            → iterable branch, coroutines scheduled, but results accumulated, not sent
          → returns full LintFilesRunResult
      → wraps in LintRunResult, returns
    → handler returns full result to outer run_action
    → outer run_action returns full result (partial results never sent)
```

## Decision

Two handler-facing APIs, plus framework support for async generator handlers:

1. **`IActionRunner.run_action_iter()`** — returns `AsyncIterator[ResultT]` that yields
   per-part results as the subaction produces them.
2. **`run_context.partial_result_sender.send()`** — sends a mapped result to the client
   via a composed `PartialResultSender` on the context. No-op when no
   `partial_result_token`.
3. **Async generator `run()`** — when a handler's `run()` is an async generator (uses
   `yield`), the framework iterates it, sends each yielded value as a partial result,
   and accumulates the full result automatically.

**The handler never branches on `partial_result_token`.** The framework decides delivery
mode based on whether partial results were produced.

### Two handler patterns

**Primary: `yield` (async generator) — for sequential / `as_completed` cases:**

```python
class LintHandler(...):
    async def run(self, payload, run_context):
        async for partial in self.action_runner.run_action_iter(
            action=lint_files_action_instance,
            payload=LintFilesRunPayload(file_paths=file_uris),
            meta=run_meta,
        ):
            yield LintRunResult(messages=partial.messages)
```

The handler only maps and yields. The framework handles accumulation (`update()`),
partial result sending, and delivery mode. No manual accumulation, no
`partial_result_sender.send()` call.

**Escape hatch: `partial_result_sender.send()` — for `TaskGroup` concurrency:**

`yield` cannot propagate out of concurrent tasks. When a handler runs multiple subactions
in parallel via `TaskGroup`, it uses the composed `partial_result_sender` instead:

```python
class QualityCheckHandler(...):
    async def run(self, payload, run_context) -> QualityCheckResult:
        async def run_and_send(action, action_payload):
            async for partial in self.action_runner.run_action_iter(action, action_payload, meta):
                await run_context.partial_result_sender.send(map_result(partial))
                return partial

        async with asyncio.TaskGroup() as tg:
            lint_task = tg.create_task(run_and_send(lint_action, lint_payload))
            typecheck_task = tg.create_task(run_and_send(typecheck_action, tc_payload))

        return QualityCheckResult(
            lint=lint_task.result(), typecheck=typecheck_task.result()
        )
```

### Flow traces (proposed)

**With token (yield path):**

```text
run_action_raw(partial_result_token=42)
  → run_action(lint, token=42)
    → execute_action_handler(partial_result_token=42)
      → LintHandler.run() is async generator
      → framework iterates:
          partial = next from run_action_iter(lint_files)
          yielded = LintRunResult(messages=partial.messages)
          → partial_result_sender.schedule_sending(42, yielded)
          → accumulate into execution_result via update()
      → send_all_immediately()
      → execution_result = None (partials sent)
  → run_action_raw returns empty response
```

**Without token (yield path):**

```text
run_action_raw(partial_result_token=None)
  → run_action(lint, token=None)
    → execute_action_handler(partial_result_token=None)
      → LintHandler.run() is async generator
      → framework iterates:
          partial = next from run_action_iter(lint_files)
          yielded = LintRunResult(messages=partial.messages)
          → no token → skip sending
          → accumulate into execution_result via update()
      → execution_result = accumulated LintRunResult
  → run_action_raw returns full LintRunResult
```

## Changes required

### 1. `PartialResultSender` protocol + composition on `RunActionContext`

**File:** `finecode_extension_api/src/finecode_extension_api/code_action.py`

Define a handler-facing protocol and a no-op default, then compose onto the context:

```python
class PartialResultSender(typing.Protocol):
    """Handler-facing interface for sending partial results to the client."""

    async def send(self, result: RunActionResult) -> None: ...


class _NoOpPartialResultSender:
    async def send(self, result: RunActionResult) -> None:
        pass

_NOOP_SENDER = _NoOpPartialResultSender()


class RunActionContext(typing.Generic[RunPayloadType]):
    def __init__(
        self,
        run_id: int,
        initial_payload: RunPayloadType,
        meta: RunActionMeta,
        info_provider: RunContextInfoProvider,
        partial_result_sender: PartialResultSender = _NOOP_SENDER,
    ) -> None:
        ...
        self.partial_result_sender = partial_result_sender
```

The framework injects a tracking implementation when a `partial_result_token` is present,
and the no-op default otherwise. Handlers call `run_context.partial_result_sender.send()`
without checking for a token — the no-op makes it safe unconditionally.

**Framework-internal implementation** (in `finecode_extension_runner`):

```python
class _TrackingPartialResultSender:
    """Wraps partial_result_sender.schedule_sending with state tracking."""

    def __init__(
        self,
        token: int | str,
        send_func: collections.abc.Callable[
            [int | str, RunActionResult], collections.abc.Awaitable[None]
        ],
    ) -> None:
        self._token = token
        self._send_func = send_func
        self.has_sent = False

    async def send(self, result: RunActionResult) -> None:
        self.has_sent = True
        await self._send_func(self._token, result)
```

The framework keeps its own reference to `_TrackingPartialResultSender` so it can check
`has_sent` after the handler returns — the `has_sent` flag replaces the previous
`_partial_results_sent` on `RunActionContext`.

### 2. `RunActionWithPartialResultsContext` — forward `partial_result_sender` to super

**File:** `finecode_extension_api/src/finecode_extension_api/code_action.py`

Add `partial_result_sender` param, pass to `super().__init__()`.

### 3. `IActionRunner` — add `run_action_iter()`

**File:** `finecode_extension_api/src/finecode_extension_api/interfaces/iactionrunner.py`

```python
class IActionRunner(service.Service, typing.Protocol):
    ...

    def run_action_iter(
        self,
        action: ActionDeclaration[code_action.Action[PayloadT, typing.Any, ResultT]],
        payload: PayloadT,
        meta: code_action.RunActionMeta,
    ) -> collections.abc.AsyncIterator[ResultT]: ...
```

`run_action()` stays unchanged — always returns full result.

### 4. `ActionRunner` — implement `run_action_iter()`

**File:** `finecode_extension_runner/src/finecode_extension_runner/impls/action_runner.py`

```python
_SENTINEL = object()

class ActionRunner(iactionrunner.IActionRunner):
    ...

    async def run_action_iter(self, action, payload, meta):
        queue = asyncio.Queue()

        async def producer():
            try:
                await self._run_action_func(
                    action, payload, meta, partial_result_queue=queue
                )
            finally:
                await queue.put(_SENTINEL)

        task = asyncio.create_task(producer())
        try:
            while True:
                item = await queue.get()
                if item is _SENTINEL:
                    break
                yield item
        finally:
            if not task.done():
                task.cancel()
                try:
                    await task
                except (asyncio.CancelledError, Exception):
                    pass
```

### 5. `run_action()` service — queue support, token injection, async generator handling

**File:** `finecode_extension_runner/src/finecode_extension_runner/_services/run_action.py`

**5a.** Add `partial_result_queue: asyncio.Queue | None = None` parameter to
`run_action()`.

**5b.** Create the sender and inject it into `RunActionContext` via `known_args`
(around line 111):

```python
if partial_result_token is not None:
    tracking_sender = _TrackingPartialResultSender(
        token=partial_result_token,
        send_func=partial_result_sender.schedule_sending,
    )
else:
    tracking_sender = None

known_args={
    "run_id": lambda _: run_id,
    "initial_payload": lambda _: payload,
    "meta": lambda _: meta,
    "info_provider": lambda _: run_context_info,
    "partial_result_sender": lambda _: tracking_sender or code_action._NOOP_SENDER,
},
```

**5c.** Pass `partial_result_queue` to `run_subresult_coros_concurrently` /
`run_subresult_coros_sequentially`. When the queue is provided, put each per-part result
into the queue instead of accumulating or sending via `partial_result_sender`:

```python
# in run_subresult_coros_concurrently / run_subresult_coros_sequentially:
if partial_result_queue is not None:
    await partial_result_queue.put(action_subresult)
    return None
elif send_partial_results:
    await partial_result_sender.schedule_sending(partial_result_token, action_subresult)
    return None
else:
    return action_subresult
```

**5d.** For non-iterable payloads with `partial_result_queue`, put the final result into the
queue so `run_action_iter` yields it:

```python
# at the end of run_action(), before return:
if partial_result_queue is not None and action_result is not None:
    await partial_result_queue.put(action_result)
    return None
return action_result
```

**5e.** Pass `partial_result_token` to `execute_action_handler()` (new parameter).

**5f.** In `execute_action_handler`, handle async generator `run()` methods:

```python
call_result = handler_run_func(**args)
if inspect.isasyncgen(call_result):
    # handler is an async generator — iterate, accumulate, optionally send partials
    execution_result = None
    async for partial_result in call_result:
        if partial_result_token is not None:
            await partial_result_sender.schedule_sending(
                partial_result_token, partial_result
            )
        if execution_result is None:
            execution_result = deep_copy(partial_result)
        else:
            execution_result.update(partial_result)
    if partial_result_token is not None:
        await partial_result_sender.send_all_immediately()
        execution_result = None  # partials already sent
elif inspect.isawaitable(call_result):
    execution_result = await call_result
else:
    execution_result = call_result
```

**5g.** After non-generator handler returns in the non-iterable branch, check
`tracking_sender.has_sent` (for the `partial_result_sender.send()` escape hatch):

```python
if tracking_sender is not None and tracking_sender.has_sent:
    await partial_result_sender.send_all_immediately()
    action_result = None
else:
    action_result = handler_result
```

### 6. `LintHandler.run()` — async generator with yield

**File:** `finecode_builtin_handlers/src/finecode_builtin_handlers/lint.py`

```python
async def run(self, payload, run_context):
    ...
    lint_files_action_instance = self.action_runner.get_action_by_source(
        lint_files_action.LintFilesAction
    )
    async for partial in self.action_runner.run_action_iter(
        action=lint_files_action_instance,
        payload=lint_files_action.LintFilesRunPayload(file_paths=file_uris),
        meta=run_meta,
    ):
        yield lint_action.LintRunResult(messages=partial.messages)
```

No manual accumulation, no `partial_result_sender.send()` call, no branching.

### 7. `FormatHandler.run()` — same pattern

**File:** `finecode_builtin_handlers/src/finecode_builtin_handlers/format.py`

Same async generator pattern, adapted for format result types.

### 8. Documentation

- **ADR-0009**: Record dual API design with rationale.
- **designing-actions.md**: Add section on partial result propagation with examples of both
  patterns (yield and `partial_result_sender.send()`).

## What does NOT change

- `LintFilesDispatchHandler` — still uses `partial_result_scheduler` to schedule per-file
  coroutines. The dispatch level is unaware of how its results are consumed.
- `PartialResultScheduler` — no changes.
- `PartialResultSender` — no changes to its interface.
- `run_action_raw()` — still passes `partial_result_token` to `run_action()` as before.
- Handlers that don't delegate to sub-actions — unaffected.

## Design notes

### Two APIs: when to use which

| Pattern | `yield` | `partial_result_sender.send()` |
| --- | --- | --- |
| Sequential iteration of subaction | **Use this** | Works but verbose |
| `asyncio.as_completed` concurrency | **Use this** | Works but verbose |
| `TaskGroup` concurrency | Cannot yield from tasks | **Use this** |
| Multiple concurrent subaction iterators | Needs merge utility | **Use this** |

`yield` is the primary recommendation. `partial_result_sender.send()` exists for cases
where `yield` is structurally impossible (concurrent tasks).

### Why AsyncIterator instead of forwarding the token?

Simpler alternative: add `partial_result_token` param to `IActionRunner.run_action()` so
the handler just forwards the token and the subaction sends partial results directly.

**Problem**: the subaction's result type (`LintFilesRunResult`) is sent as-is. If the
parent action has a different result type (extra/fewer fields), partial results have the
wrong shape. The handler has no chance to map them.

With `run_action_iter`, the handler receives each partial result, maps it to the parent's
result type, and yields/sends the mapped version. No JSON-compatibility constraint between
subaction and parent result types.

### Why a composed `partial_result_sender` instead of a method?

`RunActionContext` is a lightweight data holder — its only behaviour is the `current_result`
property (delegated to `_info_provider`). Adding a `send_partial_result()` method would
break this pattern. Composing a `PartialResultSender` is consistent with the existing
`partial_result_scheduler` on `RunActionWithPartialResultsContext`.

The concrete `PartialResultSender` implementation lives in `finecode_extension_runner` —
the extension API package cannot depend on it. Instead, `run_action()` creates a
`_TrackingPartialResultSender` wrapping `partial_result_sender.schedule_sending` and
injects it into `RunActionContext` via the existing DI `known_args` mechanism. The handler
sees only the `PartialResultSender` protocol. A no-op default (`_NOOP_SENDER`) makes the
call safe when no token is present — handlers never need to check.

### Queue-based bridging

`run_action_iter()` in `ActionRunner` creates an `asyncio.Queue` and runs `run_action()`
as a concurrent producer task. Each per-part result from the iterable-payload path goes
into the queue. The caller `async for`s over the queue.

For non-iterable subaction payloads, one result is put into the queue → one iteration
(step 5d).

### Async generator detection

The framework already checks `inspect.isawaitable()` in `execute_action_handler`. Adding
`inspect.isasyncgen()` is a natural extension. The `ActionHandler` protocol's `run()`
return type stays as `RunActionResult` — async generator handlers are supported at runtime
but not enforced by the type system. This is a pragmatic choice; a separate
`StreamingActionHandler` protocol can be added later if type strictness is needed.

### Flushing

`PartialResultSender` batches sends with a 300ms debounce (`schedule_sending`). Flushing
via `send_all_immediately()` happens:

- After iterating an async generator handler (step 5f)
- After a non-generator handler returns with `tracking_sender.has_sent=True` (step 5g)

Both are safe to call unconditionally — no-op when nothing is queued.

## Verification

After implementation:

1. Calling `lint` with `partial_result_token` → partial results arrive as `LintRunResult`
   (not `LintFilesRunResult`) as each file completes.
2. Calling `lint` without `partial_result_token` → full `LintRunResult` returned as before.
3. If `LintRunResult` is later extended with fields not in `LintFilesRunResult`, the
   handler's mapping logic populates them — partial results include the extra fields.
4. A hypothetical handler using `TaskGroup` + `partial_result_sender.send()` sends partial results
   as each concurrent task completes.
