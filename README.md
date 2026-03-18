# @mention Handoff Protocol

**A pattern for multi-agent coordination using @mentions as task triggers, with human-gated rounds.**

Created by [CODA Synthesis Labs](https://codasynthesis.com) — built in production with [OpenClaw](https://openclaw.ai).

---

## The Problem

Running multiple AI agents in a shared workspace (Discord, Slack, etc.) creates two failure modes:

1. **Infinite loops** — Agent A triggers Agent B, which triggers Agent A, burning tokens forever
2. **Silent isolation** — Agents work independently with no coordination, duplicating effort or missing context

Most multi-agent systems solve this with rigid orchestration (a single "manager" agent routes all work) or message queues. Both add complexity and single points of failure.

## The Pattern

**@mentions as structured task triggers, with human-gated rounds.**

```
Human assigns task → Agent A works → @mentions Agent B for handoff
Agent B works → @mentions Agent A with results → Human reviews
```

### Core Rules

1. **@mention = task trigger.** An agent only activates on a task when explicitly @mentioned by another agent or a human. No ambient listening, no auto-triggering.

2. **One response per thread.** When @mentioned, an agent responds once with its work product. No back-and-forth debate between agents.

3. **Human gates each round.** After an agent completes work and @mentions the next agent, a human reviews before the next agent activates. This prevents loops and ensures quality.

4. **Agents have lanes.** Each agent owns specific domains. Handoffs only happen across domain boundaries.

5. **File-based artifacts.** Agents communicate through files (reports, code, configs) — not conversational context. This makes handoffs auditable and resumable.

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  Shared Channel                  │
│              (Discord, Slack, etc.)              │
├─────────────────────────────────────────────────┤
│                                                  │
│  Human ──task──→ Agent A (Opus/Strategist)       │
│                     │                            │
│                     ├── works on task             │
│                     ├── produces artifact         │
│                     └── @mentions Agent B         │
│                            │                     │
│  Human reviews ◄───────────┘                     │
│       │                                          │
│       └── approves ──→ Agent B (GPT/QA)          │
│                           │                      │
│                           ├── reviews artifact   │
│                           ├── produces findings  │
│                           └── @mentions Agent A  │
│                                  │               │
│  Human reviews ◄─────────────────┘               │
│       │                                          │
│       └── closes task or iterates                │
│                                                  │
└─────────────────────────────────────────────────┘
```

## Implementation

### Agent Configuration

Each agent needs:
- A dedicated bot account on the platform (Discord bot, Slack app, etc.)
- An `allowFrom` list specifying which accounts can trigger it
- Clear lane definitions (what it owns, what it doesn't touch)

Example (OpenClaw config):
```json
{
  "channels": {
    "discord": {
      "allowFrom": ["human-user-id", "other-agent-bot-id"],
      "groupPolicy": "allowlist"
    }
  }
}
```

### Lane Definitions

Define each agent's domain explicitly:

| Agent | Owns | Does NOT Touch |
|-------|------|----------------|
| Coda (Strategist) | Architecture, strategy, task assignment, final review | Direct code fixes, QA reports |
| Penn (QA/Ops) | Code review, security scans, system health, deploy verification | Strategy decisions, money-touching code |

### Handoff Format

When an agent @mentions another, use a structured format:

```markdown
@Penn 📋 QA Review Needed

**Task:** Review cfo-daily-summary.cjs changes
**Branch/Commit:** `abc123`
**Changed files:** scripts/cfo-daily-summary.cjs
**What to check:**
- Discord posting logic (was broken, now uses --stdin)
- Error handling on API failures
**Artifact:** See commit diff
```

### Anti-Loop Rules

These are non-negotiable:

1. **Max 1 response per @mention.** Agent responds once, then waits.
2. **No responding to other agents' completions.** If Agent B finishes and @mentions Agent A, Agent A does NOT auto-respond — the human decides whether to continue.
3. **No spawning sub-agents in response to @mentions.** The receiving agent does the work itself.
4. **Timeout:** If an agent is @mentioned and doesn't respond within the configured timeout, the human is notified.
5. **Emergency stop:** Any human can say "stop" and all agents halt immediately.

### Human Gating

The human's role in each round:

1. **Review** — Read the agent's output. Is it correct? Complete?
2. **Decide** — Should the next agent be triggered, or is the task done?
3. **Trigger** — Either @mention the next agent directly, or approve the agent's @mention
4. **Override** — Redirect to a different agent, modify the task, or close it

**The human never needs to be in the loop for routine checks** — only for handoff decisions. This keeps the protocol lightweight while preventing runaway behavior.

## Comparison to Other Approaches

| Approach | Pros | Cons |
|----------|------|------|
| **Single orchestrator** | Centralized control | Single point of failure, bottleneck |
| **Message queue** | Reliable, ordered | Complex infra, over-engineered for <5 agents |
| **Shared context window** | Agents see everything | Token explosion, privacy issues |
| **@mention handoff** | Simple, auditable, loop-proof | Requires human availability for gating |

## Real-World Example

This protocol runs in production at CODA Synthesis Labs with:
- **Coda** (Claude Opus) — Strategy, builds, final review
- **Penn** (GPT-5.4) — QA, security scans, deploy verification, update research

### Actual handoff from production:

```
[Coda] Built cfo-daily-summary.cjs — fixes delivery config and
       Discord posting. @Penn please QA review:
       - Check the --stdin piping to discord-post.cjs
       - Verify error handling on Coinbase/Kalshi API failures
       - Confirm no secrets in the diff

[Human reviews, approves Penn to proceed]

[Penn] QA Review Complete:
       ✅ --stdin piping looks correct
       ✅ No secrets in diff
       ⚠️ Suggestion: Add timeout to execSync Discord post call
       @Coda findings above — one suggestion, no blockers

[Human reviews, closes task or asks Coda to implement suggestion]
```

## When NOT to Use This

- **Real-time systems** — The human gating adds latency. Use direct API calls instead.
- **Single-agent tasks** — If one agent can do the whole job, don't force a handoff.
- **More than 5 agents** — The @mention graph gets complex. Consider a proper orchestration layer.
- **Tasks requiring agent debate** — This protocol explicitly prevents back-and-forth. If you want agents to deliberate, use a different pattern.

## Getting Started

1. **Set up 2 agents** on a shared platform (Discord recommended for channel isolation)
2. **Define lanes** — what each agent owns
3. **Configure allowFrom** — agents can only hear @mentions from approved sources
4. **Start with one handoff type** — e.g., "build → QA review"
5. **Human gates every round** for the first week, then relax as trust builds

## License

MIT — use it however you want.

---

*Built by humans and AI, working together. Not the other way around.*

*[CODA Synthesis Labs](https://codasynthesis.com) · [@danpickett4-design](https://github.com/danpickett4-design)*
