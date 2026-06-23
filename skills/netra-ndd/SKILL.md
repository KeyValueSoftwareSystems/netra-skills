---
name: ndd-generate
description: >-
  Generate evaluation datasets from QA/dev specification documents. Analyzes
  spec sheets, discovers agent span structure from code, recommends evaluators,
  auto-configures variable mappings, and produces an editable plan for review.
  Use when the user says: NDD generate, create test suite, generate dataset
  from spec, create evaluation plan, generate NDD plan, or provides a QA
  specification document.
---

# NDD Generate — Specification-Driven Dataset Planning

This skill takes a QA/dev specification document and produces a complete,
editable evaluation plan — dataset items, evaluators, variable mappings,
and pass criteria — then creates everything in Netra on user approval.

## Workflow

Execute these phases in order. Complete each phase before moving to the next.

### Phase 1: Ingest the Specification

1. Read the specification document provided by the user (file path, pasted text, or URL).
2. Extract every testable requirement. For each, identify:
   - **Scenario name** — a concise, human-readable name describing the behavior being validated
   - **Category** — happy path, negative, edge case, safety, performance
   - **Input** — the user message or trigger that tests this scenario
   - **Expected behavior** — what the correct agent response or action should be

#### Scenario Naming Guidelines

Generate meaningful scenario names based on the behavior being tested.

Good examples:

- User Login Success
- Password Reset With Valid Email
- Product Search Returns Relevant Results
- Reject Prompt Injection Attempt
- Handle Empty User Query
- Customer Requests Refund
- Retrieve Account Balance
- Multi-Step Travel Planning Conversation

Avoid generic names such as:

- Scenario 1
- Test Case 1
- Item 3
- Validation Test
- Happy Path Test
3. Propose additional test cases the spec may have missed:
   - Boundary conditions not explicitly listed
   - Common adversarial inputs (prompt injection, jailbreak attempts)
   - Performance edge cases (very long input, empty input, special characters)
   - Multi-step reasoning scenarios if the agent supports them
4. Present the full list and ask the user to confirm before proceeding.

### Phase 2: Discover Agent Span Structure

Build a **span map** — the list of named spans the agent produces at runtime.

**Primary method — code analysis:**

Search the agent codebase for instrumentation patterns. Use Grep and Read tools.

```
Patterns to search for:

Python Netra decorators:
  @workflow, @agent, @task, @span          → span name from name= parameter
  Netra.start_span("name", ...)            → span name from first argument

Python tool definitions:
  @tool def func_name(...)                 → span name = function name
  class MyTool(BaseTool): name = "..."     → span name from class attribute

TypeScript Netra decorators:
  @Workflow(), @Agent(), @Task()           → span name from decorator argument
  netra.startSpan("name", ...)             → span name from first argument

Framework-specific:
  CrewAI Agent/Task definitions            → span name from name field
  LangGraph node definitions               → span name from node name
  Claude Agent SDK tool definitions        → span name from tool name
```

For each span found, record:
- **name** — the span name as it will appear in traces
- **type** — workflow, agent, task, tool, or generation
- **output description** — what the function returns (infer from code)
- **position in flow** — call order relative to other spans

**Fallback method — sample trace:**

If code analysis yields insufficient results (no decorators found, dynamic span names):

1. Ask the user: "I couldn't find enough span definitions in the code. Has this agent been run at least once with Netra instrumented?"
2. If yes, use `netra_query_traces` to find a recent trace, then `netra_get_trace_by_id` to fetch its spans.
3. Build the span map from the actual trace data — span names, parent-child relationships, input/output.

**Output of this phase:**

```
Agent Flow:
  {root-span-name} ({type})
  ├── {child-span-1} ({type}) → {output description}
  ├── {child-span-2} ({type}) → {output description}
  │   └── {nested-span} ({type}) → {output description}
  └── {child-span-3} ({type}) → {output description}
```

Present to user and confirm span names are accurate.

### Phase 3: Select Evaluators and Configure Mappings

For each test case from Phase 1, determine which evaluators to attach and how to wire their variables.

**Step 3a: Get available evaluators**

Try the MCP tool first. If unavailable, use the embedded reference.

```
Try:    netra_get_evaluator_library (all categories)
        netra_list_evaluators (custom evaluators in the project)

Fallback: Read references/evaluator-library.md for the full catalog
```

**Step 3b: Match evaluators to test cases**

Apply these mapping rules:

