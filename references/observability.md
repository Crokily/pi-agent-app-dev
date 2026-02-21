# Observability, Debugging & Monitoring for Agent Applications

## Why Agent Apps Need Special Observability

> "From a software engineer's point of view, you can think of LLMs as the worst database you've ever heard of, but worse."
> — PydanticAI docs

Traditional software is deterministic: same input → same output. Agent apps are **non-deterministic, multi-step, and expensive**. A subtle prompt change can completely alter behavior, and there's no `EXPLAIN` query to understand why. Agent observability must cover:

1. **Trace completeness** — every LLM call, tool execution, and decision in a single view
2. **Cost attribution** — per-session, per-user, per-tool cost breakdown
3. **Error diagnosis** — why did the agent fail? Was it the LLM, a tool, or the prompt?
4. **Performance profiling** — latency per step, time-to-first-token, tool execution duration
5. **Quality measurement** — did the agent produce the right result? How to evaluate?
6. **Production health** — alerting on degradation, drift, and anomalies

## Industry Consensus (2025-2026)

### The Trace → Span Hierarchy

Every major framework has converged on a **Traces + Spans** model, aligned with OpenTelemetry:

```
Trace (agent run / workflow)
├── Span: invoke_agent "DeployBot"
│   ├── Span: LLM generation (model=claude-sonnet-4, tokens=1200, cost=$0.004)
│   ├── Span: execute_tool "check_status" (duration=320ms, success=true)
│   ├── Span: LLM generation (model=claude-sonnet-4, tokens=800, cost=$0.003)
│   └── Span: execute_tool "deploy" (duration=4200ms, success=true)
└── metadata: { userId, sessionId, totalCost, totalTokens, success }
```

| Framework | Tracing Model | Standard |
|-----------|--------------|----------|
| **OpenAI Agents SDK** | Built-in traces/spans, auto-traces agent runs + tool calls + guardrails | Custom + 30+ external processors |
| **PydanticAI / Logfire** | OpenTelemetry native, follows GenAI Semantic Conventions | OTel GenAI v1.37+ |
| **LangChain / LangSmith** | Traces with nested runs, sessions, user tracking, agent graphs | Proprietary + OTel export |
| **Agno (AgentOS)** | Built-in tracing with per-user/session isolation | Custom via AgentOS |
| **Cloudflare AI Gateway** | Proxy-level analytics: request count, tokens, cost, latency | Gateway pattern |
| **Langfuse** | Open-source OTel-compatible traces, sessions, evaluations | OTel + custom |

### OpenTelemetry GenAI Semantic Conventions (v1.40.0)

The OTel standard now defines **agent-specific spans**:

- `create_agent` — agent creation (remote services)
- `invoke_agent` — agent invocation with full attributes
- `execute_tool` — tool execution
- Standard attributes: `gen_ai.agent.name`, `gen_ai.agent.id`, `gen_ai.conversation.id`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.usage.cache_read.input_tokens`

This is the direction the industry is heading. Pi-mono isn't OTel-native, but its event system maps cleanly to OTel spans.

## Pi-Mono Observability Primitives

Pi-mono provides raw building blocks. Your app layer builds observability on top.

### Available Primitives

| Primitive | Source | What it gives you |
|-----------|--------|-------------------|
| `AgentEvent` stream | `session.subscribe()` | agent_start/end, turn_start/end, message_start/update/end, tool_execution_start/update/end |
| `AgentSessionEvent` | `session.subscribe()` | auto_compaction_start/end, auto_retry_start/end |
| `Usage` on AssistantMessage | `event.message.usage` | input/output/cacheRead/cacheWrite tokens + cost breakdown |
| `SessionStats` | `session.getSessionStats()` | Aggregate token/cost/message counts |
| `ContextUsage` | `session.getContextUsage()` | Current context window utilization |
| `onPayload` callback | `StreamOptions.onPayload` | Raw provider HTTP payload inspection |
| `StopReason` | `event.message.stopReason` | Why the LLM stopped: "stop", "length", "toolUse", "error", "aborted" |
| Session export | `session.exportToHtml()` | Full conversation replay as HTML |

### Event-to-Span Mapping

```
Pi-Mono Event              → OTel Span Equivalent
──────────────────────────────────────────────────
agent_start                → trace start (invoke_agent)
  turn_start               → span start (agent_turn)
    message_start          → span start (generation)
    message_end            → span end (generation) + usage attributes
    tool_execution_start   → span start (execute_tool)
    tool_execution_end     → span end (execute_tool) + error attribute
  turn_end                 → span end (agent_turn)
