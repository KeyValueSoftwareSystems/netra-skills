---
name: netra-typescript-instrumentation
description: Instrument TypeScript/Node.js LLM applications with Netra. Covers auto-instrumentation, decorators, and manual spans.
---

# Netra TypeScript Instrumentation

Instrument TypeScript/Node.js LLM applications with Netra, following best practices tailored to the TypeScript SDK.

## Workflow

- - Assess the current environment. Ensure `netra-sdk` is installed and is the latest version.
- Determine the kind of instrumentation necessary.
- Instrument the application and instruct the user on steps they need to take to set up their environment.

## Auto-Instrumentation

Auto-instrumentation is the fastest way to start. Netra patches supported libraries and automatically captures spans for LLM calls, frameworks, vector DBs, HTTP, and more.

Initialize Netra once at application startup, before using libraries you want instrumented. **The init call is async — always `await` it.**

```env
NETRA_API_KEY=
NETRA_OTLP_ENDPOINT=
```

The user may create their API key by visiting the dashboard: Settings → Project → API Keys.

```typescript
import { Netra, NetraInstruments } from "netra-sdk";

// Initialize before importing providers/frameworks for best coverage
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

Valid instrument names are listed in the official Netra documentation listed in the **References** section. Always verify an instrument exists before using it.

### Blocking instruments

Use `blockInstruments` to exclude specific instrumentations while keeping defaults:

```typescript
await Netra.init({
  appName: "my-ai-app",
  blockInstruments: new Set([NetraInstruments.HTTP]),
});
```

### Controlling root spans

Use `rootInstruments` to limit which libraries can create root-level spans:

```typescript
await Netra.init({
  appName: "my-ai-app",
  rootInstruments: new Set([
    NetraInstruments.OPENAI,
    NetraInstruments.ANTHROPIC,
  ]),
});
```

## Decorators

TypeScript decorators require `"experimentalDecorators": true` in `tsconfig.json`.

- `@workflow`: top-level business process.
- `@agent`: agent/orchestrator behavior.
- `@task`: discrete unit of work / tool call.
- `@span`: generic/custom span type.

```typescript
import { Netra, SpanType } from "netra-sdk";
import { workflow, agent, task, span } from "netra-sdk/decorators";

@workflow({ name: "order-fulfillment" })
class OrderWorkflow {
  async run(order: { id: string; items: unknown[] }) {
    Netra.setRootInput(order);

    const result = await new OrderAgent().orchestrate(order);

    Netra.setRootOutput(result);
    return result;
  }
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
```

Decorators automatically capture parameters and exceptions. Focus instrumentation on high-value workflow boundaries to avoid noisy traces.

## Manual Instrumentation

Unlike Python, TypeScript has no built-in context manager pattern. When using `Netra.startSpan()`, **always** call `span.end()`, typically in a `finally` block.

```typescript
import { Netra, SpanType } from "netra-sdk";
import type { UsageModel } from "netra-sdk";

async function chatWithAI(userMessage: string): Promise<string> {
  Netra.setRootInput(userMessage);

  const span = Netra.startSpan("chat-completion", {
    asType: SpanType.GENERATION,
    moduleName: "chat",
    attributes: { entrypoint: "chatWithAI" },
  });

  span.setPrompt(userMessage);
  span.setModel("gpt-4");
  span.setLlmSystem("openai");
  span.addEvent("generation.started");

  try {
    const responseText = "Hello from model";

    span.setUsage([
      {
        model: "gpt-4",
        cost_in_usd: 0.001,
        usage_type: "chat",
        units_used: 1,
      },
    ]);
    span.setSuccess();
    Netra.setRootOutput(responseText);
    return responseText;
  } catch (error: any) {
    span.setError(error?.message || "unknown error");
    throw error;
  } finally {
    span.end();
  }
}
```

### Automatic Lifecycle Management

Use `startActiveSpan()` when possible. It **automatically** handles span completion and error tracking.

```typescript
const result = Netra.startActiveSpan(
  "my-operation",
  { asType: SpanType.TOOL },
  async (span) => {
    span.setAttribute("key", "value");
    return await doWork();
  },
);
```

## Context Tracking

Set context for trace grouping and filtering:

```typescript
Netra.setUserId("user-123");
Netra.setSessionId("session-abc");
Netra.setTenantId("tenant-xyz");
Netra.setCustomAttributes("feature", "chat-v2");
```

## Root Input & Output

Use `setRootInput` and `setRootOutput` to record the top-level input and output on the **root span** of the current trace. These values appear as the `input` and `output` attributes on the trace in the Netra dashboard, making it easy to see what went in and what came out at a glance.

**Always set root input at the entry point of your workflow and root output before returning the final result.** This should be the default practice for every instrumented application.

```typescript
import { Netra } from "netra-sdk";

Netra.setRootInput({ query: "What is the weather today?" });

// ... run your pipeline / agent / chain ...

Netra.setRootOutput("The weather today is sunny with a high of 72°F.");
```

Both methods accept any serializable value (strings, objects, arrays, etc.). The SDK serializes the value to a string internally.

## Root Span Wrapper

When `enableRootSpan` is enabled, use `runWithRootSpan()` to execute code within the root span context. Any spans created inside the callback are automatically **parented to the root span**.

```typescript
await Netra.init({ appName: "my-app", enableRootSpan: true });

Netra.runWithRootSpan(() => {
  // All spans created here are children of the root span
  doWork();
});
```

## Recommended Rollout Strategy

1. Start with auto-instrumentation for broad, immediate coverage.
2. Add decorators for business semantics (`@workflow` → `@agent` → `@task`).
3. Use manual spans only for operations requiring precise boundaries or custom metadata.

## Validation Checklist

1. `await Netra.init()` is called once at startup — always awaited.
2. Initialization happens **before** instrumented library usage.
3. `experimentalDecorators: true` is set in `tsconfig.json` if using decorators.
4. Manual spans always call `span.end()` in a `finally` block.
5. `Netra.setRootInput()` is called at the entry point with the user's input, and `Netra.setRootOutput()` is called with the final result before returning.
6. `await Netra.shutdown()` is called on graceful app termination.
7. All `NetraInstruments` values and `SpanType` values used are verified against the official Netra documentation listed in the **References** section below.

## References

- https://docs.getnetra.ai/Observability/Traces/auto-instrumentation
- https://docs.getnetra.ai/Observability/Traces/decorators
- https://docs.getnetra.ai/Observability/Traces/manual-tracing
- https://docs.getnetra.ai/Observability/Traces/configuration/initialization
- https://docs.getnetra.ai/sdk-reference/sdk/typescript
