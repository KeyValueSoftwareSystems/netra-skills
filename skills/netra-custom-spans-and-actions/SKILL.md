---
name: netra-custom-spans-and-actions
description: 'Implement manual Netra spans with rich attributes, usage tracking, action records, and events.'
argument-hint: 'Describe the operation boundaries you need to trace and what usage/actions metadata must be attached.'
---

# Netra Custom Spans And Actions

Use this skill when auto/decorator tracing is not enough and you need explicit span lifecycle control.

## When To Use
- You need spans around external APIs, tool calls, or batch steps.
- You need usage/cost tracking per operation.
- You need structured action records for DB/API side effects.

## Procedure
1. Wrap each critical operation in `Netra.start_span(...)`.
2. Set span attributes (`model`, `llm_system`, operation keys).
3. Attach usage via `UsageModel`.
4. Attach actions via `ActionModel`.
5. Add events for checkpoints and failures.

## Python Pattern
```python
from netra import Netra, UsageModel, ActionModel


def process_item(item_id: str):
    with Netra.start_span("process_item") as span:
        span.set_attribute("item.id", item_id)
        span.set_model("gpt-4o")
        span.set_llm_system("openai")

        usage = [
            UsageModel(
                model="gpt-4o",
                usage_type="generation",
                units_used=1,
                cost_in_usd=0.01,
            )
        ]
        span.set_usage(usage)

        action = ActionModel(
            action="DB",
            action_type="UPDATE",
            metadata={"table": "items"},
            success=True,
        )
        span.set_action([action])

        span.add_event("item.processed", {"item_id": item_id})
```

## TypeScript Pattern
```typescript
import { Netra, SpanType } from "netra-sdk";

async function processItem(itemId: string): Promise<void> {
    const span = Netra.startSpan(
        "process_item",
        { attributes: { "item.id": itemId }, moduleName: "items" },
        undefined,
        SpanType.TOOL
    );

    span.setModel("gpt-4o");
    span.setLlmSystem("openai");

    try {
        span.setUsage([
            {
                model: "gpt-4o",
                usageType: "generation",
                unitsUsed: 1,
                costInUsd: 0.01,
            },
        ]);

        span.setAction([
            {
                action: "DB",
                actionType: "UPDATE",
                metadata: { table: "items" },
                success: true,
            },
        ]);

        span.addEvent("item.processed", { itemId });
        span.setSuccess();
    } catch (error: any) {
        span.setError(error.message || "process_item failed");
        throw error;
    } finally {
        span.end();
    }
}
```

## Checklist
- Each custom span maps to a meaningful business boundary.
- Usage and cost fields are populated for paid model calls.
- Action records capture side effects with success/failure state.
- Spans always close cleanly on exceptions.

## References
- https://docs.getnetra.ai/Observability/Traces/manual-tracing
- https://docs.getnetra.ai/sdk-reference/sdk/overview
- https://docs.getnetra.ai/sdk-reference/sdk/typescript
