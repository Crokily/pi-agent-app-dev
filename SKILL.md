---
name: pi-agent-app-dev
description: "Best practices for building agent-powered applications (not chatbots) using pi-mono SDK (@mariozechner/pi-ai, @mariozechner/pi-agent-core, @mariozechner/pi-coding-agent). Use when the task involves: designing agent application architecture, embedding agent capabilities into web/backend services, defining tools for agents, choosing between bash and custom tools, writing system prompts for agent executors, implementing security for autonomous agents, building agent-as-backend systems, integrating pi-mono SDK into existing applications, creating Extensions or Skills for pi, or any development where an LLM agent loop drives real-world actions (deployment, infrastructure, data processing) rather than conversation."
---

# Pi-Mono Agent Application Development

## Core Philosophy

Agent applications ≠ chatbots. Build **autonomous execution engines** that receive user intent, plan steps, call tools, handle errors, and deliver results — not conversational interfaces.

**The formula:**
```
Agent App = Tools (capabilities) + System Prompt (rules) + Agent Loop (engine)
```

- **Tools**: Define what the agent CAN do (Docker ops, DB queries, API calls)
- **System Prompt**: Define what the agent SHOULD do (safety rules, scope, behavior)
- **Agent Loop**: The SDK's core cycle: prompt → LLM decision → tool call → result → continue/finish

## Pi-Mono Architecture (choose your layer)

```
Layer 3: pi-coding-agent  → createAgentSession() — full app layer (sessions, extensions, skills)
Layer 2: pi-agent-core    → Agent / agentLoop()   — engine layer (tool calling, events, state)
Layer 1: pi-ai            → stream() / complete()  — LLM interface (multi-provider, tools)
```

| Scenario | Layer | Entry Point |
|----------|-------|-------------|
| Embed agent in existing web app | 3 | `createAgentSession()` |
| Agent as isolated subprocess | 3 | `pi --mode rpc` |
| Full control over agent loop | 2 | `agentLoop()` |
| Only need LLM + tool calling | 1 | `stream()` / `complete()` |

## Tool Architecture: The Hybrid Model

Do NOT choose between bash and custom tools. Use a **three-layer hybrid**:

```
Layer 1: bash (general primitive)    → exploration, diagnostics, flexible ops
Layer 2: read/write/edit (optimized) → more reliable than bash for file ops
Layer 3: custom tools (business)     → safety-critical ops, structured output, transactions
+ Skills (knowledge injection)       → teach agent HOW to use Layer 1-3 for specific domains
```

**Decision framework for each operation:**

1. Security-sensitive? → Custom tool (hardcode checks in `execute`)
2. Needs structured output for app layer? → Custom tool (return JSON)
3. Involves DB/external API/transactions? → Custom tool (encapsulate logic)
4. Model can do it reliably with bash? → bash + skill guidance
5. High-frequency file operation? → Optimized primitive (read/write/edit)

**Why bash is powerful:** Infinite surface area, model familiarity from training data, composable via pipes, context-efficient (one 200-token tool definition). But bash has **no safety boundary** — never trust System Prompt as security; all security checks must be hardcoded in tool `execute` functions.

See [references/tool-design.md](references/tool-design.md) for detailed tool design patterns, Anthropic research analysis, and code examples.

## Development Patterns

### Pattern 1: SDK Embedded Agent (most common)

For embedding agent execution into a web service or backend:

```typescript
import { createAgentSession, SessionManager, DefaultResourceLoader, type ToolDefinition } from "@mariozechner/pi-coding-agent";
import { getModel } from "@mariozechner/pi-ai";
import { Type } from "@sinclair/typebox";

// 1. Define business tools with safety checks in execute()
const myTool: ToolDefinition = {
  name: "do_something",
  label: "Do Something",
  description: "What this tool does (shown to LLM)",
  parameters: Type.Object({
    target: Type.String({ description: "target identifier" }),
  }),
  execute: async (toolCallId, params, signal, onUpdate, ctx) => {
    // SECURITY: hardcoded checks here — LLM cannot bypass
    if (!isAuthorized(currentUserId, params.target)) throw new Error("Unauthorized");
    onUpdate?.({ content: [{ type: "text", text: "Working..." }] }); // stream progress
    const result = await performAction(params.target);
    return { content: [{ type: "text", text: JSON.stringify(result) }], details: result };
  },
};

// 2. Configure system prompt via ResourceLoader
const loader = new DefaultResourceLoader({
  systemPromptOverride: () => `You are an execution engine. Execute intent directly, verify results, auto-fix on failure.`,
});
await loader.reload();

// 3. Create session and run
const { session } = await createAgentSession({
  model: getModel("anthropic", "claude-sonnet-4-20250514"),
  resourceLoader: loader,
  customTools: [myTool],
  tools: [],  // omit built-in tools if not needed, or keep bash via createBashTool()
  sessionManager: SessionManager.inMemory(),
});

// 4. Collect results via events
const executions: any[] = [];
session.subscribe((event) => {
  if (event.type === "tool_execution_end") executions.push({ tool: event.toolName, ok: !event.isError });
});

await session.prompt("Deploy the app to staging");
```

### Pattern 2: Agent-as-Backend (RPC subprocess)

For process isolation (how OpenClaw integrates):

```bash
pi --mode rpc --no-session -e ./my-extension.ts
```

Communicate via JSON over stdin/stdout. Send `{"type":"prompt","message":"..."}`, receive event stream. See [references/integration-patterns.md](references/integration-patterns.md) for RPC protocol details.

### Pattern 3: Bare Agent Loop (maximum control)

