---
name: netra-evaluation-setup
description: >-
  Generate evaluation datasets from QA/dev specification documents. Analyzes
  spec sheets, discovers agent span structure from code, recommends evaluators,
  auto-configures variable mappings, and produces an editable plan for review.
  Use when the user says: NDD generate, create test suite, generate dataset
  from spec, create evaluation plan, generate NDD plan, or provides a QA
  specification document.
---

# Netra Evaluation Setup — Specification-Driven Dataset Planning

This skill takes a QA/dev specification document and produces a complete,
editable evaluation plan — dataset items, evaluators, variable mappings,
and pass criteria — then creates everything in Netra on user approval.

## Workflow

Execute these phases in order. Complete each phase before moving to the next.

### Phase 0: Verify Provider Configuration

Before starting any planning or creation work, verify that the organization has a default provider configured.

Call `netra_get_default_llm_configuration` via MCP.

- If a valid configuration is returned (containing `provider`, `model`, and `providerConfigurationId`), proceed to Phase 1.
- If null or an error is returned, **stop immediately** and inform the user:

```
⚠️ No default provider configured.

Evaluation and dataset creation require a default LLM provider in your organization.
Please configure a default provider in the Netra dashboard before proceeding:

  Settings → Providers → Set Default

Once configured, run this skill again.
```

Do not proceed to any subsequent phase until a valid default provider is confirmed.

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

**Step 3c: Identify configurable variables**

Some evaluators require **user-provided configuration** that cannot be auto-resolved from
conversation context or metadata. These are defined as `configurableVariables` in the evaluator
library. If a configurable variable is left empty, the evaluator will fail or produce unreliable
scores (e.g., "Cannot be evaluated" with a degraded score).

Evaluators with configurable variables:

| Evaluator | Variable | Type | What to provide |
|---|---|---|---|
| **Guideline Adherence** | `assistant_instructions` | string | The full system prompt or instructions given to the AI agent — what it must do, how it should behave, security rules, escalation guidelines |
| **Guideline Adherence** | `assistant_constraints` | string | Constraints the AI agent must respect — things it must NOT do, boundaries, prohibited actions. Also auto-resolved from `metadata.assistant_constraints` but configurable value takes precedence |
| **Factual Accuracy** (multi-turn) | `reference_facts` | json | Facts the agent should communicate correctly — product details, policies, prices, dates. Also auto-resolved from `metadata.reference_facts` but configurable value takes precedence |

Variable resolution priority for session evaluators:
1. `evaluatorConfigs[].configuredValues` (highest priority — user-provided)
2. Session metadata auto-resolution (e.g., `metadata.assistant_constraints`, `metadata.reference_facts`)
3. Empty string (fallback — causes evaluation failures)

`assistant_instructions` has NO auto-resolution path — it MUST be provided via `evaluatorConfigs`.

When an evaluator with configurable variables is recommended:
- Ask the user to provide the values
- Include them in the plan under a "Configurable Variables" section for each evaluator
- If the user cannot provide them yet, mark them as `⚠️ NEEDS CONFIGURATION` in the plan

**Step 3d: Configure variable mappings**

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

### Phase 3e: Set pass criteria and model

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

### Phase 3f: Determine Execution Strategy

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
- Set Eval Type = `session` — MUST be explicitly passed to `netra_create_evaluator`
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

For single-turn items:

| Field | Value |
|---|---|
| **Input** | {the test input / user message} |
| **Expected Output** | {the expected correct response or behavior} |
| **Provider Config** | {none — use evaluator default \| provider_id: {id}, model: {model}} |

For multi-turn items:

| Field | Value |
|---|---|
| **Input** | `{}` (empty object — simulation engine generates messages) |
| **Expected Output** | `null` |
| **ScenarioName** | {concise display name for the scenario} |
| **Scenario** | {the actual goal/scenario description that evaluators use to judge success} |
| **Persona** | {who the simulated user is} |
| **Max Turns** | {number} |
| **Behaviour Instructions** | {step-by-step script for simulated user} |
| **Provider Config** | provider_id: {id}, model: {model} |

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

If the evaluator has configurable variables, include:

**Configurable Variables (user-provided):**

