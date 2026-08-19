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
  async run(order: { id: string; items: unknown[]; customerRequest: string }) {
    // Root input: only the user's request text, not the whole order envelope.
    Netra.setRootInput(order.customerRequest);

    const result = await new OrderAgent().orchestrate(order);

    // Root output: only the message returned to the user, not the internal result object.
    Netra.setRootOutput(result.confirmationMessage);
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
    return {
      status: "queued",
      orderId: order.id,
      confirmationMessage: `Order ${order.id} is queued for shipping.`,
    };
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

`setRootInput` and `setRootOutput` record the top-level input and output on the **root span** of the current trace. They carry a specific meaning — they are not general-purpose debug attributes:

- **Root input** = the input the *user* (or calling client) gave the AI/agent system — the prompt, question, or message that started this trace.
- **Root output** = the final answer the AI/agent system sent *back to the user* — the response content as the user sees it.

These become the `input` and `output` attributes on the trace in the Netra dashboard and are what trace-level evaluators read. Everything else — routing metadata, auth context, internal state, token usage — belongs on span attributes or custom attributes, not here.

**Always set root input at the entry point of your workflow and root output before returning the final result.** This should be the default practice for every instrumented application.

```typescript
import { Netra } from "netra-sdk";

Netra.setRootInput("What is the weather today?");

// ... run your pipeline / agent / chain ...

Netra.setRootOutput("The weather today is sunny with a high of 72°F.");
```

Both methods accept any serializable value (strings, objects, arrays, etc.). The SDK serializes the value to a string internally.

### Set only the core input and output

Handlers usually receive a request *envelope* and return a response *envelope*. Extract the core content — **do not pass the whole object**. A trace whose input is the entire request body buries the actual user question inside metadata, which makes traces hard to scan and trace-level evaluators unreliable.

```typescript
// ❌ Wrong — whole envelopes; the real input/output is buried in metadata
Netra.setRootInput(req.body);    // { message, sessionId, authToken, flags }
Netra.setRootOutput(response);   // { text, usage, model, latencyMs }

// ✅ Right — only what the user sent and what the user gets back
Netra.setRootInput(req.body.message);
Netra.setRootOutput(response.text);
```

Rules for picking the value:

- Prefer a **plain string** when the user's input is text and the answer is text — the common case.
- Use an object **only when the core input genuinely has multiple parts** (e.g. `{ question, imageUrl }`), and include only those parts.
- For multi-turn chat, root input is the **current user turn**, not the whole message history. History belongs on the generation span's prompt.
- Exclude prompt scaffolding: system prompts, retrieved/RAG context, tool schemas, few-shot examples. These are already captured on the generation spans that use them.
- Exclude identity and infrastructure fields — `sessionId`, `userId`, `tenantId`, auth tokens, request IDs, feature flags. Use `Netra.setSessionId()`, `setUserId()`, `setTenantId()`, and `setCustomAttributes()` for those.
- Exclude response bookkeeping: token usage, cost, model name, finish reason, timings, trace IDs.
- Never pass secrets or raw PII — root input/output is stored and rendered on the trace.
- If the core value is nested, index into it instead of passing the parent: `payload.data.query`, not `payload`.

### Generator / SSE streaming (Express, Next.js, Hono, etc.)

In web frameworks, streaming responses are often sent as formatted SSE strings (`"data: ...\n\n"`) or via `res.write()`. Because the output is transformed before being sent, you cannot simply wrap the raw stream — you need to **manually accumulate** the content chunks and call `setRootOutput` after iteration completes.

```typescript
import { Netra } from "netra-sdk";

app.post("/api/chat/stream", async (req, res) => {
  const span = Netra.startSpan("chat-stream");
  Netra.setSessionId(req.body.sessionId);
  Netra.setRootInput(req.body.message);

  const collected: string[] = [];

  try {
    const stream = await agent.run(req.body.message, { stream: true });

    for await (const chunk of stream) {
      if (chunk.content) {
        collected.push(chunk.content);
        res.write(`data: ${chunk.content}\n\n`);
      }
    }

    Netra.setRootOutput(collected.join(""));
    res.write("data: [DONE]\n\n");
    res.end();
    span.setSuccess();
  } catch (error: any) {
    span.setError(error?.message || "unknown error");
    res.end();
  } finally {
    span.end();
  }
});
```

**Rule of thumb:** Use manual accumulation + `setRootOutput` when the handler transforms or formats chunks before sending (SSE, WebSocket frames, custom protocols). This ensures the root span captures the clean response content without framing artifacts.

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
5. `Netra.setRootInput()` is called at the entry point with the user's input, and `Netra.setRootOutput()` is called with the final result before returning. For SSE/generator streaming, chunks are accumulated and `setRootOutput` is called after iteration completes.
6. Root input/output carry **only the core input and output** — the user's message in, the user-facing response out. No request/response envelopes, no message history, no system prompts or retrieved context, no session/user/auth fields, no usage or timing metadata.
7. `await Netra.shutdown()` is called on graceful app termination.
8. All `NetraInstruments` values and `SpanType` values used are verified against the official Netra documentation listed in the **References** section below.

## References

- https://docs.getnetra.ai/Observability/Traces/auto-instrumentation
- https://docs.getnetra.ai/Observability/Traces/decorators
- https://docs.getnetra.ai/Observability/Traces/manual-tracing
- https://docs.getnetra.ai/Observability/Traces/configuration/initialization
- https://docs.getnetra.ai/sdk-reference/sdk/typescript
