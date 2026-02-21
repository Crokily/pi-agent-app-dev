# Security Architecture for Agent Applications

## The Core Problem

Agent applications give LLMs the ability to take real-world actions. LLMs can be manipulated via prompt injection. Therefore, **every action boundary must be enforced in code, not in prompts.**

## Threat Model

| Threat | Vector | Impact |
|--------|--------|--------|
| Prompt injection | Malicious content in user input or fetched data rewrites agent instructions | Agent takes unintended actions with full tool access |
| Tool abuse | LLM calls tools with harmful parameters (e.g., `rm -rf /`) | Data loss, resource exhaustion, credential theft |
| Data exfiltration | Agent sends sensitive data to external endpoints via bash/network tools | Privacy breach, secret leakage |
| Resource exhaustion | Infinite loops, fork bombs, disk-filling operations | Denial of service, cost explosion |
| Privilege escalation | Agent accesses resources outside its intended scope | Lateral movement, unauthorized data access |
| Supply chain attack | Compromised or untrusted LLM generates malicious code intentionally | Full system compromise |

## Five-Layer Defense Model

### Layer 1: Tool-Level Validation (★★★★★ Strongest)

Hardcoded checks inside each tool's `execute()` function. The LLM cannot bypass TypeScript code regardless of prompt manipulation.

```typescript
const deleteInstanceTool: ToolDefinition = {
  name: "delete_instance",
  execute: async (toolCallId, params, signal, onUpdate, ctx) => {
    // Ownership check — cannot be prompt-injected away
    const instance = await db.instance.findUnique({ where: { id: params.instanceId } });
    if (!instance) throw new Error("Instance not found");
    if (instance.userId !== currentUserId) throw new Error("Not your instance");
    if (instance.status === "protected") throw new Error("Protected instances cannot be deleted");

    await docker.getContainer(instance.containerId).remove({ force: true });
    await db.instance.delete({ where: { id: params.instanceId } });
    return { content: [{ type: "text", text: `Deleted ${params.instanceId}` }], details: {} };
  },
};
```

### Layer 2: Extension Event Gates (★★★★ Strong)

Intercept tool calls before execution via pi extensions. Cross-cutting security concerns.

```typescript
// extensions/security-gate.ts
export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash") {
      const cmd = event.input.command || "";
      const blocked = ["rm -rf /", "dd if=", "mkfs", "chmod 777", "curl|sh", "wget|sh"];
      if (blocked.some(b => cmd.includes(b)))
        return { block: true, reason: `Blocked dangerous command: ${cmd.slice(0, 50)}` };
    }
    if (event.toolName === "docker_run") {
      if ((event.input.cpuLimit || 1) > 2) return { block: true, reason: "CPU limit > 2" };
      if ((event.input.memoryLimit || 2048) > 4096) return { block: true, reason: "Memory > 4GB" };
    }
  });
}
```

### Layer 3: Code Execution Sandbox (★★★★★ Strongest for code agents)

When agents execute LLM-generated code (bash commands, scripts), run it in an isolated sandbox. This is the **industry consensus** for production agent security. See detailed sandbox comparison below.

### Layer 4: System Prompt Rules (★★ Weak — guidance only)

Behavioral guidance that can be bypassed by prompt injection. Never rely on this as a security boundary.

### Layer 5: Infrastructure Isolation (★★★★★ Strongest)

OS-level isolation for the agent process itself:

```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --memory 2g --cpus 1 \
  -v /data/workspace:/workspace:rw \
  agent-image pi --mode rpc
```

---

## Code Execution Sandboxing — The Industry Landscape (2025-2026)

