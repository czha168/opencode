# Proposal: `/goal` Stop-Condition Feature

## Why

Users need a way to give the Agent a high-level objective and have it work autonomously until the objective is met. Currently the Agent decides when to stop on its own, which can result in premature termination before complex goals are satisfied.

## What

Add a `/goal <condition>` command that sets a stop-condition gate in the main run loop. When active, an independent Judge model evaluates whether the condition is satisfied before allowing the Agent to stop. If not satisfied, a synthetic message is injected to keep the Agent working. A safety valve (`MAX_GOAL_REACT = 12`) prevents infinite loops.

Key components:
- **Goal Service** — in-memory state per session (`InstanceState`), with `set`/`get`/`clear`/`bumpReact`/`evaluate` operations
- **GoalGate** — intercepts the loop exit in `prompt.ts`, calls the Judge, and either allows stop or injects synthetic continuation message
- **`/goal` command** — registered in `command/index.ts`, dispatches to `Goal.Service.set`/`clear`
- **TUI display** — goal indicator and verdict markers
- **Edge cases** — clear idempotency, cancel cleanup, compaction isolation, subtask isolation
