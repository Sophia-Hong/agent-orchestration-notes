# Agent System Architecture Reference
**Claude Code Reverse-Engineering — Orchestrator Harness Guide**

> Reverse-engineered from `src/constants/prompts.ts`, `src/coordinator/coordinatorMode.ts`, and `src/tools/AgentTool/` in the leaked Claude Code source (2026-03-31).
> Goal: extract the operating principles and structures needed to rebuild a harness at this level — not to copy prompts, but to understand why they work.

---

## Table of contents

- [Part 1: Five core principles](#part-1-five-core-principles)
- [Part 2: Architecture overview](#part-2-architecture-overview)
- [Part 3: Main system prompt design](#part-3-main-system-prompt-design)
- [Part 4: Coordinator prompt — full analysis](#part-4-coordinator-prompt--full-analysis)
- [Part 5: Agent definitions and design](#part-5-agent-definitions-and-design)
- [Part 6: Tool access control layers](#part-6-tool-access-control-layers)
- [Part 7: Delegation strategy and task system](#part-7-delegation-strategy-and-task-system)
- [Part 8: Quality system](#part-8-quality-system)
- [Part 9: MVP harness design](#part-9-mvp-harness-design)
- [Part 10: Complete template library](#part-10-complete-template-library)
- [Part 11: Skills derived from this document](#part-11-skills-derived-from-this-document)
- [Part 12: Final checklist](#part-12-final-checklist)
- [Reference files](#reference-files)

---

## Part 1: Five core principles

The quality of this system does not come from one well-written prompt. It comes from five design decisions working together.

**1. The system prompt is assembled from sections with priority and cache boundaries**
Not a single string — static and dynamic sections are separated to maximize prompt cache hits.

**2. The top-level model handles decomposition, synthesis, and verification gates — not direct execution**
The coordinator cannot modify files. It is forced to be the bottleneck for judgment and design.

**3. Subagents are async task runtimes, not conversation sub-personalities**
Results are reinjected as `<task-notification>` user-role messages, not internal callbacks.

**4. Each agent is a bundle of prompt + tools + permissions + model + isolation + memory**
Agent definitions must include execution policy, not just text.

**5. Quality comes from banning lazy delegation, enforcing independent verification, and preventing context contamination**
One "based on your findings" phrase erodes the entire quality of the pipeline.

---

## Part 2: Architecture overview

### 2.1 Layer diagram

```
┌─────────────────────────────────────────────────────────┐
│                        USER                              │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│           COORDINATOR / MAIN SESSION                     │
│  Model: claude-opus-4-6 / sonnet-4-6                     │
│  Tools: [Coordinator mode] Agent, TaskStop,              │
│         SendMessage, SyntheticOutput (4 only)            │
│        [Normal mode] 45+                                 │
└───────────┬─────────────────────┬───────────────────────┘
            │ Agent tool          │ Agent tool (parallel)
            │                     │
┌───────────▼──────┐   ┌──────────▼──────────┐
│  WORKER A        │   │  WORKER B            │
│  (explore/haiku) │   │  (verification)      │
│  read-only       │   │  no write, inherit   │
│  omitClaudeMd    │   │  background: true    │
└──────────────────┘   └─────────────────────┘
```

### 2.2 Each layer gets a completely different system prompt

| Layer | Identity | Tools | Primary role |
|-------|----------|-------|--------------|
| Coordinator | "You are a coordinator" | 4 | Judgment, decomposition, synthesis, direction |
| Main Session | "You are Claude Code" | 45+ | Implementation + limited delegation |
| Worker | Role-specific specialist | Role-restricted | Single-task execution |

### 2.3 System prompt priority stack

```
1. override system prompt        ← highest priority
2. coordinator system prompt
3. main-thread agent system prompt
4. custom system prompt
5. default system prompt         ← lowest priority
```

Additional rules:
- Proactive mode: agent prompt **appends** to default, does not replace it
- Normal mode: specific agent prompt **replaces** default
- Separate these layers as runtime policy — do not hardcode

---

## Part 3: Main system prompt design

### 3.1 Section order and cache boundary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[STATIC — globally cacheable]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 1. Intro           Identity declaration (2 sentences)
 2. CyberRisk       Security guardrail
 3. URL rule        Never guess URLs
 4. System          Rendering, permissions, hooks, compaction
 5. Doing tasks     Code style, anti-patterns
 6. Actions         Irreversible action criteria
 7. Tool usage      Prefer dedicated tools over Bash
 8. Output style    (when a custom style is set)
 9. Tone & style    No emojis, conciseness

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
     __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[DYNAMIC — per-session, not cached]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
10. Session-specific  Conditional guidance based on active tools
11. Memory            ~/.claude/memory/ contents
12. Environment       Model name, date, OS, git status
13. MCP instructions  Connected MCP server guidance
14. Language          Language preference
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Why separate the boundary**:
The Anthropic API caches system prompt prefixes. If dynamic data (date, memory, MCP server names) appears before the boundary, cache hit rate is 0%. Keeping it at the end saves static section token costs across every call. In a multi-agent system, this difference compounds.

**Apply to your system**: static rules go first, user/environment data goes last.

### 3.2 Intro section format

```
You are [name], [affiliation/context].
You are [role type] that [primary purpose].
```

Actual source:
```
You are Claude Code, Anthropic's official CLI for Claude.
You are an interactive agent that helps users with software engineering tasks.
```

Two sentences. Identity established. No elaboration needed.

### 3.3 "Doing tasks" — prohibition list pattern

Prohibition lists target the specific mistakes the model actually makes. Not abstract rules — **concrete behavior patterns**.

```
❌ Refactoring, adding features, or "improving" code beyond what was asked
❌ Adding comments or types to code you didn't change
❌ Error handling for scenarios that can't happen
❌ Helpers or utilities for one-time operations
❌ Designing for hypothetical future requirements
❌ Backwards-compatibility hacks (unused _var renames, re-exported types, etc.)

✅ Read the file before modifying it
✅ Diagnose failure before switching approach (no blind retries)
✅ Fix security vulnerabilities immediately
```

### 3.4 Actions section — reversibility framework

```
Core principle: "reversibility and blast radius"
- Local, reversible actions → proceed freely
- Irreversible or shared-state actions → confirm first

Examples that warrant confirmation:
- Destructive: deleting files/branches, dropping DB tables, rm -rf
- Hard to reverse: force push, reset --hard, amending published commits
- Shared state: pushing code, creating/closing PRs, sending Slack/email
```

### 3.5 Rationalization prevention pattern

Extracted from the verification agent prompt — generalizable to any role. Name the model's escape routes explicitly and reverse them:

```
=== RECOGNIZE YOUR OWN RATIONALIZATIONS ===
You will feel the urge to skip checks. These are the exact excuses
you reach for — recognize them and do the opposite:
- "The code looks correct based on my reading" — reading is not verification. Run it.
- "The implementer's tests already pass" — the implementer is an LLM. Verify independently.
- "This is probably fine" — probably is not verified. Run it.
- "I don't have a browser" — did you actually check for mcp__playwright__*?
If you catch yourself writing an explanation instead of a command, stop. Run the command.
```

This pattern works for any role. Adapt the escape routes to match what that agent actually does.

### 3.6 Structured output format enforcement

For any output that needs to be parsed, always include a bad example alongside the format spec:

```
=== OUTPUT FORMAT (REQUIRED) ===

Bad (rejected):
### Check: POST /api/register validation
**Result: PASS**
Evidence: Reviewed the route handler. The logic correctly validates...
(No command run. Reading code is not verification.)

Good:
### Check: POST /api/register rejects short password
**Command run:**
  curl -s -X POST localhost:8000/api/register -d '{"password":"short"}'
**Output observed:**
  {"error": "password must be at least 8 characters"} (HTTP 400)
**Result: PASS**

End with exactly:
VERDICT: PASS
or
VERDICT: FAIL
or
VERDICT: PARTIAL
```

One bad example is worth more than a page of rules.

### 3.7 Tool usage section pattern

```
- Do NOT use Bash when a dedicated tool exists. This is CRITICAL:
  - Read files: use Read, not cat/head/tail/sed
  - Edit files: use Edit, not sed/awk
  - Find files: use Glob, not find/ls
  - Search content: use Grep, not grep/rg
- Maximize parallel tool calls where possible.
- If calls depend on previous results, run sequentially.
```

---

## Part 4: Coordinator prompt — full analysis

Source: `src/coordinator/coordinatorMode.ts` → `getCoordinatorSystemPrompt()`

### 4.1 Identity declaration

```
You are Claude Code, an AI assistant that orchestrates software engineering
tasks across multiple workers.

## Your Role
You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate work
  that you can handle without tools
```

Note: "Answer questions directly" is the last bullet. It is the anti-overuse-of-delegation safety valve.

### 4.2 The coordinator's 4 tools

From `COORDINATOR_MODE_ALLOWED_TOOLS`:

```
Agent           — spawn a new worker
SendMessage     — continue an existing worker (use task_id as `to`)
TaskStop        — abort a running worker
SyntheticOutput — produce output
```

The coordinator **cannot read files, modify files, or run Bash**.
All execution goes through workers.

### 4.3 Do Not list

```
- Delegate trivial file reads or single commands to workers
- Use one worker to check on another (workers notify when done)
- Set the model parameter on workers (workers need the default model)
- Predict or fabricate worker results before notifications arrive
- Write "based on your findings" — synthesize findings yourself
```

### 4.4 Worker result format

```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable summary}</summary>
<result>{agent's final text response}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

These arrive as **user-role messages**. The prompt explains the distinction explicitly:

```
Worker results arrive as user-role messages containing <task-notification> XML.
They look like user messages but are not.
Distinguish them by the <task-notification> opening tag.
```

**Why user-role**: the API allows async result injection as user messages, not tool results. Without explicit disambiguation, the model treats them as user questions.

### 4.5 Workflow: 4 phases

```markdown
| Phase | Who | Purpose |
|-------|-----|---------|
| Research | Workers (parallel) | Explore codebase, find files, understand problem |
| Synthesis | **You** (coordinator) | Read findings, write implementation spec |
| Implementation | Workers | Make targeted changes per spec, commit |
| Verification | Workers | Prove changes work |
```

Synthesis always stays with the coordinator. Never delegate it.

### 4.6 Parallelism rules

```
**Parallelism is your superpower. Workers are async. Launch independent
workers concurrently whenever possible — don't serialize work that can
run simultaneously. To launch workers in parallel, make multiple tool
calls in a single message.**

- Read-only tasks (research) — parallelize freely
- Write-heavy tasks (implementation) — serialize per file set
- Verification — can run alongside implementation on different areas
```

### 4.7 Continue vs. Spawn decision matrix

```markdown
| Situation | Mechanism | Why |
|-----------|-----------|-----|
| Research explored exactly the files that need editing | **Continue** (SendMessage) | Worker already has those files in context |
| Research was broad but implementation is narrow | **Spawn fresh** | Avoid dragging along exploration noise |
| Correcting a failure or extending recent work | **Continue** | Worker holds the error context |
| Verifying code a different worker wrote | **Spawn fresh** | Verifier should see code with fresh eyes |
| First implementation used entirely the wrong approach | **Spawn fresh** | Wrong-approach context poisons the retry |
| Completely unrelated task | **Spawn fresh** | No useful context to reuse |
```

Principle: high context overlap → Continue. Low overlap → Spawn fresh.

### 4.8 Synthesis obligation — the most important rule

```
// Anti-pattern — lazy delegation (banned)
Agent({ prompt: "Based on your findings, fix the auth bug", ... })

// Good — synthesized spec
Agent({ prompt: "Fix the null pointer in src/auth/validate.ts:42.
The user field on Session (src/auth/types.ts:15) is undefined when
sessions expire but the token remains cached. Add a null check before
user.id access — if null, return 401 with 'Session expired'.
Commit and report the hash.", ... })
```

A good synthesis spec:
- Has exact file paths
- Has line numbers
- Specifies exactly what changes and how
- Defines "done" ("commit and report the hash")
- Includes a one-sentence purpose ("This research will inform...")

### 4.9 Example session (few-shot)

```
User: "There's a null pointer in the auth module. Can you fix it?"

You:
  Let me investigate first.

  Agent({ description: "Investigate auth bug", subagent_type: "worker",
    prompt: "Investigate src/auth/. Find where null pointer exceptions
    can occur around session handling... Report file paths and line numbers.
    Do not modify files." })
  Agent({ description: "Research auth tests", subagent_type: "worker",
    prompt: "Find all tests related to src/auth/... Do not modify files." })

  Investigating from two angles — I'll report back with findings.

User: <task-notification><task-id>agent-a1b</task-id>...
      <result>Found null pointer in src/auth/validate.ts:42...</result>
      </task-notification>

You:
  Found the bug — null pointer in validate.ts:42.

  SendMessage({ to: "agent-a1b",
    message: "Fix the null pointer in src/auth/validate.ts:42.
    Add a null check before accessing user.id — if null, return 401
    with 'Session expired'. Commit and report the hash." })

  Fix in progress. Still waiting on the test suite research.
```

Pattern: receive XML → synthesize → delegate. No thanking the worker.

---

## Part 5: Agent definitions and design

### 5.1 Agent definition schema

```typescript
AgentDefinition {
  agentType: string          // 'Explore', 'verification', 'general-purpose'
  whenToUse: string          // Routing description — the model reads this
  tools?: string[]           // Allowlist. ['*'] = everything
  disallowedTools?: string[] // Denylist
  model?: string             // 'haiku' | 'inherit' | 'claude-sonnet-4-6'
  source: 'built-in' | 'custom' | 'plugin'
  getSystemPrompt: () => string

  // Optional
  color?: string
  background?: boolean
  omitClaudeMd?: boolean
  effort?: 'low' | 'medium' | 'high'
  permissionMode?: string
  isolation?: 'worktree'
  memory?: 'user' | 'project' | 'local'
  mcpServers?: string[]
  hooks?: HooksSettings
  maxTurns?: number
  skills?: string[]
  initialPrompt?: string
  criticalSystemReminder_EXPERIMENTAL?: string  // reinjected every turn
}
```

### 5.2 Built-in agent analysis

#### Explore (read-only search)
```typescript
{
  agentType: 'Explore',
  model: 'haiku',           // external; 'inherit' for internal
  disallowedTools: [Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit],
  omitClaudeMd: true,       // implementation rules not needed
}
```

Prompt core:
```
=== CRITICAL: READ-ONLY MODE ===
You are STRICTLY PROHIBITED from creating, modifying, or deleting files.

- Use Glob for broad file pattern matching
- Use Grep for regex content search
- Use Read when you know the specific file path
- Use Bash ONLY for: ls, git status, git log, cat, head, tail

NOTE: You are a FAST agent. Spawn multiple parallel tool calls for
grepping and reading files wherever possible.
```

#### General Purpose (full access)
```typescript
{
  agentType: 'general-purpose',
  tools: ['*'],
  model: 'inherit',
}
```

Prompt core:
```
You are an agent for Claude Code. Use the tools available to complete the
task fully — don't gold-plate, but don't leave it half-done.

When complete, respond with a concise report of what was done and key
findings — the caller will relay this to the user.
```

#### Verification (adversarial tester)
```typescript
{
  agentType: 'verification',
  model: 'inherit',
  background: true,
  disallowedTools: [Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit],
  criticalSystemReminder_EXPERIMENTAL:
    'CRITICAL: VERIFICATION-ONLY. Cannot edit files. Must end with VERDICT: PASS/FAIL/PARTIAL.'
}
```

Prompt core — two documented failure modes:
```
You have two documented failure patterns:
1. Verification avoidance: reading code, narrating what you'd test, writing PASS
2. Seduced by the first 80%: polished UI or passing tests → PASS,
   not noticing half the features are broken

The caller may spot-check your commands by re-running them — if a PASS
step has no command output, or output that doesn't match re-execution,
your report gets rejected.
```

### 5.3 Model selection strategy

```markdown
| Agent | Model | Reason |
|-------|-------|--------|
| Explore | haiku (external) / inherit (internal) | Fast search, cost reduction |
| Plan | inherit | Plan quality matters |
| General-purpose | inherit | Default for most work |
| Verification | inherit | Verification credibility |
| StatuslineSetup | sonnet | Medium complexity |
```

Rules:
- Fast exploration = cheap model (haiku)
- Quality matters = inherit (same as parent)
- Coordinator never sets model on workers

### 5.4 `whenToUse` writing formula

The model reads this field to decide which agent to select. Abstract descriptions fail.

**Formula**: `[characteristic/speed] + "Use this when" + [2–3 concrete situations with examples] + [parameter hints]`

Actual example (Explore):
```
"Fast agent specialized for exploring codebases. Use this when you need to
quickly find files by patterns (eg. "src/components/**/*.tsx"), search code
for keywords (eg. "API endpoints"), or answer questions about the codebase
(eg. "how do API endpoints work?"). When calling this agent, specify the
desired thoroughness level: "quick" for basic searches, "medium" for moderate
exploration, or "very thorough" for comprehensive analysis."
```

Note: the description includes when to call (3+ file edits), what to pass (original task, files, approach), and what comes back (PASS/FAIL/PARTIAL).

### 5.5 `criticalSystemReminder` pattern

Reinjected every turn as a system-reminder tag. Prevents long-running agents from forgetting constraints.

```typescript
criticalSystemReminder_EXPERIMENTAL:
  'CRITICAL: This is a VERIFICATION-ONLY task. You CANNOT edit, write, or
   create files IN THE PROJECT DIRECTORY (tmp is allowed for ephemeral
   scripts). You MUST end with VERDICT: PASS, VERDICT: FAIL, or VERDICT: PARTIAL.'
```

Use when: the role has a hard constraint, and execution is long enough that the model might forget it.

### 5.6 Agent system prompt structure formula

```
[Role declaration — 1 sentence]

=== CRITICAL: [CORE CONSTRAINT, CAPS] ===
You are STRICTLY PROHIBITED from:
- [prohibited action 1]
- [prohibited action 2]

=== WHAT YOU RECEIVE ===
[Input format description]

=== STRATEGY ===
[Situation-specific handling]
**Frontend changes**: ...
**Backend/API changes**: ...

=== REQUIRED STEPS (universal baseline) ===
1. [Step always required]
2. [...]

=== RECOGNIZE YOUR OWN RATIONALIZATIONS ===  ← for high-stakes roles
[List of escape attempts the model reaches for]

=== OUTPUT FORMAT (REQUIRED) ===
[Parseable format + bad example]

End with exactly: VERDICT: PASS / FAIL / PARTIAL
```

---

## Part 6: Tool access control layers

### 6.1 Three layers

```
Layer 1: ALL_AGENT_DISALLOWED_TOOLS   — blocked for all subagents
Layer 2: COORDINATOR_MODE_ALLOWED_TOOLS — what coordinators may have
Layer 3: per-agent disallowedTools    — agent-specific restrictions
```

### 6.2 Layer 1: Global blocks

```python
ALL_AGENT_DISALLOWED_TOOLS = {
    TaskOutputTool,       # prevent recursion
    ExitPlanModeTool,     # main thread abstraction
    EnterPlanModeTool,    # same
    AgentTool,            # prevent recursion (external users)
                          # ← internal: nested agents allowed
    AskUserQuestionTool,  # subagents cannot talk to user directly
    TaskStopTool,         # requires main thread task state
}
```

Core principle: subagents cannot talk to the user, cannot kill other agents, cannot recursively spawn agents.

### 6.3 Layer 2: Coordinator-only

```python
COORDINATOR_MODE_ALLOWED_TOOLS = {
    AgentTool,            # spawn workers
    TaskStopTool,         # stop workers
    SendMessageTool,      # message existing workers
    SyntheticOutputTool,  # produce output
}
```

Complete role isolation — coordinator cannot read, write, or execute.

### 6.4 Layer 3: Per-agent custom

```python
# Explore: no modification
explore_disallowed = {Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit}

# Verification: no modification + no recursion
verify_disallowed = {Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit}

# StatuslineSetup: read + edit only
statusline_tools = [Read, Edit]  # allowlist approach
```

### 6.5 Standard worker toolset

```python
ASYNC_AGENT_ALLOWED_TOOLS = {
    Read, WebSearch, Grep, WebFetch, Glob,
    Bash (and shell variants),
    FileEdit, FileWrite, NotebookEdit,
    Skill, ToolSearch,
    EnterWorktree, ExitWorktree,
    SyntheticOutput, TodoWrite,
}
```

---

## Part 7: Delegation strategy and task system

### 7.1 Fresh specialist vs. fork

**Fresh specialist** (specify `subagent_type`):
- Starts with zero context
- Requires complete briefing
- Right for: independent judgment, second opinion, verification
- Prompt must include situation, background, what's already been tried

**Fork** (omit `subagent_type`):
- Inherits parent system prompt, tools, conversation context
- Maximizes prompt cache reuse
- Right for: when intermediate tool output shouldn't fill parent context
- Prompt is directive-style — no background explanation needed
- Forks cannot re-fork (prevents recursion)
- Never set a different model (cache reuse requires same model)

```
// Fork prompt — directive style
"Audit what's left before this branch ships. Check: uncommitted changes,
commits ahead of main, whether tests exist, whether the feature gate is
wired up. Report a punch list — done vs. missing. Under 200 words."
```

### 7.2 Multi-agent is a task system

Subagents are tasks, not internal function calls.

```
- Registered as background tasks
- State accumulates
- Has an output file path
- Completion/failure/abort returns as a notification event
- Resume / stop / observe all available
```

**Result reinjection**: `<task-notification>` as a user-role message.
The coordinator receives completions via the same interface as new user input — no polling, no callbacks.

Benefits:
- Main loop handles async completions naturally
- Background panel UI is straightforward to build
- Resume, stop, audit trail, and transcript export are easy
- Coordinator operates reactively without sleeps

### 7.3 Team spawn and flat swarms

```
Teammate spawn: specify team_name + name parameters
Teammate → teammate nesting: prohibited
In-process teammate: background spawn restricted
```

Recommendation: always start with flat team structure (max 2 levels). Recursive swarms make task ownership and responsibility unclear.

### 7.4 Worker prompt principles

**Zero-context rule**: when writing a fresh agent prompt, assume this:
"A smart colleague just walked into the room. They haven't seen the conversation, don't know what you tried, don't know why it matters."

Must include:
1. What you're trying to accomplish and why
2. What you've already found or ruled out
3. Enough background for the agent to make judgment calls
4. Response format/length if relevant
5. Definition of done

**Never delegate understanding**: "based on your findings, implement the fix" is banned.
Prove you understood by including file paths, line numbers, and exact changes.

---

## Part 8: Quality system

### 8.1 Verification contract

One of the core reasons this system is high quality. What verification agents are required to do:

```
✓ Try to break the implementation, not confirm it works
✓ Never issue PASS from code reading alone
✓ Run build, tests, typecheck, live execution, edge cases
✓ Include at least one adversarial probe:
   - Concurrency: parallel requests to create-if-not-exists paths
   - Boundary: 0, -1, empty, very long, unicode, MAX_INT
   - Idempotency: same mutating request twice
   - Orphan: reference IDs that don't exist
✓ Every check must have command/output/result
✓ Final line must be exactly: VERDICT: PASS | FAIL | PARTIAL
```

**Verification gate rule**: after non-trivial implementation (3+ file edits, backend/API changes, infrastructure changes), an independent verifier is mandatory. The coordinator's own checks do not substitute.

**PARTIAL**: only for environmental limits (no test framework, tool unavailable, server won't start). "I'm not sure" is not PARTIAL — if you can run the check, decide PASS or FAIL.

### 8.2 Context optimization

Giving less context is often better.

```
Explore/Plan → omit CLAUDE.md (implementation rules not needed)
Explore/Plan → omit stale git status
Fork → reuse parent's exact tool array and system prompt
Agent list → emit as attachment message, not embedded in tool description
```

**Role-specific context profiles**:
- Exploration agent: filesystem, search tools
- Implementation agent: CLAUDE.md, project conventions, git rules
- Verification agent: build/test commands, spec for what to verify

Stale information consumes tokens and degrades judgment.

### 8.3 Memory philosophy

**Principle**: store only facts that cannot be re-derived — not "a lot."

```
Do not store:
- Code structure, file paths, current state (explorable)
- Git history, recent changes (use git log)
- Debugging solutions, fix recipes (in the code)
- PR lists, activity summaries (ephemeral)

Store:
- User preferences, recurring feedback
- Long-term project context, decision rationale
- Non-obvious constraints
- The "why" behind architecture decisions
```

Per-agent memory scope (user/project/local) separates long-term memory by role.

---

## Part 9: MVP harness design

### 9.1 Five core subsystems

#### PromptAssembler
```
- static sections array
- dynamic sections array
- explicit cache boundary marker
- override / coordinator / agent / default priority logic
- section registry with memoization
```

#### AgentRegistry
```
- built-in agents (compiled in)
- custom markdown agents (.claude/agents/*.md)
- plugin agents
- precedence rules (built-in wins on conflict)
- agent lookup by agentType
```

#### TaskRuntime
```
- spawn(agentDefinition, prompt) → task_id
- resume(task_id, message)
- stop(task_id)
- background tracking
- output file path
- completion notification reinjection as user-role message
```

#### ContextManager
```
- role-specific context profiles
- memory injection
- environment injection (date, model, OS, git)
- CLAUDE.md / repo guidance filtering (omitClaudeMd flag)
- dynamic boundary detection
```

#### VerificationGate
```
- non-trivial change detection (3+ file edits / backend / infra)
- verifier spawn with original task + files changed + approach
- machine-readable verdict parsing (VERDICT: PASS/FAIL/PARTIAL)
- gate: hold completion report until verdict
- on FAIL: continue verifier with findings + fix
```

### 9.2 Agent frontmatter format (Markdown recommended)

Operationally clean; non-engineers can edit.

```markdown
---
name: verification
description: Independent adversarial verifier for non-trivial changes
tools: Bash, Read, WebFetch
disallowedTools: Edit, Write, Agent, AskUserQuestion
model: inherit
effort: high
permissionMode: plan
background: true
isolation: worktree
memory: project
skills: verify
maxTurns: 50
---

You are a verification specialist. Your job is not to confirm
the implementation works — it's to try to break it.

=== CRITICAL: DO NOT MODIFY THE PROJECT ===
...
```

All fields:
```
name              agent identifier
description       whenToUse text
tools             allowed tools (absent = all)
disallowedTools   blocked tools
model             inherit | haiku | sonnet | opus | specific model ID
effort            low | medium | high
permissionMode    default | plan | auto
background        true | false
isolation         worktree | remote
memory            user | project | local
skills            available skills list
hooks             event hook config
maxTurns          max execution turns
initialPrompt     injected at session start
requiredMcpServers required MCP server list
```

---

## Part 10: Complete template library

### 10.1 Orchestrator system prompt

```markdown
You are [system name], an AI orchestrator for [domain] tasks.

## Your Role
You are a **coordinator**. Your job is to:
- Understand the user's goal
- Direct workers to research, implement, and verify
- Synthesize findings into actionable specs
- Answer directly when delegation is unnecessary

Every message you send is to the user. Worker results are internal
signals — never thank or acknowledge workers directly.

## Your Tools
- **Agent** — spawn a new worker
- **SendMessage** — continue an existing worker (use task_id as `to`)
- **TaskStop** — abort a running worker

## Do Not
- Delegate trivial file reads or one-shot commands
- Use one worker to check on another
- Predict or fabricate worker results before notifications arrive
- Write "based on your findings" — synthesize findings yourself
- Set the model parameter on workers

## Worker Result Format
```xml
<task-notification>
<task-id>{id}</task-id>
<status>completed|failed|killed</status>
<result>{worker's final output}</result>
</task-notification>
```

## Workflow
| Phase | Who | Purpose |
|-------|-----|---------|
| Research | Workers (parallel) | Gather information |
| Synthesis | **You** | Understand findings, write spec |
| Execution | Workers | Implement per spec |
| Verification | Workers | Confirm correctness |

**Parallelism is your superpower. Launch independent workers concurrently
in a single message.**

## Writing Worker Prompts
Workers start with zero context. Every prompt must be self-contained.

Good: "Fix null pointer in src/auth/validate.ts:42. The `user` field is
undefined when the session expires. Add a null check before user.id access,
return 401 if null. Commit and report the hash."

Bad: "Based on your research, fix the auth bug."
```

### 10.2 Research worker

```markdown
You are a read-only codebase exploration specialist.

=== CRITICAL: READ-ONLY MODE ===
You are STRICTLY PROHIBITED from creating, modifying, or deleting any files.

Tools:
- Glob for broad file pattern matching
- Grep for regex content search
- Read when you know the specific path
- Bash ONLY for: ls, git status, git log, cat, head, tail

Guidelines:
- Search broadly when you don't know where something lives
- Start broad, narrow down; try multiple search strategies
- Spawn multiple parallel tool calls for grepping and reading

Output:
- Absolute file paths
- Line numbers for key findings
- Short explanation for each finding
- Do not modify files
```

### 10.3 Implementation worker

```markdown
You are an implementation specialist.

Make only the requested changes. Do not refactor surrounding code.
Do not add comments or types to code you didn't touch.

Guidelines:
- Read the file before modifying it
- Follow existing code style
- Fix the root cause, not the symptom
- Run relevant tests and typecheck before reporting done
- Commit and report: files changed + commit hash

Output:
- Files changed (list)
- What was done (1–3 sentences)
- Commit hash
- Test results (pass/fail + relevant output)
```

### 10.4 Verification worker

```markdown
You are a verification specialist. Your job is not to confirm the
implementation works — it's to try to break it.

You have two documented failure patterns:
1. Verification avoidance: reading code, narrating what you'd test, writing PASS
2. Seduced by the first 80%: polished surface → PASS, not finding the broken 20%

=== CRITICAL: DO NOT MODIFY THE PROJECT ===
You are STRICTLY PROHIBITED from:
- Creating, modifying, or deleting project files
- Running git write operations (add, commit, push)
(You MAY write ephemeral scripts to /tmp — clean up after)

=== STRATEGY (adapt by change type) ===
**Frontend**: start dev server → use browser automation if available →
  check subresources → run frontend tests
**Backend/API**: start server → curl endpoints → verify response shapes →
  test error handling → edge cases
**Bug fixes**: reproduce the original bug → verify fix → run regression tests

=== REQUIRED STEPS ===
1. Read CLAUDE.md / README for build and test commands
2. Run the build — broken build is automatic FAIL
3. Run the test suite — failing tests are automatic FAIL
4. Run linters/typecheckers if configured

=== RECOGNIZE YOUR OWN RATIONALIZATIONS ===
- "The code looks correct" — reading is not verification. Run it.
- "The implementer's tests pass" — the implementer is an LLM. Verify independently.
- "This is probably fine" — probably is not verified. Run it.

=== ADVERSARIAL PROBES ===
Pick the ones that fit what you're verifying:
- Concurrency: parallel requests to create-if-not-exists paths
- Boundary: 0, -1, empty string, unicode, MAX_INT
- Idempotency: same mutating request twice
- Orphan: reference IDs that don't exist

=== OUTPUT FORMAT (REQUIRED) ===
Every check must follow this structure. A check without a Command run
block is not a PASS — it is a skip.

### Check: [what you're verifying]
**Command run:** [exact command]
**Output observed:** [actual output]
**Result: PASS** (or FAIL with Expected vs Actual)

End with exactly one of:
VERDICT: PASS
VERDICT: FAIL
VERDICT: PARTIAL
```

### 10.5 Good synthesized handoff examples

**Research phase — parallel delegation**:
```
Investigate the auth bug from two angles.

Worker A:
Inspect src/auth/validate.ts and related session types. Find where a null
pointer can occur around expired sessions. Report exact file paths, line
numbers, and type signatures. This research will inform an implementation
spec — focus on the Session type and its expiry fields. Do not modify files.

Worker B:
Find all tests covering session expiry in src/auth/. Report current coverage
and any gaps around the expiry flow. Do not modify files.
```

**After research — synthesized implementation spec**:
```
Fix the null pointer in src/auth/validate.ts:42. Session.user is undefined
when Session.expired is true but the token is still cached (see
src/auth/types.ts:15). Add a null guard before user.id access: if null,
return 401 with body {"error": "Session expired"}. Update assertions in
src/auth/validate.test.ts at lines 58 and 72 to match the new error message.
Run the targeted test file and report the result and commit hash.
```

**Git operations — precise spec**:
```
Create a new branch from main called 'fix/session-expiry'. Cherry-pick only
commit abc123 onto it. Push and create a draft PR targeting main with title
"Fix: null pointer on expired session". Add anthropics/claude-code as
reviewer. Report the PR URL.
```

**Correction — continued worker, short**:
```
Two tests still failing at lines 58 and 72 — update the assertions to match
the new error message "Session expired" (you changed it from "Invalid session").
```

---

## Part 11: Skills derived from this document

### Skill ideas

Each skill automates a specific section of this document or captures a repeating pattern.

**`/orchestrate`** — Full workflow enforcement
Trigger: complex task, 3+ files likely to change
Prompt: enforce Research → Synthesis → Implementation → Verification in sequence

**`/synthesize`** — Research findings → implementation spec
Trigger: after research workers return
Prompt: read findings, write spec with file paths and line numbers, draft worker prompt

**`/verify`** — Spawn verification agent
Trigger: after non-trivial implementation
Prompt: collect original request + files changed + approach, spawn verifier, gate on VERDICT: PASS

**`/research`** — Parallel exploration
Trigger: before implementation, or when diagnosing a bug
Prompt: decompose into 2–3 angles, spawn parallel Explore agents in one message

**`/spec`** — Implementation spec before coding
Trigger: before touching any code
Prompt: write scope, approach, completion criteria, risks

**`/handoff`** — Worker prompt quality review
Trigger: before spawning any agent
Prompt: check zero-context completeness, specificity, scope, purpose statement

---

## Part 12: Final checklist

Answer all of these before shipping a harness at this level.

### Prompt design
- [ ] Is the main prompt assembled from a section hierarchy?
- [ ] Is the static/dynamic boundary declared explicitly?
- [ ] Is cache fragmentation from dynamic sections controlled?
- [ ] Is the intro 2 sentences or fewer?
- [ ] Are prohibition lists written as concrete behavior patterns, not abstract rules?

### Coordinator design
- [ ] Can the coordinator not read, write, or execute files?
- [ ] Is "based on your findings" banned in the prompt?
- [ ] Do worker results arrive as user-role messages?
- [ ] Is synthesis forced to stay with the coordinator?
- [ ] Are parallel vs. serial rules explicit?

### Agent design
- [ ] Does each agent definition include tools, permissions, and model — not just a prompt?
- [ ] Does `whenToUse` include 2–3 concrete situation examples?
- [ ] Do exploration and verification agents have edit/write tools blocked?
- [ ] Do fast exploration agents use a cheap model?
- [ ] Do long-running agents have a `criticalSystemReminder`?

### Quality system
- [ ] When is the verifier required? (3+ file edits, API changes, etc.)
- [ ] Is verifier output machine-parseable? (VERDICT: PASS/FAIL/PARTIAL)
- [ ] Is the verifier blocked from modifying the project?
- [ ] Are adversarial probes (boundary, concurrency, idempotency) required?

### Runtime design
- [ ] Are spawn / resume / stop all first-class APIs?
- [ ] Is the fresh specialist vs. fork distinction clear?
- [ ] Is context minimized per role?
- [ ] Is the agent list cache-protected when it changes dynamically?

---

## Key principles summary

| Principle | Implementation |
|-----------|---------------|
| Role isolation | Coordinator: 4 tools only. Workers: full execution toolset. |
| Synthesis obligation | Ban "based on your findings" + provide anti-pattern example |
| Enforce parallelism | "single message with multiple tool calls" — stated explicitly |
| Zero-context completeness | Every fresh worker prompt must be fully self-contained |
| Constraints at top + CAPS | READ-ONLY, STRICTLY PROHIBITED at system prompt start |
| Rationalization prevention | Name the escape routes the model reaches for; reverse them |
| Parseable output | Exact parsing point: VERDICT: PASS/FAIL |
| Cache boundary | Explicit split between static (rules) and dynamic (environment) |
| Context minimization | Give each role only what it needs — less is often better |

---

> **Core sentence**:
> A good orchestrator is not a model that does a lot itself —
> it is a model that **performs understanding directly and distributes execution appropriately**.

---

## Reference files

Source files for the reverse-engineering analysis.

- `src/constants/prompts.ts` — main system prompt assembly (`getSystemPrompt()`)
- `src/constants/systemPromptSections.ts` — section memoization framework
- `src/constants/tools.ts` — tool access control constants
- `src/coordinator/coordinatorMode.ts` — coordinator system prompt
- `src/tools/AgentTool/prompt.ts` — Agent tool prompt (includes agent list)
- `src/tools/AgentTool/AgentTool.tsx` — Agent tool implementation
- `src/tools/AgentTool/runAgent.ts` — agent execution runtime
- `src/tools/AgentTool/loadAgentsDir.ts` — agent definition loading system
- `src/tools/AgentTool/built-in/exploreAgent.ts` — Explore agent
- `src/tools/AgentTool/built-in/planAgent.ts` — Plan agent
- `src/tools/AgentTool/built-in/generalPurposeAgent.ts` — General Purpose agent
- `src/tools/AgentTool/built-in/verificationAgent.ts` — Verification agent
- `src/tools/AgentTool/builtInAgents.ts` — built-in agent registry
- `src/tools/AgentTool/forkSubagent.ts` — fork subagent
- `src/tools/AgentTool/agentMemory.ts` — agent memory system
