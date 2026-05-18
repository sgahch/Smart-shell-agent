# WaLiSSH 服务端深度技术架构分析

---

## 一、完整技术栈清单

| 类别 | 技术 | 版本 | 用途 |
|------|------|------|------|
| **运行时** | Spring Boot | 3.4.3 | 应用框架 |
| **语言** | Java | 17 | - |
| **构建** | Maven | 多模块(7个) | - |
| **AI-模型接入** | Spring AI (`spring-ai-openai`) | 1.1.5 | LLM 抽象层，OpenAI兼容API调用 |
| **AI-Agent运行时** | Google ADK (`google-adk`) | 1.2.0 | Agent编排、Runner、Session、Event |
| **AI-桥接** | `google-adk-spring-ai` | 1.2.0 | 将Spring AI ChatModel适配给ADK |
| **AI-桥接** | `google-adk-contrib-langchain4j` | 0.2.0 | 依赖引入但生产代码未使用 |
| **AI-社区工具** | `spring-ai-agent-utils` | 0.4.2 | SkillsTool技能加载 |
| **AI-探索** | LangChain4j | 1.4.0 | **仅测试代码使用，生产环境零调用** |
| **工具协议** | Spring AI MCP (`spring-ai-mcp`) | via BOM | MCP客户端/服务端，工具定义 |
| **MCP传输** | `spring-ai-starter-mcp-client-webflux` | via BOM | SSE/Stdio MCP客户端 |
| **SSH** | JSch | 0.1.54 | SSH连接、SFTP、Shell通道 |
| **数据库** | MySQL + MyBatis | 8.0.28 / 3.0.4 | 会话/消息持久化 |
| **安全** | AES-256-GCM (自研) | - | SSH凭证加密 |
| **JWT** | jjwt + java-jwt | 0.9.1 / 4.4.0 | 认证(预留) |
| **设计框架** | `xfg-wrench-starter-design-framework` | 3.0.0 | 策略树路由模式 |
| **响应式** | RxJava 3 | transitive | ADK Flowable事件流 |
| **向量库** | **无** | - | 未引入任何向量数据库 |
| **缓存** | **无** | - | 无Redis/Caffeine等 |

---

