---
description: Review a worker prompt for quality before sending. Use any time you're about to delegate to an agent and want to catch problems first.
---

Review the worker prompt below before sending. Check every item.

## Zero-context completeness

This worker has not seen the conversation. The prompt must stand alone.

- [ ] Is the background context sufficient?
- [ ] Does it contain "as we discussed", "based on your findings", or "as mentioned"? → Remove them. Synthesize the relevant facts directly.
- [ ] Is it clear what the worker must do?

## Specificity

The worker should be able to execute, not interpret.

- [ ] Are file paths included? (if known)
- [ ] Are line numbers or function names included? (if known)
- [ ] Is "done" defined? (commit hash, test passage, output format, etc.)

## Scope

- [ ] Is there a "do not modify files" constraint if this is read-only work?
- [ ] Is the scope narrow enough for one worker to finish cleanly?

## Purpose statement

- [ ] Is there one sentence explaining what this work feeds into?
  ("This research will inform an implementation spec — focus on X.")

---

If any item fails, propose a revised prompt. If all pass, send the worker.
