---
name: netra-single-turn-evaluations
description: Run single-turn evaluations with the Netra SDK. Use when setting up datasets, test suites, task functions, or custom evaluators for single-input/single-output LLM evaluation.
---

# Netra Single-Turn Evaluations

Evaluate LLM or agent outputs on a per-input basis using datasets, automated test suites, and pluggable evaluators through the Netra SDK.

## Workflow

- Ensure `Netra.init()` is called before using any evaluation APIs.
- Prepare a dataset — either inline in code or managed via the Netra API.
- Define a **task function** that takes an input and returns an output (the code under test).
- Optionally implement **custom local evaluators** to score each output.
- Run `Netra.evaluation.run_test_suite()` to execute the task against every dataset item, run evaluators, and report results.
- Review results on the Netra dashboard or inspect the returned summary dict.

# Initialization

Enable evaluation by initializing Netra at application startup. The evaluation client (`Netra.evaluation`) is created automatically.

```env
NETRA_API_KEY=
NETRA_OTLP_ENDPOINT=
```

```py
import os
from netra import Netra

Netra.init(
    app_name="my-ai-app",
    environment="production",
    headers=f"x-api-key={os.getenv('NETRA_API_KEY')}",
)
```

> [!IMPORTANT]
> `Netra.evaluation` is `None` if `Netra.init()` has not been called or if the OTLP endpoint is missing. Always init first.

# Dataset Preparation

A dataset is a list of items, each with an `input` and optional `expected_output`, `metadata`, and `tags`.

## Inline dataset (no API call)

Use `Dataset` and `DatasetItem` to define items directly in code. Best for quick experiments or CI pipelines.

```py
from netra.evaluation import DatasetItem, Dataset

dataset = Dataset(items=[
    DatasetItem(
        input="What is the capital of France?",
        expected_output="Paris",
    ),
    DatasetItem(
        input="Summarize this article in one sentence.",
        expected_output="The article discusses recent advances in renewable energy.",
        metadata={"category": "summarization"},
    ),
])
```

## API-managed dataset

Create a persistent dataset on the Netra platform, add items via API, then fetch them for test runs. Useful when datasets are shared across runs or managed from the dashboard.

```py
response = Netra.evaluation.create_dataset(
    name="QA Golden Set",
    tags=["qa", "v1"],
)
dataset_id = response.id

Netra.evaluation.add_dataset_item(
    dataset_id=dataset_id,
    item=DatasetItem(
        input="What is the capital of France?",
        expected_output="Paris",
        metadata={"difficulty": "easy"},
        tags=["geography"],
    ),
)

fetched = Netra.evaluation.get_dataset(dataset_id)
dataset = Dataset(items=fetched.items)
```

# Task Function

The `task` is a callable that receives a single dataset item's `input` and returns the output to be evaluated. It can be sync or async.

```py
from openai import OpenAI

client = OpenAI()

def my_task(input):
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": input}],
    )
    return response.choices[0].message.content
```

```py
async def my_async_task(input):
    response = await async_client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": input}],
    )
    return response.choices[0].message.content
```

Each task invocation is automatically wrapped in a `TestRun.{name}` span, so the call is traced end-to-end without extra instrumentation.

# Running a Test Suite

`run_test_suite` is the main entry point. It creates a test run, executes the task for every item, runs local evaluators, submits results, and marks the run as completed.

```py
result = Netra.evaluation.run_test_suite(
    name="QA Agent v2",
    data=dataset,
    task=my_task,
    evaluators=[my_evaluator],   # optional, see Custom Evaluators below
    max_concurrency=50,          # default 50
)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | `str` | Yes | Display name for the test run. |
| `data` | `Dataset` | Yes | Dataset of items to evaluate. |
| `task` | `Callable[[Any], Any]` | Yes | Function that produces output from input. |
| `evaluators` | `List[BaseEvaluator]` | No | Local evaluators to score each item. |
| `max_concurrency` | `int` | No | Max parallel task executions (default 50). |

**Return value** on success:

```py
{
    "runId": "uuid-of-the-run",
    "items": [
        {
            "index": 0,
            "status": "completed",
            "traceId": "abc123...",
            "spanId": "def456...",
            "testRunItemId": "ghi789...",
        },
        # ...
    ],
}
```

Returns `None` if validation fails or the run could not be created.

# Custom Evaluators

Write local evaluators that run client-side before results are sent to the platform. Subclass `BaseEvaluator` and implement `evaluate()`.

```py
from netra.evaluation import BaseEvaluator, EvaluatorConfig, EvaluatorContext, EvaluatorOutput, ScoreType

class ExactMatchEvaluator(BaseEvaluator):
    def evaluate(self, context: EvaluatorContext) -> EvaluatorOutput:
        is_match = str(context.task_output).strip().lower() == str(context.expected_output).strip().lower()
        return EvaluatorOutput(
            evaluator_name=self.config.name,
            result=is_match,
            is_passed=is_match,
            reason="Exact match" if is_match else "Output did not match expected",
        )

