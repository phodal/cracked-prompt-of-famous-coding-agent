# Qodercli Go 二进制实现架构分析

## 1. 分析对象与方法

本报告分析的对象不是源码仓库，而是已经编译好的二进制文件：

- `/Users/phodal/ai/qoder-0.5.2/app/resources/bin/aarch64_darwin/qodercli`

由于当前可见的是发行包而不是完整 Go 源码，因此本报告采用以下证据链进行分析：

1. 二进制元信息：`file`、`go version -m`。
2. 二进制字符串：`strings` 输出中的包路径、命令帮助、配置键、错误信息。
3. 已抽取的 prompt 文档与分析文件。
4. 由符号字符串反推的 Go 包组织。

因此，下面内容分为两类：

- 可直接确认的事实。
- 基于符号和字符串推导出的高可信架构判断。

## 2. 可直接确认的事实

### 2.1 二进制基本信息

`qodercli` 是一个：

- macOS `Mach-O 64-bit executable arm64` 二进制。
- 使用 `go1.25.7` 构建。
- `CGO_ENABLED=1`。
- `GOOS=darwin`，`GOARCH=arm64`。
- `-trimpath=true`，说明构建时去掉了本地源码绝对路径。

这几条信息有两个意义：

1. 它不是脚本打包器，而是完整编译的 Go 原生程序。
2. 即使做了 `trimpath`，Go 的类型与包名字符串仍然大量保留在二进制里，因此仍然可以较好地逆向其模块结构。

### 2.2 已知依赖

从 `go version -m` 可以直接确认它依赖了几组非常关键的库：

- CLI：`github.com/spf13/cobra`
- TUI：`github.com/charmbracelet/bubbletea`、`bubbles`、`lipgloss`、`glamour`
- MCP：`github.com/mark3labs/mcp-go`
- 模型接入：`github.com/openai/openai-go`、`github.com/anthropics/anthropic-sdk-go`
- GitHub：`github.com/google/go-github/v57`
- Shell/脚本处理：`mvdan.cc/sh/v3`
- 文件系统与忽略规则：`afero`、`go-gitignore`
- Markdown/HTML：`goldmark`、`html-to-markdown`
- 日志与并发：`zap`、`multierr`、`sourcegraph/conc`

光看这组依赖，就已经能确认这不是一个单纯“聊天 CLI”，而是一个同时覆盖：

- 终端交互
- 文件与项目操作
- 外部模型调用
- MCP 服务接入
- GitHub 集成
- 文档解析与渲染

的综合型终端 AI 工具。

## 3. 二进制暴露出的顶层产品形态

从 `qodercli --help` 可以直接确认 CLI 表面结构。

### 3.1 顶层定位

帮助文本把它定义为：

“terminal-based AI assistant”，支持：

- 交互式聊天
- 非交互单轮 prompt
- 代码分析
- MCP 集成
- 并发 job / worktree 模式

这说明它的产品定位并不是“单次 LLM 命令行调用器”，而是“终端内开发工作台”。

### 3.2 顶层命令面

从帮助输出可见顶层命令包含：

- `jobs`
- `rm`
- `completion`
- `feedback`
- `mcp`
- `status`
- `update`

并且支持：

- `--print` 非交互模式
- `--workspace` 指定工作区
- `--continue` / `--resume` 会话恢复
- `--agents` 动态定义自定义 agent
- `--allowed-tools` / `--disallowed-tools` 工具权限控制
- `--with-claude-config` 加载 `.claude` 配置
- `--worktree` 并发工作树任务

这些选项说明它的设计不是“先有 UI，后补工程能力”，而是从一开始就把：

- agent 配置
- 工具权限
- 会话恢复
- 多工作区
- Git worktree 并发任务

作为一等功能。

## 4. 从包路径反推的源码组织

通过二进制中的内部包路径统计，可以还原出一套非常清楚的模块骨架。

### 4.1 顶层目录骨架

从字符串中能稳定看到以下内部包前缀：

- `cmd/*`
- `core/agent/*`
- `core/resource/*`
- `core/utils/*`
- `core/config`
- `core/types`
- `core/auth/*`
- `core/account`
- `tui/*`
- `sdk/*`
- `acp/*`

这说明它的源码组织大概率采用的是很典型的 Go monolith 分层：

- `cmd` 作为 CLI 入口层。
- `core` 作为核心业务层。
- `tui` 作为终端 UI 层。
- `resource` 作为外部资源与可加载资产层。

### 4.2 `cmd` 层：命令入口分发层

可见的命令包包括：

- `cmd/start`
- `cmd/update`
- `cmd/jobs`
- `cmd/mcp`
- `cmd/message_io`
- `cmd/utils`

结合 Cobra 依赖，可以基本确认：

