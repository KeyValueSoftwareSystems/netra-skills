# Multi-Turn Simulations (Netra SDK)

Use this reference when the user asks to test an AI agent with goal-oriented, multi-turn conversations.

## Outcome

Set up simulation runs where:
- A multi-turn dataset defines realistic scenarios.
- A task wrapper calls the user's agent each turn.
- Netra evaluates whole conversations with simulation evaluators.

## Prerequisites

1. Netra SDK installed and initialized.
2. Multi-turn dataset created in Netra dashboard.
3. Dataset includes scenario goal, max turns, persona, user data, and fact checker data.
4. Simulation evaluators selected (start with Goal Fulfillment + Factual Accuracy).

## Recommended workflow for the agent

1. Confirm the target agent entry point (how to send one message and get one reply).
2. Verify Netra init happens before simulation execution.
3. Implement/extend `BaseTask` wrapper around the user's agent.
4. Ensure session continuity is preserved (`session_id` / `sessionId`).
5. Run simulation with conservative concurrency.
6. Report completed/failed counts and where to inspect results.

## Python template

```python
from netra import Netra
from netra.simulation.task import BaseTask
from netra.simulation.models import TaskResult
from uuid import uuid4

Netra.init(
		app_name="my-app",
		headers="x-api-key=YOUR_NETRA_API_KEY",
)

class MyAgentTask(BaseTask):
		def __init__(self, agent):
				self.agent = agent

		def run(self, message: str, session_id: str | None = None) -> TaskResult:
				# Netra can call the first turn with session_id=None.
				# Generate a fresh per-conversation session id in that case.
				sid = session_id or str(uuid4())
				# Replace with the user's real agent call
				reply = self.agent.chat(message, session_id=sid)
				return TaskResult(
						message=reply,
						session_id=sid,
				)

result = Netra.simulation.run_simulation(
		name="Customer Support Simulation",
		dataset_id="your-multi-turn-dataset-id",
		task=MyAgentTask(agent=...),
		context={"environment": "staging"},
		max_concurrency=3,
)

print("completed:", len(result["completed"]))
print("failed:", len(result["failed"]))
```

## TypeScript template

```typescript
import { Netra } from "netra-sdk";
import { BaseTask, TaskResult } from "netra-sdk/simulation";

await Netra.init({
	appName: "my-app",
	headers: `x-api-key=${process.env.NETRA_API_KEY}`,
});

class MyAgentTask extends BaseTask {
	constructor(private agent: any) {
		super();
	}

	async run(message: string, sessionId?: string | null): Promise<TaskResult> {
		// Netra can call the first turn with sessionId=null/undefined.
		// Generate a fresh per-conversation session id in that case.
		const sid = sessionId ?? crypto.randomUUID();
		// Replace with the user's real agent call
		const reply = await this.agent.chat(message, { sessionId: sid });
		return {
			message: String(reply?.text ?? reply ?? ""),
			sessionId: sid,
		};
	}
}

const result = await Netra.simulation.runSimulation({
	name: "Customer Support Simulation",
	datasetId: "your-multi-turn-dataset-id",
	task: new MyAgentTask(/* agent */),
	context: { environment: "staging" },
	maxConcurrency: 3,
});

console.log("completed:", result?.completed.length ?? 0);
console.log("failed:", result?.failed.length ?? 0);
```

## Dataset design guidance

When instructing users to create datasets in the dashboard, include:
1. Clear scenario goal (what success looks like).
2. Realistic max turns (support: 4-6 is a good default).
3. Persona fit (neutral/friendly/frustrated/confused/custom).
4. Simulated user data (context the simulator can reference).
5. Fact checker values (critical facts that must be correct).

## Evaluator guidance

Simulation evaluators are session-level (whole conversation).

Start with:
- `Goal Fulfillment`
- `Factual Accuracy`

Then add as needed:
- `Conversation Completeness`
- `Guideline Adherence`
- `Conversational Flow`
- `Conversation Memory`
- `Profile Utilization`
- `Information Elicitation`

## What to check after setup

1. Simulation run starts and returns totals.
2. Most conversations land in `completed`.
3. Failures include actionable `error` and `turn_id`/`turnId`.
4. Evaluation scores appear under Test Runs for each scenario.
5. Per-turn traces are available for debugging.

## Troubleshooting guidance

- Repeated session resets: verify you propagate returned session id every turn.
- High failure count: reduce concurrency and inspect first failing turn trace.
- Unrealistic simulations: improve scenario goal/persona/user-data quality.
- Weak signal: raise evaluator pass thresholds once baseline quality is stable.

## References

- https://docs.getnetra.ai/quick-start/QuickStart_Simulation
- https://docs.getnetra.ai/Simulation/Datasets
- https://docs.getnetra.ai/Simulation/Evaluators
- https://docs.getnetra.ai/sdk-reference/simulation/python
- https://docs.getnetra.ai/sdk-reference/simulation/typescript
