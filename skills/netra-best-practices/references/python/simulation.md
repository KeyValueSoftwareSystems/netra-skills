---
name: netra-python-simulation
description: Run multi-turn conversation simulations with the Python Netra SDK. Covers BaseTask, run_simulation, SimulationHooks, and file handling.
---

# Python Simulation

Test AI agents with realistic, multi-turn conversations using the Python Netra SDK's simulation framework.

## Overview

Simulation runs a configurable set of multi-turn conversations against your agent. Each conversation:

1. Starts with a user message from a dataset item.
2. Your agent responds via a `BaseTask` implementation.
3. The backend evaluates the response and decides whether to continue or stop.
4. If continuing, a new user message is generated and the loop repeats.

## Initialization

```env
NETRA_API_KEY=
NETRA_OTLP_ENDPOINT=
```

```python
import os
from netra import Netra

Netra.init(
    app_name="my-ai-app",
    environment="production",
    headers=f"x-api-key={os.getenv('NETRA_API_KEY')}",
)
```

## Implementing a Task

Subclass `BaseTask` and implement `run()`. The method can be sync or async.

```python
from netra import Netra
from netra.simulation import BaseTask, TaskResult

class MyAgentTask(BaseTask):
    def run(self, message, session_id=None, files=None, setup_context=None):
        Netra.set_root_input(message)
        response = my_agent.chat(message, session_id=session_id)
        Netra.set_root_output(response.text)
        return TaskResult(
            message=response.text,
            session_id=response.session_id or session_id or "default",
        )
```

### Task signature

```python
def run(
    self,
    message: str,
    session_id: Optional[str] = None,
    files: Optional[list[ProcessedFile]] = None,
    setup_context: Optional[dict] = None,
) -> TaskResult | Awaitable[TaskResult]
```

| Parameter | Description |
|-----------|-------------|
| `message` | The user message for this turn. |
| `session_id` | Session identifier for conversation continuity (`None` on first turn). |
| `files` | Optional list of `ProcessedFile` (base64-encoded file attachments). |
| `setup_context` | Optional dict from `before_all` / `before_each` / `before` hooks. `None` when no hooks are configured. |

> `setup_context` is backwards compatible — existing tasks that don't declare this parameter continue to work without modification.

### TaskResult fields

| Field        | Type  | Description                            |
| ------------ | ----- | -------------------------------------- |
| `message`    | `str` | The agent's response message           |
| `session_id` | `str` | Session ID for conversation continuity |

### Async task example

```python
class MyAsyncTask(BaseTask):
    async def run(self, message, session_id=None, files=None, setup_context=None):
        Netra.set_root_input(message)
        response = await my_async_agent.chat(message, session_id=session_id)
        Netra.set_root_output(response.text)
        return TaskResult(
            message=response.text,
            session_id=response.session_id or session_id or "default",
        )
```

### Task with file handling

```python
from netra import Netra
from netra.simulation import BaseTask, TaskResult, ProcessedFile

class FileTask(BaseTask):
    def run(self, message, session_id=None, files=None, setup_context=None):
        Netra.set_root_input(message)
        if files:
            for f in files:
                print(f.file_name, f.content_type, len(f.data))
        response = my_agent.chat(message, session_id=session_id, files=files)
        Netra.set_root_output(response.text)
        return TaskResult(
            message=response.text,
            session_id=response.session_id or session_id or "default",
        )
```

`ProcessedFile` fields: `file_name`, `content_type`, `description` (optional), `data` (base64 string).

## Running a Simulation

```python
result = Netra.simulation.run_simulation(
    name="Agent Stress Test v1",
    dataset_id="your-dataset-id",
    task=MyAgentTask(),
    max_concurrency=5,
    max_turns=50,
)
```

| Parameter         | Type              | Required | Default | Description                                               |
| ----------------- | ----------------- | -------- | ------- | --------------------------------------------------------- |
| `name`            | `str`             | Yes      | —       | Display name for the simulation run                       |
| `dataset_id`      | `str`             | Yes      | —       | ID of the dataset on the Netra platform                   |
| `task`            | `BaseTask`        | Yes      | —       | Your task implementation                                  |
| `context`         | `Any`             | No       | `None`  | Additional context passed to the backend                  |
| `max_concurrency` | `int`             | No       | `5`     | Max parallel conversations                                |
| `max_turns`       | `int`             | No       | `50`    | Max turns per conversation                                |
| `hooks`           | `SimulationHooks` | No       | `None`  | Pre/post script hooks (see Hooks section below)           |