| Scenario category | Recommended evaluators |
|---|---|
| Happy path — correct response expected | Answer Relevance, Answer Correctness or Semantic Similarity |
| Happy path — correct action expected | Goal Accuracy, Tool Correctness |
| Negative — should reject/error gracefully | Answer Relevance, Conciseness |
| Edge case — boundary behavior | Answer Correctness, Faithfulness |
| Safety — prompt injection, jailbreak | Toxicity, Topic Adherence |
| Performance — latency/cost bounds | Latency, Cost, Token Usage |
| RAG — retrieval quality matters | Context Relevance, Context Precision, Faithfulness, Hallucination |

These are starting recommendations. Always consider the specific scenario semantics — an evaluator makes sense only if its purpose aligns with what the test case validates.

**Step 3c: Configure variable mappings**

For each evaluator attached to a test case, map every required variable to a data source.

Decision logic for the `response` / `actual_output` variable:

```
IF the evaluator checks the final user-facing answer
  → source: taskOutput
  → expression: "taskOutput"

IF the evaluator checks a specific intermediate step
  → source: span output
  → find the span from Phase 2 whose output is relevant
  → expression: "spans[?name=='{spanName}'] | [0].output"

IF the evaluator checks tool usage
  → source: trace tools
  → expression: "trace.tools"
```

Decision logic for the `reference` / `expected` variable:

```
IF ground truth is available in the dataset item
  → source: expectedOutput
  → expression: "expectedOutput"

IF reference comes from a specific span
  → expression: "spans[?name=='{spanName}'] | [0].output"
```

Decision logic for `context` (RAG evaluators):

```
→ source: retriever span output
→ expression: "spans[?name=='{retriever-span}'] | [0].output"
```

Decision logic for performance evaluators:

```
actual_latency → expression: "trace.latency"
actual_cost    → expression: "trace.cost"
actual_tokens  → expression: "trace.tokens"
expected_*     → source: literal value from the spec
```

When you cannot confidently determine a mapping, mark it as `⚠️ NEEDS CONFIGURATION` in the plan and explain why.

### Phase 3d: Set pass criteria and model

- Use the evaluator's default pass criteria unless the spec states otherwise.
- If the user specifies a model preference, apply it to all LLM evaluators shown in the plan.

The plan should display the model configuration that will be used if evaluator creation is required.

Provider/model resolution occurs during evaluator creation, not during evaluator recommendation.

#### Item-Level Provider Config

Multi-turn dataset items require a `providerConfig` because simulations execute through a model-backed task implementation.

```json
{
  "provider_id": "{providerConfigurationId}",
  "model": "{model}"
}
```

Use `netra_get_default_llm_configuration` to retrieve the organization's default provider configuration.

Rules:

- For **multi-turn datasets**, every dataset item must include a `providerConfig`.
- If the specification does not define a model, use the organization's default provider configuration.
- If a scenario explicitly requires a different model, override the default by setting the item's `providerConfig` accordingly.
- For **single-turn datasets**, `providerConfig` is optional and should only be supplied when an item requires a model different from the evaluator default.

### Phase 3e: Determine Execution Strategy

Before generating the final plan, determine whether the specification describes a single-turn evaluation or a multi-turn simulation.

#### Single-Turn Evaluations

If the specification describes a single request-response interaction:

- Set Turn Type = `single`
- Use `Netra.evaluation.run_test_suite()` for execution

Never invoke the agent directly for evaluation execution.

Rationale:

- Trace IDs are automatically captured
- Evaluators can access trace and span outputs
- Span-based variable mappings function correctly
- Evaluation results remain linked to execution traces

Execution example:

```python
Netra.evaluation.run_test_suite(
    dataset_id="dataset-123",
    ...
)
```

#### Multi-Turn Simulations

If the specification describes a conversation, workflow, support interaction, troubleshooting flow, or any multi-step exchange:

- Set Turn Type = `multi`
- Set Eval Type = `session` (auto-inferred by `netra_create_evaluator` when `turnType: "multi"` — no need to specify unless overriding)
- Use `Netra.simulation.run_simulation()` for execution

Never invoke the agent directly for simulation execution.

The user must provide a task implementation compatible with Netra Simulation.

Example:

```python
class MyAgentTask(BaseTask):
    def __init__(self, agent):
        self.agent = agent

    def run(
        self,
        message: str,
        session_id: str | None = None,
        files: list[ProcessedFile] | None = None,
    ) -> TaskResult:
        response = self.agent.chat(
            message,
            session_id=session_id,
            files=files,
        )

        return TaskResult(
            message=response.text,
            session_id=session_id or "default",
        )
```

Execution example:

```python
result = Netra.simulation.run_simulation(
    name="Customer Support Simulation",
    dataset_id="dataset-123",
    task=MyAgentTask(my_agent),
    context={"environment": "staging"},
    max_concurrency=5,
)
```

This ensures:

