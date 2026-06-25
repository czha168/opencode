# `/goal` Implementation Plan — Adapted for opencode

This plan adapts the architecture in `docs/goal_implementation_guide.md` to the actual opencode codebase. Every phase references concrete file paths, types, and patterns used in the codebase today.

---

## Gap Analysis: Guide vs. Codebase

| Guide Reference | Codebase Reality |
|---|---|
| `packages/opencode/src/session/goal.ts` | **Does not exist** — must be created |
| `Default.GOAL` command in `command/index.ts` | Only `Default.INIT` and `Default.REVIEW` exist; no `/goal` command |
| `goalGate` in `prompt.ts` lines 1866–1974 | `prompt.ts` is 1722 lines total; no goalGate exists |
| `MAX_GOAL_REACT = 12` in `prompt.ts` | No such constant |
| `Verdict` Zod schema (`z.object`) | Codebase uses `Schema.Struct` from Effect, not Zod |
| `generateObject()` / `streamObject()` from `ai` | Used once in `agent/agent.ts` for agent generation — same pattern can be reused |
| `InstanceState` with `Map<string, Goal>` | Same pattern exists in `session/status.ts` and `session/run-state.ts` |
| `Event.Updated: BusEvent.define(...)` | Codebase uses `EventV2.define(...)` from `@opencode-ai/core/event` (see `session/status.ts`) |
| Synthetic message injection | Pattern already used in `prompt.ts`, `reminders.ts`, `compaction.ts` — `synthetic: true` on `TextPart` |
| `MessageV2.filterCompactedEffect` | Exists — used to get non-compacted transcript in the loop |
| `MessageV2.toModelMessagesEffect` | Exists — converts internal messages to AI SDK `ModelMessage[]` |
| `SessionV1.TextPart` | Has `synthetic?: boolean` field; used throughout |
| `SessionV1.User` | Has `id`, `sessionID`, `role`, `agent`, `model`, `time` fields |
| `sessions.updateMessage` / `sessions.updatePart` | Used for creating all messages and parts |
| `SessionRunState.Service` | `ensureRunning(sessionID, onInterrupt, work)` drives the loop — gate return value controls continue/stop |
| `Provider.Service` + `Provider.getLanguage()` | Used to get the AI SDK language model for inference |
| `LLM.Service` | High-level streaming interface; for the Judge we need lower-level `generateObject` directly via Provider |

---

## Phase 0: Type & Schema Definitions

**File**: `packages/opencode/src/session/goal.ts` (new)

Create the data types and schemas that all other phases depend on. Follow the Effect `Schema` pattern used throughout the codebase (not Zod).

### 0.1 — `Goal` type

```typescript
export type Goal = {
  condition: string
  react: number
}
```

Minimal — same two fields as the guide. `react` tracks Judge-denial-driven re-entry count.

### 0.2 — `Verdict` schema

Use `Schema.Struct` (not Zod) per codebase convention:

```typescript
export const Verdict = Schema.Struct({
  ok: Schema.Boolean,
  impossible: Schema.optional(Schema.Boolean),
  reason: Schema.String,
})
export type Verdict = Schema.Schema.Type<typeof Verdict>
```

Three outcomes: `{ ok: true }` → allow stop; `{ ok: false }` → keep working; `{ ok: false, impossible: true }` → unachievable, allow stop.

### 0.3 — Event schema

Use `EventV2.define` following `session/status.ts` pattern:

```typescript
export const Event = {
  Updated: EventV2.define({
    type: "session.goal",
    schema: {
      sessionID: SessionID,
      goal: Schema.optional(Schema.Struct({ condition: Schema.String })),
      lastVerdict: Schema.optional(
        Verdict.pipe(Schema.extend(Schema.Struct({
          attempt: Schema.Number,
          messageID: Schema.optional(Schema.String),
          error: Schema.optional(Schema.Boolean),
        }))),
      ),
    },
  }),
}
```

### 0.4 — Judge prompt constants

```typescript
const JUDGE_SYSTEM = `You are evaluating a stop-condition hook in opencode. Read the
conversation transcript carefully, then judge whether the user-provided
condition is satisfied.

Your response must be a JSON object with one of these shapes:
- {"ok": true, "reason": "<quote evidence>"}
- {"ok": false, "reason": "<quote what is missing>"}
- {"ok": false, "impossible": true, "reason": "<explain why>"}

