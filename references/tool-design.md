# Tool Design: Bash-First, Custom Tools Only When Justified

## The Industry Consensus (2025-2026)

The trajectory is clear across every major agent framework and production deployment:

- **Vercel** deleted 17 specialized tools, replaced with 1 bash tool → 100% success, 3.5x faster, 37% fewer tokens
- **Pi** ships 4 tools (read/write/edit/bash) with <1000 tokens of system prompt → top Terminal-Bench scores
- **Claude Code** uses bash + read + write + edit as its core toolset
- **Manus** uses fewer than 20 atomic functions, offloads real work to generated scripts in a sandbox
- **Stripe Minions** run in devboxes with the same developer tooling humans use — bash, git, linters
- **CodeAct** (ICML 2024) found code-based actions achieve up to 20% higher success rates vs JSON tool calls
- **Terminus 2** gives models just a tmux session (no tools at all) and still competes with sophisticated agent harnesses

## Why Bash Is the Primary Tool

**Infinite surface area**: One tool definition (~200 tokens) gives access to every CLI tool on the system — docker, curl, git, jq, grep, awk, sed, python, node, and anything else installed.

**Training familiarity**: LLMs have been trained on vast amounts of shell usage. They know bash better than any custom tool API.

**Composability**: `docker ps | grep unhealthy | awk '{print $1}' | xargs docker logs --tail 5` — one bash call replaces 3-4 custom tools and multiple inference round trips.

**Context efficiency**: Each tool call requires an additional LLM inference. Bash lets the agent chain operations in a single invocation, saving tokens and latency.

**Progressive disclosure**: The agent uses `ls`, `find`, `grep`, `head` to discover context just-in-time instead of needing it pre-loaded.

## When Bash Is Insufficient

1. **No safety boundary**: Bash is unconstrained. `rm -rf /`, `curl attacker.com` are all valid. System Prompt rules can be bypassed. Only hardcoded checks in `execute()` are reliable.
2. **No structured output**: Bash returns strings. Your app layer may need `{ status: "running", port: 18789 }`.
3. **No transactional guarantees**: Database transactions, OAuth flows, idempotency — these need code.

## Decision: Custom Tool or Bash?

```
Does the operation require hardcoded security checks?  → Custom tool
Does your app layer need typed JSON output?             → Custom tool
Does it involve DB transactions or external API auth?   → Custom tool
Everything else                                         → Bash
```

Most applications need **0 to 3 custom tools** plus bash. If you have more than 5, reconsider.

## Custom Tool Patterns

### Pattern: Security-Critical Tool with Verification

```typescript
const deployTool: ToolDefinition = {
  name: "deploy_instance",
  label: "Deploy Instance",
  description: "Deploy a new instance. Validates input, creates container, verifies health.",
  parameters: Type.Object({
    name: Type.String({ description: "Instance name (alphanumeric, hyphens)" }),
    image: Type.String({ description: "Docker image" }),
    userId: Type.String({ description: "Owner user ID" }),
  }),
  execute: async (toolCallId, params, signal, onUpdate) => {
    // SECURITY: hardcoded — cannot be bypassed by prompt injection
    if (!/^[a-z0-9-]+$/.test(params.name)) throw new Error("Invalid name");
    if (!params.userId.startsWith("user_")) throw new Error("Invalid userId");

    // Execute
    const container = await docker.createContainer({ /* ... */ });
    await container.start();

    // VERIFY: tool itself checks the result — don't leave this to the agent
    const healthy = await waitForHealthy(container.id, 30_000);
    if (!healthy) {
      const logs = await getContainerLogs(container.id, 50);
      throw new Error(`Health check failed. Logs:\n${logs}`);
    }

    return {
      content: [{ type: "text", text: JSON.stringify({ status: "running", port }) }],
      details: { containerId: container.id, port, status: "running" },
    };
  },
};
```