| Variable | Value |
|---|---|
| {var-name} | {user-provided value or ⚠️ NEEDS CONFIGURATION} |

Example for Guideline Adherence:

**Configurable Variables (user-provided):**

| Variable | Value |
|---|---|
| `assistant_instructions` | You are a helpful customer support agent. Always greet the user professionally... |
| `assistant_constraints` | Must NOT ask for credit card details. Must escalate billing issues to human agents... |

Example for Factual Accuracy (multi-turn):

**Configurable Variables (user-provided):**

| Variable | Value |
|---|---|
| `reference_facts` | {"return_policy": "14 days", "shipping_time": "3-5 business days", ...} |

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
      → type: MUST match the library evaluator's type exactly.
        Check the library evaluator's `type` field (e.g. "llm-as-judge", "tool_accuracy",
        "regex", "code") and pass the SAME type when creating the project evaluator.
        If the library evaluator is `tool_accuracy` (rule-based), create it as `tool_accuracy`.
        If the library evaluator is `llm-as-judge`, create it as `llm-as-judge`.
        Do NOT rely on the API to inherit the type from `libraryEvaluatorId` — it may
        default to `llm-as-judge` regardless, causing rule-based evaluators to fail with
        "Missing required config: model, provider_id, or prompt" because the engine
        incorrectly treats them as LLM evaluators.
        If type mismatch is suspected, create the evaluator as a custom evaluator
        with explicit `type` and omit `libraryEvaluatorId`.
      → turnType: "single" | "multi"
      → evalType: MUST be explicitly set
        - For single-turn evaluators: pass evalType: "turn"
        - For multi-turn evaluators: pass evalType: "session"
        Do NOT omit evalType — the API does NOT auto-infer it from turnType.
        Omitting evalType for multi-turn evaluators causes it to default to "turn",
        which means evaluators run per-message instead of per-session, resulting in
        evaluation scores showing as "not available" for simulation runs.
      → config: Must match the evaluator type.
        - For LLM-based evaluators (`llm-as-judge`): ALWAYS pass explicitly using
          the resolved default from step 2a:
          {
            "model": "{model}",
            "provider_id": "{providerConfigurationId}"
          }
          Do NOT omit config or rely on auto-resolution — the evaluation engine
          requires `provider_id` (snake_case) in the config to execute LLM evaluators.
          Omitting config or using the wrong field name causes evaluators to fail
          at runtime with "Missing required config: model, provider_id, or prompt".
        - For rule-based evaluators (`tool_accuracy`): pass the evaluator-specific
          config (e.g. `{"matchType": "partial"}`). Do NOT include model or provider_id.
        - For other code/regex evaluators: pass their specific config as defined in
          the library evaluator.
      → capture evaluatorId from response

   c. Verify the created evaluator config:

      After creation, inspect the response `config` object. Confirm it contains
      `provider_id` (snake_case). If the response only shows `providerId` (camelCase)
      without `provider_id`, the evaluator will fail at eval time.

      Valid response config:
      {
        "model": "gpt-4o",
        "provider_id": "dcffa74c-..."
      }

      If verification fails, recreate the evaluator with explicit config.

3. For each item:
   netra_create_dataset_item
   → datasetId, input, expectedOutput
   → metadata (for multi-turn: scenarioName, persona, max_turns, behaviour_instructions)
   → providerConfig

  For multi-turn datasets:
    - input: MUST be an empty object `{}`
      The simulation engine generates the first user message from
      `metadata.behaviour_instructions`, NOT from the `input` field.
      If `input` is set to a string, the scenario name will appear empty
      in the Netra dashboard because the UI reads the scenario identifier
      from `metadata.scenarioName`, and a string `input` interferes with this.
    - expectedOutput: set to `null` or omit
      The simulation evaluates the conversation as a whole via session-level
      evaluators, not against a single expected output.
    - metadata: REQUIRED — this is the primary configuration for the simulation item
     {
       "scenarioName": "{scenarioName}",
       "scenario": "{scenario}",
       "persona": "...",
       "max_turns": ...,
       "behaviour_instructions": "..."
     }
      - `scenarioName`: The scenario display name shown in the Netra dashboard.
        A concise, human-readable label for the test case (e.g. "Iterative Project Refinement").
      - `scenario`: The actual scenario description / goal that the simulation must achieve.
        This is the detailed goal text that session-level evaluators (Goal Fulfillment,
        Conversation Completeness) use to judge whether the agent succeeded. It describes
        WHAT the conversation should accomplish, not just a label.
        Example: "Get a complete engineering spec for a smart plant watering system,
        refining requirements through conversation."
      - `persona`: Who the simulated user pretends to be.
      - `max_turns`: Maximum conversation turns before the simulation stops.
      - `behaviour_instructions`: Step-by-step script for the simulated user.
        The simulation engine uses this to generate all user messages including
        the first one. Be specific about what the simulated user should say or
        ask in each turn.
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
    - input: a string containing the test message
    - expectedOutput: a string containing the expected agent response
    - providerConfig is optional.
    - Include only when the item requires a model different from the evaluator default.
   → evaluators (optional): item-level evaluator overrides with variableMapping

