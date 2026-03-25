---
name: netra-best-practices
description: 'Code-first Netra best-practices playbook covering setup, instrumentation, context tracking, custom spans/metrics, integration patterns, MCP trace analysis, evaluation, simulation, and troubleshooting.'
argument-hint: 'Describe your language/framework, provider stack, use case, target quality criteria, and what issue or outcome you are working on.'
---

# Netra Best Practices

Use this skill as the default end-to-end guide for integrating, operating, and improving AI systems with Netra.

## Scope
- Code-focused integration and operations guidance only.
- Includes runnable SDK patterns for setup, tracing, metrics, evaluations, and simulations.

## When To Use
- You want one workflow that replaces separate setup and instrumentation skills.
- You need traceability from request context to span-level debugging.
- You need repeatable quality validation with evaluations and simulations.
- You need production-safe troubleshooting guidance.

## Coverage
This unified skill combines guidance for:
- SDK setup and initialization.
- Context tracking (user/session/tenant/custom attributes/events/conversation).
- Tracing method selection (auto-instrumentation, decorators, manual spans).
- Custom spans, usage/cost, and action records.
- OpenTelemetry custom metrics.
- Integration patterns (FastAPI, Express, multi-agent, batch, streaming).
- MCP trace analysis workflow (`netra_query_traces`, `netra_get_trace_by_id`).
- Evaluation setup and regression tracking.
- Multi-turn simulation setup and analysis.
- Troubleshooting and production triage.

## End-To-End Workflow
1. Install and initialize Netra once at startup.
2. Enable only required instruments for your framework/providers.
3. Set request context (`user_id`, `session_id`, `tenant_id`) at request entry.
4. Choose tracing strategy:
   - auto-instrumentation for fast coverage,
   - decorators for semantic boundaries,
   - manual spans for precise lifecycle control.
5. Add custom attributes, events, usage, and actions on key operations.
6. Add business metrics with bounded-cardinality attributes.
7. Validate trace hierarchy and metadata in Observability.
8. Build eval datasets and run test suites before releases.
9. Build multi-turn simulations for critical scenarios/personas.
10. Use MCP trace tools to debug failures and latency regressions.
11. Apply troubleshooting triage for missing telemetry or SDK errors.

## Tracing Method Decision Guide
- Auto-instrumentation: choose when you need quickest provider/framework visibility.
- Decorators (`@workflow`, `@agent`, `@task`, `@span`): choose for clear business semantics.
- Manual spans (`Netra.start_span` / `Netra.startSpan`): choose for custom operation boundaries, retries, side effects, and explicit async lifecycle control.

## Setup Baseline
```python
from netra import Netra
from netra.instrumentation.instruments import InstrumentSet

Netra.init(
    app_name="my-ai-app",
    instruments={InstrumentSet.OPENAI, InstrumentSet.FASTAPI},
)
```

```typescript
import { Netra, NetraInstruments } from "netra-sdk";

await Netra.init({
    appName: "my-ai-app",
    headers: `x-api-key=${process.env.NETRA_API_KEY}`,
    instruments: new Set([NetraInstruments.OPENAI, NetraInstruments.EXPRESS]),
});
```

## Context Tracking Pattern
```python
from netra import Netra
from netra.session_manager import ConversationType


def handle_request(request):
    Netra.set_user_id(request.user_id)
    Netra.set_session_id(request.session_id)
    Netra.set_tenant_id(request.tenant_id)

    Netra.set_custom_attributes("customer_tier", request.customer_tier)
    Netra.set_custom_attributes("region", request.region)

    Netra.add_conversation(
        conversation_type=ConversationType.INPUT,
        role="user",
        content=request.message,
    )

    response = run_agent(request.message)

    Netra.add_conversation(
        conversation_type=ConversationType.OUTPUT,
        role="assistant",
        content=response,
    )

    Netra.set_custom_event(
        event_name="request_completed",
        attributes={"status": "success"},
    )

    return response
```

