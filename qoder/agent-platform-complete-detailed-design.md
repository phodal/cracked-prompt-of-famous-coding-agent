# 基于 Qodercli 反推的 Agent 平台完整详细设计

## 1. 文档目标

本文不是对 `qodercli` 的继续描述，而是基于前面的逆向分析，沉淀出一套足够用于设计和实现 Agent 平台的完整详细设计。

目标是回答三个问题：

1. 如果要从零设计一个类似 Qodercli 的 Agent 平台，核心系统应该有哪些。
2. 这些系统在 Go 里应该如何分层、建模、协作。
3. 什么是 MVP，什么是必须一开始就做对的底层能力。

这份设计默认平台定位为：

一个本地优先、终端优先、可多 agent 编排、可接入 MCP、支持工具权限控制、支持会话与内存管理、支持 spec 工作流的 Agent Runtime 平台。

## 2. 平台目标与非目标

### 2.1 平台目标

平台应该支持：

1. 单 agent 对话执行。
2. 多 agent 编排和子任务分发。
3. 工具调用、风险识别和权限确认。
4. Session / Memory / Plan / Spec 的结构化状态管理。
5. 本地工具和 MCP 工具的统一接入。
6. CLI、TUI、非交互模式三种使用方式。
7. 项目级与全局级资源加载。
8. 可恢复、可审计、可扩展的工程工作流。

### 2.2 非目标

第一阶段不追求：

1. 分布式多节点调度。
2. 服务端多租户 SaaS。
3. 大规模向量检索基础设施。
4. 浏览器端 Web IDE 完整替代终端界面。

这里的重点是先把“本地 Agent 平台 Runtime”做扎实，而不是先做大而全的云平台。

## 3. 总体架构

建议把平台拆成九层。

### 3.1 接入层

负责用户入口与运行模式：

- CLI 命令入口
- TUI 交互入口
- 非交互 stdin/stdout 模式
- 后续可扩展 HTTP / RPC 入口

### 3.2 调度层

负责本次用户请求的运行控制：

- session 初始化
- agent 选择
- mode 切换
- turn 循环控制
- stop reason 处理

### 3.3 Agent Runtime 层

负责 agent 的执行逻辑：

- prompt 组装
- provider 调用
- tool call 解析
- subagent 分发
- state 更新

### 3.4 Tool Protocol 层

负责工具契约与执行：

- 参数模型
- 权限模型
- 结果模型
- 元数据模型
- 实际执行器

### 3.5 Permission & Policy 层

负责风险治理：

- 工具风险评估
- 用户确认请求
- allow/deny/filter 策略
- yolo / safe mode

### 3.6 Resource Layer

负责所有可加载资产：

- agents
- skills
- commands
- prompts
- hooks
- plugins
- MCP servers

### 3.7 State & Memory Layer

负责运行态数据与长期上下文：

- conversation session
- session memory
- repository memory
- plan/spec artifacts
- job/worktree state

### 3.8 Integration Layer

负责外部系统接入：

- model providers
- MCP
- GitHub
- Git / worktree
- shell
- file system

### 3.9 Observability & Governance Layer

负责可审计与运维：

- logs
- metrics
- tool breakdown
- token accounting
- audit trail
- panic / recovery report

## 4. 核心设计原则

### 4.1 Runtime First

平台的本体是 runtime，不是 prompt 仓库。prompt 只是输入资产，真正决定平台质量的是：

- state 设计
- tool 协议
- permission 机制
- provider 适配层
- resource 加载层

### 4.2 Structured Over Ad-hoc

所有关键对象都必须结构化：

- Tool 不能只有字符串名。
- Agent 不能只是 system prompt 文本。
- Session 不能只是 message 数组。
- Memory 不能只是拼接聊天摘要。

### 4.3 Artifact as State

spec、plan、memory、session 文件不是“附件”，而是平台状态的一部分。

### 4.4 Permission by Design

权限不是 UI 按钮，而是每个工具契约的一部分。

### 4.5 Human Checkpoint for Risk

有副作用的操作必须可中断、可确认、可拒绝、可恢复。

## 5. 核心领域模型

下面给出平台应有的核心对象。

### 5.1 Agent Definition

