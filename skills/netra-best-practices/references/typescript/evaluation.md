---
name: netra-typescript-evaluation
description: Run single-turn evaluations with the TypeScript Netra SDK. Covers datasets, test suites, task functions, and custom evaluators.
---

# Netra TypeScript Single-Turn Evaluations

Evaluate LLM or agent outputs on a per-input basis using datasets, automated test suites, and pluggable evaluators through the TypeScript Netra SDK.

## Workflow

- Ensure `await Netra.init()` is called before using any evaluation APIs.
- Prepare a dataset — either inline or managed via the Netra API.
- Define a **task function** that takes an input and returns an output.
- Optionally implement evaluator objects to score each output.
- Run `Netra.evaluation.runTestSuite()` to execute and report results.
- Review results on the Netra dashboard or inspect the returned response.

## Initialization

Enable evaluation by initializing Netra at application startup.

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

> [!IMPORTANT]
> `Netra.evaluation` is `None` if `await Netra.init()` has not been called or if the OTLP endpoint is missing. Always init first.

## Dataset Preparation

A dataset is a list of items, each with an `input` and optional `expectedOutput`, `metadata`, and `tags`.

### Inline dataset (no API call)

```typescript
import type { Dataset, DatasetEntry } from "netra-sdk";

const dataset: Dataset = {
  items: [
    {
      input: "What is the capital of France?",
      expectedOutput: "Paris",
    },
    {
      input: "Summarize this article in one sentence.",
      expectedOutput:
        "The article discusses recent advances in renewable energy.",
      metadata: { category: "summarization" },
    },
  ],
};
```

### API-managed dataset

Create a persistent dataset on the Netra platform, add items via API, then fetch them for test runs. Useful when datasets are shared across runs or managed from the dashboard.

```typescript
const response = await Netra.evaluation.createDataset("QA Golden Set", [
  "qa",
  "v1",
]);
const datasetId = response!.id;

await Netra.evaluation.addDatasetItem(datasetId, {
  input: "What is the capital of France?",
  expectedOutput: "Paris",
  metadata: { difficulty: "easy" },
  tags: ["geography"],
});

const fetched = await Netra.evaluation.getDataset(datasetId);
const dataset = { items: fetched!.items };
```

## Task Function

A function receiving a single dataset item's `input` and returning the output. Can be sync or async.

```typescript
import { Netra } from "netra-sdk";
import OpenAI from "openai";

const client = new OpenAI();

async function myTask(input: any): Promise<string> {
  Netra.setRootInput(input);
  const response = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: input }],
  });
  const output = response.choices[0].message.content!;
  Netra.setRootOutput(output);
  return output;
}
```

Each task invocation is automatically wrapped in a `TestRun.{name}` span, so the call is traced end-to-end without extra instrumentation.

**Root input/output must be the core values only.** `setRootInput` takes the dataset item's input as the system receives it, and `setRootOutput` takes the answer the task produces — not the raw SDK response object, and not a result envelope. If the task builds a richer prompt (system prompt, retrieved context, few-shot examples) or returns a structured result, still pass only the user-level input and the user-facing answer:

```typescript
// ❌ Wrong — prompt scaffolding in, SDK response envelope out
Netra.setRootInput({ input, systemPrompt: SYSTEM_PROMPT, context: chunks });
Netra.setRootOutput(response);

// ✅ Right
Netra.setRootInput(input);
Netra.setRootOutput(response.choices[0].message.content!);
```

This matters more here than anywhere else: evaluators score the trace's `input`/`output`, so extra metadata directly degrades evaluation results.

## Running a Test Suite

`runTestSuite` is the main entry point. It creates a test run, executes the task for every item, runs local evaluators, submits results, and marks the run as completed.

```typescript
const result = await Netra.evaluation.runTestSuite(
  "QA Agent v2",
  dataset,
  myTask,
  [myEvaluator],
  50, // maxConcurrency
);
```