agent_end                  → trace end (invoke_agent)
auto_retry_start           → span start (retry)
auto_compaction_start      → span start (compaction)
```

## Pattern 1: Comprehensive Structured Logging

The foundation — emit structured JSON logs from events:

```typescript
interface AgentTrace {
  traceId: string;
  sessionId: string;
  userId?: string;
  startedAt: number;
  endedAt?: number;
  spans: AgentSpan[];
  totalCost: number;
  totalTokens: { input: number; output: number; cacheRead: number };
  success: boolean;
  error?: string;
}

interface AgentSpan {
  spanId: string;
  parentSpanId?: string;
  type: "generation" | "tool_execution" | "compaction" | "retry";
  name: string;
  startedAt: number;
  endedAt?: number;
  attributes: Record<string, any>;
}

function createTracer(session: AgentSession, userId?: string) {
  const trace: AgentTrace = {
    traceId: `trace_${crypto.randomUUID().replace(/-/g, "")}`,
    sessionId: session.sessionId,
    userId,
    startedAt: Date.now(),
    spans: [],
    totalCost: 0,
    totalTokens: { input: 0, output: 0, cacheRead: 0 },
    success: true,
  };

  let currentTurnSpanId: string | undefined;
  const toolStartTimes = new Map<string, { spanId: string; startedAt: number }>();

  session.subscribe((event) => {
    switch (event.type) {
      case "turn_start":
        currentTurnSpanId = `span_${crypto.randomUUID().replace(/-/g, "").slice(0, 16)}`;
        trace.spans.push({
          spanId: currentTurnSpanId,
          type: "generation",
          name: "agent_turn",
          startedAt: Date.now(),
          attributes: {},
        });
        break;

      case "message_end":
        if (event.message.role === "assistant") {
          const msg = event.message as any;
          const usage = msg.usage;
          if (usage) {
            trace.totalCost += usage.cost?.total ?? 0;
            trace.totalTokens.input += usage.input ?? 0;
            trace.totalTokens.output += usage.output ?? 0;
            trace.totalTokens.cacheRead += usage.cacheRead ?? 0;
          }
          // Update turn span with model info
          const turnSpan = trace.spans.find(s => s.spanId === currentTurnSpanId);
          if (turnSpan) {
            turnSpan.attributes = {
              ...turnSpan.attributes,
              model: msg.model,
              provider: msg.provider,
              stopReason: msg.stopReason,
              inputTokens: usage?.input,
              outputTokens: usage?.output,
              cacheReadTokens: usage?.cacheRead,
              cost: usage?.cost?.total,
            };
          }
        }
        break;

      case "tool_execution_start":
        const toolSpanId = `span_${crypto.randomUUID().replace(/-/g, "").slice(0, 16)}`;
        toolStartTimes.set(event.toolCallId, { spanId: toolSpanId, startedAt: Date.now() });
        trace.spans.push({
          spanId: toolSpanId,
          parentSpanId: currentTurnSpanId,
          type: "tool_execution",
          name: event.toolName,
          startedAt: Date.now(),
          attributes: { args: event.args },
        });
        break;

      case "tool_execution_end": {
        const startInfo = toolStartTimes.get(event.toolCallId);
        if (startInfo) {
          const span = trace.spans.find(s => s.spanId === startInfo.spanId);
          if (span) {
            span.endedAt = Date.now();
            span.attributes.durationMs = Date.now() - startInfo.startedAt;
            span.attributes.isError = event.isError;
            if (event.isError) trace.success = false;
          }
          toolStartTimes.delete(event.toolCallId);
        }
        break;
      }

      case "auto_retry_start":
        trace.spans.push({
          spanId: `span_retry_${event.attempt}`,
          type: "retry",
          name: `retry_attempt_${event.attempt}`,
          startedAt: Date.now(),
          attributes: { attempt: event.attempt, maxAttempts: event.maxAttempts, errorMessage: event.errorMessage },
        });
        break;

      case "agent_end":
        trace.endedAt = Date.now();
        break;
    }
  });

  return {
    getTrace: () => trace,
    finalize: () => {
      trace.endedAt = trace.endedAt ?? Date.now();
      return trace;
    },
  };
}
```

## Pattern 2: Cost Monitoring & Budget Enforcement

### Per-Request Budget with Alerts

```typescript
interface CostPolicy {
  maxCostPerRequest: number;   // e.g., $0.50
  maxCostPerUser24h: number;   // e.g., $5.00
  warnAtPercent: number;       // e.g., 80
  onWarn?: (current: number, max: number) => void;
  onExceed?: (current: number, max: number) => void;
}