Key patterns:
- **Validation first**: Check all inputs before any side effect
- **Verification built-in**: The tool checks its own result
- **Rich errors on failure**: Include diagnostic info so the agent can reason about what went wrong
- **Structured details**: `details` field for app-layer consumption

### Pattern: Minimal Report Tool

```typescript
const reportTool: ToolDefinition = {
  name: "report_result",
  label: "Report Result",
  description: "Call when the entire task is complete. Reports structured result to the app layer.",
  parameters: Type.Object({
    success: Type.Boolean(),
    summary: Type.String({ description: "What was done" }),
    data: Type.Optional(Type.Record(Type.String(), Type.Any())),
  }),
  execute: async (toolCallId, params) => {
    await db.log.create({ data: { ...params, timestamp: new Date() } });
    return { content: [{ type: "text", text: "Recorded." }], details: params };
  },
};
```

This is a **terminal tool** — the agent calls it as the last action. It should not be the agent's primary way of doing work.

## Anti-Patterns

### ❌ Tool as Function Wrapper (The "Message Passer")

```typescript
// BAD: This tool does exactly what the old API route did, with zero added intelligence
const instanceCreateTool = {
  name: "instance_create",
  execute: async (id, params) => {
    const instance = await prisma.instance.create({ data: params });
    await createStorage(instance.id);
    await createContainer(instance.id);
    await prisma.instance.update({ where: { id: instance.id }, data: { status: "running" } });
    return { content: [{ type: "text", text: JSON.stringify(instance) }] };
  },
};
```

Problems:
- No verification that the instance actually works
- No error recovery
- The agent is just a relay — it adds latency and unreliability without adding intelligence
- Deterministic params are passed through LLM natural language (introducing errors)

### ❌ Tool Explosion

```typescript
// BAD: 7 tools that are basically CRUD wrappers
const tools = [instanceCreate, instanceStart, instanceStop, instanceDelete,
               instanceUpdate, nginxSync, reportResult];
```

Each tool definition costs tokens in every LLM call. 7 tools × ~300 tokens = 2100 tokens of context consumed before the agent even starts working. With bash + 1 report tool, total tool definitions fit in ~400 tokens.

### ❌ Passing Deterministic Data Through Natural Language

```typescript
// BAD: API creates a DB record, then tells the agent about it in natural language
const userMessage = `Execute task: instance_create\nUser: ${userId}\nParameters: ${JSON.stringify(params)}`;
// Agent has to "understand" this, then pass the params to a tool — introducing lossy translation
```

If data is deterministic (userId, instanceId), pass it programmatically — as tool arguments, environment variables, or file state. Natural language is for goals and intent, not structured parameters.

## Skill + Bash: Teaching the Agent Domain Knowledge

Instead of encoding procedures in tools, encode them in Skills (markdown) and let the agent execute via bash:

```markdown
# Skill: Container Lifecycle

## Creating an instance
1. Generate config: write openclaw.yaml and .env to /data/{instanceId}/
2. Create container: docker run -d --name {instanceId} -v /data/{instanceId}:/app/config ...
3. Verify health: curl -sf http://localhost:{port}/health (retry 3 times, 5s apart)
4. Update nginx: regenerate /etc/nginx/conf.d/instances.conf and sudo nginx -s reload
5. If health check fails: docker logs {instanceId} --tail 50, diagnose, fix, retry

## Diagnosing a failed instance
1. Check container state: docker inspect {id} --format '{{.State.Status}} {{.State.ExitCode}}'
2. Check logs: docker logs {id} --tail 100 2>&1 | grep -i -E "error|fatal|panic"
3. Check resources: docker stats {id} --no-stream
4. If OOM: docker inspect {id} --format '{{.State.OOMKilled}}'
```

This pattern works because:
- The agent already knows bash — the skill provides domain-specific procedures
- Modifying a markdown file is cheaper than modifying tool code
- Skills load on-demand, not consuming context until needed
- The agent can adapt procedures to unexpected situations (something rigid tools cannot do)