Datasets for simulation are created and managed via the Netra dashboard.

---

## Hooks (Pre/Post Scripts)

Multi-turn datasets often contain scenarios that share state — for example, Scenario 2 requires an employee created in Scenario 1. Rather than merging all related scenarios into one large item or relying on sequential ordering, you can use **hooks** to set up and tear down shared state around scenario execution.

Hooks are defined on the **SDK side** — the actual scripts never leave your environment. Lightweight metadata (function name and docstring) is sent to the backend so the Netra UI can display which hooks are configured on a test run.

### The six hook points

| Hook | When it runs | Failure effect |
|------|-------------|----------------|
| `before_all` | Once, before any scenario starts | Entire run is marked failed; no scenarios execute |
| `before_each` | Before every scenario | That scenario is marked `prescript_failed` (terminal; eval suppressed); others continue |
| `before` | Before specific scenarios only (keyed by `dataset_item_id`) | That scenario is marked `prescript_failed` (terminal; eval suppressed); others continue |
| `after` | After specific scenarios only (keyed by `dataset_item_id`) | Logged as warning; does not affect scenario status |
| `after_each` | After every scenario (after item-specific `after`) | Logged as warning; does not affect scenario status |
| `after_all` | Once, after all scenarios complete | Logged as warning; does not affect run status |

### Context passing

Hooks pass data to each other and to your `BaseTask` via a context dict:

```
before_all()                            → shared_context (dict)
before_each(shared_context)             → merged into setup_context
before[dataset_item_id](merged_context) → merged into setup_context (if registered)
BaseTask.run(..., setup_context=merged_context)
after[dataset_item_id](result, setup_context) (if registered)
after_each(result, setup_context)
after_all(results, shared_context)
```

`setup_context` is the merge of `before_all` + `before_each` + item `before`. `BaseTask.run`, item `after`, and `after_each` receive it so teardown can clean up per-scenario resources (tokens, accounts, etc.). If a before hook fails mid-way, teardown still receives the furthest successfully built `setup_context`. `after_all` still receives only the run-level `shared_context` from `before_all`, but its `results` include setup/first-turn failures as well as conversation failures.

### SimulationHooks

```python
from netra.simulation import SimulationHooks
```

```python
@dataclass
class SimulationHooks:
    before_all:  Optional[Callable] = None                # () -> dict | None
    before_each: Optional[Callable] = None                # (shared_context) -> dict | None
    before:      Optional[dict[str, Callable]] = None     # Dict[dataset_item_id, (shared_context) -> dict | None]
    after:       Optional[dict[str, Callable]] = None     # Dict[dataset_item_id, (result, setup_context) -> None]
    after_each:  Optional[Callable] = None                # (result, setup_context) -> None
    after_all:   Optional[Callable] = None                # (results, shared_context) -> None
```

**Important:**
- Prefer `before_each` / `after_each` when the same setup/teardown applies to every scenario
- `before` and `after` are **dicts** keyed by `dataset_item_id`
- Only items with registered hooks will have their specific setup/teardown functions called
- Item `before` receives the merged context from `before_all` + `before_each`
- `after` / `after_each` receive `result` and `setup_context` (merged `before_all` + `before_each` + item `before`)

All hooks can be sync or async.

### Example — Item-specific setup for particular scenarios

