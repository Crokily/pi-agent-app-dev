# Production Patterns

## Cost Control

```typescript
let totalCost = 0;
const MAX_COST = 0.50; // USD per request

session.subscribe((event) => {
  if (event.type === "message_end" && event.message.role === "assistant") {
    if (event.message.usage?.cost) {
      totalCost += event.message.usage.cost.total;
      if (totalCost > MAX_COST) session.abort();
    }
  }
});
```

**Model selection by complexity:**

```typescript
function selectModel(intent: string) {
  const isComplex = intent.length > 200
    || /diagnos|migrat|refactor|debug|troubleshoot/i.test(intent);
  return isComplex
    ? getModel("anthropic", "claude-sonnet-4-20250514")   // strong + thinking
    : getModel("anthropic", "claude-3-5-haiku-20241022");  // fast + cheap
}
```

## Timeout and Loop Protection

```typescript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5 * 60_000); // 5 min total

const settingsManager = SettingsManager.inMemory({
  retry: { enabled: true, maxRetries: 3 },
  compaction: { enabled: false }, // short tasks don't need compaction
});

try {
  await session.prompt(intent);
} finally {
  clearTimeout(timeout);
}
```

## Structured Output via "Report" Tool

Force the agent to produce structured results your app can parse:

```typescript
const reportTool: ToolDefinition = {
  name: "report_result",
  label: "Report Result",
  description: "MUST call when task completes. Report structured result.",
  parameters: Type.Object({
    success: Type.Boolean(),
    action: Type.String({ description: "What was done" }),
    data: Type.Optional(Type.Record(Type.String(), Type.Any())),
    errors: Type.Optional(Type.Array(Type.String())),
  }),
  execute: async (toolCallId, params) => {
    // Write to DB, send webhook, etc.
    await db.executionLog.create({ data: { ...params, timestamp: new Date() } });
    return { content: [{ type: "text", text: "Result recorded." }], details: params };
  },
};
```

In System Prompt: *"When the task is complete (success or failure), always call report_result with a structured summary."*

## Context Management

### Serialization for Cross-Request Memory

```typescript
import { type Context } from "@mariozechner/pi-ai";

// Save after execution
const contextJson = JSON.stringify(agentContext);
await redis.set(`agent:ctx:${userId}`, contextJson, "EX", 3600);

// Restore and continue
const saved = await redis.get(`agent:ctx:${userId}`);
if (saved) {
  const ctx: Context = JSON.parse(saved);
  ctx.messages.push({ role: "user", content: newIntent });
  const response = await complete(model, ctx);
}
```

Note: Pi-ai contexts are fully serializable including images (base64). Works across providers — you can save context from Claude and continue with GPT-4.

### Compaction Settings

For long-running tasks, configure compaction:

```typescript
const settingsManager = SettingsManager.inMemory({
  compaction: {
    enabled: true,
    // keepRecentTokens: 20000,  // how many recent tokens to keep (default 20k)
    // reserveTokens: 16384,     // buffer for LLM response (default 16k)
  },
});
```

Compaction auto-triggers when `contextTokens > contextWindow - reserveTokens`. The SDK summarizes old messages and keeps only recent ones + summary.

### Manual Compaction

```typescript
// Via session API
const result = await session.compact("Focus on deployment decisions and errors");

// Via extension
ctx.compact({ customInstructions: "Preserve all error messages and container IDs" });
```

## Observability

For comprehensive observability patterns including structured tracing, cost monitoring, error classification, performance profiling, OpenTelemetry integration, production alerting, session replay, and eval-driven quality monitoring, see **[observability.md](observability.md)**.

Basic event logging for quick setup:

```typescript
session.subscribe((event) => {
  const base = { sessionId: session.sessionId, ts: Date.now() };

  switch (event.type) {
    case "agent_start":
      logger.info({ ...base, event: "agent_start" });
      break;
    case "tool_execution_start":
      logger.info({ ...base, event: "tool_start", tool: event.toolName, args: event.args });
      break;
    case "tool_execution_end":
      logger.info({ ...base, event: "tool_end", tool: event.toolName, error: event.isError });
      break;
    case "agent_end":
      logger.info({ ...base, event: "agent_end", messageCount: event.messages.length, totalCost });
      break;
  }
});
```

## Testing Agent Applications

### Unit Test Individual Tools

Tools are regular async functions — test them independently:

```typescript
import { describe, it, expect } from "vitest";

describe("deployTool", () => {
  it("rejects unauthorized users", async () => {
    await expect(
      deployTool.execute("id", { name: "test", image: "nginx" }, undefined, undefined, mockCtx)
    ).rejects.toThrow("Unauthorized");
  });

  it("rejects disallowed images", async () => {
    await expect(
      deployTool.execute("id", { name: "test", image: "malicious:latest" }, undefined, undefined, mockCtx)
    ).rejects.toThrow("not in allowlist");
  });
});
```

### Integration Test Agent Behavior

Use `createAgentSession` with in-memory settings to test full agent flows:

```typescript
it("deploys instance on user request", async () => {
  const { session } = await createAgentSession({
    model: getModel("anthropic", "claude-sonnet-4-20250514"),
    customTools: [deployTool, statusTool],
    tools: [],
    sessionManager: SessionManager.inMemory(),
    settingsManager: SettingsManager.inMemory({ compaction: { enabled: false } }),
  });

  const events: any[] = [];
  session.subscribe(e => { if (e.type === "tool_execution_end") events.push(e); });

  await session.prompt("Deploy a new nginx instance named test-app");

  expect(events.some(e => e.toolName === "deploy_instance" && !e.isError)).toBe(true);
});
```

### Eval-Driven Tool Refinement

Following Anthropic's "Writing Tools for Agents" methodology:

1. Create diverse, realistic test prompts (not trivial ones)
2. Run agent with your tools, collect full transcripts
3. Analyze: which tools were called? Were parameters correct? Did the agent get stuck?
4. Iterate on tool descriptions, parameter names, and error messages
5. Maintain a held-out test set to avoid overfitting

## Multi-Model Handoff

Pi-ai supports seamless cross-provider context handoffs:

```typescript
// Start with fast model for initial analysis
const haiku = getModel("anthropic", "claude-3-5-haiku-20241022");
const response1 = await complete(haiku, context);
context.messages.push(response1);

// Escalate to strong model for complex decision
context.messages.push({ role: "user", content: "This needs deeper analysis" });
const sonnet = getModel("anthropic", "claude-sonnet-4-20250514");
const response2 = await complete(sonnet, context, { thinkingEnabled: true, thinkingBudgetTokens: 4096 });
```

Thinking blocks from one provider are automatically converted to `<thinking>` tagged text for other providers.
