---
name: netra-python-instrumentation
description: Instrument Python LLM applications with Netra. Use when setting up Netra in a new project, adding observability or auditing existing instrumentation.
---

# Netra Python Instrumentation

Instrument Python LLM applications with Netra, following best practices tailored to the Python SDK.

## Workflow

- Assess the current environment - Ensure `netra-sdk` is installed and is the latest version.
- Determine the kind of instrumentation necessary for the application.
- Instrument the application and instruct the user on steps they need to take to set up their environment.

## Auto-Instrumentation

Auto-instrumentation is the fastest way to start. Netra patches supported libraries and automatically captures spans for LLM calls, frameworks, vector DBs, HTTP, and more.

Initialize Netra once at application startup, before using libraries you want instrumented.

```env
NETRA_API_KEY=
NETRA_OTLP_ENDPOINT=
```

The user may create their API key by visiting the dashboard: Settings → Project → API Keys.

```python
import os
from netra import Netra
from netra.instrumentation.instruments import InstrumentSet

# Initialize before importing providers/frameworks for best coverage
Netra.init(
    app_name="my-ai-app",
    environment="production",
    headers=f"x-api-key={os.getenv('NETRA_API_KEY')}",
    trace_content=True,
    instruments={InstrumentSet.OPENAI, InstrumentSet.LANGCHAIN},
)

# Optional context for trace grouping
Netra.set_user_id("user-123")
Netra.set_session_id("session-abc")
```

Valid instrument names are listed in the official Netra documentation listed in the **References** section. Always verify an instrument exists before using it.

### Blocking instruments

Use `block_instruments` to exclude specific instrumentations while keeping defaults:

```python
Netra.init(
    app_name="my-ai-app",
    block_instruments={
        InstrumentSet.HTTPX,
        InstrumentSet.REQUESTS,
    },
)
```

### Controlling root spans

Use `root_instruments` to limit which libraries can create root-level spans:

```python
Netra.init(
    app_name="my-ai-app",
    root_instruments={
        InstrumentSet.OPENAI,
        InstrumentSet.ANTHROPIC,
    },
)
```

## Instrumentation through decorators

Use decorators for semantic application spans with minimal code overhead.

- `@workflow`: top-level business process.
- `@agent`: agent/orchestrator behavior.
- `@task`: discrete unit of work / tool call.
- `@span`: generic/custom span type.

*NOTE*: Always ensure netra decorators are placed immediately above the function definition.

```python
from netra import Netra, SpanType
from netra.decorators import workflow, agent, task, span

@workflow(name="order-fulfillment")
def fulfill_order(order: dict):
    current = Netra.get_current_span()
    if current:
        current.set_attribute("order.id", order.get("id"))
        current.add_event("order.received", {"item_count": len(order.get("items", []))})

    result = OrderAgent().orchestrate(order)

    if current:
        current.add_event("order.completed")
    return result

@agent
class OrderAgent:
    @task(name="validate-order")
    def validate(self, order: dict):
        if not order.get("items"):
            raise ValueError("Order must contain at least one item")
        return True

    @span(name="shipping-quote", as_type=SpanType.TOOL)
    def dispatch(self, order: dict):
        return {"status": "queued", "order_id": order.get("id")}

    def orchestrate(self, order: dict):
        self.validate(order)
        return self.dispatch(order)

@tool #Langchain tools if the app uses langchain
@task(name="Tool Call")
def tool_call():
	return "Tool output"
```

Decorators automatically capture parameters and exceptions. Keep instrumentation focused on high-value workflow boundaries to avoid noisy traces.

## Manual Instrumentation

Use manual tracing when you need full lifecycle and metadata control, advanced nesting, or custom span boundaries.

Python uses context managers — the span is automatically ended when the `with` block exits.

```python
from netra import Netra, SpanType, UsageModel

def chat_with_ai(user_message: str) -> str:
    with Netra.start_span(
        "chat-completion",
        as_type=SpanType.GENERATION,
        attributes={"entrypoint": "chat_with_ai"},
        module_name="chat",
    ) as span:
        span.set_prompt(user_message)
        span.set_model("gpt-4")
        span.set_llm_system("openai")
        span.add_event("generation.started")

        try:
            response_text = "Hello from model"

            span.set_usage([
                UsageModel(
                    model="gpt-4",
                    cost_in_usd=0.001,
                    usage_type="chat",
                    units_used=1,
                )
            ])
            span.set_success()
            return response_text
        except Exception as exc:
            span.set_error(str(exc))
            raise
```

## Context Tracking

Set context for trace grouping and filtering:

```python
Netra.set_user_id("user-123")
Netra.set_session_id("session-abc")
Netra.set_tenant_id("tenant-xyz")
Netra.set_custom_attributes("feature", "chat-v2")
```

## Recommended Rollout Strategy

1. Start with auto-instrumentation for broad, immediate coverage.
2. Add decorators for business semantics (`@workflow` → `@agent` → `@task`).
3. Use manual spans only for operations requiring precise boundaries or custom metadata.

## Validation Checklist

1. `Netra.init()` is called once at startup.
2. Initialization happens **before** instrumented library usage.
3. High-level operations appear as workflow spans.
4. `Netra.shutdown()` is called on graceful app termination.
5. All `InstrumentSet` values and `SpanType` values used are verified against the official Netra documentation listed in the **References** section below.

## References

- https://docs.getnetra.ai/Observability/Traces/auto-instrumentation
- https://docs.getnetra.ai/Observability/Traces/decorators
- https://docs.getnetra.ai/Observability/Traces/manual-tracing
- https://docs.getnetra.ai/Observability/Traces/configuration/initialization
- https://docs.getnetra.ai/sdk-reference/sdk/python
