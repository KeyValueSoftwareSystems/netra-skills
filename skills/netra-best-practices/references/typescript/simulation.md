---
name: netra-typescript-simulation
description: Run multi-turn conversation simulations with the TypeScript Netra SDK. Covers BaseTask abstract class and simulation options.
---

# TypeScript Simulation

Test AI agents with realistic, multi-turn conversations using the TypeScript Netra SDK's simulation framework.

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

```typescript
import { Netra } from "netra-sdk";

await Netra.init({
  appName: "my-ai-app",
  environment: "production",
  headers: `x-api-key=${process.env.NETRA_API_KEY}`,
});
```

## Implementing a Task

Extend the `BaseTask` abstract class and implement `run()`:

```typescript
import { BaseTask } from "netra-sdk";
import type { TaskResult } from "netra-sdk";

class MyAgentTask extends BaseTask {
  async run(message: string, sessionId?: string | null): Promise<TaskResult> {
    const response = await myAgent.chat(message, { sessionId });
    return {
      message: response.text,
      sessionId: response.sessionId || sessionId || "default",
    };
  }
}
```

### Task signature

```typescript
abstract run(
  message: string,
  sessionId?: string | null,
): Promise<TaskResult> | TaskResult;
```

- `message`: The user message for this turn.
- `sessionId`: Session identifier for conversation continuity (`null`/`undefined` on first turn).

### TaskResult interface

```typescript
interface TaskResult {
  message: string;
  sessionId: string;
}
```

### TaskResult fields

| Field       | Type  | Description                            |
| ----------- | ----- | -------------------------------------- |
| `message`   | `str` | The agent's response message           |
| `sessionId` | `str` | Session ID for conversation continuity |

**Note:** The TypeScript `BaseTask.run()` does not receive `files` — file handling differs from the Python SDK.

### Task example

```typescript
class MyTask extends BaseTask {
  async run(
    message: string,
    sessionId?: string,
    files?: unknown[],
  ): Promise<TaskResult> {
    const response = await myAgent.chat(message, {
      sessionId,
    });

    return {
      message: response.text,
      sessionId: response.sessionId ?? sessionId ?? "default",
    };
  }
}
```

## Running a Simulation

```typescript
import type { SimulationOptions } from "netra-sdk";

const result = await Netra.simulation.runSimulation({
  name: "Agent Stress Test v1",
  datasetId: "your-dataset-id",
  task: new MyAgentTask(),
  maxConcurrency: 5,
});
```

| Field            | Type       | Required | Default     | Description                              |
| ---------------- | ---------- | -------- | ----------- | ---------------------------------------- |
| `name`           | `string`   | Yes      | —           | Display name for the simulation run      |
| `datasetId`      | `string`   | Yes      | —           | ID of the dataset on the Netra platform  |
| `task`           | `BaseTask` | Yes      | —           | Your task implementation                 |
| `context`        | `any`      | No       | `undefined` | Additional context passed to the backend |
| `maxConcurrency` | `number`   | No       | `5`         | Max parallel conversations (capped at 5) |

### SimulationResult interface

```typescript
interface SimulationResult {
  success: boolean;
  completed: ConversationResult[];
  failed: ConversationResult[];
  totalItems: number;
}
```

Datasets for simulation are created and managed via the Netra dashboard.

## Validation Checklist

1. `await Netra.init()` is called before accessing `Netra.simulation`.
2. `NETRA_API_KEY` and `NETRA_OTLP_ENDPOINT` are set.
3. Task's `run()` always returns a `TaskResult` with both `message` and `sessionId`.
4. `await Netra.shutdown()` is called on graceful termination.

## References

- https://docs.getnetra.ai/Simulation/Simulation-overview
- https://docs.getnetra.ai/sdk-reference/simulation/typescript