exact_match = ExactMatchEvaluator(
    EvaluatorConfig(
        name="exact_match",
        label="Exact Match",
        score_type=ScoreType.BOOLEAN,
    )
)
```

**`EvaluatorConfig` fields:**

| Field | Type | Description |
|-------|------|-------------|
| `name` | `str` | Unique identifier for the evaluator. |
| `label` | `str` | Human-readable display name. |
| `score_type` | `ScoreType` | `BOOLEAN`, `NUMERICAL`, or `CATEGORICAL`. |

**`EvaluatorContext` fields (passed to `evaluate`):**

| Field | Type | Description |
|-------|------|-------------|
| `input` | `Any` | The original input from the dataset item. |
| `task_output` | `Any` | The output returned by the task function. |
| `expected_output` | `Any` | The expected output from the dataset item (may be `None`). |
| `metadata` | `Optional[Dict]` | Optional metadata from the dataset item. |

**`EvaluatorOutput` fields (returned from `evaluate`):**

| Field | Type | Description |
|-------|------|-------------|
| `evaluator_name` | `str` | Must match `config.name`. |
| `result` | `Any` | The score or value (bool, number, or string). |
| `is_passed` | `bool` | Whether this item passed the evaluator's criteria. |
| `reason` | `Optional[str]` | Human-readable explanation. |

`evaluate()` can be sync or async — the framework awaits coroutines automatically.

## Numerical evaluator example

```py
class RelevanceScoreEvaluator(BaseEvaluator):
    def evaluate(self, context: EvaluatorContext) -> EvaluatorOutput:
        score = compute_relevance(context.input, context.task_output)  # 0.0–1.0
        return EvaluatorOutput(
            evaluator_name=self.config.name,
            result=score,
            is_passed=score >= 0.7,
            reason=f"Relevance score: {score:.2f}",
        )

relevance = RelevanceScoreEvaluator(
    EvaluatorConfig(
        name="relevance",
        label="Relevance Score",
        score_type=ScoreType.NUMERICAL,
    )
)
```

# Platform Evaluators

In addition to local evaluators, Netra provides built-in platform evaluators that run server-side after trace ingestion. These are configured via the Netra dashboard and attached to datasets. Available types:

| Evaluator | Description |
|-----------|-------------|
| **LLM-as-Judge** | Uses an LLM to grade outputs against a rubric or criteria. |
| **Semantic Similarity** | Compares output to expected output using embedding similarity. |
| **Tool Accuracy** | Validates that the correct tools were called with expected arguments. |
| **Cost** | Evaluates total cost of the traced LLM calls. |
| **Latency** | Evaluates end-to-end latency of the traced execution. |
| **Token** | Evaluates total token usage across the trace. |
| **Regex** | Matches output against a regular expression pattern. |
| **JSON** | Validates output against a JSON schema. |
| **Code** | Runs custom code-based evaluation logic server-side. |

Platform evaluators with `evalType: TURN` run automatically for each single-turn test run item after the trace is ingested. No client-side code is needed — attach them to your dataset from the dashboard.

# Full Example

```py
import os
from netra import Netra
from netra.evaluation import (
    BaseEvaluator,
    DatasetItem,
    Dataset,
    EvaluatorConfig,
    EvaluatorContext,
    EvaluatorOutput,
    ScoreType,
)
from openai import OpenAI

Netra.init(
    app_name="my-ai-app",
    environment="staging",
    headers=f"x-api-key={os.getenv('NETRA_API_KEY')}",
)

client = OpenAI()

def qa_task(input):
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": input}],
    )
    return response.choices[0].message.content

class ContainsExpectedEvaluator(BaseEvaluator):
    def evaluate(self, context: EvaluatorContext) -> EvaluatorOutput:
        output = str(context.task_output).lower()
        expected = str(context.expected_output).lower()
        contains = expected in output
        return EvaluatorOutput(
            evaluator_name=self.config.name,
            result=contains,
            is_passed=contains,
            reason="Output contains expected answer" if contains else "Expected answer not found in output",
        )

dataset = Dataset(items=[
    DatasetItem(input="What is the capital of France?", expected_output="Paris"),
    DatasetItem(input="What is 2 + 2?", expected_output="4"),
    DatasetItem(input="Who wrote Hamlet?", expected_output="Shakespeare"),
])

result = Netra.evaluation.run_test_suite(
    name="QA Bot Regression v1",
    data=dataset,
    task=qa_task,
    evaluators=[
        ContainsExpectedEvaluator(
            EvaluatorConfig(
                name="contains_expected",
                label="Contains Expected Answer",
                score_type=ScoreType.BOOLEAN,
            )
        )
    ],
)

Netra.shutdown()
```

## Validation checklist

1. `Netra.init()` is called before accessing `Netra.evaluation`.
2. `NETRA_API_KEY` and `NETRA_OTLP_ENDPOINT` environment variables are set.
3. Every `DatasetItem` has a non-empty `input`.
4. The `task` function accepts a single argument (the item input) and returns the output.
5. Custom evaluator `evaluate()` returns an `EvaluatorOutput` with `evaluator_name` matching `config.name`.
6. `ScoreType` matches the type of `result` in `EvaluatorOutput` (bool for `BOOLEAN`, number for `NUMERICAL`, string for `CATEGORICAL`).
7. `Netra.shutdown()` is called on graceful termination to flush pending traces and evaluation data.
8. Test run results appear on the Netra dashboard after the run completes.

## References

- https://docs.getnetra.ai/Evaluations/overview
- https://docs.getnetra.ai/sdk-reference/evaluation/python
- https://docs.getnetra.ai/Observability/Traces/configuration/initialization
- https://docs.getnetra.ai/sdk-reference/sdk/python