1. 主程序通过 Cobra 组织命令树。
2. `start` 很可能是默认交互入口。
3. `update` 负责自更新。
4. `jobs` 负责并发作业管理。
5. `mcp` 负责 MCP 服务管理。
6. `message_io` 很可能承接非交互输入输出、流式消息序列化或会话收发。

### 4.3 `tui` 层：终端前端层

二进制中出现频率最高的内部路径之一是 `tui/components/command`，此外还包括：

- `tui/components/messages`
- `tui/components/askuser`
- `tui/components/permission`
- `tui/components/filepicker`
- `tui/components/common/dialog`
- `tui/components/common/textarea`
- `tui/components/interaction/editor`
- `tui/components/interaction/progress`
- `tui/components/interaction/status`
- `tui/components/interaction/selectors`
- `tui/theme`
- `tui/state`
- `tui/util`

这说明它的 TUI 不是简单 stdout 打印，而是有一层完整的终端“前端框架”：

- 消息展示区
- 命令与工具视图
- 权限确认面板
- Ask User 交互组件
- 文件选择器
- 富文本编辑/输入框
- 进度与状态栏
- 主题系统
- UI 状态管理

换句话说，`qodercli` 的交互体验更接近“终端 IDE 面板”，不是“命令执行后打几行字”。

## 5. `core/agent`：真正的运行内核

从包名密度看，`core/agent` 是整个系统最重要的核心之一。

可见子模块包括：

- `core/agent`
- `core/agent/tools`
- `core/agent/tools/utils`
- `core/agent/state`
- `core/agent/state/shell`
- `core/agent/state/file`
- `core/agent/state/message`
- `core/agent/state/session`
- `core/agent/state/session_memory`
- `core/agent/state/memory`
- `core/agent/provider`
- `core/agent/permission`
- `core/agent/prompt`
- `core/agent/hooks/agent_stop`
- `core/agent/option`

这几乎直接揭示了其运行时架构。

### 5.1 `provider`：模型后端抽象层

二进制中可以看到：

- `core/agent/provider.QoderClient`
- `core/agent/provider.OpenAIClient`
- `core/agent/provider.IdeaLabClient`

再结合 OpenAI 和 Anthropic 依赖，可以推断其模型接入层是做了 provider 抽象的。也就是说，agent 不直接绑死单一模型 SDK，而是通过 provider 选择实际后端。

这与命令行参数里的 `--model auto|efficient|performance|ultimate|qmodel|gmodel...` 是一致的。

### 5.2 `state`：会话状态与工作上下文层

`state/*` 路径说明它把 agent 运行所需的上下文拆成多个子状态域：

- shell 状态
- file 状态
- message 状态
- session 状态
- memory 状态

这不是一个简单的 `[]Message` 聊天循环，而是一个带工作内存的执行环境。也就是说，对它来说“会话状态”本质上是一个多维上下文容器，而不仅是聊天历史。

### 5.3 `permission`：权限控制内核

既然 CLI 上有 `--allowed-tools` / `--disallowed-tools` / `--dangerously-skip-permissions`，而 TUI 层又有 `permission` 组件，那么权限控制显然不是外围补丁，而是核心内核功能。

这也解释了为什么 prompt 里大量出现“must ask user”“risk level”“permission params”。

### 5.4 `prompt` 与 `hooks`

`core/agent/prompt` 与 `core/agent/hooks/agent_stop` 说明：

- prompt 本身被当作系统资产管理，而不是临时字符串。
- 运行时支持 hook 生命周期扩展。

这意味着 agent 体系是可配置、可注入、可扩展的，而非硬编码死逻辑。

## 6. `core/agent/tools`：最关键的工具系统实现线索

从二进制字符串中，最重要的一组证据是这种泛型类型：

- `tools.Spec[TaskParams, NotPresent, TaskResult]`
- `tools.Spec[GrepParams, GrepPermissionParams, NotPresent]`
- `tools.Spec[ReadParams, ReadPermissionParams, ReadResponseMetadata]`
- `tools.Spec[BashParams, BashPermissionsParams, BashResponseMetadata]`
- `tools.Spec[EditParams, EditPermissionsParams, EditResponseMetadata]`
- `tools.Spec[WriteParams, WritePermissionsParams, WriteResponseMetadata]`
- `tools.Spec[AskUserQuestionParams, AskUserQuestionPermissionsParams, AskUserQuestionResponseMetadata]`

### 6.1 这说明了什么

这说明 Qodercli 的工具不是“名字 + 回调函数”的简陋注册，而是一个强类型工具规格系统。每个工具至少包含三类类型参数：

1. 参数结构。
2. 权限参数结构。
3. 响应元数据结构。

也就是说，它的工具系统设计大概率类似：