```go
type AgentDefinition struct {
    Name         string
    Description  string
    Model        string
    Tools        []string
    SystemPrompt string
    Location     ResourceLocation
    StoragePath  string
    Color        string
    Labels       []string
    NeedAskUser  bool
}
```

### 5.2 Skill Definition

```go
type SkillDefinition struct {
    Name         string
    Description  string
    AllowedTools []string
    Labels       []string
    Content      string
    Location     ResourceLocation
}
```

### 5.3 Tool Specification

```go
type ToolSpec[P any, Perm any, Meta any] struct {
    Name                string
    Description         string
    ParamSchema         Schema
    PermissionSchema    Schema
    MetadataSchema      Schema
    PermissionEvaluator PermissionEvaluator[P, Perm]
    Handler             ToolHandler[P, Meta]
}
```

### 5.4 Session

```go
type Session struct {
    ID              string
    Workspace       string
    CreatedAt       time.Time
    UpdatedAt       time.Time
    Messages        []Message
    AgentState      AgentState
    SessionMemory   SessionMemoryState
    CurrentPlanPath string
    CurrentSpecPath string
    Jobs            []JobRef
    Metadata        map[string]string
}
```

### 5.5 Permission Request

```go
type PermissionRequest struct {
    ID              string
    ToolName        string
    RiskLevel       RiskLevel
    Summary         string
    CommandPreview  string
    FileTargets     []string
    Reason          string
    SuggestedPolicy PermissionPolicy
}
```

### 5.6 MCP Server

```go
type MCPServer struct {
    Name        string
    Transport   string
    Endpoint    string
    Headers     map[string]string
    Environment map[string]string
    Scope       Scope
    AuthConfig  *MCPAuthConfig
}

type StatefulMCPServer struct {
    Server      MCPServer
    Connected   bool
    LastError   error
    Tools       []RemoteTool
    LastSyncAt  time.Time
}
```

## 6. 推荐 Go 包结构

建议源码从一开始就按能力分层，不要按“页面”和“命令”横切。

```text
agent-platform/
  cmd/
    root/
    start/
    jobs/
    mcp/
    update/
    status/
    run/
  core/
    agent/
      runner/
      state/
      session_memory/
      permission/
      provider/
      prompt/
      planner/
      spec/
      hooks/
      mode/
    tools/
      registry/
      builtin/
      protocol/
      permission/
      execution/
    resource/
      loader/
      agent/
      skill/
      command/
      hook/
      plugin/
      mcp/
    session/
    memory/
    config/
    types/
    jobs/
    runtime/
    audit/
    logging/
  tui/
    app/
    state/
    components/
    theme/
  integrations/
    model/
      openai/
      anthropic/
      internal/
    github/
    git/
    shell/
    mcp/
  internal/
    storage/
    jsonschema/
    fs/
```

## 7. 运行时生命周期设计

### 7.1 启动流程

1. 解析 CLI 参数。
2. 解析工作目录与作用域配置。
3. 初始化日志、状态目录、临时目录。
4. 加载资源：agent、skill、command、hook、plugin、MCP。
5. 初始化 provider registry。
6. 初始化 tool registry。
7. 加载或创建 session。
8. 决定运行模式：interactive / print / stream-json / worktree job。
9. 启动 agent runner。

### 7.2 单轮执行流程

1. 收到用户输入。
2. 生成当前上下文快照。
3. 组装 system prompt、skills、memory、attachments、tool list。
4. 调用模型 provider。
5. 解析输出：文本、tool call、subagent 请求、stop reason。
6. 如果有 tool call，进入工具执行循环。
7. 更新 session state。
8. 根据 stop reason 决定是否进入下一轮。

### 7.3 Stop Reason 机制

建议显式定义 stop reason，不要靠字符串猜：

```go
type StopReason string

const (
    StopCompletion          StopReason = "completion"
    StopQuestionMode        StopReason = "question_mode"
    StopWaitingMode         StopReason = "waiting_mode"
    StopSummaryMode         StopReason = "summary_mode"
    StopExplanationMode     StopReason = "explanation_mode"
    StopUncertaintyMode     StopReason = "uncertainty_mode"
    StopCompletionIllusion  StopReason = "completion_illusion"
)
```

这样 UI 和 runner 可以统一处理阶段性停顿、权限确认和用户输入等待。

## 8. Provider 路由设计

平台必须支持多 provider，不要把 prompt 构造和 SDK 绑定在一起。

