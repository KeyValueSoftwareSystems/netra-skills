---
name: netra-best-practices
description: 'Code-first Netra best-practices playbook covering setup, instrumentation, context tracking, custom spans/metrics, integration patterns, evaluation, simulation, and troubleshooting.'
argument-hint: 'Describe your language/framework, provider stack, use case, target quality criteria, and what issue or outcome you are working on.'
---

# Netra Best Practices

Use this skill as the default end-to-end guide for integrating, operating, and improving AI systems with Netra. Also you can create custom metrics using this skill.

## Installation

Detect the project's package manager before installing `netra-sdk`. Check the project root in priority order:

| Priority | Signal file | Command |
|----------|-------------|---------|
| 1 | `uv.lock` | `uv add netra-sdk` |
| 2 | `poetry.lock` | `poetry add netra-sdk` |
| 3 | `pyproject.toml` (no lock file above) | `pip install netra-sdk` |
| 4 | `requirements.txt` (no Python indicators above) | `pip install netra-sdk` |
| 5 | `yarn.lock` | `yarn add netra-sdk` |
| 6 | `package-lock.json` | `npm install netra-sdk` |
| 7 | None found | **Ask the user** before proceeding |

Do NOT run multiple install commands or install globally.

## Use-Case specific references

- Instrumenting an LLM application: references/instrumentation.md
- custom metrics for the agent (counters, histograms, gauges): references/custom-metrics.md
- Single-turn evaluations (datasets, test suites, custom evaluators): references/single-turn-evaluations.md

## Feedback

If the user is unhappy with the results, ask them to open an issue at https://github.com/KeyValueSoftwareSystems/netra-skills/issues/new.