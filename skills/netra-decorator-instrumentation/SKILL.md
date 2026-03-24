---
name: netra-decorator-instrumentation
description: 'Create custom Netra tracing instrumentation using decorators. Use when choosing between auto-instrumentation, decorators, and manual tracing in Python or TypeScript, with clear semantic span design.'
argument-hint: 'Describe your language (Python/TypeScript), app flow, and which operations need semantic spans.'
---

# Netra Decorator Instrumentation

Use this skill to choose the right tracing method and implement decorator-based semantic spans with clear boundaries.

## Supported Tracing Methods (Python + TypeScript)

Based on Netra Traces and SDK overview pages, there are three tracing methods in both languages:

| Method | Python | TypeScript | Best For |
|---|---|---|---|
| Auto-Instrumentation | `Netra.init(...)` before provider/framework imports | `new Netra(...)` initialization at app startup | Fastest setup, supported libraries, zero code changes |
| Decorators | `@workflow`, `@agent`, `@task`, `@span` | `@workflow`, `@agent`, `@task`, `@span` | Semantic application spans with minimal code changes |
| Manual Tracing | `Netra.start_span(...)` (context manager) | `Netra.startSpan(...)` / explicit `end()` | Fine-grained custom spans and lifecycle control |

Decorators should be your default for application semantics. Use manual spans for edge cases and auto-instrumentation for provider/framework coverage.

## When To Use
- You need semantic, business-level spans (not only provider-level spans).
- You want consistent workflow/agent/task structure across traces.
- You need one approach that works with both Python and TypeScript codebases.

## Decorator Selection Guide
- `@workflow`: End-to-end user-facing process.
- `@agent`: AI orchestrator or decision-making component.
- `@task`: Discrete operation (tool call, API call, validation, transformation).
- `@span`: Generic tracing when you need explicit span type control.

Use `@span(as_type=SpanType.GENERATION|TOOL|AGENT|EMBEDDING|SPAN)` when semantics do not cleanly fit the first three.

## Method Decision Flow
1. Start with auto-instrumentation to capture provider/framework spans quickly.
2. Add decorators to expose your app's workflow and orchestration semantics.
3. Add manual spans only where you need custom lifecycle or metadata control.
4. Keep only high-value spans; remove noisy low-signal instrumentation.

## Procedure
1. Identify top-level entry points and decorate with `@workflow`.
2. Mark orchestrators (class/function) with `@agent`.
3. Mark major operations and tool boundaries with `@task`.
4. Use `@span` where explicit `SpanType` improves trace readability.
5. Add critical attributes/events through `Netra.get_current_span()`.
6. Validate trace hierarchy and status in Observability -> Traces.
7. Trim noise (especially broad class-level decoration).

## Language Notes
- Python:
    - Prefer decorators plus context-manager style `Netra.start_span(...)` for manual additions.
    - Context managers auto-close spans and safely capture exceptions.
- TypeScript:
    - Use decorators for semantics and explicit span ending for manual tracing (`end()` in `finally`).
    - Keep span lifecycle explicit in async paths to avoid dangling spans.

## Python Pattern
```python
from netra.decorators import workflow, agent, task, span
from netra import Netra, SpanType

@workflow(name="order-fulfillment")
def fulfill_order(order: dict):
    current = Netra.get_current_span()
    if current:
        current.set_attribute("order.id", order.get("id"))
        current.add_event("order.received", {"item_count": len(order.get("items", []))})

    result = orchestrate_order(order)

    if current:
        current.add_event("order.completed")
    return result

@agent
class OrderAgent:
    def orchestrate(self, order: dict):
        self.validate(order)
        return self.dispatch(order)

    @task(name="validate-order")
    def validate(self, order: dict):
        if not order.get("items"):
            raise ValueError("Order must contain at least one item")
        return True

    @span(name="shipping-quote", as_type=SpanType.TOOL)
    def dispatch(self, order: dict):
        return {"status": "queued", "order_id": order.get("id")}


def orchestrate_order(order: dict):
    return OrderAgent().orchestrate(order)
```

## TypeScript Manual Fallback Pattern
Use this when a specific operation needs tighter lifecycle control than a decorator can provide.

```typescript
import { Netra, SpanType } from "netra-sdk-js";

const netra = new Netra({ apiKey: process.env.NETRA_API_KEY! });

async function runWithCustomSpan(input: string) {
    const span = netra.startSpan("custom-operation", {
        asType: SpanType.TOOL,
        moduleName: "orchestration",
        attributes: { "input.length": input.length },
    });

    try {
        // custom logic
        span.setAttribute("status", "ok");
    } catch (err) {
        span.setError(String(err));
        throw err;
    } finally {
        span.end();
    }
}
```

## Verification Checklist
- Root business operation appears as a workflow span.
- Agent orchestration appears as agent spans.
- Tasks are not over-instrumented.
- Errors surface as error spans with exception details.
- Parameters are captured but do not include sensitive payloads.

## Best Practices
- Prefer semantic decorators over broad generic spans.
- Name spans when function names are not descriptive.
- Decorate classes sparingly; class-level decoration can generate excess noise.
- Combine with auto-instrumentation for provider/database calls.
- For streaming functions, verify span closes only after stream consumption.
- In TypeScript manual spans, always close spans in `finally`.

## Out Of Scope
- Creating evaluation datasets and evaluators.
- Running simulation datasets and test runs.
- Usage analytics queries (`usage` client).

## References
- https://docs.getnetra.ai/sdk-reference/sdk/overview
- https://docs.getnetra.ai/Observability/Traces/decorators
- https://docs.getnetra.ai/Observability/Traces/overview
- https://docs.getnetra.ai/Observability/Traces/auto-instrumentation
- https://docs.getnetra.ai/Observability/Traces/manual-tracing
- https://docs.getnetra.ai/quick-start/QuickStart_Tracing
- https://docs.getnetra.ai/FAQs/FAQs#what-tracing-methods-does-netra-support
