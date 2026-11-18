# The Competitive Landscape Around Pi

Pi operates at the intersection of several rapidly evolving spaces. This document maps the landscape layer by layer -- starting with what sits beneath Pi and working up to the products that compete with it.

---

## Layer 1: Talking to LLMs -- The Unified API Space

Pi's `pi-ai` package competes with a crowded field of multi-provider LLM abstraction libraries.

### The Python Heavyweight: LiteLLM

LiteLLM is the dominant player in Python. It covers 100+ providers (vs. pi-ai's ~22) and runs either as an in-process SDK or as a standalone proxy server. The proxy adds enterprise features -- rate limiting, load balancing, spend dashboards -- at the cost of a network hop. LiteLLM is what most Python teams reach for first.

Pi-ai's advantage is being **TypeScript-native** with full type safety, using native provider SDKs directly (no translation to OpenAI format), and embedding per-model pricing data for built-in cost calculation. LiteLLM also tracks costs, but pi-ai does it without an extra service.

### The TypeScript Contender: Vercel AI SDK

The Vercel AI SDK is pi-ai's most direct competitor in TypeScript. It uses a provider plugin architecture (`@ai-sdk/openai`, `@ai-sdk/anthropic`, etc.) rather than bundling everything. AI SDK 6 (February 2026) added a `ToolLoopAgent` class for multi-step tool execution and `needsApproval` for human-in-the-loop gating.

Key differences: Vercel's SDK is heavily oriented toward UI integration (React hooks like `useChat`, streaming protocols for Next.js). Pi-ai is lower-level -- it provides a raw `AssistantMessageEventStream` with 12 distinct event types (`text_start`, `thinking_delta`, `toolcall_end`, etc.) designed for driving agent UIs. Vercel tracks token counts but doesn't calculate costs. Pi-ai has code-generated model registries with embedded pricing.

### The Frameworks: LangChain and LlamaIndex

Both provide LLM abstraction as part of much larger frameworks (chains, RAG, memory, output parsers). LangChain covers 150+ integrations. The tradeoff is weight -- these are full frameworks, not focused libraries. Pi-ai deliberately stays thin: unified streaming with tool calling, nothing more.

### Gateways and Aggregators

**OpenRouter** (500+ models, hosted API gateway), **Portkey** (1,600+ LLMs, open-source TypeScript gateway with enterprise observability), and **LLM Gateway** aggregate providers behind OpenAI-compatible endpoints. Pi-ai already supports OpenRouter and Vercel AI Gateway as providers. These services are complementary -- you can route pi-ai through them or call providers directly.

### Small TypeScript Libraries

**multi-llm-ts** and **LLM.js** are focused TypeScript libraries closest to pi-ai's philosophy. multi-llm-ts provides a `LlmModel` wrapper for unified chat/streaming/vision/function-calling. LLM.js includes automatic cost tracking using LiteLLM's model database. Both are smaller in scope and provider coverage than pi-ai.

### Structured Output: Instructor

Instructor (Python/TypeScript) solves a different problem -- extracting validated typed data from LLMs using Pydantic/Zod schemas. It uses tool calling internally as its mechanism. Complementary to pi-ai rather than competitive.

### Pi-ai's Distinctive Position

Pi-ai's combination of features is unique in the TypeScript space:
- **Code-generated model registry** with 757 models, embedded pricing, and TypeScript autocompletion on provider + model ID
- **API-centric architecture** where one provider implementation (`openai-completions`) serves OpenAI, Groq, Cerebras, xAI, and any compatible endpoint via auto-detected compatibility settings
- **Fine-grained streaming events** (12 types with progressive `partial` snapshots) -- more granular than any competitor
- **Built-in cost calculation** across 4 tiers (input, output, cache read, cache write)
- **Native SDK calls** to Anthropic, OpenAI, Google, Bedrock, Mistral -- no lossy translation to OpenAI format
- **Cross-provider handoffs** with automatic thinking block conversion and tool ID normalization

---

## Layer 2: Making Agents -- The Orchestration Space

Pi's `pi-agent-core` competes with agent frameworks that range from minimal loops to full graph engines.

### Graph-Based: LangGraph

LangGraph (by LangChain) models workflows as directed graphs with cycles. You define nodes for "call LLM," "execute tools," and "check condition," connected by edges that loop back. This gives you branching logic, human-in-the-loop checkpoints, and complex multi-step workflows. The tradeoff is complexity -- you must define the graph topology explicitly.

Pi-agent-core's approach is the opposite: a simple while-loop with steering/follow-up hooks. The "graph" is implicit. At ~560 lines, it's vastly simpler but handles only single-agent, linear-with-interruption workflows.

### Multi-Agent: CrewAI, AutoGen/AG2

CrewAI builds teams of role-playing agents that collaborate autonomously. AutoGen (now split into AG2 and the Microsoft Agent Framework) pioneered multi-agent conversations in GroupChat patterns. Both are Python-only and focus on agent-to-agent delegation.

Pi-agent-core has no multi-agent capability by design. It's a single-agent loop. Multi-agent orchestration is left to the application layer.

### Provider SDKs: Claude Agent SDK, OpenAI Agents SDK, Google ADK

Each major provider now ships their own agent framework:

- **Claude Agent SDK** bundles the entire Claude Code CLI as a runtime. It's opinionated -- tools are capabilities (terminal, files, browser) rather than typed function definitions. Locked to Claude.
- **OpenAI Agents SDK** uses four primitives: Agents, Handoffs (delegation), Guardrails (parallel input/output validation), and Tracing. Available in both Python and TypeScript. Provider-agnostic via 100+ LLM support.
- **Google ADK** has a sophisticated event system where events carry state deltas, plus workflow agents (`SequentialAgent`, `ParallelAgent`, `LoopAgent`). Full TypeScript SDK. Four-scope state management (app/user/session/temp).

### TypeScript Frameworks: Vercel AI SDK, Mastra, VoltAgent

In the TypeScript agent space specifically:

- **Vercel AI SDK 6** added a lightweight `Agent` class with `ToolLoopAgent`, `stopWhen` for loop control, and `prepareStep` for per-step configuration. Closest philosophical match to pi-agent-core.
- **Mastra** (by the Gatsby founders, $13M seed, 150K weekly npm downloads) layers agents, workflows, RAG, memory, and evaluation on top of the Vercel AI SDK. Used by Replit, PayPal, Adobe.
- **VoltAgent** adds visual observability (VoltOps Console) to TypeScript agents with memory, RAG, and guardrails.

### Minimalist: smolagents

Hugging Face's smolagents takes a radically different approach -- agents write Python code as their actions rather than using structured tool calls. ~1,000 lines. Code execution runs in E2B/Docker/Pyodide sandboxes. Best for batch/task-oriented work rather than interactive sessions.

### Pi-agent-core's Distinctive Position

What makes pi-agent-core unique at ~560 lines:

1. **Steering and follow-up queues** -- mid-run user interruption with configurable modes (`all` vs `one-at-a-time`). No other framework has this exact pattern.
2. **`convertToLlm` / `AgentMessage` extensibility** -- custom message types via TypeScript declaration merging, with app messages separated from LLM messages at the boundary. Unique.
3. **Event stream granularity** -- 14 event types covering agent/turn/message/tool lifecycle, designed for driving UIs in real-time. More detailed than any competitor.
4. **Three API layers** -- high (Agent class), mid (agentLoop/agentLoopContinue), low (direct pi-ai calls). Choose your abstraction level.

What it deliberately lacks: multi-agent orchestration, guardrails, tracing, durability, MCP, A2A protocol. These are left to extensions or application code.

---

## Layer 3: Terminal Rendering -- The TUI Space

Pi's `pi-tui` package competes with a handful of terminal UI frameworks.

### React for CLI: Ink

Ink is the most popular TypeScript TUI framework. It's literally React -- function components, hooks, context -- rendered to the terminal using Yoga for Flexbox layout. Used by Gatsby, Parcel, Yarn 2, Prisma, Shopify. The Ink UI companion library provides pre-built components (TextInput, Select, Spinner).

Pi-tui takes a fundamentally different approach: imperative components with `render(width): string[]` returning raw ANSI-escaped lines. No React, no virtual DOM, no layout engine. Pi's differential rendering compares strings line-by-line; Ink diffs a virtual terminal buffer via Yoga.

The tradeoff: Ink is declarative and familiar to React developers but adds React as a dependency. Pi-tui is lower-level, giving precise control over every character at the cost of more manual layout work.

### Legacy: Blessed / Terminal Kit

Blessed reimplements ncurses entirely in JavaScript with a DOM-like widget hierarchy and screen buffer. Functionally rich but essentially abandoned (~10 years since last commit). Neo-blessed and neo-neo-blessed are maintenance forks. Terminal Kit provides a layered approach (low-level escape codes -> widgets -> Document Model) with ScreenBuffer composition. Both are heavier than pi-tui and less actively maintained.

### The Go Standard: Bubble Tea

Charm's Bubble Tea (Go) follows The Elm Architecture: immutable Model, `Update(msg)` function, `View(model) -> string`. Bubble Tea v2's "Cursed Renderer" (based on ncurses algorithms) is highly optimized with automatic color downsampling and Kitty keyboard protocol support. 18,000+ apps built on it. Not available in TypeScript, but its patterns influence the ecosystem.

Pi-tui shares some DNA with Bubble Tea -- both support Kitty keyboard protocol, both do differential rendering, both produce string output. Pi-tui uses mutable OOP rather than TEA's functional approach.

### The New Wave: OpenTUI

OpenTUI (by SST, powers OpenCode) uses a Zig core for native-speed rendering with React and SolidJS reconcilers. Built-in tree-sitter syntax highlighting, Flexbox layout, RGBA alpha blending. Currently Bun-exclusive. Represents a possible future direction but still early and platform-limited.

### Python: Textual

Textual (by the Rich/Textualize team) borrows from web dev: a DOM, CSS engine (`.tcss` files), and async event system built on Python's `asyncio`. Can serve the same TUI to a web browser via built-in WebSocket server. Very mature with a broad widget library. Not available in TypeScript.

### Pi-tui's Distinctive Position

Pi-tui's design optimizes for the specific needs of a streaming chat/agent UI:
- **Line-based rendering** -- content grows downward (chat messages, tool output), not fixed-position panels
- **CSI 2026 synchronized output** -- atomic frame updates, zero flicker
- **Overlay system** -- anchored modal positioning with margin clamping, unique among TUI frameworks
- **ANSI code tracking** -- `AnsiCodeTracker` maintains styling state across line breaks for correct wrapping
- **Minimal surface** -- no framework dependency, no layout engine, ~1200 lines total
- **Kitty protocol + IME** -- Kitty keyboard protocol for proper key detection, CURSOR_MARKER for CJK input method positioning

---

## Layer 4: The Product -- Terminal Coding Agents

Pi's `pi-coding-agent` competes in the hottest space in developer tools: AI coding agents.

### The Market Leader: Claude Code

Anthropic's official CLI surpassed $1B annualized revenue by November 2025. Open source, terminal-native, with subagent orchestration, Agent Skills, hooks, and MCP integration. Claude Code's extension model is **prompt/markdown-based** (Skills as SKILL.md files, CLAUDE.md for project context, hooks as shell scripts) rather than programmatic.

Pi's key differentiator is its **TypeScript extension API** -- custom tools with TypeBox schemas, custom UI components (full TUI widgets), event interception with blocking, session persistence, custom rendering. Claude Code's hooks and skills are simpler but less powerful. Claude Code is locked to Anthropic models; Pi supports 22+ providers.

### The IDE Approach: Cursor, Continue, Void

**Cursor** (closed source, VS Code fork) deeply integrates AI into a full IDE -- inline diffs, visual editor, browser testing, subagent orchestration, Mission Control for monitoring parallel agents. Its rules system (`.cursor/rules/*.mdc`) provides project context.

**Continue** (open source) takes a three-component architecture (core/extension/gui) for VS Code and JetBrains, with MCP integration and team rules directories.

**Void** (open source, VS Code fork) emphasizes model flexibility and prompt transparency -- you can view and edit the exact prompts sent to the AI.

Pi is terminal-only by design. IDE integration happens through RPC mode or SDK embedding, not through a VS Code extension.

### The Provider Agents: Codex CLI, Gemini CLI, Copilot CLI

Each major provider now has a terminal agent:

- **Codex CLI** (OpenAI, Rust-based) -- built-in sandboxing via Landlock/seatbelt, AGENTS.md support, MCP integration. Designed around GPT-5.2-Codex.
- **Gemini CLI** (Google, Apache 2.0) -- 1M token context, Google Search grounding, free tier with 60 req/min. Pi actually supports Gemini CLI as a subscription-based auth source.
- **GitHub Copilot CLI** (GA February 2026) -- delegates to specialized sub-agents (Explore, Task, Code Review, Plan). Multi-model: Claude Opus 4.6, GPT-5.3-Codex, Gemini 3 Pro.

All are locked to their respective provider ecosystems. Pi supports all of them as providers while offering its own independent experience.

### The Independents: Aider, Cline, Goose, OpenCode

**Aider** (Python, Apache 2.0) is the most mature independent terminal agent. Its killer feature is a **repository map** built with tree-sitter parsers and PageRank -- it ranks code entities by importance to give the LLM a concise codebase overview without sending all source. Deep git integration with automatic commits. No extension system.

**Cline** (open source) started as a VS Code extension and launched a full CLI (2.0) in February 2026. Its "approve everything" philosophy -- every file change and command requires explicit approval -- is the opposite of Pi's "no permission popups" stance.

**Goose** (Block/Square, MIT) is a general-purpose agent framework (not coding-specific) built on MCP. Goes beyond code -- API interactions, workflow orchestration. Contributed to the Linux Foundation's Agentic AI Foundation.

**OpenCode** (Go, 100K+ stars) has a native TUI similar to Pi's, LSP integration, and a growing extension ecosystem. Its TUI is actually built on **OpenTUI** (the Zig+React rendering engine mentioned above).

### The Terminal Wrapper: Warp

Warp is a Rust-based, GPU-accelerated terminal emulator with AI agents built in. It can host multiple agents simultaneously (its own + Claude Code + Codex + Gemini CLI). "Ambient agents" run in cloud sandboxes triggered by workflow events. Pi runs *inside* Warp rather than competing with it.

### Amp (Sourcegraph)

Amp differentiates with Sourcegraph's code intelligence -- deep cross-repository dependency understanding. Multi-platform (VS Code, JetBrains, Neovim, terminal). Claude Sonnet 4 focused. Closed source, commercial.

### Pi's Distinctive Position as a Coding Agent

1. **Programmatic extensibility** -- Pi's TypeScript extension API (custom tools, UI widgets, event interception, session persistence) is the most comprehensive. Everyone else uses prompt-based instructions (CLAUDE.md, rules files) or MCP.
2. **Deliberate minimalism** -- No subagents, no plan mode, no permission popups, no MCP in the core. All buildable via extensions. This is a philosophical choice, not a limitation.
3. **Broadest provider support** -- 22+ providers with subscription-based auth for Copilot, Gemini CLI, and Antigravity.
4. **Package ecosystem** -- `pi install npm:@foo/pi-tools` for sharing extensions, skills, prompts, themes. Unique among terminal agents.
5. **Session tree branching** -- JSONL sessions with in-place branching via `id`/`parentId`. Navigate with `/tree`, fork with `/fork`. More sophisticated than any competitor's session management.
6. **Four operating modes** -- Interactive TUI, print, RPC, SDK. No other agent offers all four.

---

## Layer 5: Web Chat UIs

Pi's `pi-web-ui` competes with both component libraries and full applications.

### Hooks and Primitives: Vercel AI SDK UI

Vercel provides React hooks (`useChat`, `useCompletion`, `useObject`) that manage client-side state for AI conversations. Framework adapters exist for Vue, Svelte, and SolidJS. These are primitives, not pre-built UI -- you compose your own interface on top.

Pi's `pi-web-ui` provides **concrete components** as Lit Web Components (`<pi-chat-panel>`, `<agent-interface>`). Framework-agnostic, embeddable in any web page. Includes batteries: artifact rendering, sandboxed iframe execution, IndexedDB storage, CORS proxy, tool renderer registry, attachment handling.

### Full Applications: LibreChat, Open WebUI, Lobe Chat, NextChat

The application space is dominated by open-source ChatGPT alternatives:

- **LibreChat** (33.9K stars) -- React + Express + MongoDB. Multi-provider, MCP support, code interpreter, artifacts. Most feature-complete. Recently acquired by ClickHouse.
- **Open WebUI** -- SvelteKit + FastAPI. Offline-first, 9 vector database options for RAG, 15+ web search providers. Designed for Ollama but supports OpenAI-compatible APIs.
- **Lobe Chat** (66.7K stars) -- Next.js + Ant Design. CRDT-based multi-device sync, plugin marketplace, MCP, IndexedDB or PostgreSQL storage.
- **NextChat** (87.2K stars) -- Ultra-compact (~5MB), privacy-first, all data local. Available as Tauri desktop app (Windows/macOS/Linux) plus iOS/Android. Most starred in the category.

All are monolithic applications requiring fork-and-modify to customize. Pi's components are designed to be imported, configured, and composed programmatically -- drop `<pi-chat-panel>` into any web page.

### Desktop Apps: Jan

Jan is an Electron desktop app with a built-in inference engine for running models locally. 100% offline capable, AGPL-3.0 licensed. Different category -- it's a desktop product, not a web component library.

### Pi-web-ui's Distinctive Position

- **Embeddable components** (Lit Web Components) vs. monolithic applications
- **Artifact system** with sandboxed execution (HTML, SVG, PDF, DOCX, Excel rendering in iframe)
- **JavaScript REPL sandbox** with runtime providers for attachments, artifacts, console, and file downloads
- **Online + offline modes** -- artifacts work both with bidirectional messaging and as downloaded standalone HTML with snapshot data
- **Client-side storage** -- IndexedDB with no server requirement
- **Integrated with pi-ai and pi-agent-core** -- not a standalone product

---

## Layer 6: Self-Hosted LLM Serving

Pi's `pi-pods` competes with inference engines and GPU orchestrators.

### The Inference Engines: vLLM vs. SGLang

Pi-pods uses **vLLM** as its backend -- the dominant open-source serving engine (PagedAttention, continuous batching, OpenAI-compatible API).

**SGLang** is the rising challenger. Its **RadixAttention** stores KV cache in a radix tree for automatic prefix reuse -- particularly compelling for agentic workloads where system prompts and conversation prefixes repeat. SGLang claims better throughput in multi-turn scenarios. It's deployed at trillions of tokens/day scale and joined the PyTorch ecosystem. Switching pi-pods from vLLM to SGLang would require significant refactoring but could yield performance benefits.

**TGI** (Hugging Face) entered maintenance mode in December 2025 -- validating pi-pods' vLLM choice.

### Local Inference: Ollama, LM Studio, LocalAI

**Ollama** (Go + llama.cpp) -- simplest local model serving. One-command model running, OpenAI-compatible API, native desktop app. Now offering "Ollama Cloud" for hybrid local/cloud inference. Complementary to pi-pods (local dev vs. production GPU).

**LM Studio** -- polished desktop GUI for discovering and running models. "LM Link" (with Tailscale) enables secure device-to-device model sharing.

**LocalAI** -- Go API server supporting text, images, audio, video generation. Full OpenAI + Anthropic API compatibility. Distributed/P2P inference. Broadest capability set but not optimized for any single use case.

### Cloud Orchestration: SkyPilot, Modal, Replicate

**SkyPilot** (UC Berkeley) is pi-pods' closest conceptual competitor. It abstracts 20+ cloud providers behind a YAML task specification with automatic cloud selection, failover, and cost optimization. Used by Shopify in production. More general-purpose than pi-pods but has no model-specific intelligence (tool call parsers, GPU requirements per model family).

**Modal** is fully serverless -- decorate a Python function with `@modal.function(gpu="H100")` and it handles everything. Scale to zero, pay per second. Opposite philosophy from pi-pods' bare-metal SSH approach.

**Replicate** hosts 50K+ pre-deployed models behind an API. Model marketplace approach with auto-scaling. No infrastructure control.

### The GPU Providers

**RunPod**, **Vast.ai**, **Lambda Labs**, **DataCrunch**, **CoreWeave** -- these are the infrastructure pi-pods runs on. Pi-pods adds the management layer: automatic vLLM installation, model configuration, multi-model GPU allocation with round-robin scheduling, and tool-call parser setup. DataCrunch is recommended for its NFS sharing (download once, use across pods).

### Pi-pods' Distinctive Position

- **Model-aware configuration** -- predefined configs for Qwen, GLM, GPT-OSS, Kimi with correct tool call parsers, GPU requirements, and vLLM args
- **Multi-model management** -- run multiple models on one pod with automatic GPU allocation
- **SSH-native** -- direct access, no abstraction layers, full control
- **Integrated agent CLI** -- test deployed models immediately
- **Simple** -- one `pi pods setup` + `pi start` vs. YAML authoring or web dashboards

---

## Layer 7: AI Slack Bots

Pi's `pi-mom` competes with enterprise assistants and open-source agents.

### Enterprise Platforms: Dust, Glean

**Dust** ($29/user/month) builds multi-agent systems with 50+ data connectors. Agents search and synthesize information from connected SaaS tools. 70%+ weekly adoption at companies like Qonto (3,000+ employees). Open source core.

**Glean** ($4.6B valuation) is an enterprise search + agent platform. Permissions-aware Knowledge Graph across 100+ enterprise apps. 100+ native actions (create Jira ticket, update Salesforce).

Both focus on **information retrieval** -- they search docs and connected apps. Pi-mom focuses on **action execution** -- it runs bash commands, writes files, and installs software autonomously.

### The Direct Competitor: OpenClaw

**OpenClaw** (214K+ GitHub stars) is the closest competitor. Self-hosted Node.js agent connecting to Slack, Telegram, WhatsApp, Discord, and Signal. Full bash execution, browser automation, heartbeat scheduler. MIT licensed.

Pi-mom's advantages over OpenClaw:
- **Self-managing** -- installs tools, writes scripts, configures credentials itself
- **Per-channel workspace isolation** with separate memory, skills, and context
- **Structured memory** -- MEMORY.md (global + per-channel) and searchable log.jsonl
- **Event system** -- cron, one-shot, and immediate events
- **Context management** with compaction for long conversations
- **Docker-first security** -- explicit sandbox model

OpenClaw's advantages: multi-platform support (WhatsApp, Telegram, Signal, Discord), massive community, browser automation built-in.

### Purpose-Built: Kapa.ai, Kilo for Slack

**Kapa.ai** is narrowly focused on answering technical documentation questions via RAG. 50+ connectors, analytics dashboard, SOC 2 compliant. No tool execution.

**Kilo for Slack** (backed by GitLab co-founder) turns Slack threads into code changes and GitHub PRs. Cloud-hosted agents. Narrower than pi-mom's general-purpose approach.

### Pi-mom's Distinctive Position

- **Self-managing** -- uniquely autonomous, installs its own tools and writes its own skills
- **Full system access** -- bash, file I/O, arbitrary command execution (Docker-sandboxed)
- **Two-tier memory** -- global + per-channel MEMORY.md files
- **Event scheduling** -- cron, one-shot, and immediate events with filesystem-based triggers
- **Searchable history** -- log.jsonl records all channel messages for grep-based recall
- **Context compaction** -- handles arbitrarily long conversations without losing context
- **Skills** -- self-authored CLI tools organized as workspace and channel-specific capabilities

---

## The Big Picture

Pi's competitive position comes down to a **philosophical choice**: aggressive extensibility over built-in features. Where Claude Code bakes in subagents, plan mode, and permission popups, Pi provides TypeScript extension hooks to build them. Where Cursor provides a full IDE, Pi provides a terminal harness and SDK. Where LangGraph provides graph orchestration, Pi provides a simple while-loop with steering.

This minimalism has tradeoffs. Pi lacks features that competitors bundle: multi-agent orchestration, built-in MCP, sandboxing, guardrails, tracing, visual debugging. But it offers something no competitor matches: a complete, layered, TypeScript-native stack from LLM API to agent loop to terminal UI to web components -- all composable, all extensible, all MIT-licensed.

---

## Summary Comparison Tables

### Unified LLM API Libraries

| Library | Language | Providers | Tool Calling | Streaming | Cost Tracking | Open Source |
|---------|----------|-----------|-------------|-----------|---------------|-------------|
| **pi-ai** | TypeScript | ~22 (native SDKs) | Yes (TypeBox) | Yes (12 event types) | Yes (built-in) | MIT |
| **LiteLLM** | Python | 100+ | Yes | Yes | Yes | MIT |
| **Vercel AI SDK** | TypeScript | 20+ | Yes | Yes | Token counts only | Apache 2.0 |
| **LangChain** | Python/TS | 150+ | Yes | Yes | No (external) | MIT |
| **OpenRouter** | Service | 500+ | Yes | Yes | Yes | No |
| **Portkey** | TypeScript | 1,600+ | Yes | Yes | Yes (dashboard) | MIT (gateway) |

### Agent Frameworks

| Framework | Language | Agent Loop | Multi-Agent | Event System | Core Size |
|-----------|----------|------------|-------------|--------------|-----------|
| **pi-agent-core** | TypeScript | While-loop + steering | No | EventStream (14 types) | ~560 lines |
| **LangGraph** | Python/TS | Graph with cycles | Yes | Node transitions | Large |
| **CrewAI** | Python | Role-based Crews | Yes (core) | Flow events | Medium |
| **OpenAI Agents SDK** | Python/TS | Tool loop + Handoffs | Yes | Built-in tracing | Medium |
| **Google ADK** | Python/TS | Runner event loop | Yes | Events with state deltas | Medium |
| **Vercel AI SDK 6** | TypeScript | ToolLoopAgent | No | Lifecycle callbacks | Medium |
| **smolagents** | Python | Code-as-action | Minimal | Minimal | ~1,000 lines |

### Terminal Coding Agents

| Tool | Open Source | Models | Extension System | Subagents | Permission System |
|------|-------------|--------|------------------|-----------|-------------------|
| **Pi** | MIT | 22+ providers | TypeScript extensions, skills, packages | No (buildable) | No (buildable) |
| **Claude Code** | Yes | Anthropic only | Skills (Markdown), hooks, MCP | Yes | Yes (built-in) |
| **Aider** | Apache 2.0 | Any LLM | None | No | No |
| **Cline** | Yes | Any LLM | MCP tools | No | Yes ("approve all") |
| **Cursor** | No | Curated set | Rules (.mdc files) | Yes | Sandboxed |
| **Codex CLI** | Yes | OpenAI models | MCP, AGENTS.md | No | Landlock/seatbelt |
| **Gemini CLI** | Apache 2.0 | Gemini only | MCP | No | N/A |
| **Copilot CLI** | Yes | Multi-vendor | Specialized agents | Yes | Plan + autopilot |
| **OpenCode** | Yes | 75+ models | LSP integration | No | N/A |

### TUI Frameworks

| Framework | Language | Rendering | Layout | Component Model |
|-----------|----------|-----------|--------|-----------------|
| **pi-tui** | TypeScript | Line-based diff | Width-constrained lines | `render(width) -> string[]` |
| **Ink** | TypeScript | Virtual buffer + Yoga | Flexbox | React components |
| **Blessed** | JavaScript | Screen buffer diff | Absolute/relative | Widget hierarchy |
| **Bubble Tea** | Go | ncurses-based | String concatenation | Elm Architecture |
| **Textual** | Python | Virtual buffer + CSS | CSS Flexbox/Grid | Widget + DOM + CSS |
| **OpenTUI** | Zig + TypeScript | Native double-buffer | Flexbox | React/SolidJS reconcilers |

### Web Chat UIs

| Framework | Tech Stack | Type | Storage | Providers | Stars |
|-----------|-----------|------|---------|-----------|-------|
| **pi-web-ui** | Lit + mini-lit | Embeddable components | IndexedDB | Multi (via pi-ai) | -- |
| **Vercel AI SDK UI** | React hooks | Library/primitives | BYO | Multi | -- |
| **LibreChat** | React + Express + MongoDB | Full app | MongoDB | Multi | 33.9K |
| **Lobe Chat** | Next.js + Ant Design | Full app | IndexedDB/Postgres | Multi | 66.7K |
| **NextChat** | Next.js + Tauri | Full app | Browser local | Multi | 87.2K |
| **Open WebUI** | SvelteKit + FastAPI | Full app | SQLite/Postgres | Ollama + OpenAI | -- |

### LLM Deployment

| Tool | Category | Relation to pi-pods |
|------|----------|---------------------|
| **vLLM** | Inference engine | Pi-pods' backend |
| **SGLang** | Inference engine | Rising competitor; RadixAttention for agentic workloads |
| **SkyPilot** | Cloud orchestrator | Closest competitor; more general, less model-aware |
| **Modal** | Serverless GPU | Opposite philosophy (fully managed vs bare metal) |
| **Ollama** | Local inference | Complementary (local dev vs production) |

### Slack Bots

| Tool | Category | Relation to pi-mom |
|------|----------|---------------------|
| **OpenClaw** | Self-hosted agent | Closest competitor; multi-platform; less self-managing |
| **Dust** | Enterprise platform | Data-connector focused; no bash execution |
| **Glean** | Enterprise search | Knowledge retrieval; no tool execution |
| **Kapa.ai** | Docs Q&A bot | Narrow RAG focus; no tool execution |
