# Orchestration Skills

**Version**: v1.0.0
**Date**: 2026-03-31

---

## Origin

On March 31, 2026, the Claude Code source code was exposed via a source map in the npm package. The repository at `~/projects/Claude Code/claude-code/` is that snapshot.

Reverse-engineering the source revealed how Anthropic designs orchestrator prompts, defines agents, and enforces quality gates. These skills are the repeatable patterns extracted from that analysis — turned into tools for my own orchestration work.

**Core motivation**: stopping the mistakes I keep making when running multi-agent tasks.
- Lazy delegation — forwarding research results to the next worker without reading them
- Skipping verification and assuming implementation worked
- Writing worker prompts that aren't self-contained
- Serializing work that could run in parallel

---

## Reference

### Source
`~/projects/Claude Code/claude-code/` — Claude Code source snapshot (leaked 2026-03-31)

Key files analyzed:
- `src/constants/prompts.ts` — main system prompt assembly
- `src/coordinator/coordinatorMode.ts` — coordinator system prompt
- `src/tools/AgentTool/built-in/verificationAgent.ts` — verification agent prompt
- `src/tools/AgentTool/built-in/exploreAgent.ts` — explore agent prompt
- `src/constants/tools.ts` — tool access control constants

### Architecture guide
**[ORCHESTRATOR_HARNESS_GUIDE.md](../ORCHESTRATOR_HARNESS_GUIDE.md)**
Full reverse-engineering analysis: system prompt structure, coordinator design, agent definitions, quality systems, and templates. Read this before modifying or adding skills.

---

## Intent

These skills are not for making the AI do things — they are **protocols I follow when acting as the coordinator**.

When using Claude Code, I am the orchestrator. The orchestrator's job is:
- Not to execute directly, but to **decompose, synthesize, and gate**
- To **read and understand** research findings before writing specs
- To require **independent verification** before reporting any non-trivial implementation complete

These skills encode those principles as enforced workflows.

---

## Skills

### Workflow skills

| Skill | File | When |
|-------|------|------|
| `/orchestrate` | [orchestrate.md](orchestrate.md) | Full task from start to finish — enforces all 4 phases |
| `/research` | [research.md](research.md) | Parallel codebase exploration before implementation |
| `/synthesize` | [synthesize.md](synthesize.md) | After research returns — convert findings to spec |
| `/spec` | [spec.md](spec.md) | Write a plan before touching code |
| `/verify` | [verify.md](verify.md) | After implementation — gate on VERDICT: PASS |
| `/handoff` | [handoff.md](handoff.md) | Review a worker prompt before sending |

### Typical flows

```
Full task:
  /orchestrate

Step by step:
  /research          ← parallel Explore agents
      ↓ results arrive
  /synthesize        ← read findings, write spec
      ↓
  (implementation)
      ↓
  /verify            ← VERDICT: PASS gate

Planning first:
  /spec

Prompt review:
  /handoff           ← before spawning any worker
```

---

## Design principles

What these skills enforce:

1. **Never delegate synthesis** — "based on your findings" is banned. `/synthesize` makes you do it.
2. **Verification must be independent** — `/verify` spawns a separate agent that did not do the implementation.
3. **Parallelize by default** — `/research` puts all Agent calls in one message.
4. **Worker prompts must be zero-context complete** — `/handoff` checks this.
5. **Done requires a definition** — `/spec` writes completion criteria before any code is touched.

---

## Changelog

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2026-03-31 | Initial — 6 skills from Claude Code source reverse-engineering |