### 8.1 统一 Provider 接口

```go
type ModelProvider interface {
    Name() string
    SupportsTools() bool
    SupportsStreaming() bool
    SupportsImages() bool
    CreateResponse(ctx context.Context, req ModelRequest) (ModelResponse, error)
}
```

### 8.2 Model Routing

建议把 `auto`、`efficient`、`performance`、`ultimate` 设计成逻辑档位，不直接暴露底层模型耦合：

```go
type ModelTier string

const (
    TierAuto        ModelTier = "auto"
    TierEfficient   ModelTier = "efficient"
    TierPerformance ModelTier = "performance"
    TierUltimate    ModelTier = "ultimate"
)
```

路由器根据：

- 任务复杂度
- 是否需要 tool calling
- 是否需要长上下文
- 用户显式指定
- 企业策略

决定最终 provider 和 model。

## 9. Tool Protocol 详细设计

这是整个平台最关键的子系统之一。

### 9.1 工具定义结构

每个工具必须包含：

1. 参数 schema。
2. 权限解析逻辑。
3. 执行逻辑。
4. 响应元数据。
5. UI 展示文案。

### 9.2 工具分类

建议内建工具至少分六类：

1. 文件工具：Read、Write、Edit、Delete、Glob、Grep。
2. 终端工具：Bash、BashOutput、BashKill。
3. 交互工具：AskUser、ExitMode、EnterPlanMode。
4. 计划工具：TodoWrite、GetSpecFilePath、GetPlanFilePath。
5. 外部资源工具：WebFetch、GitHub、MCP Tool。
6. 媒体工具：ReadImage、ImageGen。

### 9.3 工具执行协议

建议所有工具执行都遵守统一返回：

```go
type ToolExecutionResult[Meta any] struct {
    Success      bool
    Content      []ContentPart
    Metadata     Meta
    DurationMs   int64
    ErrorMessage string
    Retryable    bool
}
```

### 9.4 工具注册中心

```go
type ToolRegistry interface {
    Register(spec any) error
    Get(name string) (any, bool)
    List() []ToolDescriptor
    Filter(policy ToolPolicy) []ToolDescriptor
}
```

## 10. 权限系统详细设计

### 10.1 权限层级

建议分四层：

1. 平台默认策略。
2. agent/skill 允许工具范围。
3. 当前 session 的 allow / deny 设置。
4. 单次工具调用的动态风险评估。

### 10.2 风险等级

```go
type RiskLevel string

const (
    RiskLow    RiskLevel = "low"
    RiskMedium RiskLevel = "medium"
    RiskHigh   RiskLevel = "high"
)
```

风险判定维度：

- 是否写文件。
- 是否执行 shell。
- 是否访问网络。
- 是否修改 Git 状态。
- 是否调用外部 MCP。
- 是否有副作用。

### 10.3 权限流程

1. agent 产生 tool call。
2. permission evaluator 生成 `PermissionRequest`。
3. 若低风险且策略允许，自动通过。
4. 若中高风险，进入 AskUser / Permission UI。
5. 用户选择 allow once / allow session / deny / yolo。
6. policy engine 更新 session policy。

### 10.4 Command Filter

对 shell 工具必须支持 command filter：

- 禁止危险命令名单。
- 路径白名单 / 黑名单。
- 网络访问规则。
- 输出大小限制。
- 后台任务超时与 kill 策略。

## 11. Session 与 Message 模型

### 11.1 Message 结构

```go
type Message struct {
    ID          string
    Role        string
    Content     []ContentPart
    CreatedAt   time.Time
    ToolCalls   []ToolCall
    ToolResults []ToolResult
    Metadata    map[string]string
}
```

### 11.2 Session State 结构

```go
type AgentState struct {
    CurrentAgent       string
    CurrentMode        string
    CurrentStopReason  StopReason
    Iteration          int
    MaxIterations      int
    BusinessStage      string
    RecoveryCount      int
    RecoveryLimit      int
    ToolBreakdown      map[string]int
}
```

### 11.3 Session 文件设计

建议 session 持久化为：

- `session.json`
- `session.json.tmp`
- `session_memory.md`
- `session.meta.json`

这样可以把高频变化和低频摘要分开写。

## 12. Session Memory 设计

二进制里最值得借鉴的一个子系统就是 `session_memory`。

