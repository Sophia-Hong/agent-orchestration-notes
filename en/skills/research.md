---
description: Break a codebase investigation into independent angles and run Explore agents in parallel. Use before implementation, when diagnosing a bug, or when mapping blast radius of a change.
---

Decompose the current question into 2–3 independent investigation angles, then spawn parallel Explore workers.

## Decomposition criteria

- Each angle covers a different file area or module boundary
- No worker depends on another worker's output
- Each worker can finish within its context window

## Every Explore worker prompt must include

1. **Scope** — which files, directories, or patterns to look at
2. **What to find** — specifically what the worker is looking for
3. **Purpose sentence** — "This research will inform [next step] — focus on [emphasis]"
4. **Output format** — "Report file paths, line numbers, and function signatures. Do not modify files."

## Parallel launch

Put all Agent calls in a single message. Do not serialize independent research.

When results arrive, use `/synthesize` to turn findings into a spec before spawning any implementation worker.

---

Decompose now and launch the workers.
