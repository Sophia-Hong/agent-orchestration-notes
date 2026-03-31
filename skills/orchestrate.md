---
description: Run a complex task as a full multi-agent orchestration workflow. Enforces Research → Synthesis → Implementation → Verification in sequence. Use for features, bug fixes, or refactors touching 3+ files.
---

Follow the orchestration workflow below. Do not skip phases.

## Phase 1: Research (parallel)

Break the task into 2–3 independent investigation angles. Spawn parallel Explore or general-purpose agents — one per angle. Launch all workers in a single message.

Every research worker prompt must include:
- Specific scope to investigate
- "Report file paths, line numbers, and type signatures. Do not modify files."
- One purpose sentence: "This research will inform an implementation spec — focus on [X]."

## Phase 2: Synthesis (you, not a worker)

When worker results arrive, read them yourself. Do not write "based on your findings." Do not hand findings to another worker unprocessed.

A good synthesis spec includes:
- Exact file paths
- Line numbers
- Specifically what changes and how
- Definition of done ("commit and report the hash", "tests pass", etc.)

## Phase 3: Implementation

Use the synthesized spec to spawn an implementation worker. If the research worker already has the right files in context, continue it with SendMessage. If not, spawn fresh.

Tell the worker: "Run relevant tests and typecheck, then commit and report the hash."

## Phase 4: Verification

After implementation completes, spawn an independent verification worker. Do not self-verify. Do not report completion before VERDICT: PASS arrives.

Pass to the verifier:
1. The original user request — verbatim, not summarized
2. All files changed
3. The implementation approach (2–3 sentences)
4. Plan file path if one exists

On FAIL: read the findings, fix the issue, continue the same verifier with SendMessage.
On PARTIAL: report what passed and what could not be verified.

---

Start Phase 1 now.
