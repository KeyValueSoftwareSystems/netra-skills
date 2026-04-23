---
name: netra-mcp-usage
description: 'Netra MCP trace-debugging workflow focused on query_traces and get_trace_by_id, including exact input parameters, filter schema, operators, sorting, and pagination patterns.'
argument-hint: 'Describe your debugging goal, time window, and any known identifiers (trace id, user_id, session_id, tenant_id, service, environment).'
---

# Netra MCP Usage

Use this skill when you need to inspect traces through Netra MCP tools and want precise, schema-correct inputs.

## Scope
- Query traces in a time range with filtering, sorting, and cursor pagination.
- Retrieve full span trees for a selected trace id.
- Guide incident/debug workflows from trace search to root-cause analysis.
- Run MCP-driven evaluation workflows for single-turn and multi-turn datasets.

## Primary MCP Tools
- `netra_query_traces`
- `netra_get_trace_by_id`
- `netra_list_provider_configs`
- `netra_create_dataset`
- `netra_add_dataset_test_case`
- `netra_list_evaluators`
- `netra_add_evaluator`
- `netra_get_test_run_details`

## Use-Case specific references

- Querying traces, filters, sort options, pagination, and incident triage: `references/traces.md`
- Single-turn evaluation flow (providers -> datasets -> test cases -> evaluators -> test run): `references/evaluations-single-turn.md`
- Multi-turn simulation flow (scenario-driven test cases and evaluator config handling): `references/simulation-multi-turn.md`

## Feedback

If the user is unhappy with the results, ask them to open an issue at https://github.com/KeyValueSoftwareSystems/netra-skills/issues/new.