- 工具定义层：声明工具契约。
- 权限解析层：根据参数判断风险与审批。
- 执行层：实际运行。
- 元数据层：把结果附加为结构化信息，供上游 agent 或 UI 使用。

### 6.2 这套设计为什么重要

这比传统 agent 工具系统更工程化，优点是：

- 工具可以做细粒度权限控制。
- 不同工具可以返回不同元数据，而不是只有文本输出。
- UI 能根据元数据渲染状态，而不只是显示 stdout。
- 调用链可以基于结构化结果而不是字符串解析继续工作。

所以，Qodercli 的工具层不是“agent 附件”，而是 agent runtime 的核心协议层。

## 7. `core/resource`：外部资产与可加载对象层

二进制中可以看到以下资源子系统：

- `core/resource/mcp`
- `core/resource/skill`
- `core/resource/subagent`
- `core/resource/command`
- `core/resource/plugin`
- `core/resource/hook`

这几乎把产品的“可扩展对象”类型都列全了。

### 7.1 `skill` 与 `subagent`

二进制里直接暴露了两类 YAML 结构：

#### Skill 结构

- `name`
- `description`
- `allowed-tools`
- `labels`
- `content`
- `location`

#### Subagent 结构

- `name`
- `description`
- `model`
- `tools`
- `systemPrompt`
- `location`
- `storagePath`
- `color`
- `labels`
- `ask-user`

这说明：

1. Skill 更像能力包或行为模板。
2. Subagent 更像可独立运行的 agent 定义。
3. 二者都支持从不同位置加载。
4. 工具权限是资源层配置的一部分，不是后续临时附加。

### 7.2 `plugin` / `hook` / `command`

二进制中还能看到：

- `plugin registered`
- `plugin name is required`
- `loaded default hooks file`
- `loaded extra hooks file`
- `global/commands`

这表明除了 skill / agent 之外，它还支持：

- 自定义命令资源
- 插件注册
- hook 文件加载

所以它的资源模型并不是单一 prompt 配置，而是一个完整的扩展资产系统。

## 8. MCP 子系统不是外设，而是一等公民

`qodercli mcp --help` 已经直接显示其支持：

- `add`
- `auth`
- `get`
- `list`
- `remove`

而从二进制符号还能看到 `core/resource/mcp` 下有：

- `McpServer`
- `StateMcpServer`
- `Tool`

以及大量并发池结构：

- `ResultPool[StateMcpServer]`
- `ResultErrorPool[StateMcpServer]`
- `ResultContextPool[StateMcpServer]`
- `resultAggregator[StateMcpServer]`

### 8.1 架构含义

这说明 MCP 在它内部不是“远端工具列表”这么简单，而是有一层状态化 server 封装：

- 服务器对象有连接状态。
- 可能需要 OAuth 认证。
- 拥有初始化流程。
- 拥有工具列表同步。
- 支持并发聚合多个 MCP server 的结果。

这类设计通常意味着：

1. MCP server 可被视为长期连接资源，而不是一次性请求。
2. agent 工具表可能是本地工具与远端 MCP 工具的融合视图。
3. MCP 可直接参与运行时工具选择和权限决策。

## 9. 配置与持久化模型

从字符串可以确认它至少存在这些路径和持久化对象：

- `global/agents`
- `project/agents`
- `global/skills`
- `project/skills`
- `global/commands`
- `session.json`
- `-session.json.tmp`
- `session_memory`
- `mcp-auth.json`
- `feedback.txt`
- `metadata.json`

### 9.1 作用推断

这说明它采用了多层配置作用域：

- 全局作用域
- 项目作用域
- 可能还有本地或用户作用域

同时还有：

- 会话持久化
- 会话内存
- MCP 认证持久化
- 资源元数据
- 反馈产物

这和前面从 prompt 中看到的“artifact 驱动”“session 恢复”“global/project 资源加载”完全一致。

## 10. 这不是单一 CLI，而是三层系统叠加

综合以上证据，可以把 `qodercli` 还原成三层系统。

### 10.1 第一层：Cobra 命令层

负责：

- 解析 flags
- 子命令分发
- 非交互模式入口
- 管理命令如 update、mcp、jobs

### 10.2 第二层：Bubble Tea TUI 层

负责：

- 消息渲染
- 工具确认
- Ask User UI
- 输入编辑
- 文件选择
- 进度与状态交互

### 10.3 第三层：Agent Runtime 层

负责：

- provider 调度
- 会话状态
- prompt 与 skill 装配
- 工具执行
- 权限控制
- 资源加载
- MCP 整合

也就是说，`qodercli` 并不是“CLI 调一个远端 AI 接口”，而是“一个完整的本地 agent runtime，被 CLI 和 TUI 两套界面驱动”。

