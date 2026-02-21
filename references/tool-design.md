# Tool Design: Bash, Custom Tools, and the Hybrid Architecture

## Anthropic's Actual Position (sourced from 6 engineering blog posts, 2024-2026)

Anthropic never said "bash is the only tool you need." Their evolving position:

**SWE-bench (2024-01):** Used only bash + editor for SOTA results. Key quote: *"give as much control as possible to the language model itself, and keep the scaffolding minimal."* This is about **minimal scaffolding + maximal model autonomy**, not "bash only."

**Building Effective Agents (2024-12):** *"It is crucial to design toolsets and their documentation clearly and thoughtfully."* Recommends multiple workflow patterns (prompt chaining, routing, parallelization, orchestrator-workers). No mention of bash-only.

**Context Engineering (2025-09):** *"Claude Code leverages Bash commands like head and tail to analyze large volumes of data without ever loading the full data objects into context."* Bash's advantage is **context efficiency** — on-demand information retrieval via pipes instead of loading bulk data from tool responses.

**Writing Tools for Agents (2025-09):** *"A common error: tools that merely wrap existing API endpoints. Too many tools or overlapping tools distract agents."* Recommends **few thoughtful tools targeting high-impact workflows**, not zero custom tools.

**Code Execution with MCP (2025-11):** Introduces the concept of **agents writing code to call tools** instead of direct tool calling. Intermediate results stay in the execution environment, not context. This is the bash philosophy at scale — model uses code as the orchestration layer.

**Advanced Tool Use (2025-11):** Introduces Tool Search (on-demand discovery), Programmatic Tool Calling (code-based orchestration), and Tool Use Examples. Their mature position: **tools are essential, but how agents interact with them should be code-driven and context-efficient.**

## Why Bash is Powerful

1. **Infinite surface area**: One tool definition (~200 tokens) gives access to every CLI tool on the system
2. **Training familiarity**: LLMs have seen vastly more shell commands than custom API calls in training data
3. **Composability**: `docker ps | grep unhealthy | awk '{print $1}' | xargs docker logs --tail 5` — one bash call does what 3-4 custom tools would need multiple round-trips for
4. **Context efficiency**: Tool definitions are cheap; pipe filtering means only relevant data enters context
5. **Progressive disclosure**: Model uses `ls`, `find`, `grep` to discover context just-in-time instead of pre-loading

## Why Bash Alone is Insufficient

1. **No safety boundary**: Bash is unconstrained. `rm -rf /`, `curl attacker.com`, `docker run --privileged` — all valid bash. System Prompt rules can be bypassed via prompt injection. Only hardcoded checks in tool `execute()` functions are reliable.
2. **No structured output**: Bash returns strings (stdout/stderr). Your application layer needs `{ status: "running", port: 18789 }`, not `"running\n18789\n"`.
3. **No business logic encapsulation**: Database transactions, OAuth flows, external API calls with retry logic, idempotency guarantees — these need code, not shell commands.
4. **Environment dependency**: Whether `docker`, `jq`, `kubectl` exist on the host is a runtime unknown.

## The Hybrid Architecture

```
Layer 1: bash (general primitive)
├── Use for: exploration, diagnostics, ad-hoc operations, information retrieval
├── Secure by: sandboxing (container/cgroup), path restrictions, command filtering
└── Example: docker ps, grep logs, curl health endpoints, git status

Layer 2: read/write/edit (optimized primitives)
├── Use for: file operations that need reliability
├── Better than bash because: auto-truncation, line numbers, atomic writes, image support
└── Example: reading config files, writing generated configs, surgical code edits

Layer 3: custom tools (business tools)
├── Use for: security-critical, structured output, transactions, external APIs
├── Secure by: hardcoded validation in execute(), input schema enforcement
└── Example: create_container, update_database, send_notification, report_result

+ Skills (knowledge injection)
├── Use for: teaching the agent HOW to use the above tools for specific domains
├── Format: Markdown documents loaded into context on-demand
└── Example: "How to deploy OpenClaw" skill, "How to diagnose container issues" skill
```