### 12.1 为什么需要 Session Memory

不能把所有历史都永久塞进模型上下文，否则会遇到：

- token 膨胀
- tool 调用历史污染
- 当前意图被旧内容稀释

因此需要单独的 session memory 子系统做摘要与压缩。

### 12.2 Session Memory 目标

1. 控制上下文 token。
2. 保留长期相关信息。
3. 丢弃已完成但低价值细节。
4. 记录计划、结论、用户偏好、未决问题。

### 12.3 建议结构

```go
type SessionMemoryState struct {
    MemoryPath            string
    LastSummarizedMsgID   string
    LastUpdateAt          time.Time
    PendingToolCallCount  int
    PendingContent        bool
    CompactConfig         SMCompactConfig
}

type SMCompactConfig struct {
    MaxTokens             int
    TriggerAfterTurns     int
    TriggerAfterToolCalls int
    SafetyMargin          int
}
```

### 12.4 更新机制

推荐像 Qodercli 一样设计异步串行队列：

- 新 turn 完成后发 compact 请求。
- `SerialQueue` 串行化 compact 任务。
- 如果还有 pending tool use，不立即压缩。
- 只在完整 turn 结束后更新 `LastSummarizedMsgID`。

### 12.5 Memory 文件模板

建议 session memory 结构化成：

- 当前目标
- 已完成工作
- 关键决策
- 未决问题
- 后续计划
- 用户偏好

## 13. Resource Loader 设计

### 13.1 资源种类

平台应支持：

- global agents
- project agents
- global skills
- project skills
- commands
- hooks
- plugins
- MCP definitions

### 13.2 加载顺序

建议：

1. 内建资源。
2. 全局资源。
3. 用户作用域资源。
4. 项目资源。
5. 临时 CLI 注入资源。

后加载的同名资源可覆盖前者，但必须保留冲突告警。

### 13.3 资源接口

```go
type ResourceLoader interface {
    LoadAgents(ctx context.Context, scope Scope) ([]AgentDefinition, error)
    LoadSkills(ctx context.Context, scope Scope) ([]SkillDefinition, error)
    LoadCommands(ctx context.Context, scope Scope) ([]CommandDefinition, error)
    LoadHooks(ctx context.Context, scope Scope) ([]HookDefinition, error)
    LoadMCPServers(ctx context.Context, scope Scope) ([]MCPServer, error)
}
```

## 14. Subagent 与 Task 机制设计

### 14.1 Subagent 调用模型

每次 subagent 调用必须被视为新会话。调度方必须显式传入：

- skill / agent name
- 背景信息
- 当前任务
- 已有 artifacts
- 语言要求
- 工具权限

### 14.2 Task Tool 契约

建议 Task 工具参数定义为：

```go
type TaskParams struct {
    Description  string `json:"description"`
    Prompt       string `json:"prompt"`
    SubagentType string `json:"subagent_type"`
}

type TaskResult struct {
    Content            []ContentPart `json:"content"`
    TotalDurationMs    int64         `json:"totalDurationMs"`
    TotalTokens        int           `json:"totalTokens"`
    TotalToolUseCount  int           `json:"totalToolUseCount"`
}
```

### 14.3 Subagent 运行策略

必须支持：

- read-only subagent
- implementation subagent
- reviewer subagent
- browser subagent
- general-purpose subagent

子 agent 的工具集应由 agent definition 和 runtime policy 的交集得到。

## 15. Plan 与 Spec 工作流引擎设计

### 15.1 为什么要单独建模

Plan 与 Spec 不是普通聊天状态，它们有明确文件路径、生命周期和阶段切换逻辑，应该独立建模。

### 15.2 Plan 状态

```go
type PlanState struct {
    FilePath      string
    Exists        bool
    Approved      bool
    CurrentStage  string
    LastUpdatedAt time.Time
}
```

### 15.3 Spec 状态

```go
type SpecState struct {
    TaskName         string
    SpecDir          string
    RequirementPath  string
    VerifyPath       string
    HLDPath          string
    LLDOverviewPath  string
    CurrentStage     SpecStage
    ReviewRound      int
}
```

### 15.4 Spec 引擎职责

建议单独实现 `SpecEngine`，负责：

