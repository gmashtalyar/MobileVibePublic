# MobileVibe

**An iOS coding environment with an autonomous AI agent system**

MobileVibe brings the power of desktop development tools to your pocket. It's a hybrid mobile IDE that combines a terminal-style CLI with modern SwiftUI interfaces, enabling developers to code, review, and ship from anywhere.

**Contents:** [What It Does](#what-it-does) • [Technical Architecture](#technical-architecture) • [Codebase Metrics](#codebase-metrics) • [Key Technical Decisions](#key-technical-decisions) • [Technology Stack](#technology-stack) • [Project Structure](#project-structure)

---

## What It Does

MobileVibe solves a real problem: developers are mobile, but their tools aren't. Whether you're reviewing a PR on your commute, fixing a critical bug from a coffee shop, or prototyping an idea while traveling, MobileVibe keeps you productive.

### Core Capabilities

- **GitHub Integration** — Browse repositories, explore file trees, switch branches, and commit changes directly from your phone
- **Multi-Provider AI Assistant** — Chat with OpenAI, Anthropic, Google, or DeepSeek models with seamless provider switching
- **Autonomous Agent Mode** — An AI that doesn't just answer questions but actually reads your code, writes changes, and orchestrates multi-step tasks
- **Smart Diff Viewer** — Review AI-generated changes with syntax-highlighted diffs before applying them to your repo
- **Dual Interface** — Toggle between a power-user terminal and a polished SwiftUI experience while maintaining shared state

---

## Technical Architecture

### The Agent System

The heart of MobileVibe is a **stateful, self-correcting AI agent** built on [LangGraph-Swift](https://github.com/bsorrentino/LangGraph-Swift). This isn't a simple chat wrapper—it's a multi-node workflow engine with planning, execution, validation, and feedback loops.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AGENT WORKFLOW GRAPH                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│    ┌─────────┐                                                              │
│    │  START  │                                                              │
│    └────┬────┘                                                              │
│         │                                                                   │
│         ▼                                                                   │
│    ┌─────────┐                                                              │
│    │ ROUTER  │ ─────────────────────────────────────────────────────┐       │
│    └────┬────┘                                                      │       │
│         │                                                           │       │
│    ┌────┴────────────────────────┬──────────────────────────────────┤       │
│    │                             │                                  │       │
│    ▼                             ▼                                  ▼       │
│ ┌──────────┐              ┌──────────────┐                ┌─────────────┐   │
│ │ CHATTING │              │    CODING    │                │  PLANNING   │   │
│ │   LLM    │              │     LLM      │◄───────┐       │     LLM     │   │
│ └────┬─────┘              └──────┬───────┘        │       └──────┬──────┘   │
│      │                           │                │              │          │
│      │                    ┌──────┴──────┐         │       ┌──────┴──────┐   │
│      │                    ▼             │         │       ▼             │   │
│      │               ┌─────────┐        │         │  ┌─────────────┐    │   │
│      │               │  TOOL   │        │         │  │ PLAN REVIEW │    │   │
│      │               │EXECUTOR │────────┘         │  │ (User Y/N)  │    │   │
│      │               └────┬────┘                  │  └──────┬──────┘    │   │
│      │                    │                       │         │           │   │
│      │                    ▼                       │         ▼           │   │
│      │               ┌─────────┐                  │  ┌──────────────┐   │   │
│      │               │EVALUATOR│──────────────────┤  │ ORCHESTRATOR │   │   │
│      │               └────┬────┘  Needs Revision  │  └──────┬───────┘   │   │
│      │                    │                       │         │           │   │
│      │                    │ Approved              │    ┌────┴────┐      │   │
│      │                    ▼                       │    ▼         │      │   │
│      │               ┌─────────┐                  │ ┌──────┐     │      │   │
│      │               │  FINAL  │                  │ │WORKER│─────┘      │   │
│      │               │RESPONDER│                  │ │ LLM  │            │   │
│      │               └────┬────┘                  │ └──┬───┘            │   │
│      │                    │                       │    │                │   │
│      │                    │                       │    └────► EVALUATOR ┘   │
│      ▼                    ▼                       │                         │
│    ┌─────────────────────────────────────────┐    │                         │
│    │                   END                   │◄───┘                         │
│    └─────────────────────────────────────────┘                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**12 Specialized Nodes** handle distinct responsibilities:

| Node | Purpose |
|------|---------|
| **Router** | Classifies requests into chatting, coding, or planning paths |
| **Chatting LLM** | Handles conversational Q&A without tools |
| **Coding LLM** | Executes single-task coding with autonomous tool usage |
| **Planning LLM** | Breaks complex requests into multi-step plans with codebase exploration |
| **Plan Review** | Requires user confirmation before executing plans |
| **Plan Validator** | Validates plan quality with revision feedback loop |
| **Orchestrator** | Coordinates multi-step execution with retry logic |
| **Worker LLM** | Executes individual steps under orchestrator direction |
| **Tool Executor** | Dispatches `readFile`, `writeFile`, `listFiles`, `askUser` |
| **Evaluator** | Quality gate with approve/revise feedback loop (max 3 turns) |
| **Final Responder** | Generates user-friendly completion summaries |

### Provider-Native Structured Output

A key technical innovation: **all 8 LLM nodes use provider-enforced JSON schemas** rather than prompt-based formatting. This eliminates parsing failures and ensures type-safe responses.

```swift
// Each provider has a dedicated implementation
OpenAI:    response_format with strict JSON Schema
Anthropic: Tool-based enforcement with forced tool_choice
Google:    Native responseSchema in generationConfig
DeepSeek:  JSON mode + schema guidance in system message

// Unified interface abstracts provider differences
func sendMessageWithSchema(messages:nodeType:) async throws -> AgentResponse
```

**8 Node-Specific Schemas** define the exact response structure for each node:

```
RouterSchema       → { action: "complete", response: "chatting"|"coding"|"planning" }
CodingSchema       → { action: "tool"|"complete", tool?: {...}, response?: "..." }
WorkerSchema       → { action: "tool"|"status", status?: { type, message } }
EvaluatorSchema    → { action: "evaluate", evaluation: { decision, feedback } }
...
```

### Architecture Patterns

**Feature-Based Organization** — Code is organized by feature, not layer:

```
Features/
├── Home/           # Dashboard: HomeView + HomeViewModel + HomeModels
├── AIAssistant/    # Chat interface: AIAssistantView + ViewModel
├── FileExplorer/   # Tree browser: FileExplorerView + ViewModel
├── DiffBrowser/    # Change review: DiffBrowserView + ViewModel
└── Terminal/       # CLI interface: TerminalView + ViewModel
```

**Protocol-Based Dependency Injection** — Services are injected via protocols through a central `ServiceContainer`:

```swift
protocol AIServiceProtocol {
    func sendMessage(messages:provider:model:) async throws -> String
    func sendMessageWithSchema(messages:nodeType:) async throws -> AgentResponse
}

// Production
ServiceContainer.shared.register(AIService() as AIServiceProtocol)

// Testing
ServiceContainer.shared.register(MockAIService() as AIServiceProtocol)
```

**@MainActor Thread Safety** — All services use `@MainActor` for thread-safe shared mutable state without manual synchronization.

**Unified Error Handling** — A single `AppError` enum with built-in retry logic:

```swift
enum AppError: LocalizedError {
    case aiProvider(AIProviderError)
    case github(GitHubError)
    case agent(AgentError)
    case fileContext(FileContextError)
    ...

    var shouldRetry: Bool { ... }      // Is this transient?
    var retryDelay: TimeInterval { ... } // How long to wait?
}
```

---

## Codebase Metrics

| Metric | Value |
|--------|-------|
| **Lines of Swift** | ~21,000 |
| **Agent Nodes** | 12 |
| **JSON Schemas** | 8 |
| **AI Providers** | 4 (OpenAI, Anthropic, Google, DeepSeek) |
| **Service Refactoring** | AgentService: 2,367 → 258 lines (89% reduction) |

---

## Key Technical Decisions


### Why Structured Output?

Prompt-based JSON formatting fails. Models hallucinate structure, miss fields, and produce unparseable responses. Provider-native schemas:
- **Guarantee compliance** — OpenAI strict mode, Anthropic tool forcing
- **Enable type safety** — Responses decode directly to Swift structs
- **Simplify prompts** — No JSON instructions cluttering task logic

### Why Dual Interface?

Power users want a terminal. Casual users want swipe and tap. Instead of choosing, MobileVibe offers both—sharing the same services, state, and conversation history. Toggle with one tap.

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| **UI Framework** | SwiftUI with MVVM |
| **Agent Framework** | LangGraph-Swift |
| **Persistence** | Core Data |
| **Security** | Keychain Services |
| **Networking** | async/await with URLSession |
| **Version Control** | GitHub REST API |

---


## Project Structure

```
MobileVibe/
├── App/                    # Entry point
├── Core/
│   ├── Navigation/         # AppState, NavigationRouter
│   ├── DI/                 # ServiceContainer
│   ├── Errors/             # AppError unified handling
│   ├── Security/           # KeychainService
│   └── Persistence/        # Core Data stack
├── Features/               # Feature modules (View + ViewModel)
├── Components/             # Reusable UI components
└── Services/
    ├── AI/                 # AIService, FileEditService
    ├── Agent/
    │   ├── Core/           # AgentService, AgentState, GraphBuilder
    │   ├── Nodes/          # 12 node implementations
    │   ├── Tools/          # ReadFile, WriteFile, ListFiles, AskUser
    │   ├── StructuredOutput/ # Schema definitions & validation
    │   └── Providers/      # OpenAI, Anthropic, Google, DeepSeek
    ├── GitHub/             # GitHubService
    ├── Conversation/       # Message management
    └── FileContext/        # Smart file detection
```

---