## 11. Multi-agent/spec 机制如何落到代码结构上

前面我们从 prompt 层分析了 spec 工作流。二进制实现层给出的证据表明，这套机制并不是只写在 prompt 里，而是落到了运行内核中。

### 11.1 Prompt 是资源，不是常量

有 `core/resource/skill`、`core/resource/subagent`、`core/agent/prompt`，说明 prompt 体系是可加载、可管理的资源系统。

### 11.2 Task 是工具协议的一部分

存在 `tools.Spec[TaskParams, ..., TaskResult]`，说明 Task 不是“prompt 里提了一句”，而是工具系统中的正式一类工具。

### 11.3 权限不是后加的

每个工具都有 `PermissionParams`，再加上 `permission` TUI 组件，说明权限决策从设计上就是一等公民。

### 11.4 Spec 是资源编排，不是源码分支

从目前证据看，spec 工作流更可能是：

- 由资源层加载 prompt/skill。
- 由 agent runtime 组装运行上下文。
- 由 tool system 执行 Task、Read、Grep、Bash、AskUser 等工具。
- 由 TUI 或 non-interactive mode 呈现结果。

因此 spec 机制大概率并不对应某个“专门的巨大状态机文件”，而是由多个系统协作拼起来的：

- 资源层提供行为定义。
- agent 层负责调度与状态。
- tool 层负责动作。
- TUI 层负责交互确认。

## 12. 这套实现的工程特点

### 12.1 强运行时建模

它不是把一堆 prompt 文本塞进程序，而是把：

- agent
- skill
- tool
- permission
- session
- memory
- mcp server

都建模成了显式对象。

### 12.2 本地终端优先

TUI 模块的密度非常高，说明这个产品优先优化的是终端开发体验，而不是先做 API 再顺带做 CLI。

### 12.3 工具系统高度结构化

`tools.Spec` 的泛型设计说明工具系统是强约束、可扩展、可审计的。

### 12.4 配置和资源多作用域

`global/*` 与 `project/*` 路径说明它兼顾通用能力和项目定制能力。

### 12.5 MCP 深度集成

MCP 不是附属插件，而是运行时的一部分。

## 13. 可以高可信推断出的源码结构

如果把当前证据还原成大概源码目录，最可能接近下面这种结构：

```text
qodercli/
  cmd/
    start/
    update/
    jobs/
    mcp/
    message_io/
  core/
    agent/
      provider/
      tools/
      state/
      permission/
      prompt/
      hooks/
    resource/
      mcp/
      skill/
      subagent/
      command/
      plugin/
      hook/
    config/
    types/
    utils/
    auth/
    account/
  tui/
    components/
      command/
      messages/
      permission/
      askuser/
      interaction/
      common/
    theme/
    state/
    util/
```

这不是拍脑袋目录，而是和当前二进制暴露出的包名分布高度一致。

## 14. 当前无法直接确认的部分

因为现在分析的是发行二进制，不是源码，所以仍然有几类信息不能百分百确认：

1. 具体某个命令如何调用 runtime。
2. `spec-leader` 等 prompt 是编译期内嵌、运行时下载，还是二者混合。
3. provider 路由的精确策略。
4. 权限审批的完整决策树。
5. MCP 工具与本地工具的融合点在代码中的精确实现方式。

但这些不影响对整体架构的判断。

## 15. 最终结论

从 Go 二进制实现层看，`qodercli` 可以被定义为：

“一个以 Go 构建的终端 AI 运行时平台，使用 Cobra 提供命令入口，使用 Bubble Tea 提供交互式终端界面，使用 `core/agent` 作为 agent runtime，使用 `tools.Spec` 泛型系统管理工具与权限，使用 `core/resource` 管理 skills/subagents/plugins/hooks/MCP 资源，并通过状态化的 MCP 子系统把外部工具生态接入本地 agent 执行环境。”

如果只看一句最核心的实现特征，那就是：

它不是“调用模型的 CLI”，而是“把 agent 编排、工具协议、权限控制、资源加载和终端交互全部放进一个 Go 本地 runtime 里”的胖客户端。

## 16. 后续建议

如果你要继续深挖，我建议按下面顺序进行：

1. 继续从 `strings` 和 `symbol.txt` 抽 `cmd/start`、`core/agent/provider`、`core/resource/mcp` 相关函数名。
2. 单独做一份 `tools.Spec` 工具系统逆向分析，重点看参数类型、权限类型和元数据类型的分层。
3. 单独做一份 `tui/components/*` 逆向分析，把 UI 组件树和交互流画出来。
4. 单独做一份 provider 与模型路由分析，确认 `qmodel`、`gmodel`、`ultimate` 等别名如何映射到实际后端。
