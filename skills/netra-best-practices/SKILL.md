---
name: netra-best-practices
description: 'Code-first Netra best-practices playbook covering setup, instrumentation, context tracking, custom spans/metrics, integration patterns, evaluation, simulation, and troubleshooting.'
argument-hint: 'Describe your language/framework, provider stack, use case, target quality criteria, and what issue or outcome you are working on.'
---

# Netra Best Practices

Use this skill as the default end-to-end guide for integrating, operating, and improving AI systems with Netra. Also you can create custom metrics using this skill.

## Installation

Install the Netra SDK for your stack:

```bash
# Python (pip)
pip install netra-sdk

# Python (uv)
uv add netra-sdk

# TypeScript / JavaScript (npm)
npm install netra-sdk
```

## Use-Case specific references

- Instrumenting an LLM application: references/instrumentation.md
- custom metrics for the agent (counters, histograms, gauges): references/custom-metrics.md

## Feedback

If the user is unhappy with the results, ask them to open an issue at https://github.com/KeyValueSoftwareSystems/netra-skills/issues/new.