Always include a "reason" field, quoting specific text from the
transcript whenever possible. If the transcript does not contain clear
evidence that the condition is satisfied, return {"ok": false,
"reason": "insufficient evidence in transcript"}.`

const judgeUser = (condition: string) =>
  `Based on the conversation transcript above, has the following stopping condition been satisfied? Answer based on transcript evidence only.\nCondition: ${condition}`
```

### 0.5 — `MAX_GOAL_REACT` constant

```typescript
export const MAX_GOAL_REACT = 12
```

### Deliverable

- `packages/opencode/src/session/goal.ts` with types, schemas, event definition, prompts, and constant — no service yet.

---

## Phase 1: Goal Service

**File**: `packages/opencode/src/session/goal.ts` (extend Phase 0)

Implement the `Goal.Service` with five operations: `set`, `get`, `clear`, `bumpReact`, `evaluate`.

### 1.1 — Service interface

```typescript
export interface Interface {
  readonly set: (sessionID: SessionID, condition: string) => Effect.Effect<void>
  readonly get: (sessionID: SessionID) => Effect.Effect<Goal | undefined>
  readonly clear: (sessionID: SessionID) => Effect.Effect<void>
  readonly bumpReact: (sessionID: SessionID) => Effect.Effect<number>
  readonly evaluate: (input: EvaluateInput) => Effect.Effect<Verdict>
}

export type EvaluateInput = {
  condition: string
  msgs: SessionV1.WithParts[]
  model: Provider.Model
}
```

### 1.2 — State storage

Use `InstanceState.make` with `Map<SessionID, Goal>`, following the exact pattern from `session/status.ts`:

```typescript
const state = yield* InstanceState.make(
  Effect.fn("SessionGoal.state")(function* () {
    return { goals: new Map<SessionID, Goal>() }
  }),
)
```

### 1.3 — `set`, `get`, `clear`, `bumpReact` implementations

- `set(sessionID, condition)`: Set goal with `react: 0`, publish `Event.Updated` with `{ sessionID, goal: { condition } }`.
- `get(sessionID)`: Return from the Map.
- `clear(sessionID)`: Delete from Map, publish `Event.Updated` with `{ sessionID, goal: undefined }`.
- `bumpReact(sessionID)`: Increment react, return new value.

### 1.4 — `evaluate` implementation

This is the most complex operation. It must:

1. Convert `SessionV1.WithParts[]` to AI SDK `ModelMessage[]` using `MessageV2.toModelMessagesEffect(msgs, model)` — same function used in the main loop at `prompt.ts:1331`.
2. Call `generateObject` from the `ai` package with:
   - `temperature: 0` for reproducibility
   - `messages: [system, ...conversation, user]`
   - `model: language` (from `Provider.getLanguage(model)`)
   - `schema: Verdict` (converted via `Schema.toStandardSchemaV1` + `Schema.toStandardJSONSchemaV1` — same pattern as `agent/agent.ts:410-413`)
3. Handle the dual-path (standard vs. OpenAI OAuth) following the same pattern as `agent/agent.ts:416-433`:
   - Standard: `generateObject(params).then(r => r.object)`
   - OAuth: `streamObject({ ...params, providerOptions }).fullStream` then `result.object`
4. Fail-open on error: catch any exception, log warning, return `{ ok: true, reason: "judge error" }`.

### 1.5 — Service wiring

```typescript
export class Service extends Context.Service<Service, Interface>()("@opencode/SessionGoal") {}

export const layer = Layer.effect(Service, Effect.gen(function* () {
  const events = yield* EventV2Bridge.Service
  const provider = yield* Provider.Service
  // ... state and implementations
  return Service.of({ set, get, clear, bumpReact, evaluate })
}))

export const defaultLayer = layer.pipe(
  Layer.provide(EventV2Bridge.defaultLayer),
  Layer.provide(Provider.defaultLayer),
)
```

### Dependencies

- `EventV2Bridge.Service` — for event publishing
- `Provider.Service` — for `getLanguage()` in `evaluate`
- `Session.Service` — not needed (state is in-memory via InstanceState)
- `MessageV2` — for `toModelMessagesEffect`

### Deliverable

- Complete `goal.ts` with Service, Layer, and all five operations.

---

## Phase 2: `/goal` Command Registration

**File**: `packages/opencode/src/command/index.ts` (modify)

### 2.1 — Add `GOAL` to `Default` enum

```typescript
export const Default = {
  INIT: "init",
  REVIEW: "review",
  GOAL: "goal",
} as const
```

### 2.2 — Register the `/goal` command

