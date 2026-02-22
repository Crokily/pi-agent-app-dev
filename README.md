# pi-agent-app-dev

An [Agent Skill](https://agentskills.io) for building **autonomous agent-powered applications** (not chatbots) using the [pi-mono](https://github.com/badlogic/pi-mono) SDK.

## Core Philosophy: Environment Provider, Not Orchestrator

This skill advocates for the **Agent-Driven** paradigm (similar to Claude Agent SDK and Pi SDK). Instead of building rigid orchestration graphs (workflows), you provide a rich **Environment** and a robust **Harness** where an agent can autonomously decide what to do, verify its own results, and self-heal.

> "Maybe the best architecture is almost no architecture at all. Just filesystems and bash." — Vercel

## Key Concepts

- **Environment-First**: Give the agent bash, filesystem, and state access. Let it explore and act.
- **Harness Over Graph**: Use system prompts, AGENTS.md, skills, and code-level hooks to steer the agent without constraining its decision-making.
- **Bash as a Universal Tool**: Move from specialized "tool wrappers" to general-purpose bash. One tool for infinite capabilities.
- **Verification Loops**: The core value of an agent is its ability to verify its own work (e.g., `curl` health checks, `docker inspect`) and diagnose failures.
- **Filesystem as State**: Use files for agent memory, planning, and coordination across turns and sessions.

## What This Skill Covers

- **Architectural Paradigms** — Choosing between App-Driven (Orchestration) and Agent-Driven (Harness).
- **Minimalist Tool Design** — Why you only need `bash` + `read` + `write` + `edit`. When to justify custom tools.
- **Harness Design** — Crafting system prompts that encode judgment, verification procedures, and recovery strategies.
- **Integration Patterns** — Embedding pi-mono into Next.js, using RPC subprocesses for isolation, and extension-based apps.
- **Production Patterns** — Self-healing heartbeats, failure isolation via sub-agents, reinforcement loops, and cost/token management.
- **Multi-Layer Security** — Hardcoded tool validation, sandboxed code execution (Docker, Monty, CodeMode, E2B), and permission gates.

## Installation

Copy the `pi-agent-app-dev/` directory into your skills location:

```bash
# Global (all projects)
cp -r pi-agent-app-dev/ ~/.pi/agent/skills/

# Project-local
cp -r pi-agent-app-dev/ .pi/skills/
```

Or add via pi settings:

```json
{
  "skills": ["/path/to/pi-agent-app-dev"]
}
```

## Structure

```
pi-agent-app-dev/
├── SKILL.md                        — Core guide: philosophy, harness design, quick reference
└── references/
    ├── tool-design.md              — Bash-first philosophy, custom tool justifications, anti-patterns
    ├── security.md                 — Five-layer defense, sandbox comparison, human-in-the-loop
    ├── integration-patterns.md     — SDK/RPC/Extension integration, service boundary layers
    └── production.md               — Verification loops, self-healing, reinforcement, failure isolation
```

## License

MIT
