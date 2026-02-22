---
name: pi-agent-app-dev
description: "Best practices for building agent-powered applications using pi-mono SDK (@mariozechner/pi-ai, @mariozechner/pi-agent-core, @mariozechner/pi-coding-agent). Use when the task involves: designing agent application architecture, embedding agent capabilities into web/backend services, writing system prompts for agent executors, implementing security for autonomous agents, building agent-as-backend systems, integrating pi-mono SDK into existing applications, creating Extensions or Skills for pi, or any development where an LLM agent loop drives real-world actions (deployment, infrastructure, data processing) rather than conversation."
---

# Pi-Mono Agent Application Development

## Core Philosophy: Environment Provider, Not Orchestrator

Agent-driven applications ≠ traditional AI workflows. Do not build an orchestration graph that calls LLMs at each node. Instead, **provide an environment** where the agent autonomously decides what to do, verifies its own work, and self-corrects.

> "Give as much control as possible to the language model itself, and keep the scaffolding minimal." — Anthropic, SWE-bench
>
> "Claude Code embraces radical simplicity. The team deliberately minimizes business logic, allowing the underlying model to perform most work." — Pragmatic Engineer
>
> "Maybe the best architecture is almost no architecture at all. Just filesystems and bash." — Vercel

**The formula:**
```
Agent App = Environment (tools + filesystem + state) + Harness (prompt + rules + permissions + hooks) + Loop (engine)
```

The **environment** gives the agent capabilities. The **harness** steers it without constraining its approach. The **loop** runs until the agent decides it is done.

## The Two Paradigms

Understand which paradigm you are building before writing code:

| | App-Driven (Orchestration) | Agent-Driven (Harness) |
|---|---|---|
| **Who decides next step** | Your code (graph/workflow) | The model (via prompt + tools) |
| **Frameworks** | LangGraph, PydanticAI workflows, Mastra | Claude Agent SDK, Pi SDK, OpenCode |
| **Control surface** | Explicit nodes, edges, routing logic | System prompt, permissions, skills, hooks |
| **Best for** | Predictable pipelines, compliance-critical flows | Open-ended tasks, self-healing, creative problem-solving |
| **Anti-pattern** | Using an agent SDK but hardcoding every step as a tool | Giving the agent infinite freedom with no harness |

**This skill is for the agent-driven paradigm.** If your task is better served by a deterministic pipeline, use traditional orchestration.

## Tool Design: Less Is More

### The Default Toolset: bash + read + write + edit

Pi's 4 built-in tools (~1000 tokens total) are sufficient for most agent applications. Bash alone gives the agent access to the entire Unix environment — docker, curl, git, grep, jq, and anything else installed.

> Vercel deleted 17 specialized tools and replaced them with 1 bash tool. Success rate went from 80% → 100%, 3.5x faster, 37% fewer tokens.

**Start with zero custom tools. Add them only when you hit a concrete problem bash cannot solve safely.**

### When Custom Tools Are Justified

Only three situations justify a custom tool:

1. **Security-critical operations** — The tool must enforce access control that the LLM cannot bypass via prompt injection. Hardcode ownership checks, allowlists, rate limits in `execute()`.

2. **Structured output for your app layer** — Your application needs to parse a typed JSON result (e.g., `{ instanceId, port, status }`) rather than scraping stdout text.

3. **Transactional / external API operations** — Database transactions, OAuth flows, or third-party API calls with retry logic that bash cannot reliably encapsulate.

If your custom tool is just a function wrapper with no validation, verification, or structured return — **delete it and let the agent use bash**.

See [references/tool-design.md](references/tool-design.md) for detailed patterns and code examples.

### Anti-Pattern: The Message Passer

The most common mistake when building with agent-driven SDKs:

```
❌ BAD: Button click → create Task in DB → agent picks up task
   → agent calls instance_create tool (which is just the old function)
   → tool returns result → agent calls report_result → done

   Agent added zero intelligence. It's just a slow, unreliable function router.
```

