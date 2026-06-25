# Detailed Technical Architecture Guide: `/goal` Implementation

## I. Overall Architecture

`/goal` is a **stop-condition gate** embedded in the main run loop. The core idea: once a user sets a goal, the Agent cannot stop on its own — an independent judge model must verify that the goal is satisfied or genuinely impossible before the stop is allowed.

The entire system operates across three collaborating layers:

```
┌─────────────────────────────────────────────────────┐
│              Command Layer (command/index.ts)        │
│  /goal <condition>  →  Goal.Service.set()            │
│  /goal clear        →  Goal.Service.clear()          │
├─────────────────────────────────────────────────────┤
│              State Layer (goal.ts)                   │
│  Goal = { condition, react }                         │
│  Stored in InstanceState, isolated by sessionID      │
├─────────────────────────────────────────────────────┤
│              Gate Layer (prompt.ts)                   │
│  Agent attempts stop → goalGate intercepts → Judge   │
│  Satisfied → allow stop | Not satisfied → inject     │
│  synthetic message → continue working                │
└─────────────────────────────────────────────────────┘
```

---

## II. Command Registration & Entry Point

In [command/index.ts](packages/opencode/src/command/index.ts#L168-L177), `/goal` is registered as a built-in command:

```typescript
commands[Default.GOAL] = {
  name: Default.GOAL,
  description: "set a stop-condition goal; runs until a judge says it's met. /goal clear to abort",
  source: "command",
  subtask: false,
  get template() { return "$ARGUMENTS" },
  hints: ["$ARGUMENTS"],
}
```

Key design decisions:
- **`subtask: false`**: The goal command doesn't spawn a subtask — it directly manipulates session state
- **`template: "$ARGUMENTS"`**: User input is used directly as the condition string with no template substitution. e.g., `/goal all tests pass` → condition = `"all tests pass"`
- **`/goal clear`**: Clears the active goal, aborting the judge loop

After command dispatch, `Goal.Service.set(sessionID, condition)` writes the goal into state.

---

## III. Data Model & State Management

### 3.1 Goal Type Definition

[goal.ts#L27-L31](packages/opencode/src/session/goal.ts#L27-L31):

```typescript
export type Goal = {
  condition: string   // Human-readable stop condition
  react: number       // Judge-driven re-entry count, bounded by MAX_GOAL_REACT
}
```

Deliberately minimal — only two fields. `condition` is the user's raw input; `react` is the cumulative count of times the Judge has denied the stop and forced re-entry into the loop.

### 3.2 Verdict Type

[goal.ts#L33-L38](packages/opencode/src/session/goal.ts#L33-L38):

```typescript
export const Verdict = z.object({
  ok: z.boolean(),
  impossible: z.boolean().optional(),
  reason: z.string(),
})
```

Three verdict outcomes:

| `ok` | `impossible` | Meaning |
|------|-------------|---------|
| `true` | — | Goal satisfied, allow stop |
| `false` | `undefined` | Goal not satisfied, keep working |
| `false` | `true` | Goal unachievable, allow stop |

### 3.3 State Storage

State is managed through `InstanceState`, an in-memory store isolated per project instance:

```typescript
const state = yield* InstanceState.make(
  Effect.fn("SessionGoal.state")(function* () {
    return { goals: new Map<string, Goal>() }
  }),
)
```

- Keyed by `sessionID` for `Goal` object access
- Automatically cleared on instance teardown — no disk persistence
- Shares the same InstanceState pattern as [run-state.ts](packages/opencode/src/session/run-state.ts)

### 3.4 Service Interface

[goal.ts#L81-L98](packages/opencode/src/session/goal.ts#L81-L98) defines five operations:

| Method | Purpose |
|--------|---------|
| `set(sessionID, condition)` | Set goal, reset react to 0, broadcast event |
| `get(sessionID)` | Retrieve current active goal |
| `clear(sessionID)` | Clear goal, broadcast `goal: undefined` |
| `bumpReact(sessionID)` | Increment react counter, return new value |
| `evaluate(input)` | Invoke judge model against conversation transcript |

---

## IV.Judge Evaluation Model

### 4.1 System Prompt

[goal.ts#L64-L71](packages/opencode/src/session/goal.ts#L64-L71) defines the Judge's system prompt:

```
You are evaluating a stop-condition hook in Mimo Code. Read the 
conversation transcript carefully, then judge whether the user-provided 
condition is satisfied.

Your response must be a JSON object with one of these shapes:
- {"ok": true, "reason": "<quote evidence>"}
- {"ok": false, "reason": "<quote what is missing>"}
- {"ok": false, "impossible": true, "reason": "<explain why>"}

Always include a "reason" field, quoting specific text from the 
transcript whenever possible. If the transcript does not contain clear 
evidence that the condition is satisfied, return {"ok": false, 
"reason": "insufficient evidence in transcript"}.
```

Key constraint: **Judge based on transcript evidence only** — it does not trust the Agent's self-assessment.

### 4.2 User Prompt

[goal.ts#L76-L79](packages/opencode/src/session/goal.ts#L76-L79):

```typescript
const judgeUser = (condition: string) =>
  `Based on the conversation transcript above, has the following 
   stopping condition been satisfied? Answer based on transcript 
   evidence only.
   Condition: ${condition}`
```

### 4.3 Model Invocation

Implementation details in [goal.ts#L144-L218](packages/opencode/src/session/goal.ts#L144-L218):

1. **Message conversion**: `MessageV2.WithParts[]` is converted to native model messages, preserving tool calls/results/images — the Judge sees the exact same context the working Agent had

2. **Dual-path invocation**:
   - **Standard path**: Uses `generateObject()` for structured output, temperature = 0
   - **OpenAI OAuth path**: Uses `streamObject()` + `providerOptions` to pass instructions

3. **Schema enforcement**: `schema: Verdict` ensures the Judge output strictly conforms to the Zod schema

4. **Diagnostic logging**: Debug-level logging of the full transcript (long strings clipped to 200 chars), useful for debugging without polluting production logs

```typescript
const params = {
  temperature: 0,                    // Ensures stable, reproducible verdicts
  messages: [
    { role: "system", content: JUDGE_SYSTEM },
    ...conversation,                 // Full conversation transcript
    { role: "user", content: judgeUser(input.condition) },
  ],
  model: language,                   // Same model as the current Agent
  schema: Verdict,                   // Structured output constraint
}
```

---

## V. GoalGate Logic

This is the heart of the system — implemented in [prompt.ts#L1866-L1974](packages/opencode/src/session/prompt.ts#L1866-L1974).

### 5.1 Complete Flow Diagram

```
Agent completes a turn, attempts to stop
       │
       ▼
  agentID === "main" ?
       │
    No │────→ Skip (sub-agents handled by Actor preStop)
       │
      Yes
       │
       ▼
  goal.get(sessionID) exists ?
       │
    No │────→ Return false, allow stop
       │
      Yes
       │
       ▼
  Fetch transcript messages (compacted portions filtered)
  Locate the last assistant message ID
       │
       ▼
  goal.evaluate({ condition, msgs, model })
       │
       ├── Success → Verdict
       │
       └── Failure → Fail-open: { ok: true, reason: "judge error", judgeFailed: true }
                    (a flaky judge can never trap the user)
       │
       ▼
  ┌─ verdict.ok === true ──────────→ Publish verdict → clear goal → return false (allow stop) ✓
  │
  ├─ verdict.impossible === true ──→ Publish verdict → clear goal → return false (allow stop) ✘
  │
  └─ verdict.ok === false
       │
       ▼
  bumpReact(sessionID)
       │
       ▼
  react > MAX_GOAL_REACT (12) ?
       │
     Yes ──→ Warn log → publish verdict → clear goal → return false (safety valve release)
       │
      No
       │
       ▼
  Publish verdict (with attempt + messageID)
       │
       ▼
  Inject synthetic user message:
  ┌──────────────────────────────────────────┐
  │ <system-reminder>                        │
  │ Your goal is not yet satisfied: "..."    │
  │ A judge reviewed the transcript and      │
  │ reported what is still missing:          │
  │ ${verdict.reason}                        │
  │ Keep working toward the goal. Do not     │
  │ stop until it is genuinely met or        │
  │ impossible.                              │
  │ </system-reminder>                       │
  └──────────────────────────────────────────┘
       │
       ▼
  Return true → main loop continues working
```

### 5.2 Key Design Decisions

#### Fail-Open Strategy

[prompt.ts#L1893-L1901](packages/opencode/src/session/prompt.ts#L1893-L1901):

```typescript
Effect.catch((err) =>
  Effect.gen(function* () {
    yield* slog.warn("goal judge failed; allowing stop", { error: String(err) })
    return { ok: true, reason: "judge error", judgeFailed: true }
  }),
)
```

When the Judge call fails, the default is to allow the stop. This is a **safety-first** design — an unreliable Judge should never become a trap that locks the user in. The `judgeFailed` flag is broadcast to the TUI so users can see the verdict was due to an error, not genuine satisfaction.

#### Safety Valve: MAX_GOAL_REACT

[prompt.ts#L96-L102](packages/opencode/src/session/prompt.ts#L96-L102):

```typescript
const MAX_GOAL_REACT = 12
```

Even if the condition can never be satisfied, the system won't loop forever. After 12 Judge denials, the stop is force-allowed. This is more lenient than the sub-agent's `MAX_PRE_REACT = 3` because main-session goals are typically more complex.

#### Synthetic Message Injection

On denial, the Judge's reason is wrapped in a `<system-reminder>` and injected as a **synthetic user turn** (`synthetic: true`). This ensures:
- The Agent knows specifically what's missing, rather than blindly retrying
- The synthetic flag lets the system distinguish system-injected messages from genuine user input
- The message carries the same `agentID`, `model`, and `tools` as the previous turn, ensuring loop consistency

#### Verdict Anchoring

```typescript
const judgedMessageID = transcriptMsgs.findLast(
  (m) => m.info.role === "assistant"
)?.info.id
```

Each verdict carries the ID of the assistant message being evaluated. The TUI uses this to render a ✓/✘ marker next to the corresponding turn, allowing the user to trace back to exactly which piece of work was judged.

---

## VI. Event System

[goal.ts#L46-L60](packages/opencode/src/session/goal.ts#L46-L60) defines the event broadcast:

```typescript
Event.Updated: BusEvent.define("session.goal", z.object({
  sessionID: SessionID.zod,
  goal: z.object({ condition: z.string() }).optional(),   // undefined = cleared
  lastVerdict: Verdict.extend({
    attempt: z.number(),           // Which evaluation attempt
    messageID: z.string().optional(), // The assistant message that was judged
    error: z.boolean().optional(),    // Whether the Judge errored
  }).optional(),
}))
```

Event publication timing:

| Timing | `goal` | `lastVerdict` |
|--------|--------|---------------|
| `/goal` set | `{ condition }` | — |
| Judge denies | `{ condition }` | `{ ok: false, reason, attempt, messageID }` |
| Judge approves | `undefined` | `{ ok: true, reason, attempt, messageID }` |
| Impossible | `undefined` | `{ ok: false, impossible: true, reason, ... }` |
| Safety valve triggered | `undefined` | `{ ok: false, reason, attempt, messageID }` |
| `/goal clear` | `undefined` | — |

The TUI listens for this event to render the active goal indicator and verdict status. `lastVerdict` is "sticky" — even after the goal is cleared, the TUI retains the last verdict for the user to review.

---

## VII. Integration with the Main Run Loop

`goalGate` is one of several stop-condition gates, positioned in the main loop as follows:

```
Agent completes a turn (tool-use stop or text stop)
    │
    ├── autoContinueInvalidOutput (empty/reasoning-only → nudge to continue)
    ├── autoRetryStructuredOutput (structured output failure → retry)
    ├── writeContentFilterError (content filter → terminate with error)
    ├── taskGate (incomplete subtasks → keep working)        ← parallel gate
    ├── goalGate (active goal not satisfied → keep working)  ← this gate
    │
    └── All gates pass → allow stop, session enters idle
```

**Main Agent only**: `if ((agentID ?? "main") !== "main") return false`. Sub-agent stops are handled by the Actor model's `preStop` ReAct mechanism. Both share the same state management pattern (react counter + safety valve), but with different signal sources.

---

## VIII. Resource Protection Mechanisms Summary

| Mechanism | Constant | Value | Trigger |
|-----------|----------|-------|---------|
| Goal react cap | `MAX_GOAL_REACT` | 12 | Judge-denial-driven re-entry count |
| Judge failure fallback | — | fail-open | `evaluate` throws an exception |
| Instance isolation | `InstanceState` | per sessionID | Auto-cleared on instance teardown |
| Synthetic message flag | `synthetic: true` | — | Distinguishes system injection from user input |

---

## IX. Source File Index

| File | Responsibility |
|------|---------------|
| [goal.ts](packages/opencode/src/session/goal.ts) | Goal type, Verdict schema, Judge prompt, Service implementation, Event definitions |
| [prompt.ts](packages/opencode/src/session/prompt.ts#L1866-L1974) | goalGate logic, MAX_GOAL_REACT constant, synthetic message injection |
| [command/index.ts](packages/opencode/src/command/index.ts#L168-L177) | `/goal` command registration |
| [run-state.ts](packages/opencode/src/session/run-state.ts) | Run state management (Runner), goalGate return value drives continue/stop |
| [session.ts](packages/opencode/src/session/session.ts) | Session data model, provides message access interface |

---

Want to go deeper? These paths are worth exploring:

- [Goal-Driven Autonomous Loops](https://zread.ai/XiaomiMiMo/MiMo-Code/10-goal-driven-autonomous-loops) — Complete design doc and architecture diagrams
- [Session Lifecycle and LLM Loop](https://zread.ai/XiaomiMiMo/MiMo-Code/6-session-lifecycle-and-llm-loop) — Precise position of goalGate in the main loop
- [Subagent Spawn and Actor Model](https://zread.ai/XiaomiMiMo/MiMo-Code/12-subagent-spawn-and-actor-model) — Parallel stop gate for sub-agents (preStop ReAct)
- [Task Tracking and Progress](https://zread.ai/XiaomiMiMo/MiMo-Code/13-task-tracking-and-progress) — taskGate, the sibling gate running alongside goalGate