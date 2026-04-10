---
name: netra-mcp-evaluations-single-turn
description: End-to-end single-turn evaluation workflow in Netra MCP from provider selection to test run details.
---

# Netra MCP Evaluations (Single-Turn)

Use this reference for a schema-correct single-turn evaluation flow using Netra MCP tools.

## End-To-End Flow

1. List provider configurations.
2. Create a single-turn dataset.
3. Add single-turn test cases.
4. List evaluators.
5. If project evaluators are missing (or you only see library evaluators), create evaluators in the Netra dashboard first.
6. Attach evaluators to dataset or test cases.
7. Execute a test run.
8. Fetch run results using test run id.

## Step 1: List Provider Configurations

Tool: `netra_list_provider_configs`

Purpose:
- Find valid `provider_id` and `model` values for dataset items.
- Confirm the provider/model is available for your use case.

Example:

```json
{}
```

## Step 2: Create A Single-Turn Dataset

Tool: `netra_create_dataset`

Required choices:
- `turnType`: `single`
- `datasetType`: usually `text`

Example:

```json
{
  "name": "support-quality-single-turn",
  "turnType": "single",
  "datasetType": "text",
  "tags": ["support", "regression"]
}
```

## Step 3: Add Single-Turn Test Cases

Tool: `netra_add_dataset_test_case`

Important:
- For single-turn datasets, `input` is required.
- `providerConfig` is required in practice. Always pass `provider_id` and `model` from Step 1.

Example:

```json
{
  "datasetId": "<dataset-id>",
  "input": "User asks for a refund after 45 days",
  "expectedOutput": "Assistant explains policy and offers next best options",
  "contextData": {
    "policy": "30-day refund window",
    "region": "US"
  },
  "providerConfig": {
    "provider_id": "<provider-id>",
    "model": "<model-name>"
  },
  "tags": ["refund"]
}
```

## Step 4: List Evaluators

Tool: `netra_list_evaluators`

Purpose:
- Discover project evaluators available for attachment.
- Inspect available library evaluators in `libraryData`.

Example:

```json
{
  "turnType": "single",
  "page": 1,
  "limit": 20
}
```

Decision rule:
- If project evaluator results are empty and only `libraryData` has entries, stop and instruct the user to create evaluators in the Netra dashboard before continuing.

Suggested instruction to user:
- "No project evaluators are available yet. Please create/select evaluators in the Netra dashboard for this project, then rerun `netra_list_evaluators`."

## Step 5: Attach Evaluators

Tool: `netra_add_evaluator`

Options:
- Attach at dataset level (`targetType: dataset`).
- Attach at test-case level (`targetType: test_case`, requires `datasetItemId`).

Example (dataset-level):

```json
{
  "targetType": "dataset",
  "datasetId": "<dataset-id>",
  "evaluatorId": "<evaluator-id>",
  "isActive": true
}
```

Example (test-case-level):

```json
{
  "targetType": "test_case",
  "datasetId": "<dataset-id>",
  "datasetItemId": "<dataset-item-id>",
  "evaluatorId": "<evaluator-id>"
}
```

## Step 6: Execute Test Run

Use your workspace test-run execution tool (commonly named `netra_execute_test_run`) to run the dataset against the target system.

Expected output:
- A `testRunId` used for retrieval and analysis.

## Step 7: Get Test Run Details

Tool: `netra_get_test_run_details`

Required:
- `testRunId`

Optional:
- `page`, `limit`, `filters`

Example:

```json
{
  "testRunId": "<test-run-id>",
  "page": 1,
  "limit": 20
}
```

## Practical Checks

1. Always resolve `provider_id` and `model` before adding test cases.
2. For single-turn cases, verify `input` is present for every item.
3. Treat missing project evaluators as a setup blocker, not a runtime failure.
4. Attach evaluators before running test executions to avoid incomplete scoring.
5. Store and reuse `testRunId` for iterative detail queries.
