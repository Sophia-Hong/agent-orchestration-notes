# agent-orchestration-notes

Architecture reference and reusable skills for multi-agent orchestration, reverse-engineered from the Claude Code source snapshot (2026-03-31).

**Version**: v1.0.0 · **Date**: 2026-03-31

---

## Documents

- [`ORCHESTRATOR_HARNESS_GUIDE.md`](ORCHESTRATOR_HARNESS_GUIDE.md) — Core architecture reference. System prompt structure, coordinator design, agent definitions, quality systems, and complete template library.
- [`PROMPT_ANATOMY.md`](PROMPT_ANATOMY.md) — All 7 prompts extracted verbatim with structural annotations, cross-prompt patterns, and 4 reusable templates (Orchestrator, Read-only specialist, Verification, Guide).
- [`skills/`](skills/) — Orchestration skill files for Claude Code.

## Skills

Copy to `~/.claude/skills/` to invoke as `/skill-name`.

| Skill | File | When |
|-------|------|------|
| `/orchestrate` | [skills/orchestrate.md](skills/orchestrate.md) | Full 4-phase multi-agent workflow |
| `/research` | [skills/research.md](skills/research.md) | Parallel exploration before implementation |
| `/synthesize` | [skills/synthesize.md](skills/synthesize.md) | Research findings → implementation spec |
| `/spec` | [skills/spec.md](skills/spec.md) | Write a plan before touching code |
| `/verify` | [skills/verify.md](skills/verify.md) | Independent verification — gate on VERDICT: PASS |
| `/handoff` | [skills/handoff.md](skills/handoff.md) | Review a worker prompt before sending |

---

한국어 → [`ko/`](ko/)
