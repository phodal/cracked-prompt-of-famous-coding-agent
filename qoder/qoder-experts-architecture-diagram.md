# Qoder 多 Agent 架构 — Mermaid 图集

---

## 1. 全局架构总览：三种编排模式

```mermaid
graph TB
    User["👤 用户"]

    subgraph QoderCLI["qodercli (CLI 二进制)"]
        direction TB
        MainAgent["主 Agent<br/>agentRunner"]
        
        subgraph QuestMode["Quest 模式 — 轻量委派"]
            QH["Quest Task Handler<br/>评估复杂度 → 分派"]
            DA["design-agent<br/>设计文档"]
            TE["task-executor<br/>代码实现"]
        end

        subgraph SpecMode["Spec 模式 — 结构化流水线"]
            SL["spec-leader<br/>只读协调者"]
            SRA["spec-requirement-analyser"]
            SQA["spec-qa-analyser"]
            SD["spec-designer"]
            SDR["spec-design-reviewer"]
            SI["spec-implementer"]
            SV["spec-verifier"]
        end
    end

    subgraph QoderIDE["Qoder IDE (桌面二进制)"]
        direction TB
        CoderAgent["Coder Agent / Leader"]
        
        subgraph ExpertsMode["Experts 模式 — 异步团队协作"]
            IM["InboxManager<br/>消息中枢"]
            TaskA["TaskAgent<br/>代码实现"]
            ResA["ResearchAgent<br/>深度研究"]
            CRA["CodeReviewAgent<br/>代码审查"]
            VerA["VerifyAgent<br/>验证"]
        end
    end

    User -->|"CLI"| MainAgent
    User -->|"IDE"| CoderAgent
    MainAgent --> QuestMode
    MainAgent --> SpecMode
    CoderAgent --> ExpertsMode

    style QuestMode fill:#e8f5e9,stroke:#4caf50
    style SpecMode fill:#e3f2fd,stroke:#2196f3
    style ExpertsMode fill:#fff3e0,stroke:#ff9800
```

---

## 2. Experts 模式：Leader-Expert 拓扑与消息流

```mermaid
graph TB
    User["👤 用户"]
    IDE["Qoder IDE UI<br/>|Experts-Message| 通知"]

    subgraph LeaderBox["Leader Agent"]
        Leader["Coder Agent<br/><i>prompt: experts_agent_system_prompt.txt</i><br/><i>(服务端加密下发)</i>"]
        TaskTool["Task 工具<br/>启动 Expert"]
        SendMsg["SendMessage 工具<br/>发送消息"]
        AddTasks["add_tasks 工具<br/>创建任务"]
        TaskList["TaskList 工具<br/>查看任务"]
        TaskUpdate["TaskUpdate 工具<br/>更新任务"]
        WaitNode["WaitInboxNode<br/>等待回复"]
        ReadNode["ReadInboxNode<br/>消费消息"]
    end

    subgraph InboxSys["InboxManager 消息中枢"]
        LInbox["📬 Leader Inbox"]
        E1Inbox["📬 Expert-1 Inbox"]
        E2Inbox["📬 Expert-2 Inbox"]
        E3Inbox["📬 Expert-3 Inbox"]
        FileLock["🔒 FileLock<br/>并发安全"]
    end

    subgraph ExpertPool["Expert Agents (异步并行)"]
        E1["🔧 TaskAgent<br/><i>Alex (researcher)</i><br/><i>Jimmy (backend-dev)</i><br/><i>Lee (frontend-dev)</i>"]
        E2["🔬 ResearchAgent<br/><i>Sam, Tina, Eric</i>"]
        E3["📝 CodeReviewAgent<br/><i>Mark, Ryan, Daniel</i>"]
        E4["✅ VerifyAgent<br/><i>Chris, Terry, Leo</i>"]
    end

    subgraph ExpertPrompt["Expert 注入的 Prompt"]
        EP["&lt;expert_mode&gt;<br/>You are an expert agent<br/>running in a team..."]
        TW["Teammate Workflow<br/>自主领取任务"]
    end

    User -->|"用户消息"| IDE
    IDE -->|"chat/ask"| Leader
    Leader -->|"Task 工具"| E1
    Leader -->|"Task 工具"| E2
    Leader -->|"Task 工具"| E3
    Leader -->|"Task 工具"| E4
    Leader -->|"SendMessage"| E1Inbox
    Leader -->|"SendMessage"| E2Inbox
    Leader -->|"SendMessage"| E3Inbox

    E1 -->|"SendMessage"| LInbox
    E2 -->|"SendMessage"| LInbox
    E3 -->|"SendMessage"| LInbox
    E4 -->|"SendMessage"| LInbox

    WaitNode -.->|"轮询"| LInbox
    ReadNode -.->|"消费"| LInbox

    Leader -->|"通知结果"| IDE
    IDE -->|"显示"| User

    E1 --- EP
    E2 --- EP
    E3 --- EP
    EP --- TW

    InboxSys --- FileLock

    style LeaderBox fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style ExpertPool fill:#e8f5e9,stroke:#2e7d32
    style InboxSys fill:#e3f2fd,stroke:#1565c0
    style ExpertPrompt fill:#fce4ec,stroke:#c62828,stroke-dasharray: 5 5
```