The core insight driving the industry: **LLMs are better at writing code than calling tools** (see Anthropic's "Programmatic Tool Calling", Cloudflare's "Code Mode", HuggingFace's SmolAgents research). This means agents increasingly *generate and execute code*, making sandboxing critical.

### Why Sandboxing Matters

When agents write code (bash commands, Python scripts, TypeScript) to accomplish tasks:
- The generated code runs with the agent process's full permissions
- Prompt injection can cause arbitrary code execution
- Even well-intentioned LLMs sometimes generate harmful commands by mistake
- Resource exhaustion (infinite loops, disk-filling) is always possible

### Sandbox Technology Comparison

Based on Pydantic/Monty's comprehensive benchmarks (2026) and cross-industry analysis:

| Technology | Security | Start Latency | Language Completeness | Setup Complexity | Best For |
|-----------|----------|---------------|----------------------|------------------|----------|
| **Pydantic Monty** | Strict (no fs/net/env by default) | **<1μs** | Partial Python | `pip install` | Embedded code execution in agent loop |
| **Cloudflare CodeMode** | Strict (Workers isolate) | ~ms | JavaScript/TypeScript | Cloudflare Workers | Cloudflare-deployed agents |
| **Docker** | Good (process isolation) | ~195ms | Full (any language) | Docker daemon | Self-hosted, full-stack isolation |
| **E2B** | Strict (managed cloud) | ~1s cold | Full (any language) | API key | Cloud-native, no-ops sandbox |
| **Daytona** | Strict (managed cloud) | <90ms | Full (any language) | API key | Fast cloud sandbox |
| **Modal** | Strict (managed cloud) | ~200ms | Full Python | API key | Python-heavy workloads |
| **Blaxel** | Strict (hibernating VMs) | <25ms | Full (any language) | API key | Low-latency cloud sandbox |
| **WASI/Wasmer** | Strict (WASM boundary) | ~66ms | Partial | Intermediate | Portable sandboxing |
| **Pyodide** | Poor (browser sandbox) | ~2800ms | Full Python (WASM) | Intermediate | Browser-only scenarios |
| **`exec()`/YOLO** | **None** | ~0.1ms | Full | None | **Never in production** |

Source: Pydantic Monty comparison table + E2B/Daytona/Cloudflare docs.

### Approach 1: Embedded Interpreter (Monty-style)

**Concept:** A minimal, purpose-built interpreter that executes LLM-generated code with zero host access by default. The agent can only call functions you explicitly expose.

**Pydantic Monty** (Rust-based, Python/JS/Rust bindings):
```python
import pydantic_monty

m = pydantic_monty.Monty(
    code=llm_generated_code,
    inputs=['user_data'],
    external_functions=['query_database', 'send_email'],  # only these are callable
)

result = await pydantic_monty.run_monty_async(
    m,
    inputs={'user_data': sanitized_input},
    external_functions={
        'query_database': my_safe_db_query,
        'send_email': my_safe_email_sender,
    },
)
```

**Key security properties:**
- No filesystem access (unless you provide a function for it)
- No network access (unless you provide a function for it)
- No env variable access
- Execution time limits, memory limits, operation count limits
- Snapshotable: pause execution, serialize state, resume later
- Startup: <1μs (vs ~195ms Docker, ~1s cloud sandbox)

**HuggingFace SmolAgents** `LocalPythonExecutor` (AST-based):
- Re-implemented Python interpreter that executes AST node-by-node
- Import allowlist (default: math, statistics, collections, datetime, etc.)
- Blocked submodule access (e.g., `random._os.system()` → blocked)
- Operation count cap (prevents infinite loops)
- Warning: not fully escape-proof; for high-security, use Docker/E2B

### Approach 2: Isolated Worker/Container (Cloudflare CodeMode style)

**Concept:** LLM writes code → code runs in an isolated process/container with no network access. Tool calls are proxied back to the host via RPC.

**Cloudflare CodeMode** architecture:
```
┌─────────────┐        ┌──────────────────────────────────┐
│ Host Worker  │  RPC   │ Isolated Worker (sandbox)        │
│             │◄──────►│                                  │
│ Tool fns    │        │ LLM-generated code runs here     │
│ (real impl) │        │ codemode.myTool() → RPC to host  │
│             │        │ fetch() blocked (globalOutbound:  │
│             │        │    null)                          │
└─────────────┘        └──────────────────────────────────┘
```

```typescript
import { createCodeTool } from "@cloudflare/codemode/ai";
import { DynamicWorkerExecutor } from "@cloudflare/codemode";

const executor = new DynamicWorkerExecutor({
  loader: env.LOADER,
  globalOutbound: null, // ← network completely blocked
  timeout: 30000,
});

// Tools are exposed as typed API; LLM writes code against them
const codemode = createCodeTool({ tools: myTools, executor });
```

**Key security properties:**
- Network blocked at runtime level (`globalOutbound: null`)
- Code runs in separate V8 isolate
- Tool calls go through RPC dispatcher → only exposed functions callable
- Console output captured and returned
- Custom `Executor` interface — implement for any sandbox backend

### Approach 3: Container/Cloud Sandbox (E2B/Daytona/Docker)

**Concept:** Full OS-level isolation. Agent-generated code runs in a disposable container with its own filesystem, network namespace, and process tree.

**E2B:**
```typescript
import { Sandbox } from '@e2b/code-interpreter';

const sandbox = await Sandbox.create();
const result = await sandbox.runCode(llmGeneratedCode);
// Sandbox is fully isolated — separate filesystem, network
await sandbox.close();
```

**Daytona:**
```typescript
import { Daytona } from '@daytonaio/sdk';

const daytona = new Daytona({ apiKey: process.env.DAYTONA_KEY });
const sandbox = await daytona.create({ language: "python" });
const response = await sandbox.process.codeRun(llmGeneratedCode);
await daytona.delete(sandbox);
```

**Docker (self-hosted):**
```typescript
const container = await docker.createContainer({
  Image: "python:3.12-slim",
  Cmd: ["python", "-c", llmGeneratedCode],
  HostConfig: {
    ReadonlyRootfs: true,
    Memory: 256 * 1024 * 1024,  // 256MB
    NanoCpus: 500_000_000,       // 0.5 CPU
    NetworkMode: "none",          // no network
    CapDrop: ["ALL"],
    SecurityOpt: ["no-new-privileges"],
  },
});
await container.start();
const output = await container.wait();
await container.remove();
```

### Choosing the Right Sandbox for Pi-Mono Apps

| Scenario | Recommended Sandbox | Why |
|----------|-------------------|-----|
| Agent bash tool in web API | Docker container + restricted bash | Process isolation, self-hosted, no external dependency |
| Agent generates Python/JS for data processing | Monty or Cloudflare CodeMode | Sub-ms startup, tight control over exposed functions |
| Multi-tenant deployment platform | E2B or Daytona | Per-user isolation, managed infrastructure, no-ops |
| Edge/serverless agent | Cloudflare Workers isolate | V8 isolate-level isolation, global deployment |
| Development/testing | Docker or SmolAgents LocalPythonExecutor | Easy setup, good enough for dev |
| Maximum security (regulated industry) | Cloud sandbox (E2B/Daytona) + no bash tool | Full isolation, audit trail, disposable |

### Applying to Pi-Mono: Practical Patterns

**Pattern A: Sandboxed bash tool**

Replace the default bash tool with one that executes in a container:

```typescript
import { createBashTool } from "@mariozechner/pi-coding-agent";

const sandboxedBash = createBashTool(cwd, {
  operations: {
    exec: async (command, options) => {
      // Run in Docker container instead of host
      const result = await dockerExec("sandbox-container", ["bash", "-c", command], {
        timeout: options?.timeout || 30000,
      });
      return { stdout: result.stdout, stderr: result.stderr, exitCode: result.exitCode };
    },
  },
});
```

**Pattern B: Code execution as a tool**

Instead of giving bash, give a sandboxed code execution tool:

```typescript
const codeExecTool: ToolDefinition = {
  name: "execute_code",
  label: "Execute Code",
  description: "Execute Python or JavaScript code in a secure sandbox. No filesystem or network access.",
  parameters: Type.Object({
    language: Type.String({ enum: ["python", "javascript"] }),
    code: Type.String({ description: "Code to execute" }),
  }),
  execute: async (toolCallId, params) => {
    const sandbox = await Sandbox.create();
    try {
      const result = await sandbox.runCode(params.code);
      return { content: [{ type: "text", text: result.text }], details: { exitCode: result.exitCode } };
    } finally {
      await sandbox.close();
    }
  },
};
```

**Pattern C: Tiered security based on trust level**

```typescript
function createToolsForTrustLevel(level: "high" | "medium" | "low") {
  switch (level) {
    case "high":   // Internal admin
      return [createBashTool(cwd), readTool, writeTool, editTool, ...businessTools];
    case "medium": // Authenticated user
      return [sandboxedBash, readTool, ...businessTools]; // bash in container, no write
    case "low":    // Public/untrusted
      return [...businessTools]; // custom tools only, no bash, no file access
  }
}
```

---

## Human-in-the-Loop Patterns

For high-risk operations, require human approval:

```typescript
// Pi extension approach
pi.on("tool_call", async (event, ctx) => {
  if (event.toolName === "delete_instance") {
    const ok = await ctx.ui.confirm(
      "Confirm Deletion",
      `Delete instance ${event.input.instanceId}?`,
      { timeout: 30000 } // auto-cancel after 30s
    );
    if (!ok) return { block: true, reason: "Rejected by operator" };
  }
});

// Programmatic approach (for headless/API agents)
const approvalTool: ToolDefinition = {
  name: "request_approval",
  description: "Request human approval for a high-risk action. Returns approved or denied.",
  parameters: Type.Object({
    action: Type.String(),
    details: Type.String(),
    risk_level: Type.String({ enum: ["medium", "high", "critical"] }),
  }),
  execute: async (toolCallId, params) => {
    // Send to approval queue (Slack, webhook, internal tool)
    const approved = await approvalQueue.requestAndWait(params, { timeout: 300_000 });
    if (!approved) throw new Error("Action denied by human reviewer");
    return { content: [{ type: "text", text: "Approved" }], details: {} };
  },
};
```

System Prompt instruction: *"For any destructive or irreversible action, call request_approval first."*

---

## Key References

- **Cloudflare CodeMode**: https://github.com/cloudflare/agents/tree/main/packages/codemode — Isolated Worker sandbox for code execution
- **Pydantic Monty**: https://github.com/pydantic/monty — Rust-based minimal Python interpreter, <1μs startup
- **E2B**: https://github.com/e2b-dev/e2b — Cloud sandbox infrastructure
- **Daytona**: https://github.com/daytonaio/daytona — Sub-90ms cloud sandbox
- **HuggingFace SmolAgents**: https://huggingface.co/docs/smolagents/en/tutorials/secure_code_execution — Code agent security guide with multi-sandbox support
- **Anthropic "Building Effective Agents"**: https://www.anthropic.com/engineering/building-effective-agents — Agent architecture patterns
- **Anthropic "Code Execution with MCP"**: https://www.anthropic.com/engineering/code-execution-with-mcp — Code-driven tool orchestration
- **Anthropic "Programmatic Tool Calling"**: https://www.anthropic.com/engineering/advanced-tool-use — Code execution in sandbox, token-efficient
