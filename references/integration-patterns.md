# Integration Patterns

## Pattern A: SDK Embedded Agent (in-process)

Best for: Next.js API routes, Node.js services, CLI tools, backend workers.

### Stateless Request-Response

Each request creates a fresh session, executes, and returns:

```typescript
// app/api/agent/route.ts
import { createAgentSession, SessionManager, AuthStorage, ModelRegistry, DefaultResourceLoader } from "@mariozechner/pi-coding-agent";

const authStorage = AuthStorage.create("/app/config/auth.json");
const modelRegistry = new ModelRegistry(authStorage);

export async function POST(request: Request) {
  const { intent, userId } = await request.json();

  const loader = new DefaultResourceLoader({
    systemPromptOverride: () => buildPromptForUser(userId),
  });
  await loader.reload();

  const { session } = await createAgentSession({
    model: getModel("anthropic", "claude-sonnet-4-20250514"),
    thinkingLevel: "medium",
    resourceLoader: loader,
    customTools: createToolsForUser(userId),
    tools: [], // no built-in tools
    sessionManager: SessionManager.inMemory(),
    settingsManager: SettingsManager.inMemory({ compaction: { enabled: false } }),
    authStorage,
    modelRegistry,
  });

  // Collect structured results
  const toolResults: any[] = [];
  session.subscribe((event) => {
    if (event.type === "tool_execution_end") {
      toolResults.push({ tool: event.toolName, ok: !event.isError, ts: Date.now() });
    }
  });

  await session.prompt(intent);

  // Extract final response
  const last = session.messages.filter(m => m.role === "assistant").at(-1);
  const text = last?.content?.filter((b: any) => b.type === "text").map((b: any) => b.text).join("\n") ?? "";

  return Response.json({ success: toolResults.every(t => t.ok), response: text, tools: toolResults });
}
```

### With Persistent Session (stateful)

For ongoing interactions or resumable tasks:

```typescript
// Create persistent session
const { session } = await createAgentSession({
  sessionManager: SessionManager.create(process.cwd()),
});

// Continue most recent session
const { session } = await createAgentSession({
  sessionManager: SessionManager.continueRecent(process.cwd()),
});

// Open specific session
const { session } = await createAgentSession({
  sessionManager: SessionManager.open("/path/to/session.jsonl"),
});
```

## Pattern B: RPC Subprocess (process isolation)

Best for: Multi-language integration, security isolation, long-running agents. This is how OpenClaw integrates.

### Spawning and Communicating

```typescript
import { spawn } from "child_process";

class AgentSubprocess {
  private proc: ChildProcess;
  private buffer = "";

  constructor(opts: { cwd: string; extensions?: string[] }) {
    const args = ["--mode", "rpc", "--no-session"];
    for (const ext of opts.extensions ?? []) args.push("-e", ext);

    this.proc = spawn("pi", args, {
      cwd: opts.cwd,
      stdio: ["pipe", "pipe", "pipe"],
      env: { ...process.env },
    });

    this.proc.stdout.on("data", (chunk) => {
      this.buffer += chunk.toString();
      let nl;
      while ((nl = this.buffer.indexOf("\n")) !== -1) {
        this.handleEvent(JSON.parse(this.buffer.slice(0, nl)));
        this.buffer = this.buffer.slice(nl + 1);
      }
    });
  }

  async prompt(message: string): Promise<void> {
    this.send({ type: "prompt", message });
  }

  // During streaming, queue messages
  async steer(message: string): Promise<void> {
    this.send({ type: "prompt", message, streamingBehavior: "steer" });
  }

  async followUp(message: string): Promise<void> {
    this.send({ type: "prompt", message, streamingBehavior: "followUp" });
  }

  private send(cmd: any) {
    this.proc.stdin.write(JSON.stringify(cmd) + "\n");
  }

  private handleEvent(event: any) {
    // event.type: "agent_start", "tool_execution_start", "tool_execution_end",
    //             "message_update", "agent_end", "response", etc.
  }

  dispose() { this.proc.kill("SIGTERM"); }
}
```

### Key RPC Commands

```json
// Send prompt
{"type": "prompt", "message": "Deploy the app"}

// Abort current operation
{"type": "abort"}

// Get session state
{"type": "get_session_state"}

// Set model
{"type": "set_model", "model": "anthropic/claude-sonnet-4-20250514"}
```

## Pattern C: Extension-Based Agent App

For building capabilities as reusable pi extensions:

```typescript
// extensions/deploy-agent.ts
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";

export default function (pi: ExtensionAPI) {
  // Inject runtime context before each agent turn
  pi.on("before_agent_start", async (event, ctx) => {
    const systemState = await getSystemStatus();
    return {
      message: { customType: "deploy-context", content: JSON.stringify(systemState), display: false },
      systemPrompt: event.systemPrompt + `\n\nCurrent system state:\n${JSON.stringify(systemState)}`,
    };
  });

  // Register business tools
  pi.registerTool({
    name: "deploy",
    label: "Deploy",
    description: "Deploy an application instance",
    parameters: Type.Object({ name: Type.String(), image: Type.String() }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      // Tool implementation with access to ctx for user interaction
      const confirmed = await ctx.ui.confirm("Deploy?", `Deploy ${params.name} with ${params.image}?`);
      if (!confirmed) throw new Error("Cancelled by user");
      // ... deployment logic
      return { content: [{ type: "text", text: "Deployed" }], details: {} };
    },
  });

  // Register operator command
  pi.registerCommand("deploy-status", {
    description: "Show deployment status dashboard",
    handler: async (args, ctx) => {
      const status = await getAllInstanceStatus();
      ctx.ui.notify(JSON.stringify(status, null, 2), "info");
    },
  });

  // Permission gate
  pi.on("tool_call", async (event) => {
    if (event.toolName === "bash" && event.input.command?.includes("docker rm")) {
      return { block: true, reason: "Use the delete_instance tool instead" };
    }
  });
}
```

## OpenClaw: Real-World Reference Architecture

OpenClaw is the largest real-world pi-mono SDK integration. Key architecture decisions:

1. **Gateway pattern**: A Node.js gateway process manages channels (WhatsApp, Telegram, Discord, etc.) and routes messages to the Pi agent running in RPC mode as a subprocess.

2. **Multi-agent routing**: Different channels/users can be routed to isolated agent instances with separate sessions and workspaces.

3. **Tool design**: Custom tools for channel-specific actions (sending messages, managing files, canvas rendering), bash for general exploration.

4. **Session persistence**: Each conversation maintains session state, enabling the agent to recall context across interactions.

5. **Skill-based capabilities**: Domain skills (browsing, cron jobs, session management) are loaded as pi skills, extending the agent's knowledge without bloating the tool set.

Source: https://github.com/openclaw/openclaw