- 初始化 spec 目录。
- 生成 task name。
- 决定当前阶段允许哪些动作。
- 管理用户 checkpoint。
- 管理 review 循环。
- 管理 verify-fix 循环。
- 管理任务状态更新。

不要把这些逻辑散落在 prompt 文本和 UI 里。

## 16. Job 与 Worktree 并发设计

### 16.1 Job 模型

```go
type Job struct {
    ID           string
    Workspace    string
    WorktreePath string
    Branch       string
    SessionID    string
    Status       JobStatus
    StartedAt    time.Time
    UpdatedAt    time.Time
}
```

### 16.2 设计原则

1. 每个并发 job 有自己的 workspace / worktree。
2. 每个 job 绑定独立 session。
3. session memory 与 artifacts 默认不跨 job 混用。
4. 主界面只能展示 job 摘要，不要共享内部状态对象。

### 16.3 Job 管理功能

至少支持：

- create
- list current workspace jobs
- list all jobs
- remove / cleanup
- resume by session id

## 17. MCP 子系统详细设计

### 17.1 目标

把 MCP server 当作运行时资源，而不是单次 HTTP endpoint。

### 17.2 生命周期

1. 从配置加载 MCP server 定义。
2. 初始化 transport。
3. 认证，如 OAuth。
4. 拉取 tool list。
5. 注册到 unified tool registry。
6. 定期 refresh / reconnect。

### 17.3 MCP 管理器接口

```go
type MCPManager interface {
    Add(ctx context.Context, server MCPServer) error
    Remove(ctx context.Context, name string) error
    List(ctx context.Context) ([]StatefulMCPServer, error)
    Get(ctx context.Context, name string) (StatefulMCPServer, error)
    Authenticate(ctx context.Context, name string) error
    RefreshTools(ctx context.Context, name string) error
}
```

### 17.4 工具融合策略

本地工具与 MCP 工具必须进入统一 registry，但需要在 descriptor 上区分来源：

- builtin
- plugin
- mcp
- project

## 18. TUI 设计

### 18.1 组件树

建议 TUI 至少包含：

- 消息面板
- 输入编辑器
- 工具调用面板
- 权限确认弹窗
- Ask User 面板
- 文件选择器
- 进度状态栏
- job 列表视图

### 18.2 状态驱动

TUI 不要持有业务逻辑，只消费状态事件：

- message appended
- tool requested
- permission pending
- user input required
- session memory updated
- job status changed

### 18.3 交互模式

建议支持：

- normal mode
- question mode
- plan mode
- spec mode
- waiting mode

这样 UI 可以针对不同阶段切换控件和提示。

## 19. CLI 与非交互模式设计

### 19.1 非交互模式

应支持：

- 单轮 prompt 输出文本
- JSON 输出
- stream-json 输出
- 附件输入
- workspace 指定
- model 指定
- max turns 控制

### 19.2 结构化输出

建议所有非交互输出都有标准 envelope：

```json
{
  "sessionId": "...",
  "status": "ok",
  "stopReason": "completion",
  "messages": [],
  "toolBreakdown": {},
  "tokens": {}
}
```

## 20. 配置系统设计

### 20.1 配置作用域

建议至少四级：

- builtin
- global
- user
- project

### 20.2 配置内容

需要覆盖：

- 默认 model tier
- provider credentials
- tool allow/deny policy
- memory 路径
- hook 配置
- MCP server
- UI 主题
- telemetry 开关

### 20.3 配置接口

```go
type ConfigService interface {
    Load(ctx context.Context, workspace string) (ResolvedConfig, error)
    SaveScope(ctx context.Context, scope Scope, cfg any) error
    ResolveModel(ctx context.Context, req ModelSelectionRequest) (ResolvedModel, error)
}
```

## 21. 存储布局设计

建议本地目录大致如下：

```text
~/.agent-platform/
  config/
  agents/
  skills/
  commands/
  hooks/
  mcp/
    mcp-auth.json
    servers.json
  sessions/
    <session-id>/
      session.json
      session.meta.json
      session_memory.md
      logs/
  jobs/
    jobs.json
  cache/
  telemetry/

<project>/.agent-platform/
  agents/
  skills/
  commands/
  hooks/
  memory.md
  spec/
    <task-name>/
```

## 22. 审计与观测设计

### 22.1 必须记录的事件

