---
name: netra-typescript-simulation
description: Run multi-turn conversation simulations with the TypeScript Netra SDK. Covers BaseTask, runSimulation, SimulationHooks, and file handling.
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
import { BaseTask, Netra } from "netra-sdk";
import type { TaskResult } from "netra-sdk";

class MyAgentTask extends BaseTask {
  async run(
    message: string,
    sessionId?: string | null,
    files?: ProcessedFile[] | null,
    setupContext?: Record<string, any> | null,
  ): Promise<TaskResult> {
    Netra.setRootInput(message);
    const response = await myAgent.chat(message, { sessionId });
    Netra.setRootOutput(response.text);
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
  files?: ProcessedFile[] | null,
  setupContext?: Record<string, any> | null,
): Promise<TaskResult> | TaskResult;
```

| Parameter | Description |
|-----------|-------------|
| `message` | The user message for this turn. |
| `sessionId` | Session identifier for conversation continuity (`null`/`undefined` on first turn). |
| `files` | Optional list of `ProcessedFile` (base64-encoded file attachments). |
| `setupContext` | Optional dict from `beforeAll` / `beforeEach` / `before` hooks. `null` when no hooks are configured. |

> `setupContext` is backwards compatible — existing tasks that don't declare this parameter continue to work without modification.

### TaskResult interface

```typescript
interface TaskResult {
  message: string;
  sessionId: string;
}
```

### TaskResult fields

| Field       | Type     | Description                            |
| ----------- | -------- | -------------------------------------- |
| `message`   | `string` | The agent's response message           |
| `sessionId` | `string` | Session ID for conversation continuity |

### Task with file handling

```typescript
import { BaseTask } from "netra-sdk";
import type { ProcessedFile, TaskResult } from "netra-sdk";

class FileTask extends BaseTask {
  async run(
    message: string,
    sessionId?: string | null,
    files?: ProcessedFile[] | null,
    setupContext?: Record<string, any> | null,
  ): Promise<TaskResult> {
    Netra.setRootInput(message);
    if (files?.length) {
      for (const f of files) {
        console.log(f.fileName, f.contentType, f.data.length);
      }
    }
    const response = await myAgent.chat(message, { sessionId, files });
    Netra.setRootOutput(response.text);
    return {
      message: response.text,
      sessionId: response.sessionId || sessionId || "default",
    };
  }
}
```

`ProcessedFile` fields: `fileName`, `contentType`, `description` (optional), `data` (base64 string).

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

| Field            | Type              | Required | Default     | Description                                         |
| ---------------- | ----------------- | -------- | ----------- | --------------------------------------------------- |
| `name`           | `string`          | Yes      | —           | Display name for the simulation run                 |
| `datasetId`      | `string`          | Yes      | —           | ID of the dataset on the Netra platform             |
| `task`           | `BaseTask`        | Yes      | —           | Your task implementation                            |
| `context`        | `any`             | No       | `undefined` | Additional context passed to the backend            |
| `maxConcurrency` | `number`          | No       | `5`         | Max parallel conversations (capped at 5)            |
| `hooks`          | `SimulationHooks` | No       | `undefined` | Pre/post script hooks (see Hooks section below)     |

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

---

## Hooks (Pre/Post Scripts)

Multi-turn datasets often contain scenarios that share state — for example, Scenario 2 requires an employee created in Scenario 1. Rather than merging all related scenarios into one large item or relying on sequential ordering, you can use **hooks** to set up and tear down shared state around scenario execution.

Hooks are defined on the **SDK side** — the actual scripts never leave your environment. Lightweight metadata (function name and optional description) is sent to the backend so the Netra UI can display which hooks are configured on a test run.

### The six hook points

| Hook | When it runs | Failure effect |
|------|-------------|----------------|
| `beforeAll` | Once, before any scenario starts | Entire run is marked failed; no scenarios execute |
| `beforeEach` | Before every scenario | That scenario is marked `prescript_failed` (terminal; eval suppressed); others continue |
| `before` | Before specific scenarios only (keyed by `datasetItemId`) | That scenario is marked `prescript_failed` (terminal; eval suppressed); others continue |
| `after` | After specific scenarios only (keyed by `datasetItemId`) | Logged as warning; does not affect scenario status |
| `afterEach` | After every scenario (after item-specific `after`) | Logged as warning; does not affect scenario status |
| `afterAll` | Once, after all scenarios complete | Logged as warning; does not affect run status |

### Context passing

Hooks pass data to each other and to your `BaseTask` via a context object:

```
beforeAll()                            → sharedContext (Record | null)
beforeEach(sharedContext)              → merged into setupContext
before[datasetItemId](mergedContext)   → merged into setupContext (if registered)
BaseTask.run(..., setupContext)
after[datasetItemId](result, setupContext) (if registered)
afterEach(result, setupContext)
afterAll(results, sharedContext)
```

`setupContext` is the merge of `beforeAll` + `beforeEach` + item `before`. `BaseTask.run`, item `after`, and `afterEach` receive it so teardown can clean up per-scenario resources (tokens, accounts, etc.). If a before hook fails mid-way, teardown still receives the furthest successfully built `setupContext`. `afterAll` still receives only the run-level `sharedContext` from `beforeAll`, but its `results` include setup/first-turn failures as well as conversation failures.

### SimulationHooks

```typescript
import type { SimulationHooks } from "netra-sdk";
```

```typescript
interface SimulationHooks {
  beforeAll?: () => Record<string, any> | null | void | Promise<Record<string, any> | null | void>;
  beforeEach?: (sharedContext: Record<string, any> | null) => Record<string, any> | null | void | Promise<...>;
  before?: Record<string, (sharedContext: Record<string, any> | null) => Record<string, any> | null | void | Promise<...>>;
  after?: Record<string, (itemResult: Record<string, any>, setupContext: Record<string, any> | null) => void | Promise<void>>;
  afterEach?: (itemResult: Record<string, any>, setupContext: Record<string, any> | null) => void | Promise<void>;
  afterAll?: (results: Record<string, any>, sharedContext: Record<string, any> | null) => void | Promise<void>;
}
```

**Important:**
- Prefer `beforeEach` / `afterEach` when the same setup/teardown applies to every scenario
- `before` and `after` are **dicts** keyed by `datasetItemId`
- Only items with registered hooks will have their specific setup/teardown functions called
- Item `before` receives the merged context from `beforeAll` + `beforeEach`
- `after` / `afterEach` receive `result` and `setupContext` (merged `beforeAll` + `beforeEach` + item `before`)

All hooks can be sync or async (`Promise`-returning).

### Hook descriptions (required for Netra UI)

Set a `.description` property on each hook function (same pattern as the Python SDK):

```typescript
(fn as any).description = "Short explanation shown in the Netra UI";
```

Rules:
- Attach `.description` on **every** configured hook (`beforeAll`, `beforeEach`, each `before`/`after` entry, `afterEach`, `afterAll`).
- Keep it to one short sentence; the SDK truncates to **200 characters**.
- Without `.description`, the backend still receives `configured: true` and `name`, but `description` is `null` and the UI has less context.

### Example — Item-specific setup for particular scenarios

```typescript
import { BaseTask, Netra } from "netra-sdk";
import type { ProcessedFile, SimulationHooks, TaskResult } from "netra-sdk";

// ---- Hooks ----

function setupEnvironment() {
  const user = api.createUser({ name: "Test User", department: "Engineering" });
  return { userId: user.id };
}
(setupEnvironment as any).description =
  "Create a test user before any scenario runs.";

function setupEach(sharedContext: Record<string, any> | null) {
  const token = api.login({ userId: sharedContext?.userId });
  return { authToken: token };
}
(setupEach as any).description =
  "Obtain a fresh auth token before every scenario.";

function setupRefundScenario(sharedContext: Record<string, any> | null) {
  // sharedContext already includes userId + authToken from beforeAll + beforeEach
  const refundAccount = api.createRefundAccount();
  return { refundAccountId: refundAccount.id };
}
(setupRefundScenario as any).description =
  "Setup specific to refund scenarios - creates refund account.";

function setupPremiumScenario(sharedContext: Record<string, any> | null) {
  api.upgradeToPremium(sharedContext?.userId);
  return { isPremium: true };
}
(setupPremiumScenario as any).description =
  "Setup specific to premium user scenarios.";

async function teardownRefundScenario(
  result: Record<string, any>,
  setupContext: Record<string, any> | null,
) {
  try {
    api.deleteRefundAccount(setupContext?.refundAccountId);
  } catch {
    // after failures are logged but do not affect scenario status
  }
}
(teardownRefundScenario as any).description =
  "Cleanup refund-specific resources using item setup context.";

async function teardownPremiumScenario(
  result: Record<string, any>,
  setupContext: Record<string, any> | null,
) {
  try {
    api.downgradeFromPremium(setupContext?.userId);
  } catch {
    // ignore
  }
}
(teardownPremiumScenario as any).description =
  "Cleanup premium-specific resources using item setup context.";

async function teardownEach(
  result: Record<string, any>,
  setupContext: Record<string, any> | null,
) {
  try {
    api.logout({ token: setupContext?.authToken });
  } catch {
    // ignore
  }
}
(teardownEach as any).description =
  "Log out after every scenario.";

function teardownEnvironment(
  results: Record<string, any>,
  sharedContext: Record<string, any> | null,
) {
  api.deleteUser({ userId: sharedContext?.userId });
}
(teardownEnvironment as any).description =
  "Delete the test user once all scenarios are done.";

// Note: datasetItemId values come from your dataset items in the Netra dashboard
const hooks: SimulationHooks = {
  beforeAll: setupEnvironment,
  beforeEach: setupEach,
  before: {
    "refund-scenario-001": setupRefundScenario,
    "refund-scenario-002": setupRefundScenario,
    "premium-user-001": setupPremiumScenario,
  },
  after: {
    "refund-scenario-001": teardownRefundScenario,
    "refund-scenario-002": teardownRefundScenario,
    "premium-user-001": teardownPremiumScenario,
  },
  afterEach: teardownEach,
  afterAll: teardownEnvironment,
};

// ---- Task ----

class MyAgentTask extends BaseTask {
  async run(
    message: string,
    sessionId?: string | null,
    files?: ProcessedFile[] | null,
    setupContext?: Record<string, any> | null,
  ): Promise<TaskResult> {
    const ctx = setupContext || {};
    const response = await myAgent.chat(message, {
      sessionId,
      authToken: ctx.authToken,
      isPremium: ctx.isPremium ?? false,
    });
    return {
      message: response.text,
      sessionId: sessionId || response.sessionId || "default",
    };
  }
}

// ---- Run ----

const result = await Netra.simulation.runSimulation({
  name: "Multi-Scenario Workflow Test",
  datasetId: "your-dataset-id",
  task: new MyAgentTask(),
  hooks,
  maxConcurrency: 3,
});
```

### Async hooks

```typescript
async function setupEnvironment() {
  const user = await asyncApi.createUser({ name: "Test User" });
  return { userId: user.id };
}

async function setupRefundScenario(sharedContext: Record<string, any> | null) {
  const token = await asyncApi.login(sharedContext?.userId);
  return { authToken: token };
}

const hooks: SimulationHooks = {
  beforeAll: setupEnvironment,
  before: { "refund-scenario-001": setupRefundScenario },
};
```

### Hooks-only (no after hooks needed)

```typescript
const hooks: SimulationHooks = {
  beforeAll: setupEnvironment,
};
```

All hooks are optional. Omit any you don't need.

### Netra UI display

When `hooks` are passed, lightweight descriptors are sent to the backend as `lifecycleHooks`:
- `name` — from `fn.name`
- `description` — from `fn.description` (truncated to 200 characters)
- `configured: true`

The Netra dashboard shows which hook types are configured on the test run (e.g. "Has pre-script" badge) and can display the description text. The actual script code is never stored by Netra.

Always set `.description` on every hook so the UI has useful context.

---

## Validation Checklist

1. `await Netra.init()` is called before accessing `Netra.simulation`.
2. `NETRA_API_KEY` and `NETRA_OTLP_ENDPOINT` are set.
3. Task's `run()` always returns a `TaskResult` with both `message` and `sessionId`.
4. Task's `run()` calls `Netra.setRootInput(message)` at the start and `Netra.setRootOutput(response)` before returning.
5. `await Netra.shutdown()` is called on graceful termination.
6. When using hooks: `beforeAll` returns a plain object or `null`/`undefined` (other types are ignored).
7. When using hooks: `beforeEach` receives `sharedContext` and returns a plain object or `null`/`undefined`; runs for every item.
8. When using hooks: `before` dict values receive the merged context from `beforeAll` + `beforeEach` and return a plain object or `null`/`undefined`.
9. When using hooks: `after` / `afterEach` receive `result` and `setupContext` (merged `beforeAll` + `beforeEach` + item `before`); return value is ignored. They also run when the scenario fails, exceeds max turns, or a before hook fails (with the furthest successfully built `setupContext`).
10. When using hooks: `afterAll` should not throw — wrap risky cleanup in try/catch. Its `results` include setup/first-turn failures as well as conversation failures.
11. Hook dict keys must match `datasetItemId` values from your dataset.
12. When using hooks: every hook function has `(fn as any).description = "..."` set (≤ 200 chars).
13. A `prescript_failed` scenario is **terminal** — the SDK polling loop will not wait for it. Its `evalStatus` is set to `NOT_AVAILABLE` automatically and it cannot be overwritten by a timeout sweep. If all scenarios end as `failed` or `prescript_failed`, the run's evaluation status resolves to `NOT_AVAILABLE`.

## References

- https://docs.getnetra.ai/Simulation/Simulation-overview
- https://docs.getnetra.ai/sdk-reference/simulation/typescript
