# Building AI Agents with Pi: A Guided Tour

Pi is a modular ecosystem for building AI-powered coding agents. Each package builds on the ones before it, from raw LLM calls all the way up to full interactive products. This guide walks through the stack from the bottom up.

---

## Talking to LLMs: The Foundation

Everything starts with a problem: there are dozens of LLM providers, each with their own API, authentication, message formats, and quirks. The `pi-ai` package solves this by giving you one interface that works with all of them.

### Your First LLM Call

At its simplest, you pick a model and stream a response:

```typescript
import { getModel, streamSimple } from "@mariozechner/pi-ai";

const model = getModel("anthropic", "claude-sonnet-4-5-20250514");
const context = {
  systemPrompt: "You are a helpful assistant.",
  messages: [{ role: "user", content: "What is TypeScript?", timestamp: Date.now() }],
  tools: [],
};

const stream = streamSimple(model, context, { reasoning: "medium" });
for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
```

That same code works if you swap `"anthropic"` for `"openai"`, `"google"`, `"groq"`, `"mistral"`, or any of the 22+ supported providers. The model registry holds 757 predefined models, each with pricing, context window size, and capability metadata baked in.

### Tool Calling Across Providers

Where things get interesting is tool calling. Each provider has a different wire format for tools -- Anthropic uses one tool ID format, Mistral requires exactly 9 alphanumeric characters, Google wants alphanumeric plus underscores. Pi normalizes all of this.

You define tools once using TypeBox schemas:

```typescript
import { Type } from "@sinclair/typebox";

const tools = [{
  name: "read_file",
  description: "Read a file from disk",
  parameters: Type.Object({
    path: Type.String({ description: "Absolute file path" }),
  }),
}];
```

During streaming, you get `toolcall_delta` events with partially-parsed JSON arguments (via a partial JSON parser), so your UI can show what the model is typing in real-time. At `toolcall_end`, the arguments are fully validated against your TypeBox schema using AJV.

### Thinking and Reasoning

Different providers expose reasoning/thinking in different ways. Anthropic has `thinkingEnabled` with budget tokens. OpenAI has `reasoningEffort`. Google has `budgetTokens` that can be `-1` for dynamic. Pi unifies these behind a single `reasoning` parameter:

```typescript
streamSimple(model, context, { reasoning: "high" });
// Maps to: Anthropic -> thinkingBudgetTokens: 16384
//          OpenAI   -> reasoningEffort: "high"
//          Google   -> budgetTokens: 16384
```

You can override the defaults with custom `thinkingBudgets` if you need fine-grained control.

### Cost Tracking

Every response comes back with a `Usage` object that tracks tokens across four tiers -- input, output, cache read, cache write -- and calculates USD cost using per-model pricing. This flows all the way up the stack to the UI, where users see running cost totals.

### Switching Providers Mid-Conversation

One of the more powerful features is cross-provider handoffs. You can start a conversation with Claude, switch to GPT-5 mid-session, and switch back. Pi handles the message format differences automatically -- converting thinking blocks to text for providers that don't support them, normalizing tool IDs, and stripping provider-specific signatures.

---

## Making LLMs Do Things: The Agent Loop

Raw LLM calls give you text and tool call requests. But someone needs to actually *execute* those tool calls, feed the results back, handle errors, and keep the conversation going. That's what `pi-agent-core` does.

### The Core Loop

The agent operates on a simple cycle:

```
User sends message
  -> LLM responds (possibly with tool calls)
    -> Agent executes each tool
      -> Results fed back to LLM
        -> LLM responds again (more tools, or final answer)
```

In code, you create an `Agent`, give it tools, and call `prompt()`:

