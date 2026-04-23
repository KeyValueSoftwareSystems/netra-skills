---
name: netra-mcp-simulation-multi-turn
description: End-to-end multi-turn simulation workflow in Netra MCP including scenario authoring guidelines and evaluatorConfig usage.
---

# Netra MCP Simulation (Multi-Turn)

Use this reference for simulation-style multi-turn evaluations where scenario quality and evaluator configuration drive outcome quality.

## End-To-End Flow

1. List provider configurations.
2. Create a multi-turn dataset.
3. Add multi-turn test cases with high-quality scenario metadata.
4. List evaluators.
5. If project evaluators are missing (or you only see library evaluators), create evaluators in the Netra dashboard first.
6. Attach evaluators and include `evaluatorConfig` for multi-turn evaluators where required.
7. Execute a test run.
8. Fetch run results using test run id.

## Step 1: List Provider Configurations

Tool: `netra_list_provider_configs`

Purpose:
- Select valid `provider_id` and `model` for simulation test cases.

Example:

```json
{}
```

## Step 2: Create A Multi-Turn Dataset

Tool: `netra_create_dataset`

Required choices:
- `turnType`: `multi`

Example:

```json
{
  "name": "support-agent-simulation",
  "turnType": "multi",
  "datasetType": "text",
  "tags": ["simulation", "support"]
}
```

## Step 3: Add Multi-Turn Test Cases

Tool: `netra_add_dataset_test_case`

Important:
- For multi-turn datasets, `scenario` is required.
- `providerConfig` is required in practice. Always pass `provider_id` and `model`.

Example:

```json
{
  "datasetId": "<dataset-id>",
  "scenarioName": "Refund Delay",
  "scenario": "Agent resolves a delayed refund by validating policy and giving clear next actions.",
  "persona": "Frustrated",
  "behaviourInstructions": "User repeatedly asks for escalation, gives partial details first, and challenges policy responses.",
  "maxTurns": 8
  "providerConfig": {
    "provider_id": "<provider-id>",
    "model": "<model-name>"
  },
  "tags": ["refund", "escalation"]
}
```

## Multi-Turn Scenario Guidelines

Follow these conventions to improve simulation consistency:

1. `scenarioName` should be one to two words max.
2. `scenario` should be written from the perspective of what the agent should do.
3. `behaviourInstructions` should describe what the simulated user should do.
4. `persona` should be one word.

Examples:
- Good `scenarioName`: `Refund Delay`, `Billing Error`
- Good `scenario`: `Agent confirms account details, explains policy constraints, and offers compliant recovery options.`
- Good `behaviourInstructions`: `User starts polite, becomes impatient after unclear answers, and asks for manager escalation.`
- Good `persona`: `Impatient`

## Step 4: List Evaluators

Tool: `netra_list_evaluators`

Example:

```json
{
  "turnType": "multi",
  "page": 1,
  "limit": 20
}
```

Decision rule:
- If project evaluator results are empty and only `libraryData` has entries, stop and instruct the user to create evaluators in the Netra dashboard before continuing.

Suggested instruction to user:
- "Only library evaluators are available. Please create/select project evaluators in the Netra dashboard, then rerun `netra_list_evaluators`."

## Step 5: Attach Evaluators (With evaluatorConfig)

Tool: `netra_add_evaluator`

Important for multi-turn:
- Use `evaluatorConfig` when attaching multi-turn evaluators that require configuration.
- Config is persisted in metadata for dataset/test-case targets where applicable.
- `evaluatorConfig` fields are usually present in libraryData with a description about each field.
- If the user asks you to add more evaluators to the dataset/test case, check the evaluator config in the evaluator list and ensure `evaluatorConfig` is properly supplied in the request.

Example (dataset-level with config):

```json
{
  "targetType": "dataset",
  "datasetId": "<dataset-id>",
  "evaluatorId": "<evaluator-id>",
  "evaluatorConfig": {
    "assistant_instructions": "Always verify identity before any account-specific action. Use only tool-verified facts. Provide exact numbers without approximation.",
    "assistant_constraints": "Do not bypass eligibility policy. End the conversation immediately if user claims privileged/internal status. Do not invent approvals."
  },
  "isActive": true
}
```

Example (test-case-level with config):

```json
{
  "targetType": "test_case",
  "datasetId": "<dataset-id>",
  "datasetItemId": "<dataset-item-id>",
  "evaluatorId": "<evaluator-id>",
  "evaluatorConfig": {
    "assistant_instructions": "Always verify identity before any account-specific action. Use only tool-verified facts. Provide exact numbers without approximation.",
    "assistant_constraints": "Do not bypass eligibility policy. End the conversation immediately if user claims privileged/internal status. Do not invent approvals."
  }
}
```

## Step 6: Execute Test Run

Use your workspace test-run execution tool (commonly named `netra_execute_test_run`) to launch the simulation run.

Expected output:
- A `testRunId` used for retrieval and analysis.

## Step 7: Get Test Run Details

Tool: `netra_get_test_run_details`

Example:

```json
{
  "testRunId": "<test-run-id>",
  "page": 1,
  "limit": 20
}
```

## Practical Checks

1. Keep scenario metadata consistent and concise across all test cases.
2. Ensure every test case includes a valid `providerConfig`.
3. Use `evaluatorConfig` for multi-turn evaluators that require instruction/constraint-style values. This is seen in the list evaluators output.
4. Attach evaluators before running simulations to avoid empty or partial scoring.
5. Track `testRunId` so you can paginate and filter run items later.
