---
description: Write an implementation spec before touching code. Use when a task needs a clear plan first. Runs /research if the codebase isn't sufficiently understood yet.
---

Write an implementation spec before any code changes.

## Check: is the codebase understood?

- Yes → write the spec directly
- No → run `/research` first, then return here when findings arrive

## Spec structure

### Problem / goal
One sentence: what changes, and why.

### Change scope
```
Files to modify:
  - [path]: [what part, why]
  - [path]: [what part, why]

Files not to touch:
  - [path]: [reason]
```

### Approach
Step by step: what comes first and why. Call out ordering dependencies.

### Definition of done
- [ ] Test command that must pass
- [ ] Typecheck / lint
- [ ] Observable behavior to verify (curl example, CLI output, etc.)

### Risks
- Expected side effects
- Behavior that changes for existing callers
- Anything hard to reverse

---

After writing the spec, ask for confirmation or offer to start implementation immediately.
