# pi-agent-app-dev

An [Agent Skill](https://agentskills.io) for building **agent-powered applications** (not chatbots) using the [pi-mono](https://github.com/badlogic/pi-mono) SDK.

## What This Skill Does

Teaches AI coding agents how to architect and implement applications where an LLM agent loop drives real-world actions — deployment automation, infrastructure management, data processing — rather than conversation.

**Core topics covered:**

- **Pi-mono three-layer architecture** — when to use `pi-ai`, `pi-agent-core`, or `pi-coding-agent`
- **Hybrid tool design** — bash (general primitive) + optimized primitives (read/write/edit) + custom business tools + skills (knowledge injection)
- **Five-layer security model** — tool-level validation, extension event gates, code execution sandbox, system prompt guidance, infrastructure isolation
- **Integration patterns** — SDK embedding, RPC subprocess (OpenClaw-style), extension-based, bare agent loop
- **Production patterns** — cost control, timeouts, structured output, context management, compaction, testing
- **Observability** — trace/span architecture, cost monitoring, error classification, OTel bridge, eval-driven quality

Includes analysis of Anthropic's 2024–2026 engineering research on tool design, context engineering, and agent architecture.

## Installation

Copy the `pi-agent-app-dev/` directory into your skills location:

```bash
# Global (all projects)
cp -r pi-agent-app-dev/ ~/.pi/agent/skills/
# or
cp -r pi-agent-app-dev/ ~/.agents/skills/

# Project-local
cp -r pi-agent-app-dev/ .pi/skills/
# or
cp -r pi-agent-app-dev/ .agents/skills/
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
├── SKILL.md                        — Core guide: architecture, patterns, quick reference
└── references/
    ├── tool-design.md              — Bash vs custom tools, hybrid architecture, Anthropic research
    ├── security.md                 — Five-layer security model, sandbox comparison, permission gates
    ├── integration-patterns.md     — SDK/RPC/Extension integration, OpenClaw case study
    ├── observability.md            — Trace/span architecture, cost monitoring, OTel bridge, eval
    └── production.md               — Cost, timeout, testing, context management
```

## Development Feedback

This skill includes an optional feedback collection mechanism. When the agent encounters genuine friction points during real development (gaps in coverage, incorrect guidance, or areas for improvement), it may log them to a `.pidev_feedback/` directory in your project.

**This is entirely optional.** No user-identifying information is recorded — only the technical scenario, the issue encountered, and how it was resolved. If you'd prefer to disable this behavior, remove the `## Development Feedback Collection (Optional)` section from `SKILL.md`.

If you do collect feedback and want to help improve this skill, you're welcome to submit the contents as [issues](https://github.com/Crokily/pi-agent-app-dev/issues).

## License

MIT
