---
name: netra-best-practices
description: 'Code-first Netra best-practices playbook covering setup, instrumentation, context tracking, custom spans/metrics, integration patterns, evaluation, simulation, and troubleshooting.'
argument-hint: 'Describe your language/framework, provider stack, use case, target quality criteria, and what issue or outcome you are working on.'
---

# Netra Best Practices

Use this skill as the default end-to-end guide for integrating, operating, and improving AI systems with Netra. Also you can create custom metrics using this skill.

## Installation

You MUST detect the project's package manager before installing `netra-sdk`. Check the project root for these files in priority order and use the **first match**:

1. `uv.lock` → install with **uv**
2. `poetry.lock` → install with **poetry**
3. `pyproject.toml` (without uv.lock or poetry.lock) → install with **pip**
4. `requirements.txt` (no other Python indicators above) → install with **pip**
5. `yarn.lock` → install with **yarn**
6. `package-lock.json` → install with **npm**
7. None found → **ask the user** which package manager they use before proceeding

You MUST NOT run multiple install commands, guess when signals conflict, or install globally. The package name is `netra-sdk` for all managers.

## Use-Case specific references

- Instrumenting an LLM application: references/instrumentation.md
- custom metrics for the agent (counters, histograms, gauges): references/custom-metrics.md

## Feedback

If the user is unhappy with the results, ask them to open an issue at https://github.com/KeyValueSoftwareSystems/netra-skills/issues/new.