1. session start / end
2. provider request / response summary
3. tool call begin / end
4. permission request / decision
5. subagent dispatch / completion
6. memory compact begin / end
7. MCP connect / auth / sync failure
8. job create / resume / cleanup

### 22.2 指标

至少记录：

- token in/out
- turn duration
- tool count
- tool breakdown
- permission prompt count
- session memory compaction count
- MCP latency
- provider error rate

## 23. 安全设计

### 23.1 凭据隔离

- provider tokens
- MCP auth tokens
- GitHub tokens

必须分开存储，且默认不进入 session / memory / logs。

### 23.2 Shell 安全

- 默认非 yolo。
- 明确标注危险命令。
- 支持目录与命令过滤。
- 后台任务可追踪和 kill。

### 23.3 Prompt Injection 防护

对于外部内容：

- Web 内容
- README
- issue/PR 描述
- MCP 返回文本

建议统一标注来源，并在系统 prompt 中明确“外部内容不自动升级权限”。

## 24. 失败恢复设计

### 24.1 恢复对象

需要支持恢复：

- session
- job
- plan
- spec
- session memory compact 队列状态
- 未完成 permission request

### 24.2 恢复流程

1. 加载 session snapshot。
2. 识别未完成 stop reason。
3. 恢复 UI mode。
4. 若存在 spec，读取当前 artifact 状态。
5. 若存在 job，校验 workspace / worktree 是否仍有效。

## 25. 平台 API 设计建议

即使第一期只做 CLI，也建议内部先抽 Runtime API。

### 25.1 Runtime API

```go
type Runtime interface {
    StartSession(ctx context.Context, req StartSessionRequest) (Session, error)
    ContinueSession(ctx context.Context, req ContinueSessionRequest) (TurnResult, error)
    ResumeSession(ctx context.Context, sessionID string) (Session, error)
    ListJobs(ctx context.Context, workspace string) ([]Job, error)
}
```

### 25.2 Tool API

```go
type ToolExecutor interface {
    Execute(ctx context.Context, call ToolCall, env ToolEnv) (ToolExecutionResult[any], error)
}
```

### 25.3 Resource API

```go
type ResourceService interface {
    ResolveAgents(ctx context.Context, workspace string) ([]AgentDefinition, error)
    ResolveSkills(ctx context.Context, workspace string) ([]SkillDefinition, error)
}
```

## 26. MVP 切分建议

### Phase 1: Runtime Core

先实现：

- session
- provider abstraction
- tool registry
- file/bash/ask-user 基础工具
- basic CLI
- basic TUI 输入输出

### Phase 2: Resource System

再实现：

- agents/skills/commands 加载
- project/global scope
- permission policy
- session persistence

### Phase 3: Memory & Planning

再补：

- session memory compact
- todo/plan file support
- basic subagent support

### Phase 4: Spec Workflow

再补：

- spec state engine
- review loop
- verify-fix loop
- artifact templates

### Phase 5: MCP & Ecosystem

最后补：

- MCP manager
- plugin/hook system
- GitHub / browser / external integrations

## 27. 哪些地方必须一开始设计对

如果你要做 Agent 平台，有五件事必须一开始就设计对，否则后面会非常痛苦：

1. Tool 协议必须强类型，不要偷懒做字符串拼接工具。
2. Session / Memory / Artifact 必须分层，不要全塞消息历史。
3. Permission 必须是核心内核，不要最后补对话框。
4. Provider 必须抽象，不要让业务逻辑直接依赖某个 SDK。
5. Resource Loader 必须支持多作用域与覆盖顺序，不要把 agent/skill 写死在代码里。

## 28. 最终建议

如果你想真正设计出一个可扩展 Agent 平台，我建议你把整个系统理解成四个核心协议叠加：

1. `Agent Protocol`：agent 如何定义、运行、切换。
2. `Tool Protocol`：工具如何声明、授权、执行、返回。
3. `Resource Protocol`：skills、subagents、commands、hooks、plugins 如何被发现和加载。
4. `State Protocol`：session、memory、plan、spec、job 如何持久化和恢复。

只要这四个协议定义清楚，CLI、TUI、spec、多 agent、MCP、工作流治理都只是上层装配问题。

反过来说，只要这四个协议没做好，UI 再漂亮、prompt 再强，也只会得到一个不稳定、不可恢复、不可扩展的 agent 工具。
