# Netra Cursor Plugin

Netra Cursor Plugin brings Netra observability workflows into your AI coding environment through:

- An MCP server connection to Netra APIs.
- A bundled skill for Netra best practices.
- A lightweight plugin manifest for distribution.

This repository is designed to be used as a Cursor-compatible plugin package.

## What Is Included

- `mcp.json` configures the Netra MCP server endpoint and authentication header.
- `skills/netra-best-practices/SKILL.md` provides a code-first best-practices playbook for setup, tracing, context tracking, evaluations, simulations, and troubleshooting.
- `skills/netra-mcp-usage/SKILL.md` provides a focused MCP trace-debugging workflow with query/filter/schema guidance.
- `.cursor-plugin/plugin.json` contains plugin metadata (name, version, description, keywords, logo).
- `assets/` contains plugin assets such as the logo.

## Prerequisites

- A Netra account.
- A Netra API key.
- Cursor (or compatible agent environment with MCP + skill support).

## Quick Start

1. Clone this repository.

	 ```bash
	 git clone https://github.com/<your-org>/netra-cursor-plugin.git
	 cd netra-cursor-plugin
	 ```

2. Export your Netra API key and MCP url.

	 ```bash
	 export NETRA_API_KEY="your-netra-api-key"
     export NETRA_MCP_URL="https://api.getnetra.ai/mcp" # URL based on your OTLP endpoint
	 ```

3. Ensure your MCP config includes the Netra server from `mcp.json`:

	 ```json
	 {
		 "mcpServers": {
			 "netra": {
				 "url": "${NETRA_MCP_URL}",
				 "headers": {
					 "x-api-key": "${NETRA_API_KEY}"
				 }
			 }
		 }
	 }
	 ```

4. Reload your editor/agent so the MCP server and skills are discovered.

## Using The Netra Skill

This plugin includes two complementary skills:

- `netra-best-practices` for integration and production best practices.
- `netra-mcp-usage` for MCP trace querying and span-level debugging.

`netra-best-practices` helps with:

- Tracing setup and instrumentation patterns.
- Request context tracking (`user_id`, `session_id`, `tenant_id`).
- Span design using auto-instrumentation, decorators, or manual spans.
- Evaluation and simulation workflows for quality regression prevention.

`netra-mcp-usage` helps with:

- Correct input construction for `mcp_netra_netra_query_traces` and `mcp_netra_netra_get_trace_by_id`.
- Filter fields, operators, and type combinations.
- Cursor pagination and sort strategies for investigations.

Typical prompts:

- "Use Netra best practices to instrument this FastAPI app."
- "Analyze a failing trace with Netra MCP and identify the root cause."
- "Set up Netra evaluations for this retrieval workflow."
