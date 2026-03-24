---
name: netra-simulation-setup
description: 'Set up Netra multi-turn simulations with scenario goals, personas, fact checkers, evaluator configuration, and test-run analysis. Use to validate agent behavior before production.'
argument-hint: 'Describe your agent use case, scenario goals, personas, and key facts to verify.'
---

# Netra Simulation Setup

Use this skill to design realistic multi-turn simulation datasets and evaluate conversational agent behavior.

## When To Use
- You need to test multi-turn behavior, not only single-turn outputs.
- You want to compare agent performance across user personas.
- You need pre-production validation for goal achievement and factual correctness.

## Simulation Building Blocks
- Evaluators: Session-level scoring of conversation quality and outcomes.
- Multi-turn datasets: Goal, persona, max turns, user data, and facts.
- Test runs: Conversation transcript + evaluator results + trace links.

## Procedure
1. Select core simulation evaluators first:
   - Agentic: Goal Fulfillment, Information Elicitation.
   - Quality: Factual Accuracy, Conversation Completeness, Guideline Adherence.
2. Create a dataset with `Type = Multi-turn`.
3. Configure scenario:
   - goal,
   - persona (Neutral/Friendly/Frustrated/Confused/Custom),
   - max turns (start with 4-6 for support flows),
   - provider/model for simulated user.
4. Add simulated user data (table/json/text).
5. Add fact checker fields for must-communicate facts.
6. Map evaluator variables to scenario fields, agent replies, and conversation metadata.
7. Run simulations through your instrumented agent code using the dataset ID.
8. Review Test Runs:
   - pass/fail and evaluator scores,
   - exit reason (goal achieved/failed/abandoned/max turns),
   - turn-level traces for failures.

## Scenario Template
Use this template for each scenario:
- Goal: Clear user objective (single sentence).
- Persona: Behavioral style and emotional stance.
- Max Turns: Numeric limit (1-10).
- User Data: Structured context available to simulated user.
- Fact Checker: Critical facts the agent must communicate correctly.
- Success Conditions: Which evaluators must pass.

## Results Analysis Loop
1. Identify failing scenarios and failed evaluators.
2. Inspect conversation turn where behavior diverges.
3. Open trace for that turn and inspect tool/LLM behavior.
4. Classify failure cause: policy, retrieval, tool, prompt, or orchestration.
5. Apply fix, rerun the same dataset, and compare to baseline.

## Baseline Recommendations
- Start with 3-5 high-impact scenarios before scaling.
- Run the same scenarios across multiple personas.
- Keep a baseline run before each prompt/model release.
- Track success rate, average latency, cost, and turn efficiency over time.

## References
- https://docs.getnetra.ai/quick-start/QuickStart_Simulation
- https://docs.getnetra.ai/Simulation/Simulation-overview
- https://docs.getnetra.ai/Simulation/Datasets
- https://docs.getnetra.ai/Simulation/Evaluators
- https://docs.getnetra.ai/Simulation/TestRuns
