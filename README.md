<div align="center">

# WaLiSSH

**AI-Powered SSH Terminal & Server Operations Platform**

An intelligent SSH operations platform that combines a desktop terminal with AI Agent capabilities. Manage remote servers through natural language conversations — diagnose issues, install software, troubleshoot failures, and execute commands autonomously.

[![Java](https://img.shields.io/badge/Java-17-ED8B00?logo=openjdk&logoColor=white)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.3-6DB33F?logo=spring-boot&logoColor=white)](https://spring.io/projects/spring-boot)
[![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)](https://react.dev/)
[![Tauri](https://img.shields.io/badge/Tauri-2-FFC131?logo=tauri&logoColor=black)](https://tauri.app/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

[English](#features) | [中文](#特性)

</div>

---

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Project Structure](#project-structure)
- [API Reference](#api-reference)
- [Contributing](#contributing)
- [License](#license)

---

## Features

### Core Capabilities

| Feature | Description |
|---------|-------------|
| **AI-Powered Operations** | ReAct-based Agent that autonomously plans, executes, observes, and iterates to complete complex ops tasks |
| **SSH Remote Terminal** | Full-featured terminal powered by xterm.js with interactive shell, tab completion, and color output |
| **Remote File Manager** | SFTP file browsing, Monaco Editor inline editing, upload/download with sudo privilege fallback |
| **VS Code-style UI** | Integrated IDE layout with terminal, file explorer, and AI chat panel in a single desktop app |
| **Security First** | SSH connections and AI operations centralized on the server side; dangerous command interception built-in |
| **MCP Protocol** | Extensible tool integration via Local, SSE, and Stdio MCP (Model Context Protocol) transports |

### AI Agent System

- **ReAct Loop** — Reasoning + Acting cycle with configurable max steps (50) and tool calls (200)
- **Dangerous Command Detection** — Automatically blocks `rm -rf /`, `dd`, `mkfs`, and other destructive commands
- **Dual-layer Intent Recognition** — Rule-based engine (<1ms) as first pass, LLM fallback (100-500ms) for complex queries
- **Provider-Reducer Context Management** — Modular context collection with priority-based and sliding-window message trimming
- **Dynamic Prompt Enhancement** — Injects server environment info, milestones, recent commands, and tool results into prompts

### SSH & Terminal

- Password and private key authentication (AES-256-GCM encrypted credential storage)
- Heartbeat keep-alive with automatic connection health detection
- Dual-buffer isolation for AI command execution vs. frontend polling
- Large file chunked reading with automatic sudo privilege escalation

---

## Architecture

```
┌───────────────────────────────────────────────────┐
│           Tauri Desktop Client                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │ Terminal  │  │  File    │  │   AI Chat Panel  │ │
│  │ (xterm)  │  │ Explorer │  │ (ReAct Streaming)│ │
│  └──────────┘  └──────────┘  └──────────────────┘ │
└──────────────────────┬────────────────────────────┘
                       │ REST API / SSE
┌──────────────────────▼────────────────────────────┐
│           Spring Boot Server (DDD)                 │
│  ┌─────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Trigger │→ │   Case   │→ │     Domain       │  │
│  │ (REST)  │  │ (ReAct)  │  │ (Agent / SSH)    │  │
│  └─────────┘  └──────────┘  └────────┬─────────┘  │
│                                       │            │
│  ┌────────────────────────────────────▼──────────┐ │
│  │         Infrastructure (JSch / MyBatis)       │ │
│  └───────────────────────────────────────────────┘ │
└──────────────────────┬────────────────────────────┘
                       │ OpenAI-compatible API
                 ┌─────▼─────┐
                 │   LLM     │
                 └───────────┘
```

> For a detailed architecture analysis including framework responsibilities, call chains, and optimization recommendations, see [docs/architecture-analysis.md](docs/architecture-analysis.md).

<!-- Screenshot placeholder -->
<!-- ![WaLiSSH Screenshot](docs/images/screenshot.png) -->

---

## Tech Stack

### Client

| Layer | Technology |
|-------|------------|
| Desktop | [Tauri 2](https://tauri.app/) (Rust + Web) |
| Language | TypeScript 5.8 |
| UI Framework | [React 19](https://react.dev/) |
| Build Tool | [Vite 7](https://vite.dev/) |
| Styling | [Tailwind CSS 4](https://tailwindcss.com/) |
| State | [Zustand 5](https://zustand-demo.pmnd.rs/) |
| Terminal | [xterm.js 6](https://xtermjs.org/) (WebGL renderer) |
| Editor | [Monaco Editor](https://microsoft.github.io/monaco-editor/) |

### Server

| Layer | Technology |
|-------|------------|
| Language | Java 17 |
| Framework | [Spring Boot 3.4.3](https://spring.io/projects/spring-boot) |
| Architecture | DDD (Domain-Driven Design), 7 Maven modules |
| AI Agent | [Google ADK 1.2.0](https://google.github.io/adk-docs/) + [Spring AI 1.1.5](https://docs.spring.io/spring-ai/reference/) |
| Tool Protocol | [MCP](https://modelcontextprotocol.io/) (Model Context Protocol) |
| SSH | [JSch](http://www.jcraft.com/jsch/) |
| Database | MySQL 8.0 + [MyBatis](https://mybatis.org/mybatis-3/) |
| Design Pattern | Strategy Tree Router (xfg-wrench) |

---

## Prerequisites

| Component | Version | Notes |
|-----------|---------|-------|
| JDK | 17+ | [Download](https://adoptium.net/) |
| Maven | 3.8+ | [Download](https://maven.apache.org/download.cgi) |
| MySQL | 8.0+ | [Download](https://dev.mysql.com/downloads/) |
| Node.js | 22+ | [Download](https://nodejs.org/) |
| Rust | stable | [Install](https://rustup.rs/) |
| Tauri CLI | 2.x | `cargo install tauri-cli` |

---

## Quick Start

### 1. Database Setup

```bash
# Create database
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS walissh DEFAULT CHARSET utf8mb4;"
```

### 2. Server Configuration

Edit the database connection in `walissh-server/walissh-server-app/src/main/resources/application-dev.yml`:

```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/walissh?useUnicode=true&characterEncoding=utf8&autoReconnect=true&serverTimezone=UTC
    username: root
    password: your_password
```

### 3. Build & Start Server

```bash
cd walissh-server
mvn clean install -DskipTests
java -jar walissh-server-app/target/walissh-server-app.jar
```

Server starts at `http://localhost:8091`.

### 4. Start Client

**Windows:**

```cmd
cd walissh-client
docs\dev-ops\start-dev.bat
```

**macOS / Linux:**

```bash
cd walissh-client
chmod +x docs/dev-ops/start-dev.sh
./docs/dev-ops/start-dev.sh
```

### 5. Verify

```bash
# Check server health
curl http://localhost:8091/api/v1/query_ai_agent_config_list

# Expected response:
# {"code":"0000","info":"success","data":[{"agentId":"100000","agentName":"SSH AI Agent",...}]}
```

---

## Configuration

### LLM Provider

Configure your AI model in `walissh-server/walissh-server-app/src/main/resources/agent/ssh-agent.yml`:

```yaml
module:
  aiApi:
    base-url: https://your-api-endpoint    # OpenAI-compatible API URL
    api-key: sk-your-api-key               # API key
    completions-path: v1/chat/completions
    embeddings-path: v1/embeddings
  chatModel:
    model: gpt-4o                          # Model name
```

### Agent Behavior

The same YAML file configures the AI agent's system prompt, tools, and workflow:

```yaml
module:
  agents:
    - name: sshOperator
      description: "SSH command execution agent"
      instruction: |
        You are a senior Linux SRE with 10+ years of experience...
      output-key: ssh_result
  runner:
    agent-name: sshOperator
```

### Server Port

Change the server port in `application-dev.yml`:

```yaml
server:
  port: 8091
```

### Encryption Key

For production, set the SSH credential encryption key via environment variable:

```bash
export WALISSH_SECRET_KEY=your-256-bit-key
```

---

## Project Structure

```
smart-ssh-shell/
├── walissh-client/                    # Tauri Desktop Client
│   ├── src/
│   │   ├── api/                       # REST API layer (agent, ssh, terminal, file)
│   │   ├── components/                # Reusable UI components
│   │   ├── views/                     # Page-level views (MainView = primary layout)
│   │   ├── stores/                    # Zustand state (connection, agent, file, theme)
│   │   └── types/                     # TypeScript type definitions
│   └── src-tauri/                     # Rust/Tauri native layer
│       ├── src/                       # Rust entry point (main.rs, lib.rs)
│       ├── Cargo.toml                 # Rust dependencies
│       └── tauri.conf.json            # Tauri window & build config
│
├── walissh-server/                    # Spring Boot Backend
│   ├── walissh-server-api/            # Service interfaces & DTOs
│   ├── walissh-server-trigger/        # REST Controllers (HTTP entry points)
│   ├── walissh-server-case/           # ReAct use-case orchestration (5-node strategy tree)
│   ├── walissh-server-domain/         # Core domain (Agent assembly, SSH services, Intent, Prompt)
│   ├── walissh-server-infrastructure/ # Infrastructure (JSch SSH I/O, MyBatis DAO, AES encryption)
│   ├── walissh-server-types/          # Shared enums, exceptions, constants
│   └── walissh-server-app/            # Application entry, YAML configs, MCP tool registration
│
└── docs/
    └── architecture-analysis.md       # Deep architecture analysis (CN)
```

### Server Module Dependency Graph

```
types ← api ← infrastructure ← domain ← case ← trigger ← app
```

---

## API Reference

All endpoints are prefixed with `/api/v1`.

### Agent

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/query_ai_agent_config_list` | List available AI agents |
| `POST` | `/create_session` | Create a new chat session |
| `POST` | `/chat` | Send message (non-streaming) |
| `POST` | `/chat_stream` | Send message (SSE streaming with ReAct events) |

### SSH Connection

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/ssh/list` | List SSH connections |
| `POST` | `/ssh/create` | Create SSH connection |
| `POST` | `/ssh/connect` | Connect to remote server |
| `POST` | `/ssh/disconnect` | Disconnect |

### SSH Terminal

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/ssh/terminal/open` | Open interactive terminal |
| `POST` | `/ssh/terminal/exec` | Execute single command |
| `POST` | `/ssh/terminal/write` | Write to terminal stdin |
| `GET` | `/ssh/terminal/read` | Read terminal output |
| `POST` | `/ssh/terminal/resize` | Resize terminal window |
| `POST` | `/ssh/terminal/close` | Close terminal |

### SSH File

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/ssh/file/tree` | List remote directory tree |
| `GET` | `/ssh/file/content` | Read file content |
| `POST` | `/ssh/file/create-file` | Create file |
| `POST` | `/ssh/file/save-content` | Save file content |
| `POST` | `/ssh/file/upload` | Upload file |
| `GET` | `/ssh/file/download` | Download file |

### SSE Event Types (`/chat_stream`)

| Event | Description |
|-------|-------------|
| `text` | AI-generated text content |
| `tool_call` | Tool invocation with command details |
| `tool_result` | Tool execution result |
| `round_end` | End of one ReAct iteration |
| `done` | Final result with summary |
| `error` | Error information |

---

## Contributing

Contributions are welcome! Please follow these steps:

1. **Fork** this repository
2. **Create** a feature branch: `git checkout -b feature/your-feature`
3. **Commit** your changes: `git commit -m "feat: add your feature"`
4. **Push** to the branch: `git push origin feature/your-feature`
5. **Open** a Pull Request

### Development Guidelines

- **Server**: Follow DDD layered architecture; new features should be placed in the correct module
- **Client**: Use Zustand for state management; follow existing component patterns
- **Commits**: Use [Conventional Commits](https://www.conventionalcommits.org/) format (`feat:`, `fix:`, `docs:`, etc.)
- **Code Style**: Server uses Lombok; Client uses TypeScript strict mode

### Reporting Issues

If you find a bug or have a feature request, please [open an issue](https://github.com/sgahch/Smart-shell-agent/issues) with:

- Steps to reproduce (for bugs)
- Expected vs. actual behavior
- Environment details (OS, JDK version, Node version, etc.)

---

## License

This project is licensed under the [MIT License](LICENSE).

---

<div align="center">

**Built with passion for better server operations.**

</div>
