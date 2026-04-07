---
name: netra-custom-metrics
description: Emit custom metrics using the Netra Meter API. Use when tracking application-specific counters, histograms, gauges, or any numeric signal beyond what auto-instrumentation provides.
---

# Netra Custom Metrics

Emit and export application-specific metrics using the OpenTelemetry-backed `Meter` API exposed by the Netra SDK.

## Workflow

- Ensure `Netra.init()` is called with `enable_metrics=True` **before** creating any instruments or calling `Netra.get_meter()`.
- Identify which instrument type best fits each signal (counter, histogram, up-down counter, or observable gauge).
- Define instruments once at module/service level; record measurements inside request or event handlers.
- **Always** call `Netra.shutdown()` at the end of the application lifecycle to ensure metrics are flushed to the backend.
- Verify exports are reaching the backend using the Netra dashboard or OTLP endpoint logs.

# Initialization

Enable the metrics pipeline in `Netra.init` and obtain a `Meter` scoped to your service or module.

```env
NETRA_API_KEY=
NETRA_OTLP_ENDPOINT=
```

```py
import os
from netra import Netra

Netra.init(
    app_name="my-ai-app",
    environment="production",
    headers=f"x-api-key={os.getenv('NETRA_API_KEY')}",
    enable_metrics=True,
    metrics_export_interval_ms=10000,  # export every 10 s
)

# Recommended: use your service or module name
meter = Netra.get_meter("my_service")
```

> [!IMPORTANT]
> **Initialization Order Matters:** You must call `Netra.init()` before calling `Netra.get_meter()`. If you define instruments at the module level (standard practice), ensure the module is imported **after** `Netra.init()` has been executed in your application's entry point (e.g., `main.py`).

`get_meter` accepts a `name` (instrumentation scope, defaults to `"netra"`) and an optional `version` string. If `enable_metrics` is `False` or no OTLP endpoint is configured, a no-op `MeterProvider` is installed — your code can still call `get_meter` and record metrics safely, they will simply be discarded.

# Counter

Monotonically increasing value — use for request counts, completed jobs, or any value that only goes up.

```py
request_counter = meter.create_counter(
    name="http.requests",
    description="Number of HTTP requests processed",
    unit="1",
)

request_counter.add(1, attributes={"route": "/api/health", "status": "ok"})
```

# UpDownCounter

Value that can increase or decrease — use for active connections, queue depth, or in-flight requests.

```py
active_connections = meter.create_up_down_counter(
    name="connections.active",
    description="Number of active client connections",
    unit="1",
)

active_connections.add(1,  attributes={"region": "us-east-1"})  # connection opened
active_connections.add(-1, attributes={"region": "us-east-1"})  # connection closed
```

# Histogram

Distribution of measurements — use for latency, payload sizes, token counts, or any value where the distribution matters.

```py
latency = meter.create_histogram(
    name="db.query.latency_ms",
    description="Database query latency",
    unit="ms",
)

latency.record(15.3, attributes={"operation": "read",  "table": "users"})
latency.record(30.7, attributes={"operation": "write", "table": "orders"})
```

# Observable Instruments

Use observable (pull-based) instruments for system-level or slowly-changing values. Netra invokes registered callbacks periodically on each export cycle.

```py
import psutil
from opentelemetry.metrics import Observation

def _cpu_callback(options):
    yield Observation(psutil.cpu_percent(), attributes={"resource": "cpu"})

def _memory_callback(options):
    mem = psutil.virtual_memory()
    yield Observation(mem.used, attributes={"unit": "bytes"})

meter.create_observable_gauge(
    name="system.cpu.utilization",
    description="CPU utilization percentage",
    callbacks=[_cpu_callback],
)

meter.create_observable_gauge(
    name="system.memory.used_bytes",
    description="Used memory in bytes",
    callbacks=[_memory_callback],
)
```

*NOTE*: Callbacks must be fast and non-blocking. Heavy I/O inside a callback will stall the export cycle.



> [!WARNING]
> **Flush Your Metrics:** Always call `Netra.shutdown()` on graceful termination. Metrics are buffered and exported in batches (defined by `metrics_export_interval_ms`). If the application exits without a shutdown call, the last batch of metrics will likely be lost.

## Recommended instrument strategy

1. Use **Counter** for events that only accumulate (requests, errors, retries).
2. Use **Histogram** for any measurement whose distribution matters (latency, token count, payload size).
3. Use **UpDownCounter** for values that rise and fall (connections, queue depth).
4. Use **Observable Gauge** for asynchronous system-level signals (CPU, memory) polled by the SDK.

## Validation checklist

1. `enable_metrics=True` is passed to `Netra.init()`.
2. `Netra.get_meter()` and instrument creation happen **strictly after** `Netra.init()`.
3. Instruments are defined once at module/service level, not inside hot loops.
4. If instruments are in a separate module, that module is imported **after** `Netra.init()` in the entry point.
5. `attributes` dicts use only low-cardinality keys — avoid user IDs or request IDs as metric labels.
6. `Netra.shutdown()` is called on graceful termination (e.g., in a `finally` block or FastAPI `lifespan`) to flush the last export batch.
7. Metrics appear in the Netra dashboard within one export interval.

## References

- https://docs.getnetra.ai/sdk-reference/custom-metric
- https://docs.getnetra.ai/Observability/Traces/configuration/initialization
- https://opentelemetry.io/docs/specs/otel/metrics/api/
