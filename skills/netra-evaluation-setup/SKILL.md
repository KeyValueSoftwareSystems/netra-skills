---
name: netra-evaluation-setup
description: 'Set up high-quality Netra evaluations with datasets, evaluator design, variable mapping, and repeatable test runs. Use for regression detection and quality benchmarking.'
argument-hint: 'Describe your AI task, target quality criteria, and failure modes.'
---

# Netra Evaluation Setup

Use this skill to build reliable evaluation pipelines in Netra that catch regressions and measure quality over time.

## When To Use
- You need repeatable quality checks for prompts, models, or agent logic.
- You want both subjective and deterministic scoring.
- You need a baseline before deploying AI changes.

## Evaluation Design Framework
1. Define quality dimensions.
2. Build or import a dataset.
3. Select evaluator types per dimension.
4. Map variables carefully.
5. Run test suites and inspect failures.
6. Iterate prompt, policy, or tool logic.

## Choosing Evaluator Types
- Use LLM-as-Judge for subjective criteria:
  - correctness, relevance, helpfulness, hallucination checks, safety.
- Use Code Evaluators for deterministic criteria:
  - JSON schema validity, regex formats, strict business rules.

## Procedure
1. Create a dataset from production traces when possible.
2. Add edge cases and negative tests manually.
3. Select evaluators from Library or My Evaluators.
4. Configure pass thresholds and scoring output.
5. Map evaluator variables to:
   - dataset fields (`input`, `expected_output`),
   - agent response,
   - execution metadata (latency/tokens/model).
6. Test each evaluator in Playground before saving.
7. Run evaluation test suite via SDK and review Test Runs.
8. Track pass rate, latency, and cost across versions.

## Python Evaluation Run Pattern
```python
from netra import Netra

Netra.init(app_name="my-app")

def task_fn(input_data):
    # Implement your app logic and return the generated output string.
    return my_agent_response(input_data)


dataset = Netra.evaluation.get_dataset(dataset_id="YOUR_DATASET_ID")
result = Netra.evaluation.run_test_suite(
    name="baseline-eval",
    data=dataset,
    task=task_fn,
)

print(result)
```

## TypeScript Evaluation Run Pattern
```typescript
import { Netra } from "netra-sdk-js";
import OpenAI from "openai";

const client = new Netra({ apiKey: process.env.NETRA_API_KEY! });
const openai = new OpenAI();

async function taskFn(inputData: any): Promise<string> {
  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: String(inputData) }],
  });
  return response.choices[0].message.content || "";
}

const dataset = await client.evaluation.getDataset("dataset-123");

if (dataset) {
  const result = await client.evaluation.runTestSuite(
    "baseline-eval",
    dataset,
    taskFn,
    ["correctness", "relevance"],
    10
  );
  console.log(result?.runId);
}
```

## Quality Checklist
- Dataset includes realistic production examples.
- Dataset includes edge cases and refusal scenarios.
- Evaluator prompt scales are explicit (for example 0-10 with clear rubric).
- Thresholds were calibrated on representative samples.
- Variable mappings are validated on at least 5 sample rows.
- Failures are triaged as model issue, prompt issue, tool issue, or evaluator issue.

## Common Pitfalls
- Over-relying on a single evaluator.
- Vague LLM-as-Judge prompts with ambiguous scoring.
- Incorrect variable mapping causing false failures.
- Treating pass/fail as enough without reviewing traces.

## References
- https://docs.getnetra.ai/quick-start/QuickStart_Evals
- https://docs.getnetra.ai/Evaluation/Evaluation-overview
- https://docs.getnetra.ai/Evaluation/Datasets
- https://docs.getnetra.ai/Evaluation/Evaluators
- https://docs.getnetra.ai/Evaluation/TestRuns
- https://docs.getnetra.ai/sdk-reference/evaluation/typescript