function enforceCostPolicy(session: AgentSession, policy: CostPolicy) {
  let requestCost = 0;

  session.subscribe((event) => {
    if (event.type === "message_end" && event.message.role === "assistant") {
      const usage = (event.message as any).usage;
      if (usage?.cost?.total) {
        requestCost += usage.cost.total;

        // Warn threshold
        if (requestCost >= policy.maxCostPerRequest * (policy.warnAtPercent / 100)) {
          policy.onWarn?.(requestCost, policy.maxCostPerRequest);
        }

        // Hard limit
        if (requestCost >= policy.maxCostPerRequest) {
          policy.onExceed?.(requestCost, policy.maxCostPerRequest);
          session.abort();
        }
      }
    }
  });

  return { getCurrentCost: () => requestCost };
}
```

### Per-Tool Cost Attribution

Track which tools consume the most LLM tokens (tool calls generate output tokens for the model's reasoning + tool_result generates input tokens on the next turn):

```typescript
interface ToolCostTracker {
  toolName: string;
  invocationCount: number;
  totalDurationMs: number;
  errors: number;
  // LLM cost is attributed to the turn that called the tool
}

function trackToolCosts(session: AgentSession): Map<string, ToolCostTracker> {
  const trackers = new Map<string, ToolCostTracker>();
  const startTimes = new Map<string, number>();

  session.subscribe((event) => {
    if (event.type === "tool_execution_start") {
      startTimes.set(event.toolCallId, Date.now());
    }
    if (event.type === "tool_execution_end") {
      const start = startTimes.get(event.toolCallId);
      const duration = start ? Date.now() - start : 0;
      const tracker = trackers.get(event.toolName) ?? {
        toolName: event.toolName, invocationCount: 0, totalDurationMs: 0, errors: 0,
      };
      tracker.invocationCount++;
      tracker.totalDurationMs += duration;
      if (event.isError) tracker.errors++;
      trackers.set(event.toolName, tracker);
    }
  });

  return trackers;
}
```

## Pattern 3: Error Classification & Debugging

### Error Taxonomy for Agent Apps

```typescript
type AgentErrorClass =
  | "llm_api_error"       // Provider API failure (rate limit, server error)
  | "llm_context_overflow" // Context too large
  | "llm_refusal"         // Model refused to act
  | "tool_execution_error" // Tool threw an error
  | "tool_timeout"        // Tool took too long
  | "budget_exceeded"     // Cost or turn limit exceeded
  | "user_abort"          // User cancelled
  | "agent_stuck"         // Agent looping without progress
  | "prompt_injection"    // Detected prompt injection attempt
  | "unknown";

