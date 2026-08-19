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
    # Root input: only the user's request text, not the whole order envelope.
    # Envelope metadata (id, item count) goes on span attributes/events instead.
    Netra.set_root_input(order["customer_request"])

    current = Netra.get_current_span()
    if current:
        current.set_attribute("order.id", order.get("id"))
        current.add_event("order.received", {"item_count": len(order.get("items", []))})

    result = OrderAgent().orchestrate(order)

    # Root output: only the message returned to the user, not the internal result dict.
    Netra.set_root_output(result["confirmation_message"])

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
        return {
            "status": "queued",
            "order_id": order.get("id"),
            "confirmation_message": f"Order {order.get('id')} is queued for shipping.",
        }

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
    Netra.set_root_input(user_message)

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
            Netra.set_root_output(response_text)
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

## Root Input & Output

`set_root_input` and `set_root_output` record the top-level input and output on the **root span** of the current trace. They carry a specific meaning — they are not general-purpose debug attributes:

- **Root input** = the input the *user* (or calling client) gave the AI/agent system — the prompt, question, or message that started this trace.
- **Root output** = the final answer the AI/agent system sent *back to the user* — the response content as the user sees it.

These become the `input` and `output` attributes on the trace in the Netra dashboard and are what trace-level evaluators read. Everything else — routing metadata, auth context, internal state, token usage — belongs on span attributes or custom attributes, not here.

**Always set root input at the entry point of your workflow and root output before returning the final result.** This should be the default practice for every instrumented application.

```python
from netra import Netra

Netra.set_root_input("What is the weather today?")

# ... run your pipeline / agent / chain ...

Netra.set_root_output("The weather today is sunny with a high of 72°F.")
```

Both methods accept any serializable value (strings, dicts, lists, etc.). The SDK serializes the value to a string internally.

### Set only the core input and output

Entry points usually receive a request *envelope* and return a response *envelope*. Extract the core content — **do not pass the whole object**. A trace whose input is the entire request body buries the actual user question inside metadata, which makes traces hard to scan and trace-level evaluators unreliable.

```python
# ❌ Wrong — whole envelopes; the real input/output is buried in metadata
Netra.set_root_input(request_body)   # {"message": ..., "session_id": ..., "auth_token": ..., "flags": {...}}
Netra.set_root_output(response)      # {"text": ..., "usage": {...}, "model": ..., "latency_ms": ...}

# ✅ Right — only what the user sent and what the user gets back
Netra.set_root_input(request_body["message"])
Netra.set_root_output(response["text"])
```

Rules for picking the value:

- Prefer a **plain string** when the user's input is text and the answer is text — the common case.
- Use a dict **only when the core input genuinely has multiple parts** (e.g. `{"question": ..., "image_url": ...}`), and include only those parts.
- For multi-turn chat, root input is the **current user turn**, not the whole message history. History belongs on the generation span's prompt.
- Exclude prompt scaffolding: system prompts, retrieved/RAG context, tool schemas, few-shot examples. These are already captured on the generation spans that use them.
- Exclude identity and infrastructure fields — `session_id`, `user_id`, `tenant_id`, auth tokens, request IDs, feature flags. Use `Netra.set_session_id()`, `set_user_id()`, `set_tenant_id()`, and `set_custom_attributes()` for those.
- Exclude response bookkeeping: token usage, cost, model name, finish reason, timings, trace IDs.
- Never pass secrets or raw PII — root input/output is stored and rendered on the trace.
- If the core value is nested, index into it instead of passing the parent: `payload["data"]["query"]`, not `payload`.

### Streaming output

When the final output is a stream (e.g., an LLM streaming response), use `set_root_output_stream` instead. It wraps the stream transparently and records the accumulated output on the root span when iteration completes.

```python
stream = client.chat.completions.create(model="gpt-4o", messages=messages, stream=True)
stream = Netra.set_root_output_stream(stream)

for chunk in stream:
    print(chunk)  # output is auto-committed to root span when iteration ends
```

`set_root_output_stream` supports both sync and async iterables. If no active trace context exists or the value is not iterable, it returns the value unchanged — so it is always safe to reassign.

### Generator / SSE streaming (FastAPI, Starlette, etc.)

`set_root_output_stream` works by wrapping an iterable that yields raw LLM chunks. It does **not** work when the generator yields already-formatted SSE strings (e.g., `"data: ...\n\n"`), because the accumulated text would include the SSE framing.

In this pattern — common in FastAPI `StreamingResponse` generators — **manually accumulate** the content chunks and call `set_root_output` after iteration:

```python
@workflow(name="chat-stream")
def generate():
    Netra.set_root_input(user_message)
    collected: list[str] = []

    for chunk in agent.run(message, stream=True):
        if chunk and chunk.content:
            collected.append(chunk.content)
            yield f"data: {chunk.content}\n\n"

    Netra.set_root_output("".join(collected))
    yield "data: [DONE]\n\n"
```

**Rule of thumb:** Use `set_root_output_stream` when you can wrap the raw LLM iterable before consuming it. Use manual accumulation + `set_root_output` when the generator transforms or formats chunks before yielding (SSE, WebSocket frames, custom protocols).

## Recommended Rollout Strategy

1. Start with auto-instrumentation for broad, immediate coverage.
2. Add decorators for business semantics (`@workflow` → `@agent` → `@task`).
3. Use manual spans only for operations requiring precise boundaries or custom metadata.

## Validation Checklist

1. `Netra.init()` is called once at startup.
2. Initialization happens **before** instrumented library usage.
3. High-level operations appear as workflow spans.
4. `Netra.set_root_input()` is called at the entry point with the user's input, and `Netra.set_root_output()` (or `Netra.set_root_output_stream()` for streaming) is called with the final result before returning. For SSE/generator streaming, chunks are accumulated and `set_root_output` is called after iteration completes.
5. Root input/output carry **only the core input and output** — the user's message in, the user-facing response out. No request/response envelopes, no message history, no system prompts or retrieved context, no session/user/auth fields, no usage or timing metadata.
6. `Netra.shutdown()` is called on graceful app termination.
7. All `InstrumentSet` values and `SpanType` values used are verified against the official Netra documentation listed in the **References** section below.

## References

- https://docs.getnetra.ai/Observability/Traces/auto-instrumentation
- https://docs.getnetra.ai/Observability/Traces/decorators
- https://docs.getnetra.ai/Observability/Traces/manual-tracing
- https://docs.getnetra.ai/Observability/Traces/configuration/initialization
- https://docs.getnetra.ai/sdk-reference/sdk/python
