# Integration Patterns

## Pattern A: SDK Embedded Agent (in-process)

Best for: Next.js API routes, Node.js services, CLI tools, backend workers.

```typescript
import { createAgentSession, SessionManager, DefaultResourceLoader } from "@mariozechner/pi-coding-agent";

export async function POST(request: Request) {
  const { intent, userId } = await request.json();

  const loader = new DefaultResourceLoader({
    systemPromptOverride: () => buildPromptForUser(userId),
  });
  await loader.reload();

  const { session } = await createAgentSession({
    model: getModel("anthropic", "claude-sonnet-4-20250514"),
    resourceLoader: loader,
    customTools: createToolsForUser(userId),
    tools: [],
    sessionManager: SessionManager.inMemory(),
  });

  const toolResults: any[] = [];
  session.subscribe((event) => {
    if (event.type === "tool_execution_end")
      toolResults.push({ tool: event.toolName, ok: !event.isError });
  });

  await session.prompt(intent);

  const last = session.messages.filter(m => m.role === "assistant").at(-1);
  const text = last?.content?.filter((b: any) => b.type === "text").map((b: any) => b.text).join("\n") ?? "";

  return Response.json({ success: toolResults.every(t => t.ok), response: text, tools: toolResults });
}
```

## Pattern B: RPC Subprocess (process isolation)

Best for: Multi-language integration, security isolation, long-running agents.

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

  async prompt(message: string) { this.send({ type: "prompt", message }); }
  async steer(message: string) { this.send({ type: "prompt", message, streamingBehavior: "steer" }); }
  private send(cmd: any) { this.proc.stdin.write(JSON.stringify(cmd) + "\n"); }
  private handleEvent(event: any) { /* process events */ }
  dispose() { this.proc.kill("SIGTERM"); }
}
```

### RPC Commands

```json
{"type": "prompt", "message": "Deploy the app"}
{"type": "abort"}
{"type": "get_session_state"}
{"type": "set_model", "model": "anthropic/claude-sonnet-4-20250514"}
```

## Pattern C: Extension-Based Agent App

For building capabilities as reusable pi extensions:

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";

export default function (pi: ExtensionAPI) {
  // Inject runtime context before each agent turn
  pi.on("before_agent_start", async (event, ctx) => {
    const state = await getSystemStatus();
    return {
      systemPrompt: event.systemPrompt + `\n\nCurrent system state:\n${JSON.stringify(state)}`,
    };
  });

  // Register security-critical tool
  pi.registerTool({
    name: "deploy",
    label: "Deploy",
    description: "Deploy an application instance",
    parameters: Type.Object({ name: Type.String(), image: Type.String() }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      // ... deployment with verification
    },
  });

  // Permission gate
  pi.on("tool_call", async (event) => {
    if (event.toolName === "bash" && event.input.command?.includes("docker rm")) {
      return { block: true, reason: "Use the delete tool instead" };
    }
  });
}
```

## Crossing the Service Boundary

When your agent needs to be accessible over a network (not just in-process), you need additional layers:

| Layer | What It Does | Options |
|-------|-------------|---------|
| **Transport** | How client talks to agent | HTTP request/response → chunked streaming → SSE → WebSocket |
| **Routing** | How messages reach the right session | Session ID registry, Cloudflare Durable Objects |
| **Persistence** | How state survives restarts | Filesystem, SQLite, R2/S3, DB |
| **Lifecycle** | What happens when client disconnects | Background process, container, serverless |

Pick the simplest transport that fits: HTTP for job agents, SSE for streaming with reconnection, WebSocket for bidirectional real-time interaction.

Not every use case needs all layers. A simple job agent needs only HTTP + filesystem persistence.
