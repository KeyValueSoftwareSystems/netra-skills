# Netra Skills

A curated set of Copilot skill definitions for building, evaluating, and validating AI systems with Netra.

This repository currently includes skills for:
- Custom tracing instrumentation using decorators and span design
- High-quality evaluation setup for regression detection
- Multi-turn simulation design and analysis

## Repository Structure

- [skills/netra-decorator-instrumentation/SKILL.md](skills/netra-decorator-instrumentation/SKILL.md)
- [skills/netra-evaluation-setup/SKILL.md](skills/netra-evaluation-setup/SKILL.md)
- [skills/netra-simulation-setup/SKILL.md](skills/netra-simulation-setup/SKILL.md)

## Skills

### 1) Netra Decorator Instrumentation

File: [skills/netra-decorator-instrumentation/SKILL.md](skills/netra-decorator-instrumentation/SKILL.md)

Use this skill when you want clear semantic tracing for workflows, agents, and tasks, and when you need to choose between:
- Auto-instrumentation
- Decorators
- Manual tracing

Highlights:
- Python and TypeScript method guidance
- Decision flow for method selection
- Decorator semantics and verification checklist

### 2) Netra Evaluation Setup

File: [skills/netra-evaluation-setup/SKILL.md](skills/netra-evaluation-setup/SKILL.md)

Use this skill when you need repeatable quality measurement and regression detection.

Highlights:
- Dataset-first evaluation workflow
- LLM-as-Judge vs Code Evaluator guidance
- Variable mapping, threshold tuning, and test-run analysis

### 3) Netra Simulation Setup

File: [skills/netra-simulation-setup/SKILL.md](skills/netra-simulation-setup/SKILL.md)

Use this skill when validating agent behavior in multi-turn conversations before production.

Highlights:
- Scenario design (goal, persona, max turns, user data, facts)
- Evaluator selection for session-level scoring
- Failure analysis loop using transcripts and traces

## How To Use These Skills

1. Open Copilot Chat in VS Code.
2. Invoke a skill by name and pass your context, for example:
	- netra-decorator-instrumentation for tracing structure and semantic spans
	- netra-evaluation-setup for evaluator and dataset design
	- netra-simulation-setup for multi-turn scenario testing
3. Include concrete details in your prompt:
	- language (Python or TypeScript)
	- framework/provider stack
	- desired success criteria and known failure modes

## Recommended Workflow

1. Instrument first with tracing skills.
2. Build evaluation datasets and scoring criteria.
3. Stress test behavior with simulations.
4. Iterate using trace and test-run insights.

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