4. Map evaluators to the dataset:
   netra_map_evaluator_to_dataset
   → datasetId, evaluatorId, variableMapping

   For single-turn evaluators: pass the full variableMapping (user_query, response, etc.)
   For multi-turn (session) evaluators: pass an empty object `variableMapping: {}`
     Session evaluators auto-resolve their variables from conversation context.
     However, the `variableMapping` parameter MUST still be provided as `{}` —
     omitting it entirely causes "Cannot convert undefined or null to object" errors.

5. Set configurable variables for evaluators that require them:

   If any mapped evaluator has configurable variables (Guideline Adherence, Factual Accuracy multi-turn),
   include `evaluatorConfigs` in the dataset item metadata.

   `evaluatorConfigs` is an array of objects, each with:
   - `id`: the evaluator ID (the project evaluator ID created in step 2)
   - `configuredValues`: an object mapping variable names to their user-provided values

   Include `evaluatorConfigs` in the `metadata` field when calling `netra_create_dataset_item`
   or `netra_update_dataset_item`.

   Example for Guideline Adherence:

   ```json
   {
     "metadata": {
       "scenarioName": "Customer Support",
       "scenario": "...",
       "persona": "...",
       "max_turns": 5,
       "behaviour_instructions": "...",
       "evaluatorConfigs": [
         {
           "id": "{guideline-adherence-evaluator-id}",
           "configuredValues": {
             "assistant_instructions": "You are a helpful customer support agent. Always greet the user professionally, stay on topic, follow security rules...",
             "assistant_constraints": "Must NOT ask for credit card details. Must escalate billing issues to human agents..."
           }
         }
       ]
     }
   }
   ```

   Example for Factual Accuracy (multi-turn):

   ```json
   {
     "metadata": {
       "evaluatorConfigs": [
         {
           "id": "{factual-accuracy-evaluator-id}",
           "configuredValues": {
             "reference_facts": "{\"return_policy\": \"14 days\", \"shipping_time\": \"3-5 business days\"}"
           }
         }
       ]
     }
   }
   ```

   Multiple evaluators can be configured in the same `evaluatorConfigs` array:

   ```json
   {
     "metadata": {
       "evaluatorConfigs": [
         {
           "id": "{guideline-adherence-evaluator-id}",
           "configuredValues": {
             "assistant_instructions": "...",
             "assistant_constraints": "..."
           }
         },
         {
           "id": "{factual-accuracy-evaluator-id}",
           "configuredValues": {
             "reference_facts": "..."
           }
         }
       ]
     }
   }
   ```

   CRITICAL: If `evaluatorConfigs` is not set for evaluators that require configurable variables,
   those evaluators will receive empty values and produce unreliable scores. The Guideline Adherence
   evaluator will report "Cannot be evaluated" for AWARENESS and CONSISTENCY criteria, and
   the Factual Accuracy evaluator will default-pass because it sees no reference facts to check against.
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
    Once resolved, pass the config explicitly to `netra_create_evaluator` using `provider_id` (snake_case).

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