- Conversation state is maintained
- Session IDs are tracked
- Simulation traces are generated
- Evaluators can access conversation history and span outputs

#### Turn Type Detection Rules

Determine execution mode from the specification:

| Spec Pattern | Turn Type |
|-------------|-----------|
| Single request → single response | single |
| One-shot tool execution | single |
| QA validation | single |
| Customer support conversation | multi |
| Sales conversation | multi |
| Troubleshooting workflow | multi |
| Multi-step assistant interaction | multi |
| Roleplay simulation | multi |

If uncertain, ask the user before generating the plan.

### Phase 4: Generate the Plan

Produce the plan in the exact format below. This is the primary output the user will review.

---

**PLAN FORMAT — output this exactly, filling in the generated values:**

```markdown
# NDD Evaluation Plan

## Dataset Configuration

| Field | Value |
|---|---|
| **Name** | {generated-dataset-name} |
| **Description** | {generated-description} |
| **Turn Type** | {single | multi} |
| **Execution Method** | {Netra.evaluation.run_test_suite | Netra.simulation.run_simulation} |
| **Total Items** | {count} |

### Execution Strategy

Single-turn datasets:
- Execute using `Netra.evaluation.run_test_suite()`

Multi-turn datasets:
- Execute using `Netra.simulation.run_simulation()`

If Turn Type = `multi`, include:

```text
Required Task Implementation:
⚠️ User must provide a BaseTask implementation compatible with Netra Simulation.
```
---

## Items

### Item 1: {scenario-name}

**Category:** {happy-path | negative | edge-case | safety | performance}

| Field | Value |
|---|---|
| **Input** | {the test input / user message} |
| **Expected Output** | {the expected correct response or behavior} |
| **Provider Config** | {none — use evaluator default \| provider_id: {id}, model: {model}} |

**Evaluators:**

#### Evaluator: {evaluator-name}

| Config | Value |
|---|---|
| Type | {llm-as-judge / code / json / ...} |
| Model | {org-default / user-specified} |
| Pass Criteria | {operator} {threshold} |

| Variable | Source | Expression |
|---|---|---|
| {var-name} | {Agent Response / Span: {span-name} / Dataset: expectedOutput / Trace: {metric} / Value} | `{jmespath-expression}` |
| {var-name} | ... | `...` |

#### Evaluator: {evaluator-name-2}

...

---

### Item 2: {scenario-name}

...

---

## Summary

| Category | Count |
|---|---|
| Happy Path | {n} |
| Negative | {n} |
| Edge Case | {n} |
| Safety | {n} |
| Performance | {n} |
| **Total** | {n} |

| Evaluators Used | Count |
|---|---|
| {evaluator-name} | {n items} |
| ... | ... |
```

---

After outputting the plan, ask the user:

> Review the plan above. You can ask me to:
> - Add, remove, or modify any item
> - Change evaluators for a specific item
> - Adjust variable mappings (where input/output/reference comes from)
> - Change pass criteria or model
> - Edit the dataset name or description
>
> When you're satisfied, say **"proceed"** and I'll create everything in Netra.

### Phase 5: Handle User Edits

When the user requests changes:
- Apply the edit to the plan.
- Re-output only the changed section (not the full plan) so the user can verify.
- Ask if there are more changes or if they want to proceed.

### Phase 6: Create in Netra

When the user says "proceed", "create", "go ahead", or similar:

Execute these MCP calls in order:

```
1. netra_create_dataset
  → name
  → description
  → turnType: {single | multi}
  → capture datasetId from response

2. If attaching any library evaluator that does not already exist in My Evaluators:

   a. Resolve the organization's default LLM configuration:

      netra_get_default_llm_configuration

      Expected response:
      {
        "provider": "openai",
        "model": "gpt-5",
        "providerConfigurationId": "provider-config-id"
      }

      If null is returned, block creation:
      ⚠️ Evaluator creation blocked. No default provider configured.
      Do not guess provider or model values.

   b. netra_create_evaluator
      → name
      → libraryEvaluatorId
      → turnType: "single" | "multi"
      → evalType: omit — auto-inferred ("turn" for single, "session" for multi)
      → config is auto-resolved from org default when omitted
      → capture evaluatorId from response

3. For each item:
   netra_create_dataset_item
   → datasetId, input, expectedOutput
   → metadata (for multi-turn: scenario, persona, max_turns, behaviour_instructions)
   → providerConfig

  For multi-turn datasets:
    - metadata
     {
       "scenario": "{scenarioName}",
       "persona": "...",
       "max_turns": ...,
       "behaviour_instructions": "..."
     }
    - Always include providerConfig.
    - If no model is specified in the plan, use the organization's default provider configuration from netra_get_default_llm_configuration.
    - Example:

    ```json
    {
      "provider_id": "{providerConfigurationId}",
      "model": "{model}"
    }
    ```

  For single-turn datasets:
    - providerConfig is optional.
    - Include only when the item requires a model different from the evaluator default.
   → evaluators (optional): item-level evaluator overrides with variableMapping

4. Map evaluators to the dataset:
   netra_map_evaluator_to_dataset
   → datasetId, evaluatorId, variableMapping
```

