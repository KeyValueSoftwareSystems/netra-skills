---
name: netra-python-simulation
description: Run multi-turn conversation simulations with the Python Netra SDK. Covers BaseTask, run_simulation, and file handling.
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
    def run(self, message, session_id=None, files=None):
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
) -> TaskResult | Awaitable[TaskResult]
```

- `message`: The user message for this turn.
- `session_id`: Session identifier for conversation continuity (`None` on first turn).
- `files`: Optional list of `ProcessedFile` (base64-encoded file attachments).

### TaskResult fields

| Field | Type | Description |
|---|---|---|
| `message` | `str` | The agent's response message |
| `session_id` | `str` | Session ID for conversation continuity |

### Async task example

```python
class MyAsyncTask(BaseTask):
    async def run(self, message, session_id=None, files=None):
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
    def run(self, message, session_id=None, files=None):
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

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | `str` | Yes | — | Display name for the simulation run |
| `dataset_id` | `str` | Yes | — | ID of the dataset on the Netra platform |
| `task` | `BaseTask` | Yes | — | Your task implementation |
| `context` | `Any` | No | `None` | Additional context passed to the backend |
| `max_concurrency` | `int` | No | `5` | Max parallel conversations |
| `max_turns` | `int` | No | `50` | Max turns per conversation |

Datasets for simulation are created and managed via the Netra dashboard.

## Validation Checklist

1. `Netra.init()` is called before accessing `Netra.simulation`.
2. `NETRA_API_KEY` and `NETRA_OTLP_ENDPOINT` are set.
3. Task's `run()` always returns a `TaskResult` with both `message` and `session_id`.
4. `Netra.shutdown()` is called on graceful termination.

## References

- https://docs.getnetra.ai/Simulation/Simulation-overview
- https://docs.getnetra.ai/sdk-reference/simulation/python