```typescript
import { agentLoop, type AgentContext, type AgentLoopConfig } from "@mariozechner/pi-agent-core";
import { getModel } from "@mariozechner/pi-ai";

const context: AgentContext = { systemPrompt: "...", messages: [], tools: [myTool] };
const config: AgentLoopConfig = {
  model: getModel("anthropic", "claude-sonnet-4-20250514"),
  convertToLlm: (msgs) => msgs.filter(m => ["user","assistant","toolResult"].includes(m.role)),
};

for await (const event of agentLoop([userMessage], context, config)) {
  if (event.type === "tool_execution_end") console.log(`Tool ${event.toolName} done`);
}
```

### Pattern 4: Minimal LLM + Tool Loop (pi-ai only)

```typescript
import { getModel, complete, Type, type Context, type Tool } from "@mariozechner/pi-ai";

const tools: Tool[] = [{ name: "act", description: "...", parameters: Type.Object({...}) }];
const context: Context = { systemPrompt: "...", messages: [{ role: "user", content: "..." }], tools };

for (let turn = 0; turn < 10; turn++) {
  const response = await complete(getModel("anthropic", "claude-sonnet-4-20250514"), context);
  context.messages.push(response);
  const calls = response.content.filter(b => b.type === "toolCall");
  if (!calls.length) break; // agent considers task complete
  for (const call of calls) {
    const result = await executeTool(call.name, call.arguments);
    context.messages.push({ role: "toolResult", toolCallId: call.id, toolName: call.name,
      content: [{ type: "text", text: JSON.stringify(result) }], isError: false, timestamp: Date.now() });
  }
}
```

## Security Model

**Golden rule: NEVER trust System Prompt as a security boundary.** LLMs can be prompt-injected. All security MUST be hardcoded in tool `execute` functions.

Four-layer defense:

| Layer | Mechanism | Reliability |
|-------|-----------|-------------|
| Tool-level validation | Hardcoded checks in `execute()` | ★★★★★ Strongest |
| Extension event gates | `pi.on("tool_call", …)` → `{ block: true }` | ★★★★ Strong |
| System Prompt rules | "Do not delete..." in prompt | ★★ Weak (bypassable) |
| Infrastructure isolation | Sandbox, cgroup, network namespace | ★★★★★ Strongest |

See [references/security.md](references/security.md) for implementation patterns.

## Production Essentials

**Cost control**: Monitor `event.message.usage.cost.total` per turn; abort if budget exceeded.

**Timeout**: Use `AbortController` with total timeout; set max turns in settings.

**Structured output**: Define a `report_result` tool and instruct in System Prompt: "Always call report_result when task completes." This gives your app layer structured JSON instead of free text.

**Multi-model strategy**: Use cheap model (Haiku) for simple tasks, strong model (Sonnet/Opus) + thinking for complex tasks. Pi-ai's `getModel()` makes switching trivial.

**Context management**: For long tasks, enable compaction in settings. For cross-request memory, serialize `Context` with `JSON.stringify()` and restore later — works across providers.

**Observability**: Subscribe to all events and emit structured logs with `sessionId`, `toolName`, `duration`, `isError`.

See [references/production.md](references/production.md) for detailed patterns with code.

## Extension System for Agent Apps

Extensions add capabilities beyond tools. Key patterns for agent apps:

- **Permission gates**: `pi.on("tool_call")` to block dangerous operations
- **Context injection**: `pi.on("before_agent_start")` to inject runtime context (user info, system state)
- **Custom commands**: `pi.registerCommand()` for operator actions
- **State persistence**: `pi.appendEntry()` for data that survives restarts
- **Structured reporting**: `pi.registerTool()` with custom `renderResult` for rich output

See [references/integration-patterns.md](references/integration-patterns.md) for extension examples.

## Context Engineering

Context is a **finite resource with diminishing returns**. Principles:

1. **Minimal tool set**: Every tool definition costs tokens. Prefer bash + few critical custom tools over dozens of specific tools.
2. **Progressive disclosure**: Use Skills and `CLAUDE.md`/`AGENTS.md` for just-in-time context. Don't dump everything into System Prompt.
3. **Token-efficient tool responses**: Truncate large outputs (use `truncateHead`/`truncateTail` from pi-coding-agent). Always inform the LLM where to find full output.
4. **Compaction**: For long tasks, configure `compaction.enabled: true` in settings. Pi auto-summarizes old messages when context nears the limit.
5. **Agentic search over pre-loading**: Let the agent use bash (`grep`, `find`, `head`) to discover information on-demand rather than pre-loading everything into context.

## Quick Reference: Key Imports

```typescript
// Layer 3 (full SDK)
import { createAgentSession, SessionManager, SettingsManager, AuthStorage, ModelRegistry,
  DefaultResourceLoader, codingTools, readOnlyTools, createCodingTools,
  type ToolDefinition, type ExtensionAPI } from "@mariozechner/pi-coding-agent";

// Layer 2 (agent engine)
import { Agent, agentLoop, agentLoopContinue } from "@mariozechner/pi-agent-core";

// Layer 1 (LLM interface)
import { getModel, stream, complete, Type, StringEnum, type Context, type Tool } from "@mariozechner/pi-ai";
```

## References

- [references/tool-design.md](references/tool-design.md) — Bash vs custom tools analysis, hybrid architecture, Anthropic research findings, tool code examples
- [references/security.md](references/security.md) — Four-layer security model, permission gate extensions, sandbox patterns
- [references/integration-patterns.md](references/integration-patterns.md) — SDK embedding, RPC subprocess, extension patterns, OpenClaw case study
- [references/production.md](references/production.md) — Cost control, timeouts, multi-model, context management, observability, testing