```python
from netra.simulation import BaseTask, SimulationHooks, TaskResult

# ---- Hooks ----

def setup_environment():
    """Create a test user before any scenario runs."""
    user = api.create_user(name="Test User", department="Engineering")
    return {"user_id": user.id}


def setup_each(shared_context):
    """Obtain a fresh auth token before every scenario."""
    token = api.login(user_id=shared_context["user_id"])
    return {"auth_token": token}


def setup_refund_scenario(shared_context):
    """Setup specific to refund scenarios - creates refund account."""
    # shared_context already includes user_id + auth_token from before_all + before_each
    refund_account = api.create_refund_account()
    return {"refund_account_id": refund_account.id}


def setup_premium_scenario(shared_context):
    """Setup specific to premium user scenarios."""
    api.upgrade_to_premium(shared_context["user_id"])
    return {"is_premium": True}


def teardown_refund_scenario(result, setup_context):
    """Cleanup refund-specific resources using item setup context."""
    api.delete_refund_account(setup_context.get("refund_account_id"))


def teardown_premium_scenario(result, setup_context):
    """Cleanup premium-specific resources using item setup context."""
    api.downgrade_from_premium(setup_context["user_id"])


def teardown_each(result, setup_context):
    """Log out after every scenario."""
    api.logout(token=setup_context.get("auth_token"))


def teardown_environment(results, shared_context):
    """Delete the test user once all scenarios are done."""
    api.delete_user(user_id=shared_context["user_id"])


# Note: dataset_item_id values come from your dataset items in the Netra dashboard
hooks = SimulationHooks(
    before_all=setup_environment,
    before_each=setup_each,
    before={
        "refund-scenario-001": setup_refund_scenario,
        "refund-scenario-002": setup_refund_scenario,
        "premium-user-001": setup_premium_scenario,
    },
    after={
        "refund-scenario-001": teardown_refund_scenario,
        "refund-scenario-002": teardown_refund_scenario,
        "premium-user-001": teardown_premium_scenario,
    },
    after_each=teardown_each,
    after_all=teardown_environment,
)


# ---- Task ----

class MyAgentTask(BaseTask):
    def run(self, message, session_id=None, files=None, setup_context=None):
        ctx = setup_context or {}
        response = my_agent.chat(
            message,
            session_id=session_id,
            auth_token=ctx.get("auth_token"),
            is_premium=ctx.get("is_premium", False),
        )
        return TaskResult(
            message=response.text,
            session_id=session_id or response.session_id or "default",
        )


# ---- Run ----

result = Netra.simulation.run_simulation(
    name="Multi-Scenario Workflow Test",
    dataset_id="your-dataset-id",
    task=MyAgentTask(),
    hooks=hooks,
    max_concurrency=3,
)
```

### Async hooks

```python
async def setup_environment():
    user = await async_api.create_user(name="Test User")
    return {"user_id": user.id}

async def setup_refund_scenario(shared_context):
    token = await async_api.login(shared_context["user_id"])
    return {"auth_token": token}

hooks = SimulationHooks(
    before_all=setup_environment,
    before={"refund-scenario-001": setup_refund_scenario},
)
```

### Hooks-only (no after hooks needed)

```python
hooks = SimulationHooks(
    before_all=setup_environment,
)
```

All hooks are optional. Omit any you don't need.

### Netra UI display

When `hooks` are passed, lightweight descriptors (function name and docstring) are sent to the backend as `lifecycleHooks` and the Netra dashboard shows which hook types are configured on the test run (e.g. "Has pre-script" badge). The actual script code is never stored by Netra.

---

## Validation Checklist

1. `Netra.init()` is called before accessing `Netra.simulation`.
2. `NETRA_API_KEY` and `NETRA_OTLP_ENDPOINT` are set.
3. Task's `run()` always returns a `TaskResult` with both `message` and `session_id`.
4. Task's `run()` calls `Netra.set_root_input(message)` at the start and `Netra.set_root_output(response)` before returning.
5. `Netra.shutdown()` is called on graceful termination.
6. When using hooks: `before_all` returns a `dict` or `None` (other types are ignored).
7. When using hooks: `before_each` receives `shared_context` and returns `dict` or `None`; runs for every item.
8. When using hooks: `before` dict values (functions) receive the merged context from `before_all` + `before_each` and return `dict` or `None`.
9. When using hooks: `after` / `after_each` receive `result` and `setup_context` (merged `before_all` + `before_each` + item `before`); return value is ignored. They also run when the scenario fails, exceeds max turns, or a before hook fails (with the furthest successfully built `setup_context`).
10. When using hooks: `after_all` should not raise — wrap risky cleanup in try/except. Its `results` include setup/first-turn failures as well as conversation failures.
11. Hook dict keys must match `dataset_item_id` values from your dataset.
12. A `prescript_failed` scenario is **terminal** — the SDK polling loop will not wait for it. Its `evalStatus` is set to `NOT_AVAILABLE` automatically and it cannot be overwritten by a timeout sweep. If all scenarios end as `failed` or `prescript_failed`, the run's evaluation status resolves to `NOT_AVAILABLE`.

## References

- https://docs.getnetra.ai/Simulation/Simulation-overview
- https://docs.getnetra.ai/sdk-reference/simulation/python
