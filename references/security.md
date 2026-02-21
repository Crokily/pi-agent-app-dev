# Security Architecture for Agent Applications

## The Core Problem

Agent applications give LLMs the ability to take real-world actions. LLMs can be manipulated via prompt injection. Therefore, **every action boundary must be enforced in code, not in prompts.**

## Four-Layer Defense Model

### Layer 1: Tool-Level Validation (★★★★★ Strongest)

Hardcoded checks inside each tool's `execute()` function. The LLM cannot bypass TypeScript code regardless of prompt manipulation.

```typescript
const deleteInstanceTool: ToolDefinition = {
  name: "delete_instance",
  // ...
  execute: async (toolCallId, params, signal, onUpdate, ctx) => {
    // Ownership check — cannot be prompt-injected away
    const instance = await db.instance.findUnique({ where: { id: params.instanceId } });
    if (!instance) throw new Error("Instance not found");
    if (instance.userId !== currentUserId) throw new Error("Not your instance");

    // Resource validation
    if (instance.status === "protected") throw new Error("Protected instances cannot be deleted");

    await docker.getContainer(instance.containerId).remove({ force: true });
    await db.instance.delete({ where: { id: params.instanceId } });
    return { content: [{ type: "text", text: `Deleted ${params.instanceId}` }], details: {} };
  },
};
```

### Layer 2: Extension Event Gates (★★★★ Strong)

Intercept tool calls before execution via pi extensions. Useful for cross-cutting concerns.

```typescript
// extensions/security-gate.ts
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    // Block dangerous bash commands
    if (event.toolName === "bash") {
      const cmd = event.input.command || "";
      const blocked = ["rm -rf /", "dd if=", "mkfs", "chmod 777", "curl|sh", "wget|sh"];
      if (blocked.some(b => cmd.includes(b))) {
        return { block: true, reason: `Blocked dangerous command: ${cmd.slice(0, 50)}` };
      }
    }

    // Enforce image allowlist for docker operations
    if (event.toolName === "docker_run") {
      const allowed = ["openclaw:", "nginx:", "node:", "postgres:"];
      if (!allowed.some(p => event.input.image?.startsWith(p))) {
        return { block: true, reason: `Image ${event.input.image} not in allowlist` };
      }
    }

    // Resource limits
    if (event.toolName === "docker_run") {
      if ((event.input.cpuLimit || 1) > 2) return { block: true, reason: "CPU limit > 2" };
      if ((event.input.memoryLimit || 2048) > 4096) return { block: true, reason: "Memory > 4GB" };
    }
  });
}
```

### Layer 3: System Prompt Rules (★★ Weak — guidance only)

System Prompt instructions guide model behavior but are NOT security boundaries. Use them for behavioral guidance only.

```
## Safety Rules (these guide your behavior but are enforced in tools)
- Verify operations before reporting success
- Prefer non-destructive operations when possible
- Always check container ownership before modifications
```

### Layer 4: Infrastructure Isolation (★★★★★ Strongest)

Run agent processes in sandboxed environments:

- **Container isolation**: Agent process runs inside a Docker container with limited capabilities
- **Filesystem restrictions**: Read-only root, writable only in designated directories
- **Network isolation**: Restrict outbound access via iptables/network policies
- **Resource limits**: cgroups for CPU, memory, disk I/O
- **Seccomp/AppArmor**: Block dangerous syscalls

```bash
# Example: run agent in restricted container
docker run --rm \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --memory 2g --cpus 1 \
  -v /data/workspace:/workspace:rw \
  agent-image pi --mode rpc
```

## Bash Safety Patterns

When allowing bash in agent apps, apply defense-in-depth:

```typescript
import { createBashTool } from "@mariozechner/pi-coding-agent";

// 1. Restrict working directory
const safeBash = createBashTool("/data/clawdeploy/user-123");

// 2. Add a spawnHook to restrict environment
const restrictedBash = createBashTool(cwd, {
  spawnHook: ({ command, cwd, env }) => ({
    command,
    cwd,
    env: {
      ...env,
      PATH: "/usr/local/bin:/usr/bin:/bin", // no sbin
      HOME: "/data/clawdeploy/user-123",
    },
  }),
});

// 3. Combine with extension gate for command filtering
```

## Design Principle: Least Privilege Tool Set

For each agent application, define the **minimal tool set** needed:

| Application Type | Recommended Tools |
|-----------------|-------------------|
| Read-only analysis | `readOnlyTools` (read, grep, find, ls) |
| Deployment automation | custom deploy/status tools + restricted bash |
| Data processing | bash (sandboxed) + read + write + custom output tool |
| Infrastructure management | custom tools for each action, NO bash |

Use `tools: []` in `createAgentSession()` to start with zero built-in tools, then add only what's needed via `customTools`.