function classifyError(event: any, context: { turnCount: number; lastNTools: string[] }): AgentErrorClass {
  if (event.type === "auto_retry_start") {
    if (/overloaded|rate.limit|429/i.test(event.errorMessage)) return "llm_api_error";
    if (/context|overflow|too.long/i.test(event.errorMessage)) return "llm_context_overflow";
    return "llm_api_error";
  }
  if (event.type === "tool_execution_end" && event.isError) return "tool_execution_error";
  // Detect stuck loop: same tool called 3+ times in a row
  if (context.lastNTools.length >= 3 && new Set(context.lastNTools.slice(-3)).size === 1) return "agent_stuck";
  return "unknown";
}
```

### Debug Transcript Capture

For post-mortem debugging, capture full transcripts:

```typescript
function captureDebugTranscript(session: AgentSession) {
  const transcript: Array<{
    timestamp: number;
    type: string;
    data: any;
  }> = [];

  session.subscribe((event) => {
    transcript.push({
      timestamp: Date.now(),
      type: event.type,
      data: (() => {
        switch (event.type) {
          case "message_end":
            return {
              role: event.message.role,
              contentTypes: event.message.content?.map((c: any) => c.type),
              model: (event.message as any).model,
              usage: (event.message as any).usage,
              stopReason: (event.message as any).stopReason,
            };
          case "tool_execution_start":
            return { tool: event.toolName, args: event.args };
          case "tool_execution_end":
            return { tool: event.toolName, isError: event.isError, resultPreview: truncate(JSON.stringify(event.result), 500) };
          case "auto_retry_start":
            return { attempt: event.attempt, error: event.errorMessage };
          default:
            return {};
        }
      })(),
    });
  });

  return {
    getTranscript: () => transcript,
    save: async (path: string) => {
      await writeFile(path, JSON.stringify(transcript, null, 2));
    },
  };
}

function truncate(s: string, maxLen: number): string {
  return s.length > maxLen ? s.slice(0, maxLen) + "…" : s;
}
```

## Pattern 4: Performance Monitoring

### Latency Tracking

```typescript
interface PerformanceMetrics {
  totalDurationMs: number;
  timeToFirstToken: number | null;  // From prompt to first streaming text
  llmGenerationMs: number;          // Total time in LLM calls
  toolExecutionMs: number;          // Total time in tool execution
  turnCount: number;
  avgTurnDurationMs: number;
  slowestTool: { name: string; durationMs: number } | null;
}

function trackPerformance(session: AgentSession): { getMetrics: () => PerformanceMetrics } {
  const metrics: PerformanceMetrics = {
    totalDurationMs: 0, timeToFirstToken: null, llmGenerationMs: 0,
    toolExecutionMs: 0, turnCount: 0, avgTurnDurationMs: 0, slowestTool: null,
  };

  let agentStartTime: number | null = null;
  let turnStartTime: number | null = null;
  let firstTokenReceived = false;
  const toolDurations: { name: string; ms: number }[] = [];
  const toolStarts = new Map<string, number>();

  session.subscribe((event) => {
    switch (event.type) {
      case "agent_start":
        agentStartTime = Date.now();
        firstTokenReceived = false;
        break;
      case "message_update":
        if (!firstTokenReceived && agentStartTime) {
          metrics.timeToFirstToken = Date.now() - agentStartTime;
          firstTokenReceived = true;
        }
        break;
      case "turn_start":
        turnStartTime = Date.now();
        metrics.turnCount++;
        break;
      case "turn_end":
        if (turnStartTime) metrics.llmGenerationMs += Date.now() - turnStartTime;
        break;
      case "tool_execution_start":
        toolStarts.set(event.toolCallId, Date.now());
        break;
      case "tool_execution_end": {
        const start = toolStarts.get(event.toolCallId);
        if (start) {
          const ms = Date.now() - start;
          metrics.toolExecutionMs += ms;
          toolDurations.push({ name: event.toolName, ms });
        }
        break;
      }
      case "agent_end":
        if (agentStartTime) metrics.totalDurationMs = Date.now() - agentStartTime;
        metrics.avgTurnDurationMs = metrics.turnCount > 0
          ? metrics.totalDurationMs / metrics.turnCount : 0;
        const slowest = toolDurations.sort((a, b) => b.ms - a.ms)[0];
        if (slowest) metrics.slowestTool = slowest;
        break;
    }
  });

  return { getMetrics: () => metrics };
}
```

## Pattern 5: OpenTelemetry Bridge

Bridge pi-mono events to OTel for integration with any observability platform:

```typescript
import { trace, SpanStatusCode, Span, Tracer } from "@opentelemetry/api";

