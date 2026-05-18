# WaLiSSH - AI 驱动的 SSH 智能运维平台

WaLiSSH 是一个 AI 驱动的 SSH 终端与服务器运维平台。通过自然语言对话，让 AI 自动执行远程命令、诊断问题、安装软件、排查故障，像一位资深 SRE 工程师一样运维你的服务器。

## 核心特性

- **AI 智能运维** — 基于 ReAct 循环的 AI Agent，自动规划→执行→观察→迭代，完成复杂运维任务
- **SSH 远程终端** — 基于 xterm.js 的全功能终端，支持命令执行、交互式 Shell、窗口自适应
- **远程文件管理** — SFTP 文件浏览、Monaco 编辑器在线编辑、上传下载、sudo 权限回退
- **VS Code 风格 UI** — 终端 + 文件管理器 + AI 对话面板三合一的桌面 IDE 界面
- **安全优先** — SSH 连接和 AI 操作统一在服务端，集中风险管控，危险命令自动拦截
- **MCP 协议支持** — 支持 Local / SSE / Stdio 三种 MCP 工具接入方式

## 项目结构

```
smart-ssh-shell/
├── walissh-client/           # Tauri 桌面客户端
│   ├── src/                  # React + TypeScript 前端
│   │   ├── api/              # REST API 调用层
│   │   ├── components/       # UI 组件
│   │   ├── views/            # 页面视图 (MainView 为主布局)
│   │   ├── stores/           # Zustand 状态管理
│   │   └── types/            # TypeScript 类型定义
│   └── src-tauri/            # Rust/Tauri 原生层
│
├── walissh-server/           # Spring Boot 后端
│   ├── walissh-server-api/        # 接口契约与 DTO
│   ├── walissh-server-trigger/    # REST Controller
│   ├── walissh-server-case/       # ReAct 用例编排层
│   ├── walissh-server-domain/     # 核心领域层 (Agent/SSH)
│   ├── walissh-server-infrastructure/  # 基础设施 (SSH I/O/DB)
│   ├── walissh-server-types/      # 公共类型与异常
│   └── walissh-server-app/        # 启动入口与配置
│
└── docs/
    └── architecture-analysis.md  # 深度架构分析文档
```

## 技术栈

### 客户端

| 层面 | 技术 |
|------|------|
| 桌面框架 | Tauri 2 (Rust + Web) |
| 前端 | React 19 + TypeScript 5.8 |
| 构建 | Vite 7 |
| 样式 | Tailwind CSS 4 |
| 状态管理 | Zustand 5 |
| 终端 | xterm.js 6 (WebGL) |
| 编辑器 | Monaco Editor |

### 服务端

| 层面 | 技术 |
|------|------|
| 语言 | Java 17 |
| 框架 | Spring Boot 3.4.3 |
| 架构 | DDD 领域驱动设计 (7 模块) |
| AI Agent | Google ADK 1.2.0 + Spring AI 1.1.5 |
| 工具协议 | MCP (Model Context Protocol) |
| SSH | JSch |
| 数据库 | MySQL + MyBatis |
| 设计模式 | 策略树路由 (xfg-wrench) |

## 架构概览

```
┌─────────────────────────────────────────┐
│  Tauri 桌面客户端 (React + xterm.js)    │
│  终端 | 文件管理 | AI 对话面板           │
└──────────────────┬──────────────────────┘
                   │ REST API / SSE
┌──────────────────▼──────────────────────┐
│  Spring Boot 后端 (DDD 7 模块)          │
│  ┌────────┐  ┌──────────┐  ┌─────────┐ │
│  │Trigger │→ │  Case    │→ │ Domain  │ │
│  │(REST)  │  │(ReAct)   │  │(Agent/  │ │
│  │        │  │          │  │ SSH)    │ │
│  └────────┘  └──────────┘  └────┬────┘ │
│                                 │       │
│  ┌──────────────────────────────▼─────┐ │
│  │ Infrastructure (SSH I/O / MySQL)   │ │
│  └────────────────────────────────────┘ │
└──────────────────┬──────────────────────┘
                   │ OpenAI API
             ┌─────▼─────┐
             │  LLM 模型  │
             └───────────┘
```

## 快速开始

### 环境要求

| 组件 | 版本 |
|------|------|
| JDK | 17+ |
| Maven | 3.8+ |
| MySQL | 8.0+ |
| Node.js | 22+ |
| Rust | latest stable |
| Tauri CLI | 2.x |

### 1. 启动服务端

```bash
# 创建数据库
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS walissh DEFAULT CHARSET utf8mb4;"

# 修改数据库连接配置
# walissh-server/walissh-server-app/src/main/resources/application-dev.yml

# 构建并启动
cd walissh-server
mvn clean install -DskipTests
java -jar walissh-server-app/target/walissh-server-app.jar
```

服务端默认运行在 `http://localhost:8091`。

### 2. 启动客户端

```bash
# Windows
cd walissh-client
docs\dev-ops\start-dev.bat

# macOS / Linux
cd walissh-client
chmod +x docs/dev-ops/start-dev.sh
./docs/dev-ops/start-dev.sh
```

### 3. 配置 LLM

在 `walissh-server/walissh-server-app/src/main/resources/agent/ssh-agent.yml` 中配置 AI API：

```yaml
module:
  aiApi:
    base-url: https://your-api-endpoint
    api-key: your-api-key
  chatModel:
    model: gpt-4o
```

## 功能说明

### SSH 连接管理
- 支持密码和密钥两种认证方式
- 心跳保活机制，自动检测连接状态
- SSH 凭证 AES-256-GCM 加密存储

### AI Agent
- 基于 ReAct 循环的智能体，自动推理→执行→观察→迭代
- 内置危险命令拦截（`rm -rf /`、`dd`、`mkfs` 等）
- 双层意图识别：规则引擎（<1ms）优先 + LLM 兜底
- Provider-Reducer 上下文管理，自动裁剪对话历史

### 终端功能
- 交互式 Shell，支持 Tab 补全、颜色输出
- AI 命令执行双缓冲隔离，避免前端/AI 输出竞争
- 大文件分块读取，sudo 权限自动回退

## 文档

- [深度架构分析](docs/architecture-analysis.md) — 框架分工、调用链路、优化建议

## License

MIT