---

## 3. Experts 模式：任务生命周期时序图

```mermaid
sequenceDiagram
    actor User as 👤 用户
    participant IDE as Qoder IDE
    participant Leader as Leader Agent
    participant Inbox as InboxManager
    participant E1 as Expert-1 (Jimmy)
    participant E2 as Expert-2 (Alex)
    participant TaskDB as TaskTree

    User->>IDE: 用户请求 "实现登录功能"
    IDE->>Leader: chat/ask

    Note over Leader: 分析需求，规划任务
    Leader->>TaskDB: add_tasks [任务1: 后端API, 任务2: 前端页面, 任务3: 测试]
    Leader->>TaskDB: TaskUpdate: 任务2.addBlockedBy=[任务1]

    par 并行启动 Experts
        Leader->>E1: Task工具 (backend-dev, name="Jimmy")<br/>任务1: 实现后端API
        Leader->>E2: Task工具 (researcher, name="Alex")<br/>研究现有认证方案
    end

    Note over Leader: WaitInboxNode 等待...

    E2->>Inbox: SendMessage → Leader<br/>"发现项目使用 JWT，建议..."
    Inbox-->>Leader: ReadInboxNode 消费消息
    Note over Leader: "Receive Message From Alex"

    E1->>TaskDB: TaskUpdate: 任务1 → completed
    E1->>Inbox: SendMessage → Leader<br/>"后端API实现完成"
    Inbox-->>Leader: ReadInboxNode 消费消息

    Note over Leader: 任务1完成，任务2解除阻塞
    Leader->>E1: Task工具 (frontend-dev, name="Lee")<br/>任务2: 实现前端页面

    E1->>TaskDB: TaskUpdate: 任务2 → completed
    E1->>Inbox: SendMessage → Leader

    Leader->>E1: Task工具 (qa, name="Chris")<br/>任务3: 运行测试验证

    E1->>TaskDB: TaskUpdate: 任务3 → completed
    E1->>Inbox: SendMessage → Leader

    Leader->>IDE: 所有任务完成，汇总结果
    IDE->>User: 展示完成摘要
```

---

## 4. Spec 模式：结构化开发流水线

