---
name: netra-mcp-traces
description: Query and inspect traces using Netra MCP query_traces and get_trace_by_id, with schema-correct filters, sorting, and cursor pagination.
---

# Netra MCP Traces

Use this reference when you need exact input structures and practical patterns for trace debugging with Netra MCP.

## Workflow

1. Start with a narrow time window and low limit.
2. Add the minimum filters needed to isolate relevant traces.
3. Sort for your objective (recent, slowest, most expensive, or errors).
4. Page through results using returned cursor values.
5. Fetch full spans for one trace id.
6. Inspect hierarchy, status, latency, and attributes.

## query_traces Input Schema

Required:
- `startTime` (string, ISO 8601)
- `endTime` (string, ISO 8601)

Optional:
- `limit` (number, 1-100, default 20)
- `cursor` (string)
- `direction` (`up` | `down`, default `down`)
- `sortField`
- `sortOrder` (`asc` | `desc`, default `desc`)
- `filters` (array of filter objects)

### sortField Values

- `latency_ms`
- `name`
- `total_cost`
- `has_pii`
- `has_violation`
- `start_time`
- `environment`
- `service`
- `has_error`
- `total_tokens`

### Filter Object Schema

Each filter object must include:
- `field`
- `value`
- `type`
- `operator`

Optional in filter object:
- `key` (for nested/object-style filtering)

#### field Values

- `name`
- `tenant_id`
- `user_id`
- `session_id`
- `environment`
- `service`
- `metadata`
- `projectIds`
- `project_id`
- `parent_span_id`
- `has_pii`
- `has_violation`
- `has_error`
- `models`
- `total_cost`
- `latency`

#### type Values

- `string`
- `number`
- `boolean`
- `arrayOptions`
- `attributeKey`
- `object`

#### operator Values

- `equals`
- `greater_than`
- `less_than`
- `greater_equal_to`
- `less_equal_to`
- `contains`
- `not_equals`
- `any_of`
- `none_of`
- `not_contains`
- `starts_with`
- `ends_with`
- `is_null`
- `is_not_null`

## Filter Patterns

- Error traces only:
	- `field: has_error`, `type: boolean`, `operator: equals`, `value: true`
- Specific session:
	- `field: session_id`, `type: string`, `operator: equals`, `value: <session-id>`
- High latency:
	- `field: latency`, `type: number`, `operator: greater_than`, `value: 3000`
- Service scoped:
	- `field: service`, `type: string`, `operator: equals`, `value: <service-name>`
- Metadata key/value:
	- `field: metadata`, `type: object`, `key: <metadata-key>`, `operator: equals`, `value: <value>`

## Pagination Pattern

1. Run `query_traces` without `cursor`.
2. Capture a `cursor` from returned trace items.
3. Re-run `query_traces` with the cursor and `direction: down`.
4. Continue while `pageInfo.hasNextPage` is true.

## get_trace_by_id Input Schema

Required:
- `traceId` (string)

Behavior:
- Returns complete span array for the trace id.
- Use this after `query_traces` to inspect one trace deeply.
- Invalid ids return a not-found style error.

## Incident Triage Recipe

1. Query for failing traces (`has_error=true`) in the incident window.
2. Sort by `latency_ms` desc to identify worst requests.
3. Pull one trace via `get_trace_by_id`.
4. Validate root span presence and parent-child span flow.
5. Check slow spans and tool/model metadata.
6. Compare with a nearby successful trace if needed.

## Practical Tips

- Keep initial windows short (5-30 minutes) for faster narrowing.
- Use one or two filters first, then add more only if needed.
- Prefer exact-match IDs (`session_id`, `user_id`, `tenant_id`) when available.
- Use `sortField=total_cost` to find expensive traces quickly.
- If no results: widen time range first, then relax filters.

## References

- https://docs.getnetra.ai/Observability/Traces
- https://docs.getnetra.ai/netra-mcp