| Parameter        | Type           | Required | Description                               |
| ---------------- | -------------- | -------- | ----------------------------------------- |
| `name`           | `string`       | Yes      | Display name for the test run             |
| `data`           | `Dataset`      | Yes      | Dataset of items to evaluate              |
| `task`           | `TaskFunction` | Yes      | Function producing output from input      |
| `evaluators`     | `any[]`        | No       | Evaluator objects to score each item      |
| `maxConcurrency` | `number`       | No       | Max parallel task executions (default 50) |

Returns `results of the test suite` if validation succeeds and `null` if validation fails.

## Custom Evaluators

Evaluators are objects with a `.config` property and an `.evaluate()` method:

```typescript
import type { EvaluatorConfig } from "netra-sdk";

interface EvaluatorContext {
  input: any;
  taskOutput: any;
  expectedOutput?: any;
  metadata?: Record<string, any>;
}

interface EvaluatorOutput {
  evaluatorName: string;
  result: any;
  isPassed: boolean;
  reason?: string;
}

const exactMatch = {
  config: {
    name: "exact_match",
    label: "Exact Match",
    scoreType: "boolean",
  } as EvaluatorConfig,

  evaluate(context: EvaluatorContext): EvaluatorOutput {
    const isMatch =
      String(context.taskOutput).trim().toLowerCase() ===
      String(context.expectedOutput).trim().toLowerCase();
    return {
      evaluatorName: this.config.name,
      result: isMatch,
      isPassed: isMatch,
      reason: isMatch ? "Exact match" : "Output did not match expected",
    };
  },
};
```

## Full Example

```typescript
import { Netra } from "netra-sdk";
import type { Dataset } from "netra-sdk";
import OpenAI from "openai";

await Netra.init({
  appName: "my-ai-app",
  environment: "staging",
  headers: `x-api-key=${process.env.NETRA_API_KEY}`,
});

const client = new OpenAI();

async function qaTask(input: any): Promise<string> {
  Netra.setRootInput(input);
  const response = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: input }],
  });
  const output = response.choices[0].message.content!;
  Netra.setRootOutput(output);
  return output;
}

const containsExpected = {
  config: {
    name: "contains_expected",
    label: "Contains Expected",
    scoreType: "boolean",
  },
  evaluate(ctx: { input: any; taskOutput: any; expectedOutput: any }) {
    const output = String(ctx.taskOutput).toLowerCase();
    const expected = String(ctx.expectedOutput).toLowerCase();
    const contains = output.includes(expected);
    return {
      evaluatorName: this.config.name,
      result: contains,
      isPassed: contains,
      reason: contains
        ? "Contains expected answer"
        : "Expected answer not found",
    };
  },
};

const dataset: Dataset = {
  items: [
    { input: "What is the capital of France?", expectedOutput: "Paris" },
    { input: "What is 2 + 2?", expectedOutput: "4" },
    { input: "Who wrote Hamlet?", expectedOutput: "Shakespeare" },
  ],
};

const result = await Netra.evaluation.runTestSuite(
  "QA Bot Regression v1",
  dataset,
  qaTask,
  [containsExpected],
);

await Netra.shutdown();
```

## Validation Checklist

1. `await Netra.init()` is called before accessing `Netra.evaluation`.
2. `NETRA_API_KEY` and `NETRA_OTLP_ENDPOINT` environment variables are set.
3. Every dataset item has a non-empty `input`.
4. The task function accepts a single argument and returns the output.
5. The task function calls `Netra.setRootInput(input)` at the start and `Netra.setRootOutput(output)` before returning, passing **only the core input and output** — the dataset item's input and the user-facing answer, never prompt scaffolding, retrieved context, or a raw SDK response/result envelope.
6. Evaluator `.evaluate()` returns an object with `evaluatorName` matching `config.name`.
6. `await Netra.shutdown()` is called on graceful termination.

## References

- https://docs.getnetra.ai/Evaluation/Evaluation-overview
- https://docs.getnetra.ai/sdk-reference/evaluation/typescript
- https://docs.getnetra.ai/Observability/Traces/configuration/initialization
- https://docs.getnetra.ai/sdk-reference/sdk/typescript