Inside the `init` function (after the existing `commands[Default.REVIEW]` block), add:

```typescript
commands[Default.GOAL] = {
  name: Default.GOAL,
  description: "set a stop-condition goal; runs until a judge says it's met. /goal clear to abort",
  source: "command",
  subtask: false,
  get template() {
    return "$ARGUMENTS"
  },
  hints: ["$ARGUMENTS"],
}
```

Key design decisions (from guide):
- `subtask: false` — goal manipulates session state directly, doesn't spawn a subtask.
- `template: "$ARGUMENTS"` — user input becomes the condition string directly. `/goal all tests pass` → condition = `"all tests pass"`.

### 2.3 — Command dispatch wiring

The `/goal` command currently flows through the generic `prompt()` path in `SessionPrompt`. We need to intercept it **before** it creates a user message and instead call `Goal.Service.set()` or `Goal.Service.clear()`.

**Option A** (recommended): Handle `/goal` dispatch in `SessionPrompt.command()` at `prompt.ts:1417`. When `input.command === "goal"`, instead of the normal template → prompt flow:

```typescript
if (input.command === "goal") {
  const goal = yield* Goal.Service
  if (input.arguments.trim() === "clear") {
    yield* goal.clear(input.sessionID)
  } else {
    yield* goal.set(input.sessionID, input.arguments.trim())
  }
  // Return a lightweight message to confirm the goal was set
  const session = yield* sessions.get(input.sessionID).pipe(Effect.orDie)
  return { info: ... , parts: [...] }
}
```

**Option B**: Add a `preDispatch` hook to the Command service that short-circuits before template resolution. More architecturally clean but requires Command service changes.

**Recommendation**: Option A is simpler and follows how other special-case commands could be handled. It's a 10-line block in the existing command handler.

### Deliverable

- `/goal` and `/goal clear` commands registered and wired to `Goal.Service`.

---

## Phase 3: GoalGate in the Main Loop

**File**: `packages/opencode/src/session/prompt.ts` (modify)

This is the heart of the system — the stop-condition gate.

### 3.1 — Import Goal service

Add `Goal` import alongside existing session imports at top of `prompt.ts`.

### 3.2 — Wire Goal.Service into the layer

Add `Goal.Service` to the `yield*` block in the `layer` definition (around line 99-127), and add `Goal.defaultLayer` to `defaultLayer` and `Goal.node` to `node`.

### 3.3 — Implement `goalGate` function

Add a local function inside the `layer` Effect.gen closure (has access to `sessions`, `events`, `state`, etc.):

```typescript
const goalGate = Effect.fn("SessionPrompt.goalGate")(function* (input: {
  sessionID: SessionID
  agentID?: string
  msgs: SessionV1.WithParts[]
  model: Provider.Model
}): Effect.Effect<boolean> {
  // Only gate the main agent
  if ((input.agentID ?? "main") !== "main") return false

  const goal = yield* Goal.Service
  const active = yield* goal.get(input.sessionID)
  if (!active) return false

  // Filter compacted messages, locate last assistant message
  const transcriptMsgs = input.msgs.filter(
    (m) => !m.parts.some((p) => p.type === "compaction")
  )
  const judgedMessageID = transcriptMsgs.findLast(
    (m) => m.info.role === "assistant"
  )?.info.id

  // Evaluate with fail-open
  const verdict = yield* goal
    .evaluate({ condition: active.condition, msgs: transcriptMsgs, model: input.model })
    .pipe(
      Effect.catchCause((cause) =>
        Effect.gen(function* () {
          yield* Effect.logWarning("goal judge failed; allowing stop", {
            error: Cause.squash(cause).toString(),
          })
          return { ok: true, reason: "judge error", error: true } as Verdict & { error: boolean }
        }),
      ),
    )

  // Satisfied or impossible → allow stop
  if (verdict.ok || verdict.impossible) {
    yield* events.publish(Goal.Event.Updated, {
      sessionID: input.sessionID,
      goal: undefined,
      lastVerdict: {
        ...verdict,
        attempt: active.react + 1,
        messageID: judgedMessageID,
      },
    })
    yield* goal.clear(input.sessionID)
    return false
  }

  // Not satisfied → check safety valve
  const react = yield* goal.bumpReact(input.sessionID)
  if (react > MAX_GOAL_REACT) {
    yield* Effect.logWarning("goal safety valve triggered", {
      sessionID: input.sessionID,
      react,
    })
    yield* events.publish(Goal.Event.Updated, {
      sessionID: input.sessionID,
      goal: undefined,
      lastVerdict: {
        ...verdict,
        attempt: react,
        messageID: judgedMessageID,
      },
    })
    yield* goal.clear(input.sessionID)
    return false
  }

  // Publish denial verdict
  yield* events.publish(Goal.Event.Updated, {
    sessionID: input.sessionID,
    goal: { condition: active.condition },
    lastVerdict: {
      ...verdict,
      attempt: react,
      messageID: judgedMessageID,
    },
  })

  // Inject synthetic user message
  const lastUser = transcriptMsgs.findLast((m) => m.info.role === "user")
  if (!lastUser) return false

  const syntheticMsg: SessionV1.User = {
    id: MessageID.ascending(),
    sessionID: input.sessionID,
    role: "user",
    time: { created: Date.now() },
    agent: lastUser.info.role === "user" ? lastUser.info.agent : undefined,
    model: lastUser.info.role === "user" ? lastUser.info.model : undefined,
  }
  yield* sessions.updateMessage(syntheticMsg)
  yield* sessions.updatePart({
    id: PartID.ascending(),
    messageID: syntheticMsg.id,
    sessionID: input.sessionID,
    type: "text",
    text: [
      "<system-reminder>",
      `Your goal is not yet satisfied: "${active.condition}"`,
      "A judge reviewed the transcript and reported what is still missing:",
      verdict.reason,
      "Keep working toward the goal. Do not stop until it is genuinely met or impossible.",
      "</system-reminder>",
    ].join("\n"),
    synthetic: true,
  } satisfies SessionV1.TextPart)

  return true // keep the loop running
})
```

