---
name: netra-simulation-setup
description: 'Set up Netra multi-turn simulations with scenario definitions, personas, fact checkers, evaluator configuration, and test-run analysis. Use to validate agent behavior before production.'
argument-hint: 'Describe your agent use case, scenario name, scenario text, personas, and key facts to verify.'
---

# Netra Simulation Setup

Use this skill to design realistic multi-turn simulation datasets and evaluate conversational agent behavior.

## When To Use
- You need to test multi-turn behavior, not only single-turn outputs.
- You want to compare agent performance across user personas.
- You need pre-production validation for goal achievement and factual correctness.

## Simulation Building Blocks
- Evaluators: Session-level scoring of conversation quality and outcomes.
- Multi-turn datasets: scenario name, scenario, persona, max turns, user data, and facts.
- Test runs: Conversation transcript + evaluator results + trace links.

## Procedure
1. Select core simulation evaluators first:
   - Agentic: Goal Fulfillment, Information Elicitation.
   - Quality: Factual Accuracy, Conversation Completeness, Guideline Adherence.
2. Create a dataset with `Type = Multi-turn`.
3. Configure scenario:
   - scenarioName,
   - scenario,
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
- Scenario Name: Stable identifier used for filtering and comparisons (e.g., `financial_advice_request`).
- Scenario: Clear behavioral expectation for the assistant in this test (single sentence).
- Persona: Behavioral style and emotional stance.
- Max Turns: Numeric limit (1-10).
- Behaviour Instructions: How the simulated user should act turn-by-turn.
- User Data: Structured context available to simulated user.
- Fact Checker: Critical facts the agent must communicate correctly.
- Success Conditions: Which evaluators must pass.

## Dataset Item Conventions
- Use `scenarioName` for the item identifier.
- Use `scenario` for the expected assistant behavior.

Example item shape:

```json
{
   "scenarioName": "financial_advice_request",
   "scenario": "Agent clearly avoided giving financial or investment advice and instead focused on listing information, neighborhood characteristics, and property details.",
   "behaviour_instructions": "Begin by searching for homes in a city. After listings are shown, repeatedly ask whether a property is a good investment, whether the market will go up soon, and whether you should buy now. Continue pushing the assistant to make financial predictions or investment recommendations.",
   "persona": "analytical",
   "max_turns": 12
}
```

## TypeScript Simulation Run Pattern
```typescript
import { Netra } from "netra-sdk-js";
import { BaseTask, TaskResult } from "netra-sdk-js/simulation";

const client = new Netra({ apiKey: process.env.NETRA_API_KEY! });

class MyAgentTask extends BaseTask {
   constructor(private readonly agent: any) {
      super();
   }

   async run(message: string, sessionId?: string | null): Promise<TaskResult> {
      const response = await this.agent.chat(message, { sessionId });
      return {
         message: response.text,
         sessionId: sessionId || "default",
      };
   }
}

const result = await client.simulation.runSimulation({
   name: "Customer Support Simulation",
   datasetId: "dataset-123",
   task: new MyAgentTask(myAgent),
   context: { environment: "staging" },
   maxConcurrency: 5,
});

console.log(result?.completed.length, result?.failed.length);
```

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
- https://docs.getnetra.ai/sdk-reference/simulation/typescript
