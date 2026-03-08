# ReAct AgentState FSM Proposal

## 1. Background

The current ReAct Agent runtime state is chained implicitly through stage outputs: the output of `think` becomes the input of `act`, and the output of `act` is then passed to `observe`. `OrderOrchestrator` itself does not maintain a unified Agent state model. This is simple, but as more stages and more complex flows are introduced, the structure has started to show clear weaknesses:

- The implicit state chain is fragile. Once the output schema of one stage changes, downstream failures only surface at runtime, usually as failed type assertions, nil propagation, or behavior drift.
- Dependencies between stages are not enforced at the system level. Right now the only rule is "the previous output becomes the next input", without an explicit contract for which state a stage depends on and which state it produces.
- Intermediate state is hard to reuse and extend. Adding a new stage, skipping one, supporting conditional branches, or resuming execution all require ad hoc stitching on top of existing output structures.
- Auditability and traceability are weak. The system cannot answer questions like "which intermediate decisions did this response depend on", "who wrote this field", or "when did the state transition happen".

This needs to be handled now for three reasons:

- The Agent already includes multi-stage orchestration, tool calling, streaming output, and session memory. Keeping the current implicit-state pattern will amplify complexity.
- If later workflows need conditional branches, retries, human intervention, checkpoints, or resumable execution, state and state transitions must first be separated from prompt outputs.
- Once Agent state becomes a runtime-level abstraction, testing, audit logs, SSE events, replay, metrics, and persistence can all be built around the same model.

If left unresolved, the system will stay in a "the flow runs, but the state is not provable" phase: short term, stage changes become increasingly risky; long term, the Agent framework becomes harder to extend and operate.

## 2. Goals

- Goal 1: introduce a unified, explicit, strongly typed `AgentState` interface for the ReAct Agent, inspired by LangGraph's `AgentState`.
- Goal 2: move orchestration from "stage output chaining" to a state-transition-based FSM where state, stages, and transition conditions are first-class concepts.
- Goal 3: define the minimum state set and its invariants so required capabilities are preserved without turning shared state into a dumping ground.
- Goal 4: make each stage declare its read and write permissions against `AgentState`.
- Goal 5: provide a consistent foundation for debugging, audit, replay, resumable execution, and future extension through state snapshots and transition logs.

## 3. Non-goals

- Non-goal 1: this proposal does not rewrite the existing think, act, or observe prompt content; it only changes how they connect to runtime state.
- Non-goal 2: this does not expand ReAct into a full DAG orchestration engine; the primary model remains a finite-state machine.
- Non-goal 3: this does not introduce distributed persistent state storage; the initial target is an in-process state model plus optional event logs.
- Non-goal 4: this does not unify every historical schema in the project; the focus is Agent runtime state, not the entire business data model.

## 4. Current State and Constraints

Technical state:

- `component/agent/agent.go` uses `schema.Schema` as the shared input/output type in `OrderOrchestrator.Run`, passing stage results through `FlowChan` sequentially.
- `component/agent/react/react.go` only declares prompt input and output types in `stageTypeRegistry`; it does not declare shared-state dependencies, permissions, or transition rules.
- The current `think -> act -> observe` order is hard-coded execution order, not an explicit state machine.
- Intermediate runtime state is scattered across `schema.ThinkInput`, `schema.ThinkOutput`, `schema.ToolOutputs`, `schema.Observation`, and memory context.

Dependency state:

- The project already has `genkit` flow, streaming, memory, tool calling, and SSE feedback. The new design must stay compatible with those capabilities.
- A large amount of current logic depends on `schema.Schema` and runtime type assertions, so the transition cannot break every interface at once.

Compatibility constraints:

- Existing ReAct configuration and default think/act/observe behavior must remain compatible.
- Streaming output must continue to work; the FSM must not block incremental observe output.
- Migration must be incremental, allowing legacy stage outputs and new state writes to coexist for a while.

## 5. Design

### 5.1 Overall Approach

The core idea is to change the Agent runtime from "each stage directly consumes the previous stage output" to "all stages operate on a shared `AgentState`". `AgentState` stores the minimum state the system recognizes for the current turn. Every stage must declare which state it reads, which state it writes, what preconditions it requires, and which lifecycle state it transitions to on success.

An FSM model is introduced at the orchestration layer: a stage is no longer just a flow, but a state transformer with `Preconditions`, `ReadSet`, `WriteSet`, and `Transition`. Before scheduling a stage, the orchestrator validates that the current state satisfies invariants and preconditions. After execution, it commits the state patch and records the transition event. That makes dependency violations, illegal states, unauthorized writes, and missing state visible as framework errors instead of prompt-time surprises.

### 5.2 Architecture Diagram or Flow

```mermaid
stateDiagram-v2
    [*] --> "Initialized"
    "Initialized" --> "Thinking": "load user input"
    "Thinking" --> "Acting": "thought ready and tools requested"
    "Thinking" --> "Observing": "thought ready and no tool needed"
    "Acting" --> "Observing": "tool outputs ready"
    "Observing" --> "Finished": "final answer ready"
    "Observing" --> "Thinking": "need another iteration"
    "Thinking" --> "Failed": "invariant violation / model error"
    "Acting" --> "Failed": "tool error unrecoverable"
    "Observing" --> "Failed": "render error unrecoverable"
```

### 5.3 Key Changes

#### Module A: strongly typed `AgentState`

- Define a unified state container instead of sequentially passing `schema.Schema`.
- Prefer a split model of lifecycle state, business state, and metadata rather than one oversized struct.
- Example:

```go
type LifecycleState int

const (
    StateInitialized LifecycleState = iota
    StateThinking
    StateActing
    StateObserving
    StateFinished
    StateFailed
)

type SessionID string
type Iteration int

type ThoughtState struct {
    Intent         schema.Intent
    SuggestedTools []string
    Reasoning      string
}

type ActionState struct {
    RequestedTools []string
    ToolOutputs    schema.ToolOutputs
}

type ObservationState struct {
    Summary     string
    FinalAnswer string
    NeedLoop    bool
}

type AgentState struct {
    Lifecycle   LifecycleState
    SessionID   SessionID
    UserInput   *schema.UserInput
    Iteration   Iteration
    Thought     *ThoughtState
    Action      *ActionState
    Observation *ObservationState
    LastError   error
}
```

#### Module B: minimum state set and invariants

- Keep only facts that matter across stages, across iterations, or for audit; do not store transient prompt assembly data in shared state.
- The recommended minimum state set is `Lifecycle`, `SessionID`, `UserInput`, `Iteration`, `Thought`, `Action`, `Observation`, and `LastError`.
- Example invariants:
- When `Lifecycle == StateInitialized`, `UserInput != nil` and `Thought/Action/Observation == nil`.
- When `Lifecycle == StateActing`, `Thought != nil` and `Thought.Intent` is already defined.
- When `Lifecycle == StateObserving` and the path came from `act`, `Action != nil`.
- When `Lifecycle == StateFinished`, `Observation != nil` and `Observation.FinalAnswer != ""`.
- `Iteration >= 0`, and every transition back to `Thinking` increments it monotonically.

#### Module C: upgrade the stage interface into a state transformer

- Replace `Stage.Execute(ctx, chans, input schema.Schema)` with a state-oriented interface.
- Suggested interface:

```go
type StateStage interface {
    Name() string
    ReadSet() []StateField
    WriteSet() []StateField
    Check(*AgentState) error
    Execute(context.Context, *AgentState, *Channels) (*StatePatch, error)
    Next(*AgentState) (LifecycleState, error)
}
```

- `StatePatch` expresses the legal change set so stages do not mutate the full state object arbitrarily.

#### Module D: permission and dependency validation

- Every stage must declare read and write boundaries, and the orchestrator validates them before and after execution.
- The initial permission matrix can be expressed as a list:
- `think` may read `SessionID`, `UserInput`, `Iteration`, and previous `Observation`; it may write `Thought` and `Lifecycle`; it must not write `Action` or `Observation`.
- `act` may read `SessionID`, `Iteration`, and `Thought`; it may write `Action` and `Lifecycle`; it must not write `UserInput`, `Thought`, or `Observation`.
- `observe` may read `SessionID`, `Iteration`, `Thought`, and `Action`; it may write `Observation` and `Lifecycle`; it must not write `UserInput`, `Thought`, or `Action`.
- `error-handler` may read any field; it may write `LastError` and `Lifecycle`; it must not write business-state fields.
- If a stage writes undeclared fields or changes read-only fields, the framework should fail immediately.

#### Module E: FSM orchestrator

- Evolve `OrderOrchestrator` into `FSMOrchestrator`.
- Responsibilities include validating current-state invariants, selecting the executable stage from `Lifecycle`, executing the stage and merging `StatePatch`, recording state transition events, and handling loop exit and failure states.
- Loop conditions then depend on explicit state such as `Observation.NeedLoop`, not on implicit output shapes.

#### Module F: auditability and traceability

- Record a standardized event per stage execution:

```go
type StateTransitionEvent struct {
    SessionID   string
    Stage       string
    From        LifecycleState
    To          LifecycleState
    ReadSet     []StateField
    WriteSet    []StateField
    Patch       map[string]any
    StartedAt   time.Time
    FinishedAt  time.Time
    Err         string
}
```

- Start with in-memory logs or debug output, then wire them into persistence or SSE later.

### 5.4 Data and Interface Changes

New interfaces:

- `AgentState`
- `StateStage`
- `StatePatch`
- `FSMOrchestrator`
- `StateValidator` or `InvariantChecker`

Field changes:

- Existing stages no longer pass full business state directly as `schema.Schema`.
- `think/act/observe` should read from `AgentState` and write into `StatePatch`.
- `StageInfo` can later grow declarative metadata such as:

```yaml
stages:
  - name: think
    flow_type: think
    enters: Thinking
    exits:
      - Acting
      - Observing
    reads: [session_id, user_input, iteration, observation]
    writes: [thought, lifecycle]
```

Compatibility impact:

- Existing call sites that assume stage chaining via `schema.Schema` need gradual migration.
- Streaming interfaces should stay compatible because lifecycle and output transport do not need to change together.

Migration path:

- Phase 1: introduce `AgentState`, state validation, and adapter layers while keeping existing stage flows.
- Phase 2: migrate `think/act/observe` to declare read/write sets and return `StatePatch`.
- Phase 3: replace sequential orchestration with `FSMOrchestrator` and add transition logs, replay hooks, and observability.

### 5.5 Error Handling and Fallback

Possible failure points:

- State invariant validation failure.
- A stage attempting unauthorized writes.
- A legacy stage adapter producing incomplete state.
- FSM transitions getting stuck in an illegal lifecycle state.

Failure handling:

- Validation failures should stop execution immediately and surface a framework-level error with enough context to debug.
- Unauthorized writes should be rejected instead of silently merged.
- Adapter failures should be logged explicitly and fall back to the previous orchestration path only during the migration window.
- Transition dead-ends should move the lifecycle to `Failed` with a structured error record.

Degradation strategy:

- Keep a migration window where the legacy sequential path can still run behind a feature flag.
- If state transition enforcement proves too strict for one stage, disable it for that stage explicitly rather than weakening global guarantees.
- If replay or audit logging fails, the runtime flow should still complete; logging should not become a hard dependency for the Agent answer path.