```
✅ GOOD: Button click → agent receives goal + environment context
   → agent inspects current state (bash: docker ps, curl health endpoints)
   → agent decides what to create and how
   → agent creates instance (bash or minimal custom tool)
   → agent verifies instance is healthy (curl, docker inspect)
   → agent diagnoses and fixes problems if unhealthy
   → agent reports structured result only after verification
```

The difference: the agent **thinks, verifies, and self-corrects**. That is the value proposition.

## Harness Design: Steering Without Constraining

In agent-driven systems, the harness replaces the workflow graph. It is the developer's primary control surface.

### The Five Harness Elements

| Element | Purpose | Example |
|---------|---------|---------|
| **System prompt** | Rules, goals, verification procedures, behavioral constraints | "After creating an instance, verify it responds to health checks" |
| **AGENTS.md / Skills** | Domain knowledge injected on-demand into context | Deployment procedures, error diagnosis playbooks |
| **Filesystem state** | Shared memory between agent turns, sessions, and sub-agents | `/tmp/deploy-state.json`, checklist files, plan files |
| **Hooks / Extensions** | Code-level interception points for security gates and context injection | `pi.on("tool_call")` to block dangerous bash commands |
| **Permissions** | Restrict tool access based on trust level or context | Read-only mode for exploration, full access for execution |

### System Prompt Design

The system prompt is the most important piece of an agent-driven application. It is where you encode **judgment**, not just instructions.

**Principles:**
- **Be specific about verification**: "After deploying, run `curl -sf http://localhost:{port}/health` and confirm HTTP 200"
- **Encode recovery strategies**: "If health check fails, check `docker logs --tail 50` for errors. If OOM, increase memory limit. If config error, regenerate config and restart."
- **Set boundaries, not steps**: "Never delete user data without explicit instruction" rather than "Step 1: check, Step 2: delete"
- **Keep it under 1000 tokens**: Pi's own system prompt is ~200 tokens. Models are RL-trained for agentic behavior — they don't need 10K tokens of instruction.

### Filesystem as Agent Memory

The filesystem is the universal persistence layer for agents. It is unlimited, persistent, directly operable, and requires no special tools.

**Patterns:**
- **State files**: Agent writes `state.json` with current progress; reads it on next invocation for continuity
- **Plan files**: Agent writes `PLAN.md` with task breakdown; updates checkboxes as it progresses
- **Shared memory**: Multiple agent sessions read/write the same workspace directory; filesystem is the coordination mechanism
- **Reinforcement via files**: After each tool call, inject filesystem state into context to keep the agent aware of overall progress

> "File System as Extended Memory: unlimited in size, persistent by nature, and directly operable by the agent itself." — Manus

## Pi-Mono SDK Integration

### Choose Your Layer

```
Layer 3: pi-coding-agent  → createAgentSession() — full app (sessions, extensions, skills)
Layer 2: pi-agent-core    → agentLoop()           — engine (tool calling, events, state)
Layer 1: pi-ai            → stream() / complete()  — LLM interface (multi-provider, tools)
```

| Scenario | Layer | Entry Point |
|----------|-------|-------------|
| Full agent app with sessions, extensions | 3 | `createAgentSession()` |
| Agent as isolated subprocess | 3 | `pi --mode rpc` |
| Custom agent loop with full control | 2 | `agentLoop()` |
| Single LLM call with tool use | 1 | `stream()` / `complete()` |

### Pattern 1: Minimal Agent App (most recommended)

```typescript
import { createAgentSession, SessionManager, DefaultResourceLoader } from "@mariozechner/pi-coding-agent";
import { getModel } from "@mariozechner/pi-ai";

const loader = new DefaultResourceLoader({
  systemPromptOverride: () => SYSTEM_PROMPT,
});
await loader.reload();

const { session } = await createAgentSession({
  model: getModel("anthropic", "claude-sonnet-4-20250514"),
  resourceLoader: loader,
  customTools: securityCriticalToolsOnly, // minimal — 0 to 3 tools max
  tools: [],  // or include bash via createBashTool() for full capability
  sessionManager: SessionManager.inMemory(),
});

await session.prompt(userIntent);
```

