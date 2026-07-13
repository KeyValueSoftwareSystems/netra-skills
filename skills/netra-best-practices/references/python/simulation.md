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
from netra.simulation import BaseTask, TaskResult

class MyAgentTask(BaseTask):
    def run(self, message, session_id=None, files=None, setup_context=None):
        response = my_agent.chat(message, session_id=session_id)
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
| `setup_context` | Optional dict from `before_all` / `before` hooks. `None` when no hooks are configured. |

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
        response = await my_async_agent.chat(message, session_id=session_id)
        return TaskResult(
            message=response.text,
            session_id=response.session_id or session_id or "default",
        )
```

### Task with file handling

```python
from netra.simulation import BaseTask, TaskResult, ProcessedFile

class FileTask(BaseTask):
    def run(self, message, session_id=None, files=None, setup_context=None):
        if files:
            for f in files:
                print(f.file_name, f.content_type, len(f.data))
        response = my_agent.chat(message, session_id=session_id, files=files)
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

### The four hook points

| Hook | When it runs | Failure effect |
|------|-------------|----------------|
| `before_all` | Once, before any scenario starts | Entire run is marked failed; no scenarios execute |
| `before` | Before each individual scenario | That scenario is marked `prescript_failed`; others continue |
| `after` | After each individual scenario (success or failure) | Logged as warning; does not affect scenario status |
| `after_all` | Once, after all scenarios complete | Logged as warning; does not affect run status |

### Context passing

Hooks pass data to each other and to your `BaseTask` via a context dict:

```
before_all()            → shared_context (dict)
before(id, shared_ctx)  → item_context merged into shared_context
BaseTask.run(..., setup_context=merged_context)
after(id, result, shared_context)
after_all(results, shared_context)
```

### SimulationHooks

```python
from netra.simulation import SimulationHooks
```

```python
@dataclass
class SimulationHooks:
    before_all: Optional[Callable] = None   # () -> dict | None
    before:     Optional[Callable] = None   # (run_item_id, shared_context) -> dict | None
    after:      Optional[Callable] = None   # (run_item_id, result, shared_context) -> None
    after_all:  Optional[Callable] = None   # (results, shared_context) -> None
```

All hooks can be sync or async.

### Example — correlated scenarios with shared state

```python
from netra.simulation import BaseTask, SimulationHooks, TaskResult

# ---- Hooks ----

def setup_environment():
    """Create a test employee and assign the admin role before any scenario runs."""
    employee = api.create_employee(name="Test User", department="Engineering")
    api.assign_role(employee.id, "admin")
    return {"employee_id": employee.id}


def setup_scenario(run_item_id, shared_context):
    """Log in as the test employee before each scenario."""
    token = api.login(employee_id=shared_context["employee_id"])
    return {"auth_token": token}


def teardown_scenario(run_item_id, result, shared_context):
    """Log out after each scenario regardless of outcome."""
    api.logout(token=shared_context.get("auth_token"))


def teardown_environment(results, shared_context):
    """Delete the test employee once all scenarios are done."""
    api.delete_employee(employee_id=shared_context["employee_id"])


hooks = SimulationHooks(
    before_all=setup_environment,
    before=setup_scenario,
    after=teardown_scenario,
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
        )
        return TaskResult(
            message=response.text,
            session_id=session_id or response.session_id or "default",
        )


# ---- Run ----

result = Netra.simulation.run_simulation(
    name="Employee Workflow Simulation",
    dataset_id="your-dataset-id",
    task=MyAgentTask(),
    hooks=hooks,
    max_concurrency=3,  # scenarios run in parallel, each getting its own auth_token
)
```

### Async hooks

```python
async def setup_environment():
    employee = await async_api.create_employee(name="Test User")
    return {"employee_id": employee.id}

async def setup_scenario(run_item_id, shared_context):
    token = await async_api.login(shared_context["employee_id"])
    return {"auth_token": token}

hooks = SimulationHooks(
    before_all=setup_environment,
    before=setup_scenario,
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

When `hooks` are passed, the Netra dashboard shows which hook types are configured on the test run (e.g. "Has pre-script" badge). The actual script code is never stored by Netra.

---

## Validation Checklist

1. `Netra.init()` is called before accessing `Netra.simulation`.
2. `NETRA_API_KEY` and `NETRA_OTLP_ENDPOINT` are set.
3. Task's `run()` always returns a `TaskResult` with both `message` and `session_id`.
4. `Netra.shutdown()` is called on graceful termination.
5. When using hooks: `before_all` and `before` return a `dict` or `None` (other types are ignored).
6. When using hooks: `after` and `after_all` should not raise — wrap risky cleanup in try/except.

## References

- https://docs.getnetra.ai/Simulation/Simulation-overview
- https://docs.getnetra.ai/sdk-reference/simulation/python