```typescript
import { Agent } from "@mariozechner/pi-agent-core";

const agent = new Agent({
  initialState: {
    systemPrompt: "You are a coding assistant.",
    model: getModel("anthropic", "claude-sonnet-4-5-20250514"),
    tools: [readFileTool, bashTool],
    // ...
  },
  convertToLlm: (messages) => messages.filter(m => ["user", "assistant", "toolResult"].includes(m.role)),
});

agent.subscribe((event) => {
  if (event.type === "message_update") {
    // Stream text to UI in real-time
  }
  if (event.type === "tool_execution_start") {
    // Show "Running bash: ls -la" in the UI
  }
});

await agent.prompt("What files are in the current directory?");
```

Under the hood, this runs a **dual-loop architecture**. The inner loop handles LLM turns and tool execution. The outer loop handles follow-up messages queued while the agent was working.

### Steering and Follow-ups

This is where things get sophisticated. While the agent is executing tools, the user might want to interrupt:

```typescript
// User types while agent is running a long bash command
agent.steer("Actually, skip that and look at package.json instead");
```

Steering messages are checked between each tool execution. If one arrives, remaining tools are skipped (with error results so the LLM knows they didn't run), and the steering message is injected into the conversation.

Follow-up messages work differently -- they queue up and execute after the agent finishes its current work:

```typescript
agent.followUp("Now run the tests");
```

Both support two modes: `"one-at-a-time"` (deliver one message per turn) or `"all"` (batch deliver everything queued).

### Custom Message Types

The agent system separates "app messages" from "LLM messages." Your app might have message types the LLM shouldn't see -- notifications, artifacts, UI state. You define these via TypeScript declaration merging:

```typescript
declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    notification: { role: "notification"; text: string; timestamp: number };
  }
}
```

Then your `convertToLlm()` function filters or transforms these before they reach the LLM. The agent stores the full message history (including custom types), but the LLM only sees what you send it.

### Context Transformation

Before each LLM call, messages pass through an optional pipeline:

```
All messages -> transformContext() -> convertToLlm() -> LLM
```

`transformContext` is where you'd implement context window management -- dropping old messages, summarizing history, injecting external knowledge. `convertToLlm` handles the final conversion from your app's message types to standard LLM format.

### The Event System

The agent emits 14 event types covering the full lifecycle: agent start/end, turn start/end, message start/update/end, tool execution start/update/end. UIs subscribe to these for real-time rendering. Tool execution events include streaming updates via an `onUpdate` callback, so long-running tools can show progress.

---

## Drawing to the Terminal: The TUI Framework

Before we can build the interactive coding agent, we need a way to render UIs in the terminal. The `pi-tui` package is a minimal framework built for this purpose, with one core goal: **no flickering**.

### How Rendering Works

Terminal UIs typically flicker because they clear the screen and redraw everything on each update. Pi-tui avoids this with a three-strategy differential renderer:

1. **First render** -- Output lines without clearing (preserves scrollback history)
2. **Width change** -- Full clear and re-render (line wrapping invalidated everything)
3. **Normal update** -- Find the first and last changed lines, move the cursor there, clear and redraw only those lines

All output is wrapped in the **CSI 2026 synchronized output protocol**, which tells the terminal to buffer updates and flush atomically. The result is buttery smooth rendering even during heavy streaming.

### Components

Everything is a component with a simple interface:

```typescript
interface Component {
  render(width: number): string[];  // Return lines that fit within width
  handleInput?(data: string): void;
  invalidate?(): void;
}
```

The framework ships with 13 built-in components. The most important ones:

**Editor** -- A multi-line text editor with vertical scrolling, autocomplete (slash commands + file paths), paste detection (large pastes become `[paste #1 +50 lines]` markers), history navigation, and Emacs-style kill ring. This is what powers the input area in the coding agent.

**Markdown** -- Full markdown rendering with syntax highlighting, headings, bold/italic, code blocks, lists, blockquotes, and links. Used for rendering assistant messages.

**Image** -- Inline image rendering using Kitty graphics protocol (Kitty, Ghostty, WezTerm), iTerm2 protocol fallback, or text placeholder on unsupported terminals.

**SelectList** and **SettingsList** -- Interactive menus for model selection, settings panels, session browsers.

### Overlays

Modal dialogs use the overlay system, which supports sophisticated positioning:

```typescript
tui.addOverlay(component, {
  anchor: "center",
  width: "80%",
  maxHeight: "90%",
  margin: { top: 2, bottom: 2 },
  visible: (cols, rows) => cols > 40,  // Hide on narrow terminals
});
```

Overlays are composited in a single pass using segment extraction -- the framework tracks active ANSI escape codes so styling doesn't bleed across overlay boundaries.

### Input Handling

The framework automatically detects and enables the **Kitty keyboard protocol**, which disambiguates keys that traditional terminals can't (like Escape vs Ctrl+[) and reports press, repeat, and release events. On Windows, it uses Koffi FFI to enable virtual terminal input for proper modifier key support.

All text utilities -- `visibleWidth()`, `truncateToWidth()`, `wrapTextWithAnsi()` -- properly handle Unicode via native `Intl.Segmenter`, East Asian double-width characters, and ANSI escape codes. An `AnsiCodeTracker` maintains active styling state across line breaks so wrapping doesn't lose colors.

---

## The Product: An Interactive Coding Agent

Now we have all the pieces: LLM communication, agent orchestration, and terminal rendering. The `pi-coding-agent` package puts them together into an interactive AI coding assistant.

### Running It

Launch `pi` in any project directory and you get a full TUI:

```
+---------------------------------------------------+
| Startup header (shortcuts, loaded files, model)   |
|---------------------------------------------------|
|                                                   |
| Chat history (scrollable)                         |
|   User messages, assistant responses,             |
|   collapsible tool output                         |
|                                                   |
|---------------------------------------------------|
| Multi-line editor (@ for files, ! for commands)   |
|---------------------------------------------------|
| Footer: cwd | session | tokens | cost | model    |
+---------------------------------------------------+
```

### The Built-in Tools

The agent comes with 7 tools that cover the core coding workflow:

**read** reads files (text or images, auto-resized). **write** creates files. **edit** does surgical text replacement with exact-match (it generates unified diffs so you can see what changed). **bash** runs shell commands with streaming output. **grep** uses ripgrep for fast content search. **find** uses `fd` for fast file search. **ls** lists directories.

All tools have a pluggable operations interface. This means you can swap out the file system backend -- for example, to make all tools work over SSH:

```typescript
const readTool = createReadTool("/remote/path", {
  operations: mySSHReadOperations,
});
```

Output is capped at 8192 lines or 524KB per tool call, with overflow metadata so the model can request more with offset/limit.

### Session Branching

Sessions are stored as JSONL files, but they're not linear -- they form a **tree**. Every message has an `id` and optional `parentId`. When you want to try a different approach, you branch:

Press Escape twice to open the `/tree` navigator. You see your entire conversation history as a tree. Navigate to any point, and the agent loads that branch's context. You can label important points, search across branches, and fork sessions into separate files.

### Compaction

Long sessions eventually exceed the model's context window. Pi handles this automatically: when token count approaches the limit, it summarizes older messages while keeping recent ones intact. The algorithm finds a cut point (keeping ~20KB of recent context), asks the LLM to summarize everything before that point, and inserts a `CompactionEntry`. The full history remains in the JSONL file -- `/tree` can always navigate back.

If a single turn is so large it exceeds the keep-recent budget, the system splits it and generates two summaries: one for prior history and one for the early part of the oversized turn.

### Making It Your Own

Pi's philosophy is "aggressively extensible rather than feature-complete." The core is intentionally minimal -- no MCP, no sub-agents, no permission popups built in. Instead, you extend it:

**Extensions** are TypeScript modules that can register tools, commands, keyboard shortcuts, UI components, and event hooks. They can intercept tool calls (to build permission gates), inject context, and persist custom data in sessions:

```typescript
// ~/.pi/agent/extensions/safety.ts
export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("Dangerous", "Allow rm -rf?");
      if (!ok) return { block: true, reason: "User denied" };
    }
  });
}
```

**Skills** are markdown files with YAML frontmatter that describe capabilities the agent can invoke on demand (via `/skill:name`). They follow the Agent Skills standard.

**Prompt templates** are reusable markdown snippets with variable expansion, typed `/name` to expand.

**Packages** bundle all of the above and are installable from npm or git:

```bash
pi install npm:@foo/pi-tools
```

### Context Files

The system prompt is built dynamically from:
1. Tool descriptions
2. `AGENTS.md` / `CLAUDE.md` files (walked up from cwd to root, plus `~/.pi/agent/AGENTS.md`)
3. Active skills
4. Current date and working directory

You can replace the entire system prompt with `.pi/SYSTEM.md` or append to it with `.pi/APPEND_SYSTEM.md`.

### Beyond the Terminal

The coding agent also runs headless. **RPC mode** (`pi --mode rpc`) speaks JSON-RPC over stdin/stdout -- you send `prompt`, `steer`, `follow_up` commands and receive streamed events. This is how IDE integrations work.

**SDK mode** embeds pi programmatically:

```typescript
import { createAgentSession, SessionManager } from "@mariozechner/pi-coding-agent";

const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  // ...
});

session.subscribe((event) => { /* handle events */ });
await session.prompt("Refactor the auth module");
```

---

## The Same Thing, In a Browser

The `pi-web-ui` package provides the same chat experience as web components, built with mini-lit and Tailwind CSS v4.

### Chat Interface

Drop `<pi-chat-panel>` into your page and wire it to an Agent:

```html
<pi-chat-panel></pi-chat-panel>
```

The panel handles message rendering, streaming, model selection, thinking level control, and file attachments (drag-drop or paste). On desktop, it shows a side-by-side layout with an artifacts panel; on mobile, artifacts overlay the chat.

### Attachments

Users can attach PDFs, Word docs, PowerPoints, Excel spreadsheets, and images. Each format has a text extractor that preserves document structure:

- PDFs -> `<pdf><page number="1">extracted text</page>...</pdf>`
- DOCX -> `<docx><page>text</page>...</docx>`
- PPTX -> `<pptx><slide>text</slide><notes>speaker notes</notes></pptx>`
- Excel -> `<excel><sheet name="Sheet1">CSV data</sheet>...</excel>`

The extracted text is sent to the LLM as structured content, so it understands the document's organization.

### Artifacts and the REPL

The artifacts system lets the LLM create, update, and manage files that render inline -- HTML pages in sandboxed iframes, SVGs, markdown, spreadsheets, PDFs. HTML artifacts get a console, preview/code toggle, and reload button.

The JavaScript REPL tool executes code in a sandboxed iframe with access to user attachments and artifacts. Runtime providers give sandbox code APIs like `listAttachments()`, `readTextAttachment()`, `getArtifact()`, and `createOrUpdateArtifact()`. This works in both online mode (bidirectional messaging between iframe and host) and offline mode (downloaded standalone HTML with snapshot data baked in).

### Storage

Everything persists in IndexedDB via four stores: settings (key-value), provider API keys, sessions (dual-store: full data + lightweight metadata for fast listing), and custom provider definitions (Ollama, LM Studio, vLLM endpoints). A smart CORS proxy system auto-detects which providers need proxying based on the key type -- OAuth tokens get proxied, regular API keys don't.

---

## Deploying to Slack

The `pi-mom` package takes the same agent core and deploys it as a Slack bot. Mention `@mom` in a channel or DM her, and she responds with full bash, file read/write, and edit capabilities.

### How It Works

Mom connects via Slack Socket Mode (WebSocket). Every message in her channels is logged to `log.jsonl` -- she sees the full conversation history, not just @mentions. When triggered, she syncs the log to a `context.jsonl` session file, rebuilds her system prompt with fresh memory and skills, and runs the agent loop.

Tool output goes to threads (keeping main messages concise), and she posts usage summaries when done. Each channel gets its own independent runner, queue, and state.

### Memory

Mom has a two-tier memory system. **Global memory** (`data/MEMORY.md`) is shared across all channels -- team conventions, architecture decisions. **Channel memory** (`data/<channel>/MEMORY.md`) holds channel-specific context. Tell her "Remember that we deploy on Fridays" and she writes it to the appropriate file. Both are injected into her system prompt at each run.

### Self-Managing

Mom installs her own tools. Need `jq`? She runs `apk add jq`. Need a Python library? She `pip install`s it. Need a custom workflow? She writes a skill script to `data/skills/` and uses it going forward. All of this happens inside Docker for safety:

```bash
mom --sandbox=docker:mom-container ./data
```

The host process handles Slack and LLM API calls; actual command execution happens inside the container via `docker exec`.

### Scheduled Events

Mom can schedule work -- immediate events (for webhooks), one-shot events (run at a specific time), and periodic events (cron syntax with timezone). Events are JSON files in `data/events/`, watched by a filesystem watcher. Periodic events support `[SILENT]` responses -- if there's nothing to report, the message is deleted instead of cluttering the channel.

---

## Running Your Own Models

The `pi-pods` package handles the last mile: deploying open-source LLMs on GPU infrastructure so you don't depend on API providers.

### Setup

Point it at any Ubuntu machine with NVIDIA GPUs:

```bash
pi pods setup my-pod "ssh root@1.2.3.4" --mount "mount /dev/sda1 /models"
```

This SSHes in, installs CUDA, Python, vLLM, and configures model storage. It auto-detects your GPUs and stores the pod configuration locally in `~/.pi/pods.json`.

### Deploying Models

For predefined models, it's one command:

```bash
pi start Qwen/Qwen3-Coder-30B-A3B-Instruct --name qwen3
```

Pi selects the best GPU configuration for your hardware, downloads the model via HuggingFace, and starts a vLLM server with the right tool calling parser. The process runs in its own session (`setsid`) so it survives SSH disconnects.

For large models, it handles multi-GPU allocation automatically:

```bash
pi start Qwen/Qwen3-Coder-480B-A35B-Instruct --name qwen3-480b --gpus 8
```

It uses round-robin allocation across least-used GPUs, so you can run multiple models on the same machine without conflicts.

### Connecting

Every deployed model exposes an OpenAI-compatible API. You can point pi-ai at it:

```typescript
const model = {
  id: "Qwen/Qwen3-Coder-30B-A3B-Instruct",
  api: "openai-completions",
  provider: { name: "custom", baseUrl: "http://my-pod:8001/v1" },
  // ...
};
```

Or use any OpenAI-compatible tool. The models support tool calling through provider-specific parsers (`hermes`, `qwen3_coder`, `glm45`, `kimi_k2`), so they work with the full agent stack.

### Provider Support

DataCrunch pods can share model storage via NFS -- download once, use everywhere. RunPod uses network volumes. Vast.ai, Prime Intellect, and AWS EC2 are also supported. Three vLLM build variants handle different model families: release (stable), nightly (latest), and gpt-oss (special build for OpenAI's open-source models).

---

## How It All Connects

The dependency graph tells the story:

```
pi-ai                    <- Foundation: talk to any LLM
  ^
pi-agent-core            <- Orchestration: tool execution loop
  ^
pi-tui                   <- Terminal rendering (independent)
  ^
pi-coding-agent          <- The product: interactive coding agent
                            (uses pi-ai + pi-agent-core + pi-tui)

pi-web-ui                <- Browser alternative (uses pi-ai + pi-agent-core)
pi-mom                   <- Slack deployment (uses pi-ai + pi-agent-core)
pi-pods                  <- Self-hosted models (uses pi-agent-core)
```

All packages share lockstep versioning (currently v0.55.3) and are managed as npm workspaces. The lower layers are designed to be used independently -- you can use `pi-ai` without the agent, or `pi-agent-core` without the TUI. The higher-level packages are opinionated products built on the composable foundation.