20. When creating evaluators for multi-turn datasets, ALWAYS explicitly pass `evalType: "session"`.

    The `netra_create_evaluator` API does NOT auto-infer `evalType` from `turnType`.
    If `evalType` is omitted, it defaults to `"turn"` regardless of the `turnType` value.
    This causes multi-turn simulation evaluators to fail silently — conversations are
    recorded correctly but evaluation scores show as "not available" because turn-level
    evaluators cannot score session-level simulation data.

    Correct (multi-turn):
    ```json
    {
      "turnType": "multi",
      "evalType": "session"
    }
    ```

    Wrong (will produce "not available" scores):
    ```json
    {
      "turnType": "multi"
    }
    ```

    Only use `evalType: "turn"` within a multi-turn dataset when you explicitly need
    per-message evaluation rather than whole-conversation evaluation.

21. Item-level `providerConfig` (`{ provider_id, model }`) overrides the evaluator's default model for that specific item only. Use it only when a specific scenario requires a different model. For uniform model usage across all items, configure the model at the evaluator level — do not set `providerConfig` on every item.

22. Multi-turn dataset items must always be created with:
    - `input`: an empty object `{}` (NOT a string — the simulation engine generates user messages from metadata)
    - `expectedOutput`: `null` or omitted (session-level evaluators score the full conversation)
    - `metadata`: containing `scenarioName`, `scenario`, `persona`, `max_turns`, and `behaviour_instructions`
      - `scenarioName`: concise display label (e.g. "Iterative Project Refinement")
      - `scenario`: the actual goal/scenario description that evaluators use to judge success
        (e.g. "Get a complete engineering spec for a smart plant watering system, refining requirements through conversation")
    - `providerConfig`: always required

    If the specification does not provide a providerConfig, resolve the organization's default provider configuration using `netra_get_default_llm_configuration` and attach it to every multi-turn dataset item during creation.

    Setting `input` to a string causes the scenario name to appear empty in the Netra
    dashboard because the UI reads the scenario display name from `metadata.scenarioName`,
    and a string input overrides the expected structure.

23. Before creating a multi-turn dataset item, the system must ensure a valid provider configuration exists.
    
    If `netra_get_default_llm_configuration` returns null and no item-level provider configuration is specified:
    
    ```
    ⚠️ Multi-turn dataset creation blocked.
    No default provider configured and no item-level providerConfig supplied.
    ```
    
    Do not create the dataset items until a valid provider configuration is available.

24. When creating LLM-based evaluators, ALWAYS pass explicit `config` with `model` and `provider_id` (snake_case).

    Do NOT rely on auto-resolution of provider config — the MCP tool stores the field as `providerId` (camelCase),
    but the evaluation engine requires `provider_id` (snake_case). This mismatch causes evaluators to silently fail
    at runtime with "Missing required config: model, provider_id, or prompt", resulting in null scores on all items.

    Correct:
    ```json
    {
      "model": "gpt-4o",
      "provider_id": "dcffa74c-..."
    }
    ```

    Wrong (will fail at eval time):
    ```json
    {
      "model": "gpt-4o",
      "providerId": "dcffa74c-..."
    }
    ```

25. MCP tools (`netra_create_test_run`, `netra_submit_test_run_item`) are for manual/programmatic
    result submission only. They require a `trace_id` from an existing trace. Do NOT use them as a
    substitute for `Netra.evaluation.run_test_suite()` or `Netra.simulation.run_simulation()` — those
    SDK methods handle agent execution, trace capture, and result submission automatically.

26. When using `Netra.evaluation.run_test_suite()` with a remote Netra dataset:

    ```python
    response = Netra.evaluation.get_dataset(dataset_id)
    dataset = Dataset(items=response.items)
    result = Netra.evaluation.run_test_suite(
        name="...",
        data=dataset,
        task=agent_task_function,
        max_concurrency=3,
    )
    ```

    The `data` parameter requires a `Dataset` object, not a dataset ID string.
    Use `Netra.evaluation.get_dataset()` to fetch items, then wrap in `Dataset(items=...)`.

27. When mapping multi-turn (session) evaluators to a dataset using `netra_map_evaluator_to_dataset`,
    ALWAYS pass `variableMapping: {}` (empty object). Session-level evaluators auto-resolve their
    variables from the conversation context and do not require explicit variable mappings.
    However, the `variableMapping` parameter itself MUST be provided — omitting it entirely
    causes the API to throw "Cannot convert undefined or null to object".

