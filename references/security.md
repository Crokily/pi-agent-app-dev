# Security Architecture for Agent Applications

## Core Problem

Agent applications give LLMs real-world action capabilities. LLMs can be manipulated via prompt injection. Therefore, **every action boundary must be enforced in code, not in prompts.**

## Five-Layer Defense

### Layer 1: Tool-Level Validation (★★★★★)

Hardcoded checks in `execute()`. The LLM cannot bypass TypeScript code.

```typescript
const deleteTool: ToolDefinition = {
  name: "delete_instance",
  execute: async (toolCallId, params) => {
    const instance = await db.instance.findUnique({ where: { id: params.instanceId } });
    if (!instance) throw new Error("Not found");
    if (instance.userId !== currentUserId) throw new Error("Not your instance");
    if (instance.status === "protected") throw new Error("Protected — cannot delete");
    // ... proceed with deletion
  },
};
```

### Layer 2: Extension Event Gates (★★★★)

Cross-cutting security via pi extensions:

```typescript
export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event) => {
    if (event.toolName === "bash") {
      const cmd = event.input.command || "";
      const blocked = ["rm -rf /", "dd if=", "mkfs", "chmod 777"];
      if (blocked.some(b => cmd.includes(b)))
        return { block: true, reason: `Blocked: ${cmd.slice(0, 50)}` };
    }
  });
}
```

### Layer 3: Infrastructure Isolation (★★★★★)

Run the agent in a restricted container:

```bash
docker run --rm --read-only \
  --tmpfs /tmp:rw,noexec,nosuid \
  --cap-drop ALL --security-opt no-new-privileges \
  --memory 2g --cpus 1 \
  -v /data/workspace:/workspace:rw \
  agent-image pi --mode rpc
```

### Layer 4: Code Execution Sandbox (★★★★★ for code agents)

When agents generate and execute code, sandbox it. Options:

| Technology | Latency | Best For |
|-----------|---------|----------|
| Docker (NetworkMode: none) | ~195ms | Self-hosted, full isolation |
| Pydantic Monty | <1μs | Embedded interpreter, tight control |
| Cloudflare CodeMode | ~ms | Cloudflare-deployed agents |
| E2B / Daytona | ~90ms-1s | Cloud-native, managed, no-ops |

### Layer 5: System Prompt Rules (★★ Weak)

Behavioral guidance only. Never rely on this for security.

## Human-in-the-Loop

For high-risk operations:

```typescript
pi.on("tool_call", async (event, ctx) => {
  if (event.toolName === "delete_instance") {
    const ok = await ctx.ui.confirm("Confirm Deletion", `Delete ${event.input.instanceId}?`);
    if (!ok) return { block: true, reason: "Rejected by operator" };
  }
});
```

## Trust-Tiered Tool Access

```typescript
function createToolsForTrust(level: "admin" | "user" | "public") {
  switch (level) {
    case "admin":  return [createBashTool(cwd), readTool, writeTool, editTool, ...businessTools];
    case "user":   return [sandboxedBash, readTool, ...businessTools];
    case "public": return [...businessTools]; // no bash, no file access
  }
}
```
