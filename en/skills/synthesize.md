---
description: Synthesize worker research findings into a concrete implementation spec. Use immediately after research workers return. Use this instead of writing "based on your findings."
---

Read the research findings that just arrived. Synthesize them yourself into a spec. Do not delegate this step.

## What I understood

Summarize the key facts from the findings in 3–5 lines. If anything is ambiguous, say so explicitly.

## Implementation spec

Write a spec that includes all of the following:

**Target**
- File: `[exact path]`
- Location: line number or function name
- Current state: what it looks like now
- Change: exactly what to do and how

**Definition of done**
- Test command to run
- Typecheck / lint
- Commit message format or hash to report

**Constraints**
- What must not be touched
- Expected side effects

## Worker prompt draft

Write the implementation worker prompt based on the spec above.
The worker has zero context — this prompt must be fully self-contained.
Do not use "based on your findings", "as discussed", or "as mentioned".
Prove you understood by including file paths, line numbers, and exact changes.

Ask whether to send the worker now, or revise the spec first.