28. Multi-turn dataset items MUST use `input: {}` (empty object).
    The simulation engine generates all user messages — including the first one — from
    `metadata.behaviour_instructions`. The `input` string is NOT sent as the first message.
    Using a string for `input` causes the Netra dashboard to show an empty scenario name
    because the UI expects `metadata.scenarioName` to be the primary identifier when `input`
    is an empty object.

    Both `scenarioName` and `scenario` MUST be present in metadata:
    - `scenarioName` = the short display label shown in the dashboard
    - `scenario` = the actual goal/scenario description that session evaluators
      (Goal Fulfillment, Conversation Completeness) read to judge success

    Omitting `scenario` causes evaluators to receive an empty goal, resulting in
    inaccurate or failing evaluation scores.

29. A default provider configuration MUST exist before any MCP-based creation (datasets, evaluators, dataset items, evaluator mappings).
    Phase 0 verifies this upfront by calling `netra_get_default_llm_configuration`. If no default provider
    is configured, the entire workflow is blocked — do not proceed to planning phases or creation.
    This check prevents wasted effort generating plans that cannot be materialized.

30. When adding a library evaluator to My Evaluators, the project evaluator's `type` MUST match the
    library evaluator's original type exactly.

    - If the library evaluator is `tool_accuracy` (rule-based), create it with `type: "tool_accuracy"`
      and pass its specific config (e.g. `{"matchType": "partial"}`). Do NOT include model/provider_id.
    - If the library evaluator is `llm-as-judge`, create it with `type: "llm-as-judge"` and pass
      model/provider_id config.
    - If the library evaluator is `regex` or `code`, create it with the matching type.

    The `netra_create_evaluator` API may silently default to `llm-as-judge` when using
    `libraryEvaluatorId`, regardless of the library evaluator's actual type. This causes rule-based
    evaluators (like `tool_accuracy`) to fail at runtime because the engine expects LLM config
    (`model`, `provider_id`, `prompt`) that rule-based evaluators don't use.

    If you encounter this type mismatch, create the evaluator as a **custom evaluator** with explicit
    `type` set to the correct value and omit `libraryEvaluatorId`. This bypasses the auto-defaulting
    behavior and ensures the evaluator runs with the correct execution logic.

31. When using evaluators with configurable variables, ALWAYS include `evaluatorConfigs` in dataset
    item metadata (or dataset metadata) with the correct evaluator ID and variable values.

    Evaluators with configurable variables:

    | Evaluator | Configurable Variables |
    |---|---|
    | Guideline Adherence | `assistant_instructions` (REQUIRED — no auto-resolution), `assistant_constraints` (optional — auto-resolved from `metadata.assistant_constraints` but configurable value takes precedence) |
    | Factual Accuracy (multi-turn) | `reference_facts` (optional — auto-resolved from `metadata.reference_facts` but configurable value takes precedence) |

    `evaluatorConfigs` format in metadata:

    ```json
    {
      "evaluatorConfigs": [
        {
          "id": "{evaluator-id}",
          "configuredValues": {
            "variable_name": "value"
          }
        }
      ]
    }
    ```

    Without `evaluatorConfigs`:
    - **Guideline Adherence** will receive empty `assistant_instructions`, causing the evaluator
      to report "The evaluation is severely limited because the assistant's instructions were not
      provided" and produce a degraded score (typically 0.5) that fails the >= 0.6 pass criteria.
    - **Factual Accuracy (multi-turn)** will receive empty `reference_facts` if not set in
      `metadata.reference_facts` either, causing it to default-pass with no meaningful evaluation.

    The `assistant_instructions` variable is the most critical — it has NO fallback auto-resolution
    from any metadata field. It MUST be explicitly provided via `evaluatorConfigs[].configuredValues`.

    When the plan includes Guideline Adherence or Factual Accuracy (multi-turn), ALWAYS prompt the
    user to provide these values before proceeding to creation. If the user cannot provide them,
    warn that the evaluator will produce unreliable scores.

## Reference

For the full evaluator catalog with variables and thresholds, see [evaluator-library.md](references/evaluator-library.md).
