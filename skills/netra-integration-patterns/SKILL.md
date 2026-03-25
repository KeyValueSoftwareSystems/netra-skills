---
name: netra-integration-patterns
description: 'Apply proven Netra integration patterns for RAG, FastAPI, multi-agent, streaming, and batch workflows.'
argument-hint: 'Describe your application architecture (RAG/API/agent/batch/streaming) and provider/tooling stack.'
---

# Netra Integration Patterns

Use this skill to choose a practical integration pattern and implement it with low-risk defaults.

## Supported Scenarios
- RAG pipelines with vector DB + LLM instrumentation.
- FastAPI request handlers with context and guardrails.
- Multi-agent orchestration with workflow/agent decorators.
- Batch processing with per-item custom spans.
- Streaming responses with active span lifecycle through iteration.

## Procedure
1. Choose the closest scenario pattern.
2. Initialize Netra once at startup with required instruments.
3. Apply decorators to workflow boundaries.
4. Add request context and security pipeline where needed.
5. Add custom spans only for external boundaries not already covered.
6. Validate traces for hierarchy and error visibility.

## FastAPI Baseline Pattern
```python
from fastapi import FastAPI
from netra import Netra
from netra.instrumentation.instruments import InstrumentSet

Netra.init(
    app_name="fastapi-chatbot",
    instruments={InstrumentSet.FASTAPI, InstrumentSet.OPENAI, InstrumentSet.REDIS},
)

app = FastAPI()
```

## Express Baseline Pattern
```typescript
import express from "express";
import { Netra, NetraInstruments } from "netra-sdk";

async function main() {
    await Netra.init({
        appName: "my-api",
        headers: `x-api-key=${process.env.NETRA_API_KEY}`,
        instruments: new Set([NetraInstruments.EXPRESS, NetraInstruments.OPENAI]),
    });

    const app = express();

    app.use((req, _res, next) => {
        if (req.headers["x-user-id"]) {
            Netra.setUserId(req.headers["x-user-id"] as string);
        }
        if (req.headers["x-session-id"]) {
            Netra.setSessionId(req.headers["x-session-id"] as string);
        }
        next();
    });

    app.listen(3000);
}

main();
```

## Checklist
- `Netra.init()` called exactly once at startup.
- Instrument set matches all major providers/frameworks used.
- Request-level IDs are set for every incoming request.
- Security scanning and PII masking run before model execution.

## References
- https://docs.getnetra.ai/quick-start/QuickStart_Tracing
- https://docs.getnetra.ai/Observability/Traces/auto-instrumentation
- https://docs.getnetra.ai/Observability/Traces/decorators
- https://docs.getnetra.ai/sdk-reference/sdk/typescript