## Decorator + Manual Span Pattern
```python
from netra import Netra, SpanType, UsageModel, ActionModel
from netra.decorators import workflow, agent, task, span


@workflow(name="order-fulfillment")
def fulfill_order(order: dict):
    return OrderAgent().orchestrate(order)


@agent
class OrderAgent:
    @task(name="validate-order")
    def validate(self, order: dict):
        if not order.get("items"):
            raise ValueError("Order must contain at least one item")

    @span(name="shipping-quote", as_type=SpanType.TOOL)
    def dispatch(self, order: dict):
        with Netra.start_span("persist-order") as custom:
            custom.set_usage([
                UsageModel(
                    model="gpt-4o",
                    usage_type="generation",
                    units_used=1,
                    cost_in_usd=0.01,
                )
            ])
            custom.set_action([
                ActionModel(
                    action="DB",
                    action_type="INSERT",
                    metadata={"table": "orders"},
                    success=True,
                )
            ])
            custom.add_event("order.persisted", {"order_id": order.get("id")})
        return {"status": "queued", "order_id": order.get("id")}

    def orchestrate(self, order: dict):
        self.validate(order)
        return self.dispatch(order)
```

## Custom Metrics Pattern
```python
from netra import Netra

Netra.init(app_name="my-service", enable_metrics=True, metrics_export_interval_ms=30000)
meter = Netra.get_meter("my-service")

requests = meter.create_counter("llm.requests.total", unit="1")
errors = meter.create_counter("llm.errors.total", unit="1")
latency = meter.create_histogram("llm.latency_ms", unit="ms")
active = meter.create_up_down_counter("llm.active_requests", unit="1")
```

## Integration Pattern Checks
- FastAPI/Express: initialize once and add request context middleware.
- Multi-agent: set `@workflow` root and `@agent` boundaries.
- Batch: create per-item spans and capture failure metadata.
- Streaming: ensure spans close only after stream completion.

## MCP Debugging Workflow
1. Query traces with a narrow time range and specific filters.
2. Select the most relevant trace.
3. Fetch full span tree by trace ID.
4. Inspect parent-child relationships, errors, and slow spans.
5. Summarize root cause and next fix.

Primary tools:
- `netra_query_traces`
- `netra_get_trace_by_id`

## Evaluation Code Sample (Python)
```python
from netra import Netra

Netra.init(app_name="my-app")


def task_fn(input_data):
    # Replace with your app invocation and return output text.
    return my_agent_response(input_data)


dataset = Netra.evaluation.get_dataset(dataset_id="YOUR_DATASET_ID")
result = Netra.evaluation.run_test_suite(
    name="baseline-eval",
    data=dataset,
    task=task_fn,
)

print(result)
```

## Evaluation Code Sample (TypeScript)
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

## Simulation Code Sample (TypeScript)
```typescript
import { Netra } from "netra-sdk-js";
import { BaseTask, TaskResult } from "netra-sdk-js/simulation";

const client = new Netra({ apiKey: process.env.NETRA_API_KEY! });

class MyAgentTask extends BaseTask {
    constructor(private readonly agent: any) {
        super();
    }

    async run(message: string, sessionId?: string | null): Promise<TaskResult> {
        const response = await this.agent.chat(message, { sessionId });
        return {
            message: response.text,
            sessionId: sessionId || "default",
        };
    }
}

const result = await client.simulation.runSimulation({
    name: "Customer Support Simulation",
    datasetId: "dataset-123",
    task: new MyAgentTask(myAgent),
    context: { environment: "staging" },
    maxConcurrency: 5,
});

console.log(result?.completed.length, result?.failed.length);
```

## Troubleshooting Triage
1. Confirm package installation and runtime compatibility.
2. Confirm single `Netra.init()` call before critical imports.
3. Verify API key headers and endpoint configuration.
4. Verify enabled instruments match runtime dependencies.
5. Verify network egress and shutdown flush.
6. For missing traces/metrics, check environment flags and export intervals.

## Production Checklist
- Initialization occurs once per process.
- IDs are set at every request boundary and not leaked across users.
- Span naming and boundaries reflect business intent.
- No sensitive payloads are exported unintentionally.
- Metrics dimensions are bounded and low cardinality.
- Evaluations and simulations run before release promotion.
- MCP trace review is part of incident/debug workflow.

## References
- https://docs.getnetra.ai/quick-start/QuickStart_Tracing
- https://docs.getnetra.ai/Observability/Traces/overview
- https://docs.getnetra.ai/Observability/Traces/auto-instrumentation
- https://docs.getnetra.ai/Observability/Traces/decorators
- https://docs.getnetra.ai/Observability/Traces/manual-tracing
- https://docs.getnetra.ai/Evaluation/Evaluation-overview
- https://docs.getnetra.ai/Simulation/Simulation-overview
- https://docs.getnetra.ai/sdk-reference/sdk/overview
- https://docs.getnetra.ai/sdk-reference/sdk/typescript