function createOTelBridge(session: AgentSession, tracerName = "pi-agent") {
  const tracer: Tracer = trace.getTracer(tracerName);
  let rootSpan: Span | undefined;
  const spanStack = new Map<string, Span>();

  session.subscribe((event) => {
    switch (event.type) {
      case "agent_start":
        rootSpan = tracer.startSpan("invoke_agent", {
          attributes: {
            "gen_ai.operation.name": "invoke_agent",
            "gen_ai.conversation.id": session.sessionId,
          },
        });
        break;

      case "message_end":
        if (event.message.role === "assistant" && rootSpan) {
          const msg = event.message as any;
          const genSpan = tracer.startSpan("generation", {
            attributes: {
              "gen_ai.operation.name": "chat",
              "gen_ai.request.model": msg.model,
              "gen_ai.provider.name": msg.provider,
              "gen_ai.usage.input_tokens": msg.usage?.input,
              "gen_ai.usage.output_tokens": msg.usage?.output,
              "gen_ai.usage.cache_read.input_tokens": msg.usage?.cacheRead,
              "gen_ai.response.finish_reasons": [msg.stopReason],
            },
          });
          genSpan.end();
        }
        break;

      case "tool_execution_start":
        const toolSpan = tracer.startSpan(`execute_tool ${event.toolName}`, {
          attributes: {
            "gen_ai.operation.name": "execute_tool",
            "gen_ai.tool.name": event.toolName,
            "gen_ai.tool.call.arguments": JSON.stringify(event.args),
          },
        });
        spanStack.set(event.toolCallId, toolSpan);
        break;

      case "tool_execution_end": {
        const span = spanStack.get(event.toolCallId);
        if (span) {
          if (event.isError) {
            span.setStatus({ code: SpanStatusCode.ERROR });
            span.setAttribute("error.type", "tool_execution_error");
          }
          span.end();
          spanStack.delete(event.toolCallId);
        }
        break;
      }

      case "agent_end":
        rootSpan?.end();
        break;
    }
  });
}
```

### Connecting to Observability Platforms

With the OTel bridge, export to any platform:

```typescript
// Example: Export to Langfuse, Logfire, or any OTel-compatible backend
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { BatchSpanProcessor } from "@opentelemetry/sdk-trace-base";

const provider = new NodeTracerProvider();
provider.addSpanProcessor(
  new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? "http://localhost:4318/v1/traces",
      headers: { Authorization: `Bearer ${process.env.OTEL_API_KEY}` },
    })
  )
);
provider.register();
// Now createOTelBridge() spans are exported automatically
```

**Compatible platforms** (non-exhaustive): Langfuse, Pydantic Logfire, LangSmith, Arize Phoenix, Weights & Biases, MLflow, Braintrust, AgentOps, Datadog, New Relic, Grafana, SigNoz.

## Pattern 6: Raw Provider Payload Inspection

PydanticAI's Hamel Husain-inspired principle: "Show me the prompt." Use pi-ai's `onPayload` to inspect exactly what's sent to providers:

```typescript
const { session } = await createAgentSession({
  model: getModel("anthropic", "claude-sonnet-4-20250514"),
  // ... other config
});