```mermaid
graph LR
    User["👤 用户"]

    subgraph Pipeline["Spec 流水线 (spec-leader 协调)"]
        direction LR
        S1["Stage 1.1<br/>spec-requirement-analyser<br/>→ 01-requirement.md"]
        S1Q["Stage 1.2<br/>spec-qa-analyser<br/>→ 01-quality-assurance.md"]
        CP1{"🔒 Checkpoint 1<br/>用户确认"}
        S2["Stage 2<br/>spec-designer<br/>→ 02-design.md"]
        S3["Stage 3<br/>spec-design-reviewer<br/>→ 03-review-round-N.md"]
        CP2{"🔒 Checkpoint 2<br/>用户确认"}
        S4["Stage 4<br/>spec-implementer<br/>→ 代码实现"]
        S5["Stage 5<br/>spec-verifier<br/>→ 测试验证"]
    end

    User --> S1
    S1 --> S1Q
    S1Q --> CP1
    CP1 -->|"确认"| S2
    S2 --> S3
    S3 -->|"最多10轮"| S3
    S3 --> CP2
    CP2 -->|"确认"| S4
    S4 --> S5
    S5 -->|"失败 (最多3次)"| S4
    S5 -->|"通过"| Done["✅ 完成"]

    style CP1 fill:#fff9c4,stroke:#f9a825
    style CP2 fill:#fff9c4,stroke:#f9a825
    style Done fill:#c8e6c9,stroke:#2e7d32
```

---

## 5. Quest 模式：智能分派

```mermaid
graph TB
    User["👤 用户请求"]
    QH["Quest Task Handler<br/>评估复杂度"]

    User --> QH

    QH -->|"简单"| Direct["直接处理<br/>回答问题/简单修改"]
    QH -->|"中等"| TE["task-executor<br/>实现代码"]
    QH -->|"复杂"| DA["design-agent<br/>设计文档"]
    DA --> TE2["task-executor<br/>按设计实现"]

    subgraph Confirm["⚠️ 用户确认"]
        Note["NEVER call Task()<br/>without explicit<br/>user approval"]
    end

    QH -.-> Confirm

    style Direct fill:#c8e6c9
    style Confirm fill:#fff9c4,stroke:#f9a825,stroke-dasharray: 5 5
```

---

## 6. Prompt 加载与注入架构

```mermaid
graph TB
    subgraph Server["☁️ 服务端"]
        EncPrompts["加密 Prompt 模板<br/>prompt/*.txt"]
    end

    subgraph IDE["Qoder IDE"]
        EES["EncryptedEngineService<br/>解密 + 渲染"]
        
        subgraph Templates["Prompt 模板 Key"]
            T1["experts_agent_system_prompt.txt<br/>(Leader 核心 Prompt)"]
            T2["coder_agent_*_system_prompt.txt<br/>(多模型变体: Opus/G5/Codex/Gemini3)"]
            T3["research_agent_system_prompt.txt"]
            T4["code_review_agent_system_prompt.txt"]
            T5["design_agent_system_prompt.txt"]
            T6["long_run_agent_system_prompt.txt"]
        end

        subgraph Injection["运行时注入"]
            GoTpl["Go 模板变量<br/>{{ .CodeFileContent }}<br/>{{ .Memory }}<br/>{{ .WorkspacePath }}"]
            XMLCtx["XML 上下文标签<br/>&lt;teamDocs&gt;<br/>&lt;wiki_knowledge&gt;<br/>&lt;conversation&gt;"]
            Rules["Rules 注入<br/>&lt;always_on_rules&gt;<br/>&lt;rule name=...&gt;"]
            SysRem["System Reminders<br/>&lt;system_reminder&gt;<br/>动态状态提醒"]
        end

        subgraph Embedded["二进制嵌入 Prompt"]
            EP1["&lt;expert_mode&gt; 块"]
            EP2["SendMessage 工具描述"]
            EP3["Task/TaskList/TaskUpdate 工具描述"]
            EP4["Teammate Workflow"]
            EP5["CodeReview Agent 系统提示词"]
            EP6["Agent 命名规范"]
        end
    end

    Server -->|"加密下发"| EES
    EES -->|"解密渲染"| Templates
    Templates --> Injection
    Injection -->|"组装完整 Prompt"| FinalPrompt["最终 Agent Prompt"]
    Embedded -->|"直接拼接"| FinalPrompt

    style Server fill:#e8eaf6,stroke:#3f51b5
    style Embedded fill:#fce4ec,stroke:#c62828
    style Templates fill:#e3f2fd,stroke:#1565c0
    style Injection fill:#f3e5f5,stroke:#7b1fa2
```

---

## 7. Expert Agent 工具集与权限矩阵