### Pattern 2: Bare Loop (maximum control)

```typescript
import { agentLoop, type AgentContext, type AgentLoopConfig } from "@mariozechner/pi-agent-core";

const context: AgentContext = { systemPrompt: PROMPT, messages: [], tools: myTools };
const config: AgentLoopConfig = {
  model: getModel("anthropic", "claude-sonnet-4-20250514"),
  convertToLlm: (msgs) => msgs.filter(m => ["user","assistant","toolResult"].includes(m.role)),
  getApiKey: (provider) => keys[provider],
};

for await (const event of agentLoop([userMessage], context, config)) {
  if (event.type === "tool_execution_end") console.log(`${event.toolName}: ${event.isError ? "FAIL" : "OK"}`);
}
```

### Pattern 3: RPC Subprocess (process isolation)

```bash
pi --mode rpc --no-session -e ./my-extension.ts
```

Communicate via JSON over stdin/stdout. Send `{"type":"prompt","message":"..."}`, receive event stream.

See [references/integration-patterns.md](references/integration-patterns.md) for full protocol details and examples.

## Security Model

**Golden rule: NEVER trust System Prompt as a security boundary.** All security MUST be hardcoded in tool `execute()` functions or enforced via extension event gates.

| Layer | Mechanism | Reliability |
|-------|-----------|-------------|
| Tool-level validation | Hardcoded checks in `execute()` | ★★★★★ |
| Extension event gates | `pi.on("tool_call", …)` → `{ block: true }` | ★★★★ |
| Infrastructure isolation | Container, cgroup, network namespace | ★★★★★ |
| System Prompt rules | "Do not delete..." in prompt | ★★ Weak |

For code execution sandboxing options (Docker, E2B, Monty, Cloudflare CodeMode), see [references/security.md](references/security.md).

## Production Essentials

**Cost control**: Monitor `event.message.usage.cost.total` per turn; abort if budget exceeded. Use cheap models for simple tasks, strong models for complex ones.

**Timeout**: `AbortController` with total timeout; set max turns as a safety net.

**Structured output**: Use a `report_result` tool only as the final structured reporting mechanism — not as the agent's primary communication channel.

**Reinforcement**: After tool calls, inject reminders of the overall objective and current state. Armin Ronacher: *"Every time the agent runs a tool you have the opportunity to feed more information back into the loop — remind it about the overall objective and the status of individual tasks."*

**Failure isolation**: Run subtasks that might fail repeatedly in sub-agents. Report only the success plus a brief summary of what didn't work, avoiding context pollution from failed attempts.

**Verification loops**: The single most important production pattern. Encode in system prompt:
1. Execute the action
2. Verify the result (curl health check, run tests, inspect state)
3. If verification fails: diagnose (read logs), attempt fix, re-verify
4. If fix fails after N attempts: report failure with diagnostic context

See [references/production.md](references/production.md) for detailed patterns, observability, and testing guidance.

## Quick Reference: Key Imports

```typescript
// Layer 3 (full SDK)
import { createAgentSession, SessionManager, SettingsManager, AuthStorage, ModelRegistry,
  DefaultResourceLoader, createBashTool, type ToolDefinition, type ExtensionAPI } from "@mariozechner/pi-coding-agent";

// Layer 2 (agent engine)
import { agentLoop } from "@mariozechner/pi-agent-core";

// Layer 1 (LLM interface)
import { getModel, stream, complete, Type, type Context, type Tool } from "@mariozechner/pi-ai";
```

## References

- [references/tool-design.md](references/tool-design.md) — Bash-first philosophy, when to create custom tools, anti-patterns, code examples
- [references/security.md](references/security.md) — Five-layer defense, sandbox comparison (Monty/CodeMode/Docker/E2B), human-in-the-loop
- [references/integration-patterns.md](references/integration-patterns.md) — SDK embedding, RPC subprocess, extension patterns
- [references/production.md](references/production.md) — Verification loops, cost control, reinforcement, failure isolation, observability, testing
