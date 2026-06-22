---
change: goal-stop-condition
design-doc: docs/goal-implementation-plan.md
base-ref: f50e4accf3860a0297c70a1833fdc02782deb0e1
---

# Implementation Plan: `/goal` Stop-Condition Feature

Derived from `docs/goal-implementation-plan.md`.

## Task 1: Create `goal.ts` — Types, Schemas, Events, Prompts

**File**: `packages/opencode/src/session/goal.ts` (new)

- Define `Goal` type: `{ condition: string; react: number }`
- Define `Verdict` schema using `Schema.Struct` (not Zod): `{ ok: Boolean, impossible?: Boolean, reason: String }`
- Define `Event.Updated` using `EventV2.define` with type `"session.goal"`, schema containing `sessionID`, `goal?`, `lastVerdict?`
- Define Judge prompt constants: `JUDGE_SYSTEM` and `judgeUser(condition)`
- Define `MAX_GOAL_REACT = 12`
- Self-reexport: `export * as Goal from "./goal"`

## Task 2: Implement Goal.Service — State + Operations

**File**: `packages/opencode/src/session/goal.ts` (extend)

- Define `Interface` with `set`, `get`, `clear`, `bumpReact`, `evaluate`
- Define `EvaluateInput` type: `{ condition, msgs: SessionV1.WithParts[], model: Provider.Model }`
- Implement `InstanceState.make` with `Map<SessionID, Goal>` (same pattern as `session/status.ts:65-67`)
- Implement `set(sessionID, condition)`: store goal with `react: 0`, publish `Event.Updated`
- Implement `get(sessionID)`: return from Map
- Implement `clear(sessionID)`: delete from Map, publish `Event.Updated` with `goal: undefined`
- Implement `bumpReact(sessionID)`: increment react, return new value
- Implement `evaluate(input)`:
  1. Convert msgs via `MessageV2.toModelMessagesEffect(msgs, model)`
  2. Call `generateObject` with `temperature: 0`, system+user+conversation messages, `schema: Verdict` (via `Schema.toStandardSchemaV1` + `Schema.toStandardJSONSchemaV1`)
  3. Dual-path: standard → `generateObject(params).then(r => r.object)`; OpenAI OAuth → `streamObject({...params, providerOptions})` then `result.object`
  4. Fail-open: catch errors → return `{ ok: true, reason: "judge error" }`
- Wire `Service` class via `Context.Service`, create `layer` and `defaultLayer`/`node` exports

## Task 3: Register `/goal` Command

**File**: `packages/opencode/src/command/index.ts` (modify)

- Add `GOAL: "goal"` to `Default` object
- Register command in `init` function: `name: "goal"`, `description: "set a stop-condition goal..."`, `source: "command"`, `subtask: false`, `template: "$ARGUMENTS"`
- Handle dispatch in `SessionPrompt.command()` at `prompt.ts`:
  - When `input.command === "goal"`:
    - If `input.arguments.trim() === "clear"` → `goal.clear(sessionID)`
    - Else → `goal.set(sessionID, input.arguments.trim())`
  - Return lightweight confirmation message

## Task 4: Add goalGate to prompt.ts Run Loop

**File**: `packages/opencode/src/session/prompt.ts` (modify)

- Import `Goal` from `session/goal`
- Add `Goal.Service` to `yield*` dependencies in layer (~line 99-127)
- Add `Goal.defaultLayer` to `defaultLayer` providers
- Add `Goal.node` to `node` exports
- Implement `goalGate` function inside the layer `Effect.gen` closure:
  - Gate only fires for `agentID === "main"`
  - If no active goal → return `false` (allow stop)
  - Filter compacted messages from transcript
  - Call `goal.evaluate()` with fail-open catch
  - If `verdict.ok || verdict.impossible` → publish `Event.Updated`, clear goal, return `false`
  - If `bumpReact > MAX_GOAL_REACT` → safety valve, publish, clear, return `false`
  - Otherwise → publish denial verdict, inject synthetic user message, return `true`
- Integrate at `outcome === "break"` check (~line 1380):
  - Replace `if (outcome === "break") break` with goalGate call
  - If gate returns `false` → `break`; if `true` → `continue`

## Task 5: TUI Goal Display

**File**: `packages/tui/src/` (various)

- Add `GoalState` type to TUI session store
- Subscribe to `Goal.Event.Updated` events
- Create goal indicator component (active goal with spinner, verdict markers)
- Render verdict badges on evaluated assistant messages

## Task 6: Edge Cases

**Files**: Various modifications

- `Goal.Service.clear()` — idempotent (no-op when no goal)
- Goal auto-cleared on session end (InstanceState teardown)
- Compaction isolation — already handled by `filterCompactedEffect`
- Subtask isolation — gate guard `agentID !== "main"` → skip
- Cancel cleanup — add `goal.clear(sessionID)` to `SessionPrompt.cancel`

## Task 7: Tests

**Files**: `packages/opencode/test/session/goal.test.ts`, `packages/opencode/test/session/goal-gate.test.ts` (new)

- Unit tests for `Goal.Service` (set/get/clear/bumpReact/evaluate with mock Provider)
- Integration tests for `goalGate` (mock evaluate → verify synthetic injection, fail-open, safety valve)
- E2E test for `/goal` command (set, clear, persistence)
