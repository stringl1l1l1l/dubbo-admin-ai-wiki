# ReAct AgentState FSM Proposal

## 1. 背景

当前 ReAct Agent 的运行状态由各个 stage 的输出隐式串联：`think` 的输出直接成为 `act` 的输入，`act` 的输出再交给 `observe`，`OrderOrchestrator` 本身并不维护统一的 Agent 状态模型。这种实现虽然简单，但随着 stage 增多、流程变复杂，已经暴露出明显的结构性问题：

- 隐式状态链路十分脆弱。某个 stage 的输出结构一旦变化，后续 stage 会在运行时才暴露错误，问题通常表现为类型断言失败、空值传播或行为偏移。
- stage 之间的依赖关系不是系统级硬约束。当前只有“上一个输出作为下一个输入”的约定，没有显式声明“某个 stage 依赖哪些状态、产出哪些状态”，导致排障高度依赖人工理解流程。
- 中间状态难以复用和扩展。想引入新的 stage、跳过某个 stage、支持条件分支或恢复执行时，都需要依赖已有输出结构做拼接，缺少稳定的共享状态接口。
- 审计和可追踪性较弱。系统无法直接回答“某次响应依赖了哪些中间决策”“某个字段是谁写入的”“状态在什么时刻发生了转移”，这会限制调试、回放和后续治理能力。

这个问题需要现在处理，原因有三点：

- 当前 Agent 已经具备多 stage 编排、工具调用、流式输出和会话记忆，继续沿用隐式状态模式会放大复杂度。
- 后续如果要支持更复杂的工作流，例如条件分支、重试、人工介入、checkpoint 或恢复执行，必须先把“状态”和“状态转移”从 prompt 输出中抽离出来。
- Agent 状态一旦成为运行时核心抽象，后续测试、审计、SSE 事件、回放、指标和持久化都可以围绕同一模型建设；如果继续拖延，未来迁移成本会更高。

如果不处理，系统会继续处于“流程能跑，但状态不可证明”的阶段：短期会让 stage 改动越来越危险，长期会阻碍 Agent 框架化和可运维化。

## 2. 目标

- 目标 1：参考 LangGraph `AgentState` 思路，为 ReAct Agent 引入统一、显式、强类型的 `AgentState` 接口。
- 目标 2：将 Agent 编排模型从“stage 输出串联”升级为“基于状态转移的 FSM”，让状态、阶段和转移条件成为一等公民。
- 目标 3：定义最小状态集及其 invariant，保证必要能力完整，同时避免把所有临时数据都塞入共享状态。
- 目标 4：明确每个 stage 对 `AgentState` 的读写权限，形成可校验的依赖边界。
- 目标 5：为调试、审计、回放、恢复执行和后续扩展提供统一的状态快照与转移日志基础。

## 3. 非目标

- 非目标 1：本次不重写现有 think、act、observe 的 prompt 内容，只调整其输入输出接入方式。
- 非目标 2：本次不把 ReAct 直接扩展为完整 DAG 编排系统，仍以有限状态机模型为主。
- 非目标 3：本次不引入分布式持久化状态存储，初期只要求进程内状态模型与可选事件日志。
- 非目标 4：本次不解决所有历史 schema 的统一问题，重点是 Agent 运行态而不是全局业务数据模型。

## 4. 现状与约束

技术现状：

- `component/agent/agent.go` 中的 `OrderOrchestrator.Run` 以 `schema.Schema` 为统一输入输出，通过 `FlowChan` 顺序传递 stage 结果。
- `component/agent/react/react.go` 中的 `stageTypeRegistry` 只声明了 prompt 输入输出类型，没有声明共享状态依赖、权限和状态转移规则。
- 当前 `think -> act -> observe` 的阶段关系是硬编码的执行顺序，不是显式状态机。
- 运行中间态主要散落在 `schema.ThinkInput`、`schema.ThinkOutput`、`schema.ToolOutputs`、`schema.Observation` 和 memory 上下文里。

依赖现状：

- 已有 `genkit` flow、streaming、memory、tool calling、SSE 反馈机制，新方案需要继续兼容这些能力。
- 当前大量逻辑基于 `schema.Schema` 和运行时类型断言实现，新方案不能一次性破坏全部接口。

兼容性约束：

- 需要兼容现有 ReAct 配置和 `think/act/observe` 的默认行为。
- 需要保留现有流式输出能力，不能因为引入状态机而阻断 observe 的增量返回。
- 新方案应允许渐进迁移，至少在一段时间内兼容“老 stage 输出”和“新状态写入”并存。

## 5. 方案设计

### 5.1 总体方案

核心思路是将 Agent 的运行核心从“每个 stage 直接接收上一个 stage 的输出”改为“所有 stage 围绕同一个 `AgentState` 工作”。`AgentState` 保存当前轮次中被系统认可的最小状态集；每个 stage 必须声明自己读取哪些状态、写入哪些状态、满足什么前置条件、成功后把状态转移到哪里。