If MCP tools for creation are not yet available, output a message:

> The dataset creation MCP tools are not registered yet. Here's what would be created:
> - Dataset: {name} with {n} items
> - {n} evaluator mappings
>
> You can create this manually in the Netra dashboard, or wait for the MCP tools to be available.

After successful creation, output:

```
✅ Dataset created: {name} ({datasetId})
   Items: {count} created
   Evaluator mappings: {count} configured
   
   View in Netra: https://app.getnetra.ai/datasets/{datasetId}
   
   Next: Run `netra run {dataset-name}` to execute the test suite.
```

## Important Rules

1. **Never skip the preview.** Always show the full plan before creating anything.
2. **Never guess span names.** If code analysis doesn't find clear span definitions, use the fallback method or ask the user.
3. **Mark uncertainty explicitly.** Use `⚠️ NEEDS CONFIGURATION` for any mapping you're not confident about.
4. **Respect the spec.** If the QA spec defines specific pass thresholds, use those instead of defaults.
5. **Include the reasoning.** When recommending an evaluator, briefly explain why it fits the test case.
6. **Don't over-evaluate.** Not every item needs 5 evaluators. Match evaluator count to scenario complexity — simple happy paths may need 1-2, complex edge cases may need 3-4.
7. **Prefer span output over taskOutput** when the evaluator checks an intermediate step. Use `taskOutput` only when evaluating the final user-facing response.
8. Never execute the agent directly for evaluation runs.
   - Single-turn evaluations must use `Netra.evaluation.run_test_suite()`.
   - Multi-turn simulations must use `Netra.simulation.run_simulation()`.

9. Trace IDs are required for evaluator execution.
   Any execution strategy that bypasses Netra Evaluation or Simulation APIs is invalid.

10. Multi-turn simulations require a task implementation that extends `BaseTask`.
    The generated plan must explicitly mention this requirement.

11. When generating execution examples, use the official Netra SDK execution APIs.
    Do not generate examples that directly invoke the agent for evaluation execution.

12. Variable mappings that reference spans are only valid when execution occurs through Netra Evaluation or Simulation APIs that generate trace data.

13. For multi-turn simulations, include a task implementation example using:
    - `BaseTask`
    - `TaskResult`
    - `session_id`
    - `Netra.simulation.run_simulation()`

14. Resolve the organization's default provider/model only when creating a project evaluator from a library evaluator.

15. Do not fetch provider/model information during evaluator recommendation unless evaluator creation is required.

16. Library evaluators attached to a dataset must first be materialized into My Evaluators before mappings are created.

17. Every generated scenario must have a meaningful, human-readable name.

18. Scenario names should describe the behavior being validated rather than the test category.

Good:
- Retrieve Customer Order Status
- Reject Unauthorized Data Access
- Handle Empty Input

Bad:
- Scenario 1
- Test Case 3
- Happy Path
- Negative Test

19. Scenario names should be unique within the dataset whenever possible.

20. When creating evaluators for multi-turn datasets, `evalType` is automatically set to `"session"` by `netra_create_evaluator` when `turnType: "multi"` is passed. Do not specify `evalType` unless you explicitly need `"turn"`-level evaluation within a multi-turn dataset.

21. Item-level `providerConfig` (`{ provider_id, model }`) overrides the evaluator's default model for that specific item only. Use it only when a specific scenario requires a different model. For uniform model usage across all items, configure the model at the evaluator level — do not set `providerConfig` on every item.

22. Multi-turn dataset items must always be created with a `providerConfig`.

    If the specification does not provide one, resolve the organization's default provider configuration using `netra_get_default_llm_configuration` and attach it to every multi-turn dataset item during creation.

23. Before creating a multi-turn dataset item, the system must ensure a valid provider configuration exists.
    
    If `netra_get_default_llm_configuration` returns null and no item-level provider configuration is specified:
    
    ```
    ⚠️ Multi-turn dataset creation blocked.
    No default provider configured and no item-level providerConfig supplied.
    ```
    
    Do not create the dataset items until a valid provider configuration is available.

## Reference

For the full evaluator catalog with variables and thresholds, see [evaluator-library.md](references/evaluator-library.md).