## 二、层级结构图 + 每层职责明细

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      walissh-server-app (启动层)                        │
│  Application.java / AiAgentAutoConfig / application.yml / agent/*.yml  │
│  职责：Spring Boot启动、Bean注册、配置绑定、AI依赖聚合                    │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ 依赖 trigger + infrastructure
┌──────────────────────────────▼──────────────────────────────────────────┐
│                    walissh-server-trigger (接口触发层)                    │
│  AgentServiceController / SshConnectionController / SshTerminal...     │
│  职责：REST API入口、请求参数校验、DTO转换、SSE响应管理                    │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ 依赖 case + domain
┌──────────────────────────────▼──────────────────────────────────────────┐
│                    walissh-server-case (用例编排层)                       │
│  AIAgentReActServiceCase / RootNode / AiCallNode / ToolCallNode /       │
│  LoopDecisionNode / UserFeedbackNode / DefaultReActFactory.DynamicContext│
│  职责：ReAct循环编排、SSE事件流发射、步骤控制、终止决策                     │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ 依赖 domain + api
┌──────────────────────────────▼──────────────────────────────────────────┐
│                   walissh-server-domain (核心领域层)                      │
│                                                                         │
│  ┌─ agent子域 ─────────────────────────────────────────────────────┐    │
│  │  Agent装配流水线: RootNode→AiApiNode→ChatModelNode→AgentNode   │    │
│  │  →AgentWorkflowNode→RunnerNode (动态注册Spring Bean)            │    │
│  │                                                                  │    │
│  │  ChatService (ADK Runner调用)                                    │    │
│  │  IntentService (规则+LLM双层意图识别)                             │    │
│  │  PromptService / DynamicPromptBuilder / MilestoneTracker         │    │
│  │  ChatContextService (Provider-Reducer上下文管理)                  │    │
│  │  MCP客户端工厂: Local/SSE/Stdio三种MCP连接                       │    │
│  │  ADK工具: SshExecuteAdkTool (@Schema注解)                        │    │
│  │  Spring AI工具: SshExecuteMcpService (@Tool注解)                 │    │
│  │  Spring AI补丁: MySpringAI / MyMessageConverter /                │    │
│  │                 SpringAiToAdkToolConverter                       │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌─ ssh子域 ────────────────────────────────────────────────────────┐   │
│  │  SshConnectionService / SshFileService / SshTerminalService      │   │
│  │  端口接口: ISshSessionPort / ISshFilePort / ITerminalSessionPort │   │
│  │  仓储接口: IChatHistoryRepository / ISshConnectionRepository     │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ 依赖 types
┌──────────────────────────────▼──────────────────────────────────────────┐
│                walissh-server-infrastructure (基础设施层)                 │
│  SshSessionPort(JSch) / SshFilePort(SFTP) / TerminalSessionPort(Shell) │
│  ChatHistoryRepository(MyBatis) / SshConnectionRepository(AES加密)     │
│  DAO: IChatMessageDao / IChatSessionDao / ISshConnectionDAO 等         │
│  职责：SSH I/O实现、数据库持久化、加密、外部系统适配                       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                walissh-server-api (契约定义层)                            │
│  IAgentService / ISshConnectionService / ISshTerminalService 等         │
│  DTO: ChatRequestDTO / ReActEventDTO / TerminalExecRequestDTO 等       │
│  职责：服务接口契约、数据传输对象、不含任何实现                             │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                walissh-server-types (基础类型层)                          │
│  ResponseCode枚举 / AppException / Constants                            │
│  职责：全局枚举、异常定义、常量，零依赖                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 三、Spring AI / Google ADK / LangChain4j 框架分工对照表

| 维度 | Spring AI 1.1.5 | Google ADK 1.2.0 | LangChain4j 1.4.0 |
|------|-----------------|-------------------|---------------------|
| **架构角色** | **底层基座** — LLM抽象层 | **上层运行时** — Agent编排引擎 | **未使用** — 仅测试代码 |
| **所在模块** | domain层 (直接依赖) | domain层 + app层 | app层 (仅test/) |
| **核心职责** | 模型调用、工具定义、MCP协议 | Agent生命周期、Runner、Session、Event流 | 无生产职责 |
| **模型管理** | `OpenAiApi` → `OpenAiChatModel` 构建 | 通过 `SpringAI(chatModel)` 桥接使用 | - |
| **工具定义** | `@Tool` 注解 → `ToolCallback` / `ToolCallbackProvider` | `FunctionTool.create()` / `@Schema`注解 | - |
| **Agent编排** | 无（纯模型层） | `LlmAgent` / `SequentialAgent` / `ParallelAgent` / `LoopAgent` | - |
| **执行引擎** | 无 | `InMemoryRunner` + `Runner.runAsync()` → `Flowable<Event>` | - |
| **会话管理** | 无 | `Session` (ADK内置内存Session) | - |
| **插件系统** | 无 | `BasePlugin` / `LoggingPlugin` | - |
| **MCP支持** | `spring-ai-mcp` + `SyncMcpToolCallbackProvider` (客户端) | 无原生MCP，依赖Spring AI桥接 | - |
| **RAG功能** | **未使用**（无向量库依赖） | 无 | - |
| **生产代码调用** | 高频：模型构建、工具注册、MCP连接、意图分类 | 高频：Agent组装、Runner执行、事件处理 | **零** |

**关键发现：LangChain4j 是冗余依赖，生产代码无任何 import。**

---

## 四、三者真实调用关系与数据流

### 4.1 依赖关系图

```
                    ┌──────────────────────┐
                    │   Google ADK 1.2.0   │  ← Agent运行时（上层）
                    │  LlmAgent / Runner   │
                    │  Event / Session     │
                    └──────────┬───────────┘
                               │ 通过 google-adk-spring-ai 桥接
                               │ new SpringAI(chatModel)
                               ▼
                    ┌──────────────────────┐
                    │   Spring AI 1.1.5    │  ← LLM抽象（底层基座）
                    │  OpenAiChatModel     │
                    │  @Tool / MCP Client  │
                    └──────────┬───────────┘
                               │ HTTP调用
                               ▼
                    ┌──────────────────────┐
                    │  OpenAI兼容API       │  ← gpt-5.1 via apis.itedus.cn
                    └──────────────────────┘
```

### 4.2 完整调用链路流程图（ReAct流式路径）

```
客户端 POST /api/v1/chat_stream
        │
        ▼
┌─ AgentServiceController ──────────────────────────────────────────────────┐
│  注入 IChatService + IAIAgentReActServiceCase                             │
│  调用 reactServiceCase.chatStream(chatRequestDTO)                         │
└───────────────────────┬───────────────────────────────────────────────────┘
                        │
        ┌───────────────▼───────────────┐
        │  AIAgentReActServiceCase      │
        │  创建 ResponseBodyEmitter     │
        │  初始化 DynamicContext         │
        │  新线程运行 RootNode           │
        └───────────────┬───────────────┘
                        │
        ┌───────────────▼───────────────┐
        │  RootNode (react)             │  ← ReAct入口
        │  1. 提取sessionId/userId      │
        │  2. 绑定终端session到ThreadLocal│
        │  3. 从DB加载历史消息(最近50条)  │
        │  4. 初始化计数器               │
        │     maxSteps=50               │
        │     maxToolCalls=200          │
        └───────────────┬───────────────┘
                        │ 路由 → reactAiCallNode
        ┌───────────────▼───────────────┐
        │  AiCallNode (核心节点)         │
        │                               │
        │  1. intentService.classify()  │  ← 意图识别
        │     ├─ RuleIntentClassifier   │     (规则匹配, <1ms)
        │     └─ LLMIntentClassifier    │     (LLM兜底, 100-500ms)
        │                               │     直接调用 Spring AI ChatModel
        │  2. chatContextService         │  ← 上下文裁剪
        │     .trimHistory(8000 tokens) │     HybridReducer = Priority ∩ SlidingWindow
        │                               │
        │  3. promptService              │  ← 动态Prompt增强
        │     .buildEnrichedMessage()   │     注入: 环境信息/里程碑/最近命令/工具结果
        │                               │
        │  4. 从DefaultArmoryFactory     │  ← 获取组装好的Agent
        │     .getAiAgentRegisterVO()   │     返回 AiAgentRegisterVO
        │     .getRunner()              │     获取 ADK InMemoryRunner
        │                               │
        │  5. runner.runAsync(           │  ← ★ 核心调用 ★
        │     userId, sessionId,        │     Google ADK Runner
        │     Content, RunConfig)       │
        │     → Flowable<Event>         │
        │                               │
        │  6. 遍历 Event 流:            │
        │     ├─ text内容 → SSE text事件│
        │     ├─ stateDelta → 检测工具结果│  ← SpringAI自动执行工具后
        │     │   (functionCalls始终为空) │     结果在stateDelta中
        │     └─ 完成 → 保存消息到DB     │
        │                               │
        │  7. 路由决策:                  │
        │     ├─ 有stopReason → UserFeedbackNode
        │     ├─ 超maxSteps → UserFeedbackNode
        │     ├─ 有toolCalls → ToolCallNode
        │     └─ 否则 → LoopDecisionNode│
        └───────────────┬───────────────┘
                        │
        ┌───────────────▼───────────────┐
        │  ToolCallNode                 │
        │  处理ADK自动执行的工具结果      │
        │  或手动调用 SshExecuteAdkTool  │
        │  保存工具调用/结果到DB         │
        └───────────────┬───────────────┘
                        │ 路由 → reactAiCallNode (循环)
                        │
        ┌───────────────▼───────────────┐
        │  LoopDecisionNode             │
        │  检查终止条件:                 │
        │  ├─ maxSteps / maxToolCalls   │
        │  ├─ finish命令 / 错误         │
        │  └─ 无toolCalls → completed   │
        │  有toolCalls → 继续循环        │
        │  终止 → UserFeedbackNode      │
        └───────────────┬───────────────┘
                        │
        ┌───────────────▼───────────────┐
        │  UserFeedbackNode (终止节点)   │
        │  构建 ReActResultDTO          │
        │  发送 done SSE事件            │
        │  关闭 Emitter                 │
        │  清理 ThreadLocal             │
        └───────────────────────────────┘
```

### 4.3 Agent装配流水线（启动时执行）

```
application.yml + agent/*.yml
        │
        ▼
AiAgentAutoConfig (ApplicationReadyEvent监听)
        │
        ▼
ArmoryService.acceptArmoryAgents()
        │
        ▼ (策略树链式执行)
┌─ RootNode ───────────────────────────────────────────────────────────────┐
│  入口，路由到 AiApiNode                                                   │
└───────┬──────────────────────────────────────────────────────────────────┘
        ▼
┌─ AiApiNode ──────────────────────────────────────────────────────────────┐
│  读取 YAML 中 ai-api 配置                                                │
│  构建 OpenAiApi(baseUrl, apiKey, completionsPath, embeddingsPath)        │
│  存入 DynamicContext.openAiApi                                           │
└───────┬──────────────────────────────────────────────────────────────────┘
        ▼
┌─ ChatModelNode ──────────────────────────────────────────────────────────┐
│  构建 OpenAiChatModel(openAiApi, model="gpt-5.1")                       │
│  加载MCP工具: DefaultMcpClientFactory                                    │
│    ├─ LocalToolMcpCreateService → 从Spring容器获取ToolCallbackProvider    │
│    ├─ SSEToolMcpCreateService → McpSyncClient + HttpClientSseTransport   │
│    └─ StdioToolMcpCreateService → McpSyncClient + StdioTransport         │
│  加载Skills: DefaultToolSkillsCreateService → SkillsTool                 │
│  存入 DynamicContext.chatModel + toolCallbacks                           │
└───────┬──────────────────────────────────────────────────────────────────┘
        ▼
┌─ AgentNode ──────────────────────────────────────────────────────────────┐
│  遍历 YAML 中 agents[] 配置                                              │
│  对每个agent:                                                            │
│    LlmAgent.builder()                                                    │
│      .name(name)                                                         │
│      .model(new SpringAI(chatModel))     ← ★ Spring AI→ADK桥接 ★       │
│      .instruction(instruction)                                           │
│      .outputKey(outputKey)                                               │
│      .tools(FunctionTool.create(sshExecuteAdkTool, "executeCommand"))    │
│      .build()                                                            │
│  存入 DynamicContext.agentGroup                                           │
└───────┬──────────────────────────────────────────────────────────────────┘
        ▼
┌─ AgentWorkflowNode ──────────────────────────────────────────────────────┐
│  遍历 YAML 中 workflows[] 配置                                           │
│  按type分发:                                                             │
│    ├─ "Loop" → LoopAgentNode → ADK LoopAgent(subAgents, maxIterations)  │
│    ├─ "Parallel" → ParallelAgentNode → ADK ParallelAgent(subAgents)     │
│    └─ "Sequential" → SequentialAgentNode → ADK SequentialAgent(subAgents)│
└───────┬──────────────────────────────────────────────────────────────────┘
        ▼
┌─ RunnerNode ─────────────────────────────────────────────────────────────┐
│  InMemoryRunner(rootAgent, appName, plugins)                             │
│  创建 AiAgentRegisterVO(runner, chatModel, agentId, agentName)           │
│  registerBean(agentId, aiAgentRegisterVO)  ← ★ 动态注册为Spring Bean ★   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 五、功能职责归属明细表

| 功能模块 | 负责框架/组件 | 具体类 | 说明 |
|----------|--------------|--------|------|
| **RAG** | **未实现** | - | 无向量库依赖，无Embedding，无文档检索 |
| **Agent智能体** | **Google ADK** | `LlmAgent`, `SequentialAgent`, `ParallelAgent`, `LoopAgent` | ADK全权负责Agent定义与组合 |
| **Agent执行** | **Google ADK** | `InMemoryRunner`, `Runner.runAsync()` | ADK Runner是唯一执行引擎 |
| **LLM调用** | **Spring AI** | `OpenAiChatModel`, `OpenAiApi` | 通过`SpringAI(chatModel)`桥接给ADK |
| **工具定义(@Tool)** | **Spring AI** | `SshExecuteMcpService`, `MyTestMcpService` | `@Tool`注解 → `ToolCallbackProvider` |
| **工具定义(ADK原生)** | **Google ADK** | `SshExecuteAdkTool` | `@Schema`注解 → `FunctionTool.create()` |
| **工具桥接** | **自研补丁** | `SpringAiToAdkToolConverter` | 反射提取`MethodToolCallback`的method/object，转为ADK `FunctionTool` |
| **MCP协议** | **Spring AI** | `spring-ai-mcp`, `SyncMcpToolCallbackProvider` | MCP客户端连接（SSE/Stdio/Local） |
| **会话记忆** | **自研** | `ChatHistoryRepository` + MyBatis | 手动持久化到MySQL，无框架内置Memory |
| **模型管理** | **Spring AI** | `OpenAiApi` + `OpenAiChatModel` | 由`ChatModelNode`在启动时构建 |
| **意图识别** | **自研** | `IntentService` + `RuleIntentClassifier` + `LLMIntentClassifier` | 规则优先，LLM兜底，不依赖任何AI框架 |
| **上下文管理** | **自研** | `ChatContextService` + Provider/Reducer | Provider收集 → Reducer裁剪，纯自研 |
| **Prompt增强** | **自研** | `DynamicPromptBuilder` + `MilestoneTracker` | 环境注入、里程碑追踪、动态前缀 |
| **ReAct循环** | **自研(case层)** | `AIAgentReActServiceCase` + 5个Node | 策略树实现，非ADK内置ReAct |
| **SSE流式** | **自研** | `ResponseBodyEmitter` + `DynamicContext.emitter` | 自定义SSE事件格式 |
| **ADK模型适配** | **自研补丁** | `MySpringAI` (extends `BaseLlm`) | 替换ADK默认`SpringAI`，自定义消息转换 |
| **ADK消息转换** | **自研补丁** | `MyMessageConverter` (extends `MessageConverter`) | 处理inline media(images/audio) |
| **SSH连接** | **JSch** | `SshSessionPort` | 密码/密钥认证、心跳保活 |
| **终端会话** | **JSch** | `TerminalSessionPort` | ChannelShell、双缓冲、守护线程读取 |
| **文件管理** | **JSch** | `SshFilePort` | SFTP + sudo回退、二进制检测 |

---

## 六、统一抽象层、配置中心、路由分发、消息转换

| 抽象层 | 位置 | 机制 |
|--------|------|------|
| **Agent装配抽象** | `AbstractArmorySupport` (extends `AbstractMultiThreadStrategyRouter`) | 策略树模式，每个Node决定下一个Node |
| **ReAct节点抽象** | `AbstractAIAgentReActSupport` (extends `AbstractMultiThreadStrategyRouter`) | 同构策略树，SSE事件辅助方法 |
| **MCP客户端抽象** | `TooMcpCreateService` 接口 + `DefaultMcpClientFactory` | 按type(sse/local/stdio)路由到具体实现 |
| **Skills抽象** | `ToolSkillsCreateService` 接口 + `DefaultToolSkillsCreateService` | 按type(directory/resource)加载 |
| **SSH端口抽象** | `ISshSessionPort` / `ISshFilePort` / `ITerminalSessionPort` | 六角架构端口，infrastructure层实现 |
| **上下文Provider抽象** | `ContextProvider` 接口 + `@Order`排序 | TerminalState/Task/Milestone/ToolResult四个Provider |
| **消息Reducer抽象** | `MessageReducer` 接口 | SlidingWindow/Priority/Hybrid三种裁剪策略 |
| **配置中心** | `AiAgentAutoConfigProperties` (`@ConfigurationProperties`) | YAML → `Map<String, AiAgentConfigTableVO>` |
| **Agent注册表** | `DefaultArmoryFactory` + `AiAgentRegisterVO` (动态Bean) | `agentId → Runner + ChatModel` 的运行时注册表 |
| **消息实体转换** | `MyMessageConverter` | ADK `LlmRequest` → Spring AI `Prompt` |
| **工具转换** | `SpringAiToAdkToolConverter` | Spring AI `ToolCallback` → ADK `FunctionTool`（反射） |

---

## 七、架构设计优点

1. **DDD分层清晰**：types → api → infrastructure → domain → case → trigger → app，依赖方向严格单向，外层依赖内层
2. **六角架构实践**：SSH领域通过端口-适配器模式隔离，ISshSessionPort/SshFilePort由infrastructure层实现，domain层零JSch import
3. **Agent动态装配**：YAML配置驱动的Agent组装流水线，启动时自动构建并注册为Spring Bean，支持运行时切换
4. **双层意图识别**：规则引擎(<1ms)优先 + LLM兜底(100-500ms)，兼顾速度和准确率
5. **Provider-Reducer上下文管理**：模块化收集+策略化裁剪，可扩展性强
6. **双缓冲终端**：前端轮询缓冲与AI命令执行缓冲隔离，避免竞争条件

---

## 八、架构问题分析

### 问题1：框架职责错位 — ADK越位成为"基座"

```
当前实际关系:
  ADK (上层运行时) ──依赖──→ Spring AI (底层)

  但项目中:
  - ADK Runner 是唯一执行入口 (Runner.runAsync)
  - ReAct循环自研在case层，又调用ADK Runner
  - ADK的内置Agent循环被绕过，只用了Runner的单次调用

  结果: ADK的Agent编排能力(内置ReAct)被浪费，
        自研的ReAct循环反而成了实际编排层
```

**问题**：项目引入了ADK的`SequentialAgent`/`LoopAgent`等编排能力，但ReAct流式路径完全自研，ADK的内置循环机制被绕过。ADK Runner仅被当作"单次LLM调用+自动工具执行"的封装使用。

### 问题2：SSH工具双重实现

```
SshExecuteMcpService  → @Tool (Spring AI) → ToolCallbackProvider → MCP Local
SshExecuteAdkTool     → @Schema (ADK)     → FunctionTool.create → ADK Agent

两套工具做同一件事，通过 SpringAiToAdkToolConverter 桥接
```

**问题**：同一个SSH命令执行能力维护了两套实现（Spring AI版和ADK版），增加了维护成本和不一致风险。

### 问题3：LangChain4j冗余依赖

```
pom.xml 引入:
  langchain4j 1.4.0
  langchain4j-core 1.4.0
  langchain4j-open-ai 1.4.0
  google-adk-contrib-langchain4j 0.2.0

生产代码 import 数量: 0
```

**问题**：LangChain4j三个artifact + ADK桥接包共4个依赖，生产代码零使用，徒增构建时间和JAR体积。

### 问题4：ADK补丁侵入性

```
MySpringAI extends BaseLlm        ← 替换ADK默认模型适配器
MyMessageConverter extends MessageConverter ← 替换ADK默认消息转换器
SpringAiToAdkToolConverter        ← 反射提取私有字段(toolMethod/toolObject)
```

**问题**：通过继承+反射对ADK内部类进行patch，ADK版本升级时极易break。`SpringAiToAdkToolConverter`依赖反射访问`MethodToolCallback`的私有字段，属于脆弱耦合。

### 问题5：会话记忆原始

```
当前: ChatHistoryRepository → MySQL → 手动查询 → List<Map>
缺少: 无语义缓存、无摘要压缩、无重要性排序、无向量检索
```

**问题**：会话记忆是纯数据库轮询，无Redis缓存层，无消息摘要/压缩机制，高频对话场景下DB压力大。

---

## 九、与标准企业级架构对比 + 优化方案

### 标准架构参考

```
┌─────────────────────────────────────────────────┐
│  业务编排层 (Agent Orchestration)                 │
│  Spring AI Agent / 自研ReAct / LangChain4j Agent │
│  职责: 流程编排、工具调度、记忆管理                  │
└──────────────────────┬──────────────────────────┘
                       │ 调用
┌──────────────────────▼──────────────────────────┐
│  能力抽象层 (AI Abstraction)                      │
│  Spring AI (统一基座)                             │
│  ChatModel / EmbeddingModel / ToolCallback       │
│  MCP Client / VectorStore / ChatMemory           │
│  职责: 模型调用、工具协议、向量存储、记忆管理        │
└──────────────────────┬──────────────────────────┘
                       │ 适配
┌──────────────────────▼──────────────────────────┐
│  模型提供商 (LLM Provider)                       │
│  OpenAI / Anthropic / 本地模型                    │
└─────────────────────────────────────────────────┘
```

### 当前架构 vs 标准架构

| 维度 | 当前架构 | 标准企业级架构 |
|------|---------|--------------|
| **底层基座** | Spring AI (模型调用) + ADK (Runner执行) **双基座** | Spring AI **单一基座** |
| **Agent编排** | ADK Agent + 自研ReAct循环 **双路径并存** | 统一编排层（Spring AI Agent 或 自研） |
| **工具系统** | Spring AI @Tool + ADK @Schema **双实现** | Spring AI @Tool **统一定义** |
| **会话记忆** | 自研MySQL轮询 | Spring AI ChatMemory / Redis |
| **RAG** | 无 | Spring AI VectorStore + EmbeddingModel |
| **框架耦合** | 3个AI框架互相桥接 | 1-2个框架，职责清晰 |

### 优化调整方案

#### 方案A：以Spring AI为核心基座，精简ADK（推荐）

```
目标架构:
┌─ 业务编排层 ─────────────────────────────────────────┐
│  自研ReAct (case层) + Spring AI Agent (原生)          │
│  去掉ADK Runner依赖，ReAct直接调用ChatModel           │
└───────────────────────┬──────────────────────────────┘
                        │
┌─ 能力抽象层 ──────────▼──────────────────────────────┐
│  Spring AI (唯一基座)                                 │
│  OpenAiChatModel / @Tool / MCP Client                │
│  ChatMemory(新增) / VectorStore(新增RAG)              │
└───────────────────────┬──────────────────────────────┘
                        │
┌─ 工具层 ──────────────▼──────────────────────────────┐
│  SshExecuteMcpService (@Tool, 唯一实现)               │
│  删除 SshExecuteAdkTool                              │
│  删除 SpringAiToAdkToolConverter                     │
│  删除 MySpringAI / MyMessageConverter                │
└──────────────────────────────────────────────────────┘
```

**具体改动**:

| 改动项 | 操作 | 收益 |
|--------|------|------|
| 移除 `google-adk` 全套依赖 | 从pom.xml删除5个ADK artifact | 减少~15MB JAR体积，消除框架桥接复杂度 |
| 移除 `langchain4j` 全套依赖 | 从pom.xml删除4个LC4j artifact | 消除冗余 |
| AiCallNode直接调用ChatModel | `chatModel.call(prompt)` 替代 `runner.runAsync()` | 消除ADK Runner中间层 |
| 删除 `SshExecuteAdkTool` | 只保留 `SshExecuteMcpService`(@Tool) | SSH工具单一实现 |
| 删除3个ADK补丁类 | `MySpringAI` / `MyMessageConverter` / `SpringAiToAdkToolConverter` | 消除反射脆弱耦合 |
| 新增 `ChatMemory` | 引入Spring AI ChatMemory + Redis | 结构化会话记忆 |
| 新增 `VectorStore` | 引入Spring AI VectorStore + EmbeddingModel | RAG能力 |

#### 方案B：以ADK为核心运行时，Spring AI仅做模型接入

```
目标架构:
┌─ Agent运行时 ────────────────────────────────────────┐
│  Google ADK (唯一运行时)                              │
│  LlmAgent内置ReAct + Runner + Session + Memory       │
│  不再自研ReAct循环，使用ADK原生能力                    │
└───────────────────────┬──────────────────────────────┘
                        │
┌─ 模型接入层 ──────────▼──────────────────────────────┐
│  Spring AI (仅做模型接入)                             │
│  OpenAiChatModel → SpringAI(chatModel) 桥接          │
│  去掉@Tool定义，统一用ADK FunctionTool               │
└──────────────────────────────────────────────────────┘
```

**方案A更适合本项目**，原因：

1. **ReAct循环已自研成熟**：case层5个Node的策略树实现完善，SSE事件格式已定义，前端已对接
2. **Spring AI生态更贴合Spring Boot**：`@Tool`注解、MCP Client、`ToolCallbackProvider`等与Spring天然集成
3. **ADK的Agent编排能力未被充分利用**：当前仅用Runner单次调用，SequentialAgent/LoopAgent在YAML中配置但ReAct流式路径不经过它们
4. **降低维护风险**：消除3个ADK补丁类的反射脆弱耦合
