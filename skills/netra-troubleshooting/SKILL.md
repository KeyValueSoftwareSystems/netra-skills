---
name: netra-troubleshooting
description: 'Diagnose and fix common Netra SDK integration failures for initialization, imports, traces, and metrics.'
argument-hint: 'Describe the exact error, environment, installed extras, and whether traces/metrics are missing or partial.'
---

# Netra Troubleshooting

Use this skill to resolve common integration issues quickly and systematically.

## Common Problems
- `ModuleNotFoundError` for `netra`.
- Missing extras for Presidio or prompt-injection scanner.
- `Netra.init()` called multiple times.
- Traces or metrics not visible in dashboard.
- Async or streaming flow not appearing as expected.

## Triage Flow
1. Confirm package installation and Python version compatibility.
2. Confirm one-time initialization happens before provider imports.
3. Verify API key/header and OTLP endpoint configuration.
4. Verify enabled instruments match runtime libraries.
5. Confirm network egress and collector compatibility.
6. Confirm shutdown flush path for final telemetry.

## Quick Fix Commands
```bash
pip install netra-sdk
pip install 'netra-sdk[presidio]'
pip install 'netra-sdk[llm_guard]'
npm install netra-sdk
```

## TypeScript Initialization Check
```typescript
import { Netra } from "netra-sdk";

await Netra.init({
	appName: "my-app",
	headers: `x-api-key=${process.env.NETRA_API_KEY}`,
});

await Netra.shutdown();
```

## Debug Checklist
- `Netra.is_initialized()` returns `True` after startup.
- `trace_content` is configured for the environment policy.
- Metrics enabled with `enable_metrics=True` where expected.
- Export interval and shutdown flush are configured.
- No duplicate app initializers across worker processes.

## References
- https://docs.getnetra.ai/FAQs/FAQs
- https://docs.getnetra.ai/quick-start/QuickStart_Tracing
- https://docs.getnetra.ai/sdk-reference/sdk/overview
- https://docs.getnetra.ai/sdk-reference/sdk/typescript