// In StreamOptions or via the session's stream configuration:
// onPayload captures the raw API request body
const payloads: unknown[] = [];
// Use this when calling pi-ai directly:
const response = await stream(model, context, {
  onPayload: (payload) => {
    payloads.push(payload);
    console.log("[RAW PAYLOAD]", JSON.stringify(payload, null, 2));
  },
});
```

This is invaluable for debugging model behavior — you see the exact system prompt, tool definitions, and messages sent to the provider.

## Pattern 7: Production Alerting

Build alert rules on top of the trace/metrics data:

```typescript
interface AlertRule {
  name: string;
  condition: (trace: AgentTrace) => boolean;
  severity: "warning" | "critical";
  action: (trace: AgentTrace) => void;
}

const alertRules: AlertRule[] = [
  {
    name: "high_cost",
    condition: (t) => t.totalCost > 1.0,
    severity: "critical",
    action: (t) => sendAlert(`Agent trace ${t.traceId} cost $${t.totalCost.toFixed(2)}`),
  },
  {
    name: "high_turn_count",
    condition: (t) => t.spans.filter(s => s.type === "generation").length > 15,
    severity: "warning",
    action: (t) => sendAlert(`Agent trace ${t.traceId} used ${t.spans.length} turns — possible loop`),
  },
  {
    name: "tool_error_rate",
    condition: (t) => {
      const tools = t.spans.filter(s => s.type === "tool_execution");
      const errors = tools.filter(s => s.attributes.isError);
      return tools.length > 0 && errors.length / tools.length > 0.3;
    },
    severity: "warning",
    action: (t) => sendAlert(`Agent trace ${t.traceId} had >30% tool error rate`),
  },
  {
    name: "slow_execution",
    condition: (t) => (t.endedAt ?? Date.now()) - t.startedAt > 120_000,
    severity: "warning",
    action: (t) => sendAlert(`Agent trace ${t.traceId} took >2 minutes`),
  },
];

function evaluateAlerts(trace: AgentTrace) {
  for (const rule of alertRules) {
    if (rule.condition(trace)) {
      rule.action(trace);
    }
  }
}
```

## Pattern 8: Session Replay & Post-Mortem

Pi-mono's built-in HTML export + custom debug captures:

```typescript
// Built-in: Export full conversation as HTML
const htmlPath = await session.exportToHtml("/path/to/debug");

// Custom: Combine trace + transcript for complete post-mortem
async function savePostMortem(session: AgentSession, tracer: ReturnType<typeof createTracer>, transcript: any[]) {
  const stats = session.getSessionStats();
  const trace = tracer.finalize();
  const postMortem = {
    summary: {
      sessionId: session.sessionId,
      cost: stats.cost,
      tokens: stats.tokens,
      messages: stats.totalMessages,
      toolCalls: stats.toolCalls,
      success: trace.success,
      durationMs: trace.endedAt! - trace.startedAt,
    },
    trace,
    transcript,
  };

  await writeFile(`/var/log/agent/postmortem-${trace.traceId}.json`, JSON.stringify(postMortem, null, 2));
  // Also export HTML for visual replay
  await session.exportToHtml(`/var/log/agent/replay-${trace.traceId}`);
}
```

## Pattern 9: Eval-Driven Quality Monitoring

Inspired by Anthropic's "Writing Tools for Agents" and LangSmith's evaluation framework:

### Automated Quality Checks

```typescript
interface EvalResult {
  traceId: string;
  score: number;        // 0-1
  passed: boolean;
  checks: { name: string; passed: boolean; detail: string }[];
}

