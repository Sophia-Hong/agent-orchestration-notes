---
description: Spawn an independent verification agent after implementation. Required after 3+ file edits, backend/API changes, or infrastructure changes. Do not report completion before VERDICT: PASS.
---

Spawn a verification worker now for the implementation that just completed.

Collect the following and build the worker prompt:

1. **Original request** — the user's exact words, not a summary
2. **Files changed** — complete list of modified, created, or deleted files
3. **Approach** — how it was implemented, in 2–3 sentences
4. **Plan file** — path if one exists

Tell the worker explicitly:
- "Your job is to break this, not to confirm it works."
- "Do not modify any project files."
- "You MAY write ephemeral scripts to /tmp — clean up after."
- "End with exactly: VERDICT: PASS, VERDICT: FAIL, or VERDICT: PARTIAL"

=== AFTER THE VERDICT ===

**PASS**
Spot-check: re-run 2–3 commands from the verifier's report. Every PASS check must have a Command run block — if any lacks one or the output diverges, resume the verifier with the specifics. If spot-check passes, report completion to the user.

**FAIL**
Read the failure. Fix the root cause — not the symptom. Continue the same verifier with SendMessage (it holds the error context). Repeat until PASS.

**PARTIAL**
Report to the user: what passed, what could not be verified, and why.
Do not upgrade PARTIAL to PASS — only the verifier assigns verdicts.
