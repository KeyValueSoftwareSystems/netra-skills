# Netra Skills

A curated set of Copilot skill definitions for integrating, securing, observing, evaluating, and validating AI systems with Netra.

This repository currently includes skills for:
- SDK setup and initialization
- Context tracking (user/session/tenant/conversation)
- PII and prompt-injection guardrails
- Manual custom spans, usage, and action records
- OpenTelemetry custom metrics through Netra
- Integration blueprints for common app architectures
- Tracing instrumentation using decorators and span design
- High-quality evaluation setup for regression detection
- Multi-turn simulation design and analysis
- Troubleshooting common integration failures

## Repository Structure

- [skills/netra-setup/SKILL.md](skills/netra-setup/SKILL.md)
- [skills/netra-context-tracking/SKILL.md](skills/netra-context-tracking/SKILL.md)
- [skills/netra-pii-and-input-guardrails/SKILL.md](skills/netra-pii-and-input-guardrails/SKILL.md)
- [skills/netra-custom-spans-and-actions/SKILL.md](skills/netra-custom-spans-and-actions/SKILL.md)
- [skills/netra-custom-metrics/SKILL.md](skills/netra-custom-metrics/SKILL.md)
- [skills/netra-integration-patterns/SKILL.md](skills/netra-integration-patterns/SKILL.md)
- [skills/netra-decorator-instrumentation/SKILL.md](skills/netra-decorator-instrumentation/SKILL.md)
- [skills/netra-evaluation-setup/SKILL.md](skills/netra-evaluation-setup/SKILL.md)
- [skills/netra-simulation-setup/SKILL.md](skills/netra-simulation-setup/SKILL.md)
- [skills/netra-troubleshooting/SKILL.md](skills/netra-troubleshooting/SKILL.md)

## Skills

### 1) Netra Setup

File: [skills/netra-setup/SKILL.md](skills/netra-setup/SKILL.md)

Use this skill for first-time SDK integration.

Highlights:
- Installation and initialization templates
- Environment variable guidance
- Startup and shutdown checklist

### 2) Netra Context Tracking

File: [skills/netra-context-tracking/SKILL.md](skills/netra-context-tracking/SKILL.md)

Use this skill when tracking user/session/tenant and conversation-level context.

Highlights:
- Request-bound ID setting
- Custom attributes and custom events
- Conversation input/output logging

### 3) Netra PII And Input Guardrails

File: [skills/netra-pii-and-input-guardrails/SKILL.md](skills/netra-pii-and-input-guardrails/SKILL.md)

Use this skill for input security and privacy controls.

Highlights:
- PII detection strategy
- Prompt-injection scanning
- Block/mask policy flow

### 4) Netra Custom Spans And Actions

File: [skills/netra-custom-spans-and-actions/SKILL.md](skills/netra-custom-spans-and-actions/SKILL.md)

Use this skill when you need explicit manual spans and side-effect tracking.

Highlights:
- Span lifecycle boundaries
- Usage and cost metadata
- Action event modeling

### 5) Netra Custom Metrics

File: [skills/netra-custom-metrics/SKILL.md](skills/netra-custom-metrics/SKILL.md)

Use this skill for KPI instrumentation with OpenTelemetry meters.

Highlights:
- Counter/histogram/up-down patterns
- Attribute cardinality guidance
- Export and flush practices

### 6) Netra Integration Patterns

File: [skills/netra-integration-patterns/SKILL.md](skills/netra-integration-patterns/SKILL.md)

Use this skill to apply architecture-specific integration playbooks.

Highlights:
- FastAPI/RAG/multi-agent/batch/streaming alignment
- Instrument selection baseline
- Request-flow checkpoints

### 7) Netra Decorator Instrumentation

File: [skills/netra-decorator-instrumentation/SKILL.md](skills/netra-decorator-instrumentation/SKILL.md)

Use this skill when you want clear semantic tracing for workflows, agents, and tasks, and when you need to choose between:
- Auto-instrumentation
- Decorators
- Manual tracing

Highlights:
- Python and TypeScript method guidance
- Decision flow for method selection
- Decorator semantics and verification checklist

### 8) Netra Evaluation Setup

File: [skills/netra-evaluation-setup/SKILL.md](skills/netra-evaluation-setup/SKILL.md)

Use this skill when you need repeatable quality measurement and regression detection.

Highlights:
- Dataset-first evaluation workflow
- LLM-as-Judge vs Code Evaluator guidance
- Variable mapping, threshold tuning, and test-run analysis

### 9) Netra Simulation Setup

File: [skills/netra-simulation-setup/SKILL.md](skills/netra-simulation-setup/SKILL.md)

Use this skill when validating agent behavior in multi-turn conversations before production.

Highlights:
- Scenario design (goal, persona, max turns, user data, facts)
- Evaluator selection for session-level scoring
- Failure analysis loop using transcripts and traces

### 10) Netra Troubleshooting

File: [skills/netra-troubleshooting/SKILL.md](skills/netra-troubleshooting/SKILL.md)

Use this skill to diagnose integration and telemetry visibility issues.

Highlights:
- Failure-mode triage flow
- Installation and extras checks
- Trace/metrics visibility checklist

## How To Use These Skills

1. Open Copilot Chat in VS Code.
2. Invoke a skill by name and pass your context, for example:
	- netra-setup for first-time integration
	- netra-context-tracking for identity/session/conversation enrichment
	- netra-pii-and-input-guardrails for security controls
	- netra-custom-metrics for KPI instrumentation
	- netra-decorator-instrumentation for semantic span structure
	- netra-evaluation-setup for evaluator and dataset design
	- netra-simulation-setup for multi-turn scenario testing
3. Include concrete details in your prompt:
	- language (Python or TypeScript)
	- framework/provider stack
	- desired success criteria and known failure modes

## Recommended Workflow

1. Set up SDK, instruments, and environment configuration.
2. Add context tracking and security guardrails.
3. Add decorators/manual spans and custom metrics.
4. Build evaluation datasets and scoring criteria.
5. Stress test behavior with simulations.
6. Iterate using trace and test-run insights.

## Source Documentation

These skills are grounded in Netra docs and SDK references:
- Tracing: https://docs.getnetra.ai/Observability/Traces/overview
- Decorators: https://docs.getnetra.ai/Observability/Traces/decorators
- Manual Tracing: https://docs.getnetra.ai/Observability/Traces/manual-tracing
- Evaluation: https://docs.getnetra.ai/Evaluation/Evaluation-overview
- Simulation: https://docs.getnetra.ai/Simulation/Simulation-overview
- SDK Overview: https://docs.getnetra.ai/sdk-reference/sdk/overview

## Contributing New Skills

Add a new folder under skills with a SKILL.md file that includes:
- YAML frontmatter with name and description
- Clear trigger conditions for when the skill should be used
- Step-by-step procedure
- Verification checklist and references

Keep each skill focused on one workflow and avoid mixing unrelated responsibilities.