function evaluateAgentRun(trace: AgentTrace, expectedTools?: string[]): EvalResult {
  const checks: EvalResult["checks"] = [];

  // Check 1: Did it complete successfully?
  checks.push({
    name: "completion",
    passed: trace.success,
    detail: trace.success ? "Agent completed" : `Agent failed: ${trace.error}`,
  });

  // Check 2: Were expected tools called?
  if (expectedTools) {
    const calledTools = new Set(trace.spans.filter(s => s.type === "tool_execution").map(s => s.name));
    const missing = expectedTools.filter(t => !calledTools.has(t));
    checks.push({
      name: "expected_tools",
      passed: missing.length === 0,
      detail: missing.length === 0 ? "All expected tools called" : `Missing: ${missing.join(", ")}`,
    });
  }

  // Check 3: Cost reasonable?
  checks.push({
    name: "cost_reasonable",
    passed: trace.totalCost < 0.50,
    detail: `Cost: $${trace.totalCost.toFixed(3)}`,
  });

  // Check 4: No stuck loops?
  const toolNames = trace.spans.filter(s => s.type === "tool_execution").map(s => s.name);
  const hasLoop = toolNames.some((name, i) =>
    i >= 2 && toolNames[i - 1] === name && toolNames[i - 2] === name
  );
  checks.push({
    name: "no_loops",
    passed: !hasLoop,
    detail: hasLoop ? "Detected tool call loop" : "No loops detected",
  });

  const passed = checks.every(c => c.passed);
  const score = checks.filter(c => c.passed).length / checks.length;
  return { traceId: trace.traceId, score, passed, checks };
}
```

### Eval Dataset for Regression Testing

```typescript
interface EvalCase {
  name: string;
  prompt: string;
  expectedTools: string[];
  maxCost: number;
  maxTurns: number;
  validator?: (trace: AgentTrace) => boolean;
}

const evalDataset: EvalCase[] = [
  {
    name: "simple_deploy",
    prompt: "Deploy nginx to staging",
    expectedTools: ["deploy_instance"],
    maxCost: 0.10,
    maxTurns: 5,
  },
  {
    name: "error_recovery",
    prompt: "Deploy invalid-image and handle the error",
    expectedTools: ["deploy_instance", "report_result"],
    maxCost: 0.20,
    maxTurns: 8,
    validator: (trace) => trace.spans.some(s => s.name === "report_result" && !s.attributes.isError),
  },
];

// Run eval suite and report results
async function runEvalSuite(evalCases: EvalCase[], createSession: () => Promise<AgentSession>) {
  const results: EvalResult[] = [];
  for (const evalCase of evalCases) {
    const session = await createSession();
    const tracer = createTracer(session);
    await session.prompt(evalCase.prompt);
    const trace = tracer.finalize();
    results.push(evaluateAgentRun(trace, evalCase.expectedTools));
  }
  return results;
}
```

## Quick Reference: Observability Checklist

### Development Phase
- [ ] Capture full debug transcripts for every test run
- [ ] Inspect raw provider payloads (`onPayload`) to verify prompts are correct
- [ ] Track tool call sequences — are tools called in expected order?
- [ ] Verify cost per operation stays within budget
- [ ] Build eval dataset with representative test cases

### Staging / Pre-Production
- [ ] Run eval suite against real models (not mocks)
- [ ] Verify error classification covers all failure modes
- [ ] Test cost budget enforcement (abort + alert)
- [ ] Test auto-retry behavior with simulated provider errors
- [ ] Validate context compaction doesn't lose critical information

### Production
- [ ] Structured logging with trace IDs for every agent run
- [ ] Cost alerting per-request AND per-user/24h
- [ ] Tool error rate monitoring with thresholds
- [ ] Latency tracking with P50/P95/P99 dashboards
- [ ] Session replay capability for debugging user issues
- [ ] Regular eval suite runs to detect quality regression
- [ ] Context window utilization monitoring (detect creep)

## External Resources

- **OpenTelemetry GenAI Semantic Conventions**: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- **OpenAI Agents SDK Tracing**: https://openai.github.io/openai-agents-python/tracing/
- **PydanticAI Logfire Integration**: https://ai.pydantic.dev/logfire/
- **LangSmith Observability**: https://docs.smith.langchain.com/observability
- **Langfuse (open-source)**: https://langfuse.com/docs
- **Cloudflare AI Gateway**: https://developers.cloudflare.com/ai-gateway/