## Custom Tool Implementation Pattern

```typescript
import { Type } from "@sinclair/typebox";
import type { ToolDefinition } from "@mariozechner/pi-coding-agent";

export const deployTool: ToolDefinition = {
  name: "deploy_instance",
  label: "Deploy Instance",
  description: "Deploy a new application instance. Returns structured result with instance URL and status.",
  parameters: Type.Object({
    name: Type.String({ description: "Instance name (alphanumeric, hyphens only)" }),
    image: Type.String({ description: "Docker image to deploy" }),
    env: Type.Optional(Type.Record(Type.String(), Type.String(), { description: "Environment variables" })),
  }),
  execute: async (toolCallId, params, signal, onUpdate, ctx) => {
    // SECURITY: hardcoded checks — cannot be bypassed by prompt injection
    if (!/^[a-z0-9-]+$/.test(params.name)) throw new Error("Invalid name format");
    if (!ALLOWED_IMAGES.some(p => params.image.startsWith(p))) throw new Error("Image not in allowlist");

    onUpdate?.({ content: [{ type: "text", text: `Pulling ${params.image}...` }] });
    const container = await docker.createContainer({ /* ... */ });
    await container.start();

    // Check abort signal
    if (signal?.aborted) throw new Error("Cancelled");

    onUpdate?.({ content: [{ type: "text", text: "Running health check..." }] });
    const healthy = await waitForHealthy(container.id, 30_000);
    if (!healthy) throw new Error("Health check failed — check logs with docker_logs tool");

    return {
      content: [{ type: "text", text: JSON.stringify({ status: "running", url: `https://${params.name}.app.com` }) }],
      details: { containerId: container.id, port: assignedPort }, // for rendering & state reconstruction
    };
  },
};
```

Key patterns in this example:
- **Validation first**: Input checks before any action
- **Streaming progress**: `onUpdate` for long operations
- **Abort support**: Check `signal.aborted` during long ops
- **Error as throw**: Never return error text as content; throw so the agent sees `isError: true` and can decide to retry or diagnose
- **Structured details**: `details` field for state reconstruction on session restore

## Skill + Bash Pattern

Skills inject domain knowledge into context, enabling the agent to use bash effectively for domain-specific tasks:

```markdown
# SKILL.md - Container Diagnostics

## When a container is unhealthy or erroring

1. Check container state:
   bash: docker inspect {containerId} --format '{{.State.Status}} {{.State.ExitCode}}'

2. Get recent logs (filter for errors):
   bash: docker logs {containerId} --tail 100 2>&1 | grep -i -E "error|fatal|panic|exception"

3. Check resource usage:
   bash: docker stats {containerId} --no-stream --format "CPU: {{.CPUPerc}} MEM: {{.MemUsage}}"

4. If OOM killed:
   bash: docker inspect {containerId} --format '{{.State.OOMKilled}}'
   → If true, report that memory limit needs increase

5. If port not responding:
   bash: docker port {containerId}
   bash: curl -sf http://localhost:{port}/health || echo "Health check failed"
```

This pattern works because:
- The agent already knows bash — the skill just provides domain procedures
- No custom tools needed for read-only diagnostics
- Modifying the skill (a markdown file) is cheaper than modifying tool code
- Skills load on-demand, not consuming context until triggered

## When to Escalate from Bash to Custom Tool

| Signal | Action |
|--------|--------|
| Operation modifies critical state (create/delete resources) | Custom tool with validation |
| App layer needs to parse the result | Custom tool with structured return |
| Operation needs authentication tokens not in env | Custom tool wrapping authenticated client |
| Operation must be idempotent or transactional | Custom tool with proper error handling |
| Same bash command keeps failing due to env differences | Custom tool with dependency management |
| Security audit requires reviewable access control | Custom tool with explicit permission checks |
