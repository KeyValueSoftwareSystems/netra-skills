---
name: netra-setup
description: Instrument LLM applications with Netra. Use when setting up Netra in a new project, adding observability or auditing existing instrumentation.
---

# Netra Observability

Instrument LLM applications/AI agents with Netra, following best practices and tailored to your use case.

## Workflow
- Assess the current environment - Ensure netra-sdk is installed and is the latest version, and figure out which libraries are installed for the LLM application.
- Determine the kind of instrumentation necessary for the application.
- Instrument the application and instruct the user on steps they need to take to set up their environment.

## Auto Instrumentation

Auto-instrumentation is the fastest way to start. Netra patches supported libraries and automatically captures spans for LLM calls, frameworks, vector DBs, HTTP, and more.

Initialize Netra once at application startup, before using libraries you want instrumented.

```env
NETRA_API_KEY=
NETRA_OTLP_ENDPOINT=
```

The user may create their api key by visiting the dashboard and going to Settings -> Project -> API Keys.

```py
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

```typescript
import { Netra, NetraInstruments } from "netra-sdk";

// Always await init in TypeScript so instrumentations are ready
await Netra.init({
	appName: "my-ai-app",
	environment: "production",
	headers: `x-api-key=${process.env.NETRA_API_KEY}`,
	traceContent: true,
	instruments: new Set([NetraInstruments.OPENAI, NetraInstruments.LANGCHAIN]),
});

// Optional context for trace grouping
Netra.setUserId("user-123");
Netra.setSessionId("session-abc");
```

# Instrumentation through decorators

Use decorators when you want semantic application spans with very little code overhead.

- `@workflow`: top-level business process.
- `@agent`: agent/orchestrator behavior.
- `@task`: discrete unit of work/tool call.
- `@span`: generic/custom span type.

*NOTE*: Always ensure netra decorators are placed immediately above the function definition.

## Python decorators

```py
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

## TypeScript decorators

TypeScript decorators require `"experimentalDecorators": true` in `tsconfig.json`.

```typescript
import { SpanType } from "netra-sdk";
import { workflow, agent, task, span } from "netra-sdk/decorators";

@workflow({ name: "order-fulfillment" })
async function fulfillOrder(order: { id: string; items: unknown[] }) {
	const result = await new OrderAgent().orchestrate(order);
	return result;
}

@agent({ name: "order-agent" })
class OrderAgent {
	@task({ name: "validate-order" })
	async validate(order: { items: unknown[] }) {
		if (!order.items?.length) {
			throw new Error("Order must contain at least one item");
		}
	}

	@span({ name: "shipping-quote", asType: SpanType.TOOL })
	async dispatch(order: { id: string }) {
		return { status: "queued", orderId: order.id };
	}

	async orchestrate(order: { id: string; items: unknown[] }) {
		await this.validate(order);
		return this.dispatch(order);
	}
}

@task({ name: "Tool Call" })
async function toolCall() {
	return "Tool output";
}
```

Decorators automatically capture parameters and exceptions. Keep instrumentation focused on high-value workflow boundaries to avoid noisy traces.


# Manual Instrumentation

Use manual tracing when you need full lifecycle and metadata control, advanced nesting, or custom span boundaries.

## Python manual tracing

```py
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

## TypeScript manual tracing

In TypeScript, always call `end()` (preferably in `finally`).

```typescript
import { Netra, SpanType } from "netra-sdk";

async function chatWithAI(userMessage: string): Promise<string> {
	const span = Netra.startSpan(
		"chat-completion",
		{
			asType: SpanType.GENERATION,
			moduleName: "chat",
			attributes: { entrypoint: "chatWithAI" },
		}
	);

	span.setPrompt(userMessage);
	span.setModel("gpt-4");
	span.setLlmSystem("openai");
	span.addEvent("generation.started");

	try {
		const responseText = "Hello from model";

		span.setUsage([
			{
				model: "gpt-4",
				costInUsd: 0.001,
				usageType: "chat",
				unitsUsed: 1,
			},
		]);
		span.setSuccess();
		return responseText;
	} catch (error: any) {
		span.setError(error?.message || "unknown error");
		throw error;
	} finally {
		span.end();
	}
}
```

## Recommended rollout strategy

1. Start with auto-instrumentation for broad, immediate coverage.
2. Add decorators for business semantics (`workflow -> agent -> task`).
3. Use manual spans only for operations requiring precise boundaries or custom metadata.

## Validation checklist

1. `Netra.init()` / `await Netra.init()` is called once at startup.
2. Initialization happens before instrumented library usage.
3. High-level operations appear as workflow spans.
4. TypeScript manual spans always call `span.end()`.
5. `shutdown()` is called on graceful app termination.

## References

- https://docs.getnetra.ai/Observability/Traces/auto-instrumentation
- https://docs.getnetra.ai/Observability/Traces/decorators
- https://docs.getnetra.ai/Observability/Traces/manual-tracing
- https://docs.getnetra.ai/Observability/Traces/configuration/initialization
- https://docs.getnetra.ai/sdk-reference/sdk/python
- https://docs.getnetra.ai/sdk-reference/sdk/typescript