在编排层引入 FSM 模型：每个 stage 不再只是一个 flow，而是一个带有 `Preconditions`、`ReadSet`、`WriteSet`、`Transition` 的状态转换器。编排器每次调度时先校验当前状态是否满足 invariant 和前置条件，再执行 stage，最后提交状态变更并记录状态转移事件。这样做可以把“依赖关系”“非法状态”“阶段越权写入”“状态缺失”等问题提前暴露为框架错误，而不是把风险推迟到 prompt 执行后。

### 5.2 架构图或流程图

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

### 5.3 关键改动

#### 模块 A：新增强类型 `AgentState`

- 定义统一状态容器，替代当前依赖 `schema.Schema` 顺序传递的方式。
- 建议拆分为“生命周期状态 + 业务状态 + 元数据”三层，而不是一个巨大结构体。
- 示例：

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

#### 模块 B：定义最小状态集与 invariant

- 最小状态集原则：只保留“跨 stage、跨迭代、可审计”的事实，不保留纯临时 prompt 拼接数据。
- 建议初期最小状态集包括 `Lifecycle`、`SessionID`、`UserInput`、`Iteration`、`Thought`、`Action`、`Observation`、`LastError`。
- 关键 invariant 示例如下：
- `Lifecycle == StateInitialized` 时，`UserInput != nil`，`Thought/Action/Observation == nil`。
- `Lifecycle == StateActing` 时，`Thought != nil` 且 `Thought.Intent` 已确定。
- `Lifecycle == StateObserving` 时，若进入路径来自 `act`，则 `Action != nil`。
- `Lifecycle == StateFinished` 时，`Observation != nil` 且 `Observation.FinalAnswer != ""`。
- `Iteration >= 0`，且每次回到 `Thinking` 时只能单调递增。

#### 模块 C：stage 接口升级为状态转换器

- 当前 `Stage.Execute(ctx, chans, input schema.Schema)` 需要升级为围绕 `AgentState` 的接口。
- 建议接口：

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

- `StatePatch` 用于表达本 stage 合法提交的变更集，避免 stage 直接持有整个状态对象并任意修改。

#### 模块 D：读写权限与依赖校验

- 每个 stage 需要显式声明读写边界，并由 orchestrator 在运行前后校验。
- 建议首版权限矩阵改为列表描述：
- `think` 可读取 `SessionID`、`UserInput`、`Iteration`、历史 `Observation`；可写入 `Thought`、`Lifecycle`；禁止写入 `Action`、`Observation`。
- `act` 可读取 `SessionID`、`Iteration`、`Thought`；可写入 `Action`、`Lifecycle`；禁止写入 `UserInput`、`Thought`、`Observation`。
- `observe` 可读取 `SessionID`、`Iteration`、`Thought`、`Action`；可写入 `Observation`、`Lifecycle`；禁止写入 `UserInput`、`Thought`、`Action`。
- `error-handler` 允许任意只读；可写入 `LastError`、`Lifecycle`；禁止写入业务状态字段。
- 如果 stage 提交了未声明字段、修改了只读字段，框架应立即报错。

#### 模块 E：FSM 编排器

- 将 `OrderOrchestrator` 演进为 `FSMOrchestrator`。
- 编排器职责包括校验当前状态 invariant、根据 `Lifecycle` 选择可执行 stage、执行 stage 并合并 `StatePatch`、记录状态转移事件和 stage 执行结果、处理循环退出与失败态。
- 这样一来，循环条件不再依赖某个 stage 的隐式输出结构，而是依赖显式状态判断，例如 `Observation.NeedLoop`。

#### 模块 F：审计与可追踪性

- 为每次 stage 执行记录标准化事件，例如：

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

- 事件先支持内存日志或 debug 输出，后续可接入持久化或 SSE。

### 5.4 数据与接口变化

新增接口：

- `AgentState`
- `StateStage`
- `StatePatch`
- `FSMOrchestrator`
- `StateValidator` 或 `InvariantChecker`

字段变更：

- 现有 stage flow 不再把完整业务状态作为 `schema.Schema` 直接串联。
- `think/act/observe` 的输入输出需要改为“从 `AgentState` 取输入，向 `StatePatch` 写结果”。
- `StageInfo` 建议后续扩展声明式元数据，例如：

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

兼容性影响：

- 现有 `OrderOrchestrator`、`Stage.Execute`、`buildStagesFromConfig` 都需要调整。
- 现有基于 `schema.Schema` 的测试需要改造为断言状态快照和状态转移。
- 现有流式 observe 仍可保留，只是最终结果要写回 `AgentState.Observation`。

迁移方式：

