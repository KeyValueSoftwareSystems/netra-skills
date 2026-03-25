---
name: netra-setup
description: 'Install and initialize the Netra SDK with environment-safe defaults, instrument selection, and shutdown handling.'
argument-hint: 'Describe your framework/provider stack, deployment environment, and whether traces or metrics are required.'
---

# Netra Setup

Use this skill to perform first-time Netra SDK integration for Python and TypeScript applications.

## When To Use
- You are adding Netra to a new or existing AI service.
- You need one-time startup initialization with correct instrument selection.
- You need environment-variable-based configuration for dev/staging/prod.

## Procedure
1. Install base package and optional extras.
2. Initialize `Netra.init(...)` once at app startup.
3. Configure auth headers and environment tags.
4. Enable only the instruments your app uses.
5. Register shutdown flush for clean exit.

## Installation
```bash
pip install netra-sdk
pip install 'netra-sdk[presidio]'
pip install 'netra-sdk[llm_guard]'
```

## TypeScript Installation
```bash
npm install netra-sdk
```

## Minimal Initialization
```python
from netra import Netra
from netra.instrumentation.instruments import InstrumentSet

Netra.init(
	app_name="my-ai-app",
	instruments={InstrumentSet.OPENAI},
)
```

## TypeScript Minimal Initialization
```typescript
import { Netra, NetraInstruments } from "netra-sdk";

await Netra.init({
	appName: "my-ai-app",
	environment: "production",
	instruments: new Set([NetraInstruments.OPENAI]),
});
```

## Production Initialization
```python
import os
from netra import Netra
from netra.instrumentation.instruments import InstrumentSet

Netra.init(
	app_name=os.getenv("NETRA_APP_NAME", "my-ai-app"),
	headers=f"x-api-key={os.environ['NETRA_API_KEY']}",
	environment=os.getenv("NETRA_ENV", "production"),
	trace_content=os.getenv("ENV", "dev") == "dev",
	instruments={InstrumentSet.OPENAI, InstrumentSet.FASTAPI},
	enable_metrics=False,
)
```

## TypeScript Production Initialization
```typescript
import { Netra, NetraInstruments } from "netra-sdk";

await Netra.init({
	appName: process.env.NETRA_APP_NAME || "my-ai-app",
	headers: `x-api-key=${process.env.NETRA_API_KEY}`,
	environment: process.env.NETRA_ENV || "production",
	traceContent: process.env.ENV === "dev",
	instruments: new Set([NetraInstruments.OPENAI, NetraInstruments.EXPRESS]),
});

process.on("SIGTERM", async () => {
	await Netra.shutdown();
	process.exit(0);
});
```

## Environment Variables To Prioritize
- `NETRA_OTLP_ENDPOINT`
- `NETRA_API_KEY`

## Checklist
- `Netra.init()` is called exactly once.
- Initialization runs before provider/framework imports where required.
- API key headers are set in non-local environments.
- Chosen instruments match actual runtime dependencies.

## References
- https://docs.getnetra.ai/quick-start/QuickStart_Tracing
- https://docs.getnetra.ai/sdk-reference/sdk/overview
- https://docs.getnetra.ai/sdk-reference/sdk/typescript
- https://docs.getnetra.ai/Observability/Traces/configuration/initialization
- https://pypi.org/project/netra-sdk/
