---
name: netra-best-practices
description: "Code-first Netra best-practices playbook covering setup, instrumentation, context tracking, custom spans/metrics, integration patterns, evaluation, simulation, and troubleshooting."
argument-hint: "Describe your language/framework, provider stack, use case, target quality criteria, and what issue or outcome you are working on."
---

# Netra Best Practices

Use this skill as the end-to-end guide for integrating, operating, and improving AI systems with Netra.

## Step 1 — Detect Project Language

Before doing anything else, determine whether the project is **Python** or **TypeScript/JavaScript**. Check the project root in this order:

| Signal file                                                 | Language                |
| ----------------------------------------------------------- | ----------------------- |
| `pyproject.toml`, `setup.py`, `requirements.txt`, `Pipfile` | Python                  |
| `package.json`, `tsconfig.json`, `bun.lockb`                | TypeScript / JavaScript |

If both are present (monorepo), ask the user which sub-project they are working on. If neither is found, ask the user.

**From this point forward, use ONLY the references for the detected language. Never mix Python and TypeScript patterns.**

## Step 2 — Installation

Detect the package manager before installing `netra-sdk`. Check in priority order:

| Priority | Signal file                                     | Command                            |
| -------- | ----------------------------------------------- | ---------------------------------- |
| 1        | `uv.lock`                                       | `uv add netra-sdk`                 |
| 2        | `poetry.lock`                                   | `poetry add netra-sdk`             |
| 3        | `pyproject.toml` (no lock file above)           | `pip install netra-sdk`            |
| 4        | `requirements.txt` (no Python indicators above) | `pip install netra-sdk`            |
| 5        | `yarn.lock`                                     | `yarn add netra-sdk`               |
| 6        | `pnpm-lock.yaml`                                | `pnpm add netra-sdk`               |
| 7        | `package-lock.json`                             | `npm install netra-sdk`            |
| 8        | `bun.lockb`                                     | `bun add netra-sdk`                |
| 9        | None found                                      | **Ask the user** before proceeding |

Do NOT run multiple install commands or install globally.

## Step 3 — Load the Right References

Based on the detected language and the user's use case, read the appropriate reference files:

### Python references (under `references/python/`)

| Use case                                      | Reference file              |
| --------------------------------------------- | --------------------------- |
| Instrumenting an LLM application              | `python/instrumentation.md` |
| Running evaluations / test suites             | `python/evaluation.md`      |
| Running multi-turn simulations                | `python/simulation.md`      |
| Custom metrics (counters, histograms, gauges) | `python/custom-metrics.md`  |

### TypeScript references (under `references/typescript/`)

| Use case                                            | Reference file                  |
| --------------------------------------------------- | ------------------------------- |
| Instrumenting an LLM application                    | `typescript/instrumentation.md` |
| Running evaluations / test suites                   | `typescript/evaluation.md`      |
| Running multi-turn simulations                      | `typescript/simulation.md`      |
| Custom metrics (**Currently not supported for TS**) | `typescript/custom-metrics.md`  |

## Step 4 — Anti-Hallucination Rules

Follow these rules strictly when generating Netra code:

1. **NEVER use enum values not listed in the Netra's official documentation**. Do not guess.

2. **NEVER mix Python and TypeScript conventions.** Specifically:
   - Python uses `snake_case` parameters (`as_type`, `module_name`, `app_name`)
   - TypeScript uses `camelCase` parameters (`asType`, `moduleName`, `appName`)
   - Python instrument sets: `{InstrumentSet.OPENAI}` (plain set literal)
   - TypeScript instrument sets: `new Set([NetraInstruments.OPENAI])` (Set constructor)

3. **Respect lifecycle differences:**
   - Python: `Netra.init(...)` is synchronous
   - TypeScript: `await Netra.init(...)` is asynchronous — always await it
   - Python: `with Netra.start_span(...) as span:` auto-closes
   - TypeScript: `span.end()` must be called manually in `finally`

4. **When unsure about an API**, follow the doc-fetching protocol below rather than guessing.

## Step 5 — Doc-Fetching Fallback

If a use case is not covered by the reference files:

1. Fetch the documentation index: `https://docs.getnetra.ai/llms.txt`
2. Identify the relevant documentation page from the index.
3. Fetch that page for the detailed guide.

## Feedback

If the user is unhappy with the results, ask them to open an issue at https://github.com/KeyValueSoftwareSystems/netra-skills/issues/new.