```mermaid
graph LR
    subgraph LeaderTools["Leader 工具集"]
        LT1["Task — 启动 Expert"]
        LT2["SendMessage — 发消息"]
        LT3["add_tasks — 创建任务"]
        LT4["TaskList — 查看任务"]
        LT5["TaskUpdate — 更新任务"]
        LT6["Read/Grep — 读代码"]
        LT7["Codebase — 符号搜索"]
    end

    subgraph ExpertTools["Expert 通用工具集"]
        ET1["SendMessage — 与 Leader 通信"]
        ET2["TaskUpdate — 标记任务完成"]
        ET3["TaskList — 查找下一个任务"]
        ET4["claimTask — 领取任务"]
        ET5["Read/Write/Grep — 文件操作"]
        ET6["Terminal — 命令执行"]
    end

    subgraph Restrictions["权限限制"]
        R1["disallowedTools<br/>按 Agent 类型限制"]
        R2["Expert 不能启动<br/>其他 Expert"]
        R3["Leader 不直接<br/>写代码"]
    end

    LeaderTools --- Restrictions
    ExpertTools --- Restrictions

    style LeaderTools fill:#fff3e0,stroke:#e65100
    style ExpertTools fill:#e8f5e9,stroke:#2e7d32
    style Restrictions fill:#ffebee,stroke:#c62828,stroke-dasharray: 5 5
```

---

## 8. 记忆系统与 Expert 经验

```mermaid
graph TB
    subgraph ExpertExec["Expert 执行"]
        E1["Expert Agent 完成任务"]
    end

    subgraph MemoryPipeline["记忆提取 Pipeline"]
        Extract["记忆提取<br/>memoryExtractExpertExperience*"]
        Eval["质量评估<br/>memoryQualityEvaluate*"]
        Store["存储<br/>agent_memory 表"]
    end

    subgraph MemoryCategories["记忆类别"]
        Cat1["expert_experience<br/>Expert 专属经验"]
        Cat2["task_flow_experience<br/>任务流程经验"]
        Cat3["common_pitfalls_experience<br/>常见陷阱"]
        Cat4["learned_skill_experience<br/>学到的技能"]
    end

    subgraph Recall["记忆召回"]
        Prefetch["预取<br/>memory-prefetch"]
        Rerank["重排序<br/>memory-rerank"]
        Inject["注入 Prompt<br/>{{ .Memory }}"]
    end

    E1 --> Extract
    Extract --> Eval
    Eval -->|"quality_score > 阈值"| Store
    Store --> Cat1
    Store --> Cat2
    Store --> Cat3
    Store --> Cat4

    Cat1 --> Prefetch
    Prefetch --> Rerank
    Rerank --> Inject
    Inject -->|"下次对话"| NewSession["新 Session<br/>Leader/Expert"]

    style MemoryPipeline fill:#e8f5e9,stroke:#2e7d32
    style MemoryCategories fill:#e3f2fd,stroke:#1565c0
    style Recall fill:#fff3e0,stroke:#ff9800
```

---

## 9. 三种模式对比总览

```mermaid
graph TB
    subgraph Quest["Quest 模式"]
        direction TB
        Q1["单层委派"]
        Q2["同步调用"]
        Q3["无状态 Agent"]
        Q4["用户确认后执行"]
        Q5["适合: 中小型任务"]
    end

    subgraph Spec["Spec 模式"]
        direction TB
        S1["5阶段流水线"]
        S2["串行执行"]
        S3["Checkpoint 用户确认"]
        S4["Review 循环 (≤10轮)"]
        S5["适合: 大型功能开发"]
    end

    subgraph Experts["Experts 模式"]
        direction TB
        X1["Leader-Expert 星形拓扑"]
        X2["异步并行执行"]
        X3["Inbox 双向消息"]
        X4["任务依赖 blockedBy"]
        X5["适合: 复杂多文件协作"]
    end

    style Quest fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style Spec fill:#e3f2fd,stroke:#2196f3,stroke-width:2px
    style Experts fill:#fff3e0,stroke:#ff9800,stroke-width:2px
```
