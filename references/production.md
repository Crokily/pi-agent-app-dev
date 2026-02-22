# Production Patterns

## Verification Loops: The Core Value Proposition

The single most important pattern for agent-driven applications. An agent that only executes without verifying is just a slow, unreliable function call. **Verification is what makes the agent worth having.**

### Encode Verification in System Prompt

```
After every state-changing operation, verify the result:

1. Execute the action
2. Verify: check that the expected outcome actually occurred
   - Created a container? → docker inspect + curl health endpoint
   - Updated config? → read the file back, validate syntax
   - Deployed a service? → curl the public URL, check HTTP status
3. If verification fails:
   - Diagnose: read logs, check error messages, inspect state
   - Fix: attempt the most likely remedy
   - Re-verify: check again
4. If fix fails after 2 attempts: report failure with full diagnostic context
```

### Pattern: Self-Healing Heartbeat

```
You are the Health Monitor. Perform a thorough health check:

For each instance in the database with status "running":
1. docker inspect the container — is it actually running?
2. curl the health endpoint — does the service respond?
3. If container is running but service doesn't respond:
   - Check docker logs for errors
   - If config issue: regenerate config and restart
   - If resource issue: report for manual intervention
4. If container is dead but DB says "running":
   - Attempt restart with docker start
   - If restart fails: check logs, set status to "error" with diagnostic
5. After all checks: verify nginx routing matches actual running instances

Report: which instances were checked, which had issues, what was fixed.
```

This is fundamentally different from "check docker state matches DB state" — it tests **actual service functionality**.

## Reinforcement in the Agent Loop

After each tool execution, inject context to keep the agent on track:

```typescript
// Armin Ronacher's reinforcement pattern
session.subscribe((event) => {
  if (event.type === "tool_execution_end") {
    // Inject a reminder of the overall objective
    session.queueMessage(`Reminder: your objective is to ${taskDescription}. 
Current progress: ${readProgressFile()}. 
Continue with the next step.`);
  }
});
```

**Use cases for reinforcement:**
- Remind the agent of the overall task after a long tool execution
- Inform the agent of background state changes (other tasks completed, environment changed)
- Provide hints when a tool fails ("the docker socket requires the docker group — try with sudo")
- Self-reinforcement: have the agent write its own task list to a file, which it reads back to stay on track

## Failure Isolation

Run subtasks that might fail repeatedly in isolation. Don't pollute the main context with failed attempts.

```typescript
// Run a risky subtask in a sub-agent
const subResult = await runSubAgent({
  prompt: `Fix the failing test in ${testFile}. Try up to 3 approaches.`,
  model: cheapModel,
  maxTurns: 10,
});

// Only inject the summary into the main context
mainSession.queueMessage(
  `Subtask result: ${subResult.success ? "Fixed" : "Could not fix"}. ` +
  `Approaches tried: ${subResult.summary}. ` +
  (subResult.success ? "" : `Errors: ${subResult.errors.join(", ")}`)
);
```

As Armin Ronacher notes: *"It is helpful for an agent to learn about what did not work in a subtask because it can then feed that information into the next task to hopefully steer away from those failures."*

Pi supports this naturally — spawn `pi --mode rpc` as a subprocess, or just have the agent run `pi --print` via bash.

## Cost Control

```typescript
let totalCost = 0;
const MAX_COST = 0.50;

session.subscribe((event) => {
  if (event.type === "message_end" && event.message.role === "assistant") {
    totalCost += event.message.usage?.cost?.total ?? 0;
    if (totalCost > MAX_COST) session.abort();
  }
});
```

**Model selection by task complexity:**
```typescript
const isComplex = /diagnos|migrat|refactor|debug|troubleshoot/i.test(intent);
const model = isComplex
  ? getModel("anthropic", "claude-sonnet-4-20250514")
  : getModel("anthropic", "claude-3-5-haiku-20241022");
```

**Important**: A better tool caller does the job in fewer tokens. Cheaper models are not necessarily cheaper in a loop. (Armin Ronacher)

## Timeout and Loop Protection

```typescript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5 * 60_000);

try {
  await session.prompt(intent);
} finally {
  clearTimeout(timeout);
}
```

## Observability

Agent apps are non-deterministic. You need observability to understand failures.

**Essential (implement first):**
- Structured logging: traceId, tool name, duration, success/failure for every tool call
- Cost tracking: per-request and per-day totals
- Error logging: capture full tool errors and LLM stop reasons

```typescript
session.subscribe((event) => {
  const base = { sessionId, ts: Date.now() };
  if (event.type === "tool_execution_start")
    logger.info({ ...base, tool: event.toolName, args: event.args });
  if (event.type === "tool_execution_end")
    logger.info({ ...base, tool: event.toolName, error: event.isError });
  if (event.type === "message_end" && event.message.role === "assistant")
    logger.info({ ...base, cost: event.message.usage?.cost?.total, model: event.message.model });
});
```

**Useful (add when needed):**
- Debug transcripts: full event capture for post-mortem analysis
- Session HTML export: `session.exportToHtml()` for visual replay
- Raw payload inspection: `onPayload` callback to see exact prompts sent to providers

**Advanced (add in production):**
- Cost alerting with per-user/day budgets
- OpenTelemetry bridge for integration with Langfuse/Logfire/Datadog
- Eval datasets for regression testing agent behavior

## Testing

### Unit Test Tools

Tools are async functions — test them directly:

```typescript
it("rejects unauthorized users", async () => {
  await expect(
    deployTool.execute("id", { name: "test", userId: "bad" }, undefined, undefined)
  ).rejects.toThrow("Invalid userId");
});
```

### Integration Test Agent Behavior

```typescript
it("deploys and verifies instance", async () => {
  const { session } = await createAgentSession({
    model: getModel("anthropic", "claude-sonnet-4-20250514"),
    customTools: [deployTool],
    sessionManager: SessionManager.inMemory(),
  });

  const events: any[] = [];
  session.subscribe(e => { if (e.type === "tool_execution_end") events.push(e); });

  await session.prompt("Deploy a test nginx instance and verify it's healthy");

  expect(events.some(e => e.toolName === "deploy_instance" && !e.isError)).toBe(true);
});
```

### Eval-Driven Refinement

Following Anthropic's methodology:
1. Create diverse, realistic test prompts
2. Run agent, collect full transcripts
3. Analyze: which tools called, were params correct, did agent get stuck?
4. Iterate on tool descriptions, prompt wording, error messages
5. Maintain a held-out test set

## Context Management

**Compaction**: For long-running tasks, enable `compaction.enabled: true`. Pi auto-summarizes old messages when context nears the limit.

**Cross-request memory**: Serialize context with `JSON.stringify(agentContext)` and restore later. Works across providers.

**Filesystem as memory**: For multi-session continuity, have the agent write state files. These survive session boundaries, can be shared across agents, and are inspectable by humans.