### 3.4 — Integrate goalGate into the run loop

In `runLoop` (starting ~line 1134), the loop exit condition is at lines 1164-1183. After the existing exit check breaks the loop, but **before** `break`, insert the goalGate call.

The integration point is after `outcome === "break"` is determined (~line 1395). Currently:

```typescript
if (outcome === "break") break
```

Change to:

```typescript
if (outcome === "break") {
  const gateResult = yield* goalGate({
    sessionID,
    agentID: msg.agent,
    msgs,
    model,
  })
  if (!gateResult) break
  // gateResult === true means goalGate injected a synthetic message; continue loop
  continue
}
```

**Important**: `goalGate` must be called with the `msgs` snapshot from *before* the current assistant turn is finalized, but it needs the current turn's content for evaluation. The `msgs` variable already includes the latest assistant message at this point since `handle.message` has been updated through the processor. The Judge sees the same context the Agent had.

### 3.5 — Pass `Goal.Service` through the layer dependency chain

Add `Goal.Service` to:
- The `yield*` block in `prompt.ts` layer (line ~99)
- `defaultLayer` providers (line ~1556)
- `node` exports (line ~1693)

### Deliverable

- `goalGate` function in `prompt.ts`
- Gate wired into the main loop at the break point
- Goal.Service dependency wired into layer graph

---

## Phase 4: TUI Event Display

**File**: `packages/tui/src/` (various)

This phase makes the goal visible in the terminal UI. The TUI listens for `session.goal` events.

### 4.1 — Add goal state to TUI session store

In whichever TUI file manages session state subscriptions (likely a session store or context), subscribe to `Goal.Event.Updated` and maintain:

```typescript
type GoalState = {
  condition?: string
  lastVerdict?: {
    ok: boolean
    impossible?: boolean
    reason: string
    attempt: number
    messageID?: string
    error?: boolean
  }
}
```

### 4.2 — Goal indicator component

Create a small SolidJS component that renders:
- Active goal: `[Goal: "all tests pass"]` with spinner/indicator
- Verdict received: `✓` (satisfied) or `✘ attempt N` (denied) next to the judged message
- Safety valve triggered: `⚠ goal aborted after 12 attempts`

### 4.3 — Render verdict markers on messages

When `lastVerdict.messageID` matches an assistant message in the message list, render a badge next to it showing the verdict status.

### Deliverable

- TUI goal indicator visible during active goal
- Verdict markers on evaluated messages

---

## Phase 5: Config & `/goal clear` Edge Cases

### 5.1 — Handle `/goal clear` when no goal is active

`Goal.Service.clear()` should be idempotent — clearing when no goal exists is a no-op (publishes `Event.Updated` with `goal: undefined` anyway for TUI consistency).

### 5.2 — Goal cleared on session end