- 第一阶段：引入 `AgentState` 和 invariant，但保留 `OrderOrchestrator`，通过适配层把 stage 输出回填到 state。
- 第二阶段：把 stage 改造成 `StateStage`，引入 `StatePatch` 和读写权限校验。
- 第三阶段：以 `FSMOrchestrator` 替换现有顺序编排器，并补齐状态转移日志与测试。

### 5.5 错误处理与降级

可能失败的环节：

- stage 输出无法映射到合法状态补丁。
- 某个 stage 试图在非法生命周期下运行。
- stage 写入未声明字段，或破坏 invariant。
- 迁移期间新旧接口并存，导致状态与旧输出不一致。

失败后的处理方式：

- invariant 校验失败时立即进入 `StateFailed`，并记录 `LastError`，不再继续后续 stage。
- 状态补丁非法时，丢弃本次提交并返回框架错误，避免污染共享状态。
- 对于可恢复错误，例如工具调用失败，可由专门的错误处理 stage 将状态转回 `Thinking` 或直接进入 `Observing` 产出解释性答复。

降级策略：

- 首次落地时保留旧的顺序执行模式作为 fallback，便于对比验证。
- 如果状态机编排出现不可接受的兼容性问题，可短期切回旧 orchestrator，但 `AgentState` 结构仍保留，避免返工。
- 在 observe streaming 路径上，可先允许“流式输出 + 最终一次性提交 Observation”模式，避免一次性重构整个流控逻辑。

## 6. 实现要点

### 6.1 最小状态集原则

最小状态集不是“字段越少越好”，而是“只保留必须由系统负责正确性的状态”。判断标准如下：

- 会被多个 stage 共享读取。
- 会影响状态转移或循环控制。
- 需要被日志、回放、审计或测试引用。
- 丢失后会导致本轮无法继续执行。

反过来说，下列数据不应默认进入共享状态：

- 仅用于 prompt 模板渲染、不会跨 stage 复用的临时字符串。
- 可以从其他状态纯函数推导出来的冗余字段。
- 只用于本地 debug、对业务无语义的原始中间变量。

### 6.2 invariant 应以强类型表达

本次设计需要尽量避免用裸字符串或 `map[string]any` 表达核心状态。原因是 invariant 的价值在于“编译期尽量约束，运行期明确失败”，如果核心状态继续是弱类型，FSM 只会变成表面抽象。

建议：

- 生命周期用枚举类型表达，而不是字符串常量。
- `SessionID`、`Iteration`、`StateField` 等关键概念用独立类型封装。
- `ThoughtState`、`ActionState`、`ObservationState` 分别建模，而不是统一 `map[string]any`。
- 仅在事件日志、调试输出、序列化边界上做弱类型投影。

### 6.3 stage 权限必须成为框架约束

“明确 stage 读写权限”不能只写在文档里，必须变成框架行为：

- 执行前校验 `ReadSet` 对应字段是否满足前置条件。
- 执行后校验 `StatePatch` 是否只包含 `WriteSet` 内字段。
- 合并状态后再次跑 invariant 校验。

这样才能把“难以调试的隐式依赖”变成“可定位的框架错误”。

### 6.4 增加 checkpoint 与回放预留

一旦状态机落地，建议预留两类能力接口，哪怕首版不完整实现：

- `Snapshot()`：导出当前 `AgentState`，为断点恢复和测试夹具服务。
- `Replay(events []StateTransitionEvent)`：基于事件重建状态，服务调试与审计。

这两项不是首版必须上线的用户功能，但应在接口设计时预留，否则后续又会退回侵入式改造。

## 7. 实施步骤

1. 在 `component/agent` 下引入 `AgentState`、`LifecycleState`、`StateField` 和 invariant 校验器。
2. 为当前 `think/act/observe` 增加适配层，把既有输出映射成 `StatePatch`，先不改 prompt 逻辑。
3. 将 `Stage` 升级为带有 `ReadSet`、`WriteSet`、`Check` 的 `StateStage`。
4. 实现 `FSMOrchestrator`，支持显式状态转移、最大迭代和失败态。
5. 补充状态转移日志、调试输出和单元测试，重点覆盖非法状态、越权写入、循环控制和 streaming observe。
6. 在验证稳定后移除旧的“stage 输出直接串联”主路径。

## 8. 风险与后续任务

- 风险 1：如果最小状态集定义不当，可能出现“状态过肥”或“状态仍不足以支撑流程”的两头失衡。
- 风险 2：迁移期新旧模型并存，会增加一段时间的理解和测试成本。
- 风险 3：observe 的流式输出与最终状态提交存在天然双轨，需要仔细处理一致性。

后续任务建议包括：

- 为状态机引入声明式配置校验，防止无效 stage 图进入运行时。
- 为状态转移增加指标，例如 stage 成功率、转移次数、平均迭代数、失败类型分布。
- 评估是否将 Memory、Tool 调用结果和 SSE 事件统一纳入状态事件流，形成完整回放链路。
