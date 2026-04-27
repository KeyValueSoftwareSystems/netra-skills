---
name: netra-mcp-usage
description: 'Netra MCP trace-debugging workflow focused on query_traces, get_trace_by_id, and get_session_details, including exact input parameters, filter schema, operators, sorting, and pagination patterns.'
argument-hint: 'Describe your debugging goal, time window, and any known identifiers (trace id, session_id, user_id, tenant_id, service, environment).'
---

# Netra MCP Usage

Use this skill when you need to inspect traces through Netra MCP tools and want precise, schema-correct inputs.

## Scope
- Query traces in a time range with filtering, sorting, and cursor pagination.
- Retrieve full span trees for a selected trace id.
- Retrieve session-level details and all traces belonging to a session.
- Guide incident/debug workflows from trace search to root-cause analysis.

## Primary MCP Tools
- `netra_query_traces`
- `netra_get_trace_by_id`
- `netra_get_session_details`

## Workflow
1. Start with a narrow time window and low limit.
2. Add the minimum filters needed to isolate relevant traces.
3. Sort for your objective (recent, slowest, most expensive, errors).
4. Page through results using returned cursor values.
5. Fetch full spans for one trace id.
6. Inspect hierarchy, status, latency, and attributes.
7. Use `get_session_details` when you need all traces for a session or want session-level cost/token totals.

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

## get_session_details Input Schema
Provide exactly one of:
- `session_id` (string, optional): The session ID to retrieve.
- `trace_id` (string, optional): A trace ID belonging to the target session. The parent session will be resolved automatically.

Providing neither or both is an error.

Optional:
- `startTime` (string, ISO 8601): Start of the time range. Defaults to the organization's data retention window.
- `endTime` (string, ISO 8601): End of the time range. Defaults to now.
- `include_raw_inputs` (boolean, default false): If true, include full span details for each trace. Can produce large responses.
- `limit` (number, 1-100, default 50): Max traces to return per page.
- `cursor` (string): Pagination cursor from a previous response.

### Behavior Notes
- Traces within the session are sorted by `start_time` ascending.
- `status` is `errored` if any trace in the session has `has_error=true`, otherwise `completed`.
- `totals` aggregate cost, input tokens, and output tokens across all returned traces.
- Pagination works identically to `query_traces`: pass the returned `cursor` to fetch the next page.

### Session Lookup Patterns
- By session ID:
  - `session_id: <session-id>`
- By trace ID (auto-resolves to its parent session):
  - `trace_id: <trace-id>`
- With full span trees:
  - `session_id: <session-id>`, `include_raw_inputs: true`

## Incident Triage Recipe
1. Query for failing traces (`has_error=true`) in the incident window.
2. Sort by `latency_ms` desc to identify worst requests.
3. Pull one trace via `get_trace_by_id`.
4. Validate root span presence and parent-child span flow.
5. Check slow spans and tool/model metadata.
6. Compare with a nearby successful trace if needed.
7. If the failing trace belongs to a multi-turn session, call `get_session_details` with the `trace_id` to see all related traces, aggregate cost, and whether other turns also errored.

## Practical Tips
- Keep initial windows short (5-30 minutes) for faster narrowing.
- Use one or two filters first, then add more only if needed.
- Prefer exact-match IDs (`session_id`, `user_id`, `tenant_id`) when available.
- Use `sortField=total_cost` to find expensive traces quickly.
- If no results: widen time range first, then relax filters.
- When you have a `trace_id` but need the full session context, use `get_session_details` with `trace_id` instead of manually filtering `query_traces` by `session_id`.
- Use `include_raw_inputs: true` sparingly — it fetches full span trees for every trace in the session, which can be very large.
- Check `session.status` to quickly determine whether any trace in the session errored without scanning each trace individually.