Since `Goal.Service` uses `InstanceState`, goals are automatically cleared when the instance is torn down (same as `SessionRunState` and `SessionStatus`). No explicit cleanup needed.

### 5.3 — Goal interaction with compaction

When compaction occurs, compacted messages are excluded from the Judge transcript via `MessageV2.filterCompactedEffect`. This is already how `msgs` is built in the run loop (line 1145-1147). No special handling needed — the Judge sees the same filtered transcript the Agent sees.

### 5.4 — Goal interaction with subtasks

The gate only fires for `agentID === "main"`. Sub-agent stops are handled by the Actor model's `preStop` mechanism. The `if ((agentID ?? "main") !== "main") return false` guard ensures this.

### 5.5 — Goal interaction with cancel

When `SessionPrompt.cancel` is called, `SessionRunState.cancel` interrupts the runner. The goal should be cleared on cancel so it doesn't persist into the next loop invocation. Add to `cancel`:

```typescript
const goal = yield* Goal.Service
yield* goal.clear(sessionID)
```

### Deliverable

- Edge cases handled: clear idempotency, cancel cleanup, compaction, subtask isolation.

---

## Phase 6: Testing

### 6.1 — Unit tests for Goal.Service

**File**: `packages/opencode/test/session/goal.test.ts`

Use `testEffect` from `packages/opencode/test/lib/effect.ts`:

- `set` then `get` returns the goal
- `clear` removes the goal
- `bumpReact` increments and returns
- `bumpReact` exceeds `MAX_GOAL_REACT` (verify the value is returned correctly; the gate logic handles the cap)
- `evaluate` with mock Provider returns a valid Verdict

### 6.2 — Integration test for goalGate

**File**: `packages/opencode/test/session/goal-gate.test.ts`

- Mock `Goal.Service.evaluate` to return `{ ok: false, reason: "missing X" }` — verify synthetic message is injected and loop continues
- Mock `evaluate` to return `{ ok: true, reason: "done" }` — verify gate returns `false` and goal is cleared
- Mock `evaluate` to throw — verify fail-open behavior (returns `false`)
- After `MAX_GOAL_REACT` denials — verify safety valve releases

### 6.3 — End-to-end test for `/goal` command

- `/goal all tests pass` sets a goal
- `/goal clear` clears it
- Goal persists across loop iterations until satisfied or cleared

### Deliverable

- Test files covering Goal.Service, goalGate, and `/goal` command.

---

## File Change Summary

| File | Action | Phase |
|---|---|---|
| `packages/opencode/src/session/goal.ts` | **Create** | 0, 1 |
| `packages/opencode/src/command/index.ts` | **Modify** (add `Default.GOAL` + command registration) | 2 |
| `packages/opencode/src/session/prompt.ts` | **Modify** (add `goalGate`, wire into loop, add imports/deps) | 3 |
| `packages/tui/src/*` | **Modify** (goal indicator, verdict markers) | 4 |
| `packages/opencode/test/session/goal.test.ts` | **Create** | 6 |
| `packages/opencode/test/session/goal-gate.test.ts` | **Create** | 6 |

---

## Dependency Graph

```
Phase 0 (types/schemas)
  └── Phase 1 (Goal.Service)
        ├── Phase 2 (command registration)
        └── Phase 3 (goalGate in prompt.ts loop)
              └── Phase 4 (TUI display)
                    └── Phase 5 (edge cases)
                          └── Phase 6 (testing)
```

Phases 0–3 are the critical path. Phases 4–6 can proceed in parallel once Phase 3 lands.

---

## Risk Mitigation

| Risk | Mitigation |
|---|---|
| `generateObject` fails for some providers | Fail-open strategy: return `{ ok: true }` on any Judge error. Logged via `Effect.logWarning`. |
| Judge loops indefinitely | `MAX_GOAL_REACT = 12` safety valve releases the stop. |
| Judge sees compacted transcript gaps | `MessageV2.filterCompactedEffect` already excludes compacted portions; Judge sees what Agent sees. |
| Synthetic message counted as real user turn | `synthetic: true` flag on `TextPart`; TUI and title generation already filter these out (line 186-188 of prompt.ts). |
| `/goal` called while loop is idle | `Goal.Service.set` stores the goal; next `loop()` invocation will trigger `goalGate` on the Agent's first stop attempt. |
| Provider doesn't support `generateObject` | Same dual-path fallback as `agent/agent.ts`: `streamObject` with `providerOptions` for OpenAI OAuth. |
