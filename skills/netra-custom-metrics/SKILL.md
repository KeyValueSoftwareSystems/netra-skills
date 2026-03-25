---
name: netra-custom-metrics
description: 'Enable Netra custom metrics with OpenTelemetry Meter instruments and production-safe export settings.'
argument-hint: 'Describe which KPIs to track, export interval requirements, and expected metric dimensions.'
---

# Netra Custom Metrics

Use this skill to instrument business and operational KPIs alongside Netra traces.

## When To Use
- You need request, latency, token, and cost metrics.
- You need queue depth or in-flight gauges.
- You want periodic metrics export via OTLP/HTTP JSON.

## Procedure
1. Initialize Netra with `enable_metrics=True`.
2. Get a meter via `Netra.get_meter("service-name")`.
3. Define instruments: counter, histogram, up-down counter, observable counter.
4. Record values with low-cardinality attributes.
5. Call `Netra.shutdown()` on process exit to flush final export.

## Python Pattern
```python
from netra import Netra

Netra.init(
    app_name="my-service",
    enable_metrics=True,
    metrics_export_interval_ms=30000,
)

meter = Netra.get_meter("my-service")
requests = meter.create_counter("llm.requests.total", unit="1")
errors = meter.create_counter("llm.errors.total", unit="1")
latency = meter.create_histogram("llm.latency_ms", unit="ms")
active = meter.create_up_down_counter("llm.active_requests", unit="1")


def call_model(model: str):
    active.add(1, {"model": model})
    try:
        # call provider
        requests.add(1, {"model": model, "status": "success"})
        latency.record(120, {"model": model})
    except Exception as exc:
        errors.add(1, {"model": model, "error_type": type(exc).__name__})
        raise
    finally:
        active.add(-1, {"model": model})
```

## TypeScript Usage Metrics Pattern
```typescript
import { Netra } from "netra-sdk-js";

const client = new Netra({ apiKey: process.env.NETRA_API_KEY! });

const sessionUsage = await client.usage.getSessionUsage(
    "session-123",
    "2026-01-01T00:00:00.000Z",
    "2026-01-31T23:59:59.000Z"
);

if (sessionUsage) {
    console.log(sessionUsage.tokenCount, sessionUsage.requestCount, sessionUsage.totalCost);
}
```

## Checklist
- Metrics are enabled in `Netra.init()`.
- Attribute dimensions are bounded and stable.
- Export interval is tuned for workload and cost.
- Shutdown flush is registered (`atexit` or explicit stop path).

## References
- https://docs.getnetra.ai/sdk-reference/sdk/overview
- https://docs.getnetra.ai/Observability/Traces/overview
- https://docs.getnetra.ai/sdk-reference/usage/typescript
