# Single-Turn Evaluations (Netra SDK)

Use this reference when the user asks to set up or automate single-turn evaluations (input -> output) with Netra.

## Outcome

Set up a repeatable evaluation loop where:
- Test cases live in a Netra dataset.
- A task function runs the user's app logic for each dataset item.
- Netra executes a test run and scores outputs with evaluators.

## Prerequisites

1. Netra SDK installed (`netra-sdk` for Python, `netra-sdk` for TypeScript).
2. Netra initialized with API key/header.
3. At least one dataset configured in the dashboard, or created programmatically.
4. Evaluators attached in the dashboard (recommended), or passed in code.

## Recommended workflow for the agent

1. Confirm language/runtime (Python or TypeScript).
2. Ensure `Netra.init(...)` / `await Netra.init(...)` is called once at startup.
3. Fetch dataset via SDK.
4. Define `task(input)` that returns the system output string/value.
5. Run test suite with a clear run name and safe concurrency.
6. Return run id and quick status summary to the user.
7. Direct the user to Evaluation -> Test Runs for detailed scores.

## Python template

```python
from netra import Netra

Netra.init(
		app_name="my-app",
		headers="x-api-key=YOUR_NETRA_API_KEY",
)

def my_task(input_data):
		# Call the user's app/agent here and return generated output
		return f"response for: {input_data}"

dataset = Netra.evaluation.get_dataset(dataset_id="your-dataset-id")

result = Netra.evaluation.run_test_suite(
		name="My Single-Turn Eval",
		data=dataset,
		task=my_task,
		evaluators=["correctness", "relevance"],  # optional
		max_concurrency=5,
)

print(result["runId"])
```

## TypeScript template

```typescript
import { Netra } from "netra-sdk";

await Netra.init({
	appName: "my-app",
	headers: `x-api-key=${process.env.NETRA_API_KEY}`,
});

async function myTask(inputData: any): Promise<string> {
	// Call the user's app/agent here and return generated output
	return `response for: ${String(inputData)}`;
}

const dataset = await Netra.evaluation.getDataset("your-dataset-id");

const result = await Netra.evaluation.runTestSuite(
	"My Single-Turn Eval",
	dataset,
	myTask,
	["correctness", "relevance"], // optional
	5
);

console.log(result?.runId);
```

## Programmatic dataset management (optional)

Use these SDK APIs when the user wants setup fully in code:
- Python: `create_dataset`, `add_dataset_item`, `get_dataset`, `run_test_suite`
- TypeScript: `createDataset`, `addDatasetItem`, `getDataset`, `runTestSuite`

Minimal pattern:
1. Create dataset.
2. Add dataset items with `input` and `expected_output`/`expectedOutput`.
3. Fetch dataset and execute test suite.

### Python example (fully programmatic)

```python
from netra import Netra

Netra.init(
		app_name="my-app",
		headers="x-api-key=YOUR_NETRA_API_KEY",
)

created = Netra.evaluation.create_dataset(name="Support QA Dataset")
dataset_id = created["datasetId"]

Netra.evaluation.add_dataset_item(
		dataset_id=dataset_id,
		item={
				"input": "What is your refund window?",
				"expected_output": "You can request a refund within 30 days of purchase.",
		},
)

Netra.evaluation.add_dataset_item(
		dataset_id=dataset_id,
		item={
				"input": "Do you support overnight shipping?",
				"expected_output": "Yes, overnight shipping is available in select regions.",
		},
)

def task(input_data):
		# Replace with your real app/agent call.
		return f"response for: {input_data}"

dataset = Netra.evaluation.get_dataset(dataset_id=dataset_id)

result = Netra.evaluation.run_test_suite(
		name="Support QA Programmatic Eval",
		data=dataset,
		task=task,
		max_concurrency=3,
)

print(result["runId"])
```

### TypeScript example (fully programmatic)

```typescript
import { Netra } from "netra-sdk";

await Netra.init({
	appName: "my-app",
	headers: `x-api-key=${process.env.NETRA_API_KEY}`,
});

const created = await Netra.evaluation.createDataset("Support QA Dataset");
const datasetId = created?.datasetId;

if (!datasetId) {
	throw new Error("Dataset creation failed: missing datasetId");
}

await Netra.evaluation.addDatasetItem(datasetId, {
	input: "What is your refund window?",
	expectedOutput: "You can request a refund within 30 days of purchase.",
});

await Netra.evaluation.addDatasetItem(datasetId, {
	input: "Do you support overnight shipping?",
	expectedOutput: "Yes, overnight shipping is available in select regions.",
});

const dataset = await Netra.evaluation.getDataset(datasetId);

const result = await Netra.evaluation.runTestSuite(
	"Support QA Programmatic Eval",
	dataset,
	async (inputData: any) => {
		// Replace with your real app/agent call.
		return `response for: ${String(inputData)}`;
	},
	undefined,
	3
);

console.log(result?.runId);
```

## What to check after setup

1. `runId` is returned.
2. Items are mostly `completed` (not `failed`).
3. Traces are linked for each test item.
4. Evaluator scores appear in Test Runs.

## Troubleshooting guidance

- No test runs: confirm dataset has evaluators and the correct dataset id is used.
- Empty/invalid outputs: ensure the `task` function returns output for every item.
- Too many failures/timeouts: lower concurrency first (`max_concurrency` / `maxConcurrency`).

## References

- https://docs.getnetra.ai/quick-start/QuickStart_Evals
- https://docs.getnetra.ai/Evaluation/Datasets
- https://docs.getnetra.ai/sdk-reference/evaluation/python
- https://docs.getnetra.ai/sdk-reference/evaluation/typescript
