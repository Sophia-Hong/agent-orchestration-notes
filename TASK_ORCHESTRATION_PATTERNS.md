# Task Orchestration Patterns for Knowledge Work

**Derived from**: Claude Code prompt engineering (reverse-engineered 2026-03-31)
**Domain**: General — research, analysis, legal drafting, case review, report writing
**Version**: v1.0.0

> The patterns behind Claude Code's multi-agent system are not about code.
> They solve a universal problem: models skip work, delegate understanding,
> fabricate verification, and serialize what could be parallel.
> This document extracts those patterns for any knowledge task.

---

## Table of Contents

- [Origin: what the code patterns actually solve](#origin-what-the-code-patterns-actually-solve)
- [Five roles for knowledge work](#five-roles-for-knowledge-work)
- [Nine universal patterns](#nine-universal-patterns)
- [Role templates with examples](#role-templates-with-examples)
- [Workflow blueprints](#workflow-blueprints)
- [Anti-pattern catalog](#anti-pattern-catalog)

---

## Origin: what the code patterns actually solve

Claude Code has 7 system prompts across 7 roles. Each prompt solves a
specific failure mode that has nothing to do with programming:

| Code role | Actual problem solved | Knowledge work equivalent |
|-----------|----------------------|--------------------------|
| Coordinator | Forwards results without reading them | "Based on the research, draft the letter" |
| Explore agent | Modifies files when it should only search | Researcher edits source quotes instead of finding them |
| Plan agent | Skips exploration, jumps to design | Analyst recommends strategy without reading the record |
| Verification agent | Reads code and writes PASS without running it | Reviewer skims a draft and says "looks good" |
| General Purpose | Over-engineers or abandons mid-task | Drafter adds unrequested sections or stops mid-argument |
| Guide agent | Hallucinates answers instead of checking docs | Reference lookup that invents citations |

The prompts don't say "be careful." They name the exact failure, script
the counter-behavior, and structurally prevent the shortcut.

That is the transferable insight.

---

## Five roles for knowledge work

```
┌──────────────────────────────────────────────────┐
│  Coordinator (you)                                │
│  Decomposes, synthesizes, gates — never executes  │
│                                                   │
│  ┌────────────┐ ┌──────────┐ ┌────────────────┐  │
│  │ Researcher │ │ Analyst  │ │    Drafter      │  │
│  │ (find)     │ │ (reason) │ │    (write)      │  │
│  └────────────┘ └──────────┘ └────────────────┘  │
│                                                   │
│  ┌──────────────────────────────────────────────┐ │
│  │ Reviewer (verify — independent, adversarial) │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

| Role | What it does | What it must NOT do |
|------|-------------|---------------------|
| **Coordinator** | Reads all results, synthesizes into specs, decides next step | Execute any task directly; forward results unread |
| **Researcher** | Finds sources, extracts quotes, reports with citations | Interpret, recommend, or alter what it finds |
| **Analyst** | Reads researcher output, identifies patterns, recommends strategy | Skip source material; recommend without evidence |
| **Drafter** | Writes the deliverable to spec | Add unrequested content; deviate from spec |
| **Reviewer** | Tries to break the deliverable — factual, logical, structural | Rubber-stamp; review without checking sources |

The Coordinator is always you. The others are workers — agents, or
yourself in a different mode. The separation is logical, not necessarily
mechanical.

---

## Nine universal patterns

### Pattern 1: Identity → Constraint → Strategy → Format

Every role prompt follows this sequence. Reversing any element degrades performance.

```
[Identity]     Who you are and your one-line purpose
[Constraint]   What you are PROHIBITED from doing
[Strategy]     How you approach the work (role-specific)
[Format]       What your output must look like
```

**Why this order matters:**
- Strategy before Constraint → the model rationalizes past prohibitions
  ("I know I shouldn't interpret, but this finding clearly means...")
- Format before Identity → the model optimizes structure before understanding role
- Missing Constraint → the model drifts toward default helpful-assistant behavior,
  which means doing everything at once

**Knowledge work example — Researcher:**
```
[Identity]   You are a legal researcher. Your job is to find and report,
             not to interpret or recommend.
[Constraint] You must NOT: state opinions, recommend actions, editorialize
             findings, omit sources, or paraphrase without quoting.
[Strategy]   Search broadly first. Use multiple search terms. When you find
             a relevant source, extract the exact quote with citation.
             Report what you found and what you didn't find.
[Format]     For each finding: Source → Exact quote → Page/section →
             Relevance to query. End with: Sources checked but yielded
             nothing: [list].
```

### Pattern 2: Constraint depth ∝ accident probability

Don't make every constraint list the same length. The depth of prohibition
should match how easy it is to accidentally violate.

| Role | High-risk accident | Constraint depth |
|------|-------------------|-----------------|
| Researcher | Interpreting instead of reporting | Deep — 5+ specific prohibitions |
| Analyst | Recommending without evidence | Medium — 3 prohibitions + required evidence format |
| Drafter | Adding unrequested content | Light — 2 prohibitions (stick to spec, nothing extra) |
| Reviewer | Rubber-stamping | Deepest — named failure modes, scripted counter-arguments |

A researcher will naturally interpret what it finds. This is a high-frequency
accident. The constraint block must be proportionally detailed.

A drafter given a good spec rarely adds unrequested content. Light constraint.

### Pattern 3: Name the failure mode, don't describe the principle

Claude Code doesn't say "be thorough in verification." It says:

> "You have two documented failure patterns. First, **verification avoidance**:
> when faced with a check, you find reasons not to run it. Second, **seduced
> by the first 80%**: you see a polished surface and feel inclined to pass it."

The name is the tool. Once a model reads "verification avoidance," it can
catch itself: "I'm about to commit verification avoidance." A principle
("be thorough") gives no handle.

**Knowledge work failure modes to name:**

| Role | Named failure mode | What it looks like |
|------|-------------------|-------------------|
| Researcher | **Source amnesia** | Reports a claim without a citation, or cites a source it didn't actually read |
| Researcher | **First-result satisfaction** | Finds one relevant source and stops searching |
| Analyst | **Conclusion-first reasoning** | Decides the answer, then cherry-picks evidence |
| Analyst | **Authority substitution** | "Based on the research..." without stating what the research found |
| Drafter | **Scope creep** | Adds arguments, sections, or caveats not in the spec |
| Drafter | **Template echo** | Produces generic language that could apply to any case |
| Reviewer | **Comfort pass** | "This looks good" without checking a single factual claim |
| Reviewer | **Effort theater** | Lists 5 things checked but none were actually verified against source |

### Pattern 4: The synthesis obligation

This is the most important pattern. It comes from one line in the
coordinator prompt:

> "Never write 'based on your findings' or 'based on the research.'"

The reason: these phrases delegate understanding. The coordinator's
**entire value** is that it reads results, extracts meaning, and writes
specs that prove comprehension.

**In knowledge work, this means:**

When research comes back, the coordinator must:
1. Read every finding
2. State what was found, in their own words, with specific references
3. Write the next instruction that proves they understood

```
// BAD — lazy delegation
"Based on the research, draft a demand letter for the client."

// GOOD — synthesized spec
"Draft a demand letter for Kim v. Park Construction.

Key facts from research:
- Contract signed 2025-08-15, completion deadline 2025-12-01 (Exhibit A, §3.2)
- Final payment of ₩45M withheld; contractor claims additional work (email chain, 2025-11-28)
- Contractor abandoned site 2025-11-20, 11 days before deadline (site log, confirmed by photos dated 11/20-11/25)
- No written change order for additional work (confirmed: searched all project correspondence)

Letter structure:
1. Breach identification: missed deadline + site abandonment
2. Damages: ₩45M withheld payment + ₩12M remediation estimate (Han Engineering quote, 2025-12-10)
3. Demand: full remediation or ₩57M within 14 days
4. Consequence: litigation under Civil Act §544 (right to terminate for breach)

Tone: firm, factual. No emotional language. Every claim must cite a source from the research."
```

The spec doesn't say "write a good letter." It proves the coordinator read
the research and made decisions about structure, tone, and legal basis.

### Pattern 5: Positive allowed list + negative denied list (both required)

Claude Code doesn't just say "don't write files." It also says what the
agent IS allowed to do with each tool.

Without the positive list, the model infers what's "probably safe." Wrong.

**Knowledge work example — Researcher:**
```
You MAY:
- Search databases, case law repositories, document archives
- Read and extract verbatim quotes from sources
- Report that a search yielded no results
- Ask for clarification on the search scope

You must NOT:
- State opinions on what the findings mean
- Recommend a course of action
- Omit a relevant finding because it's unfavorable
- Paraphrase a source without including the original quote
- Report a source you did not actually read
```

Both lists are necessary. The positive list prevents overcaution
("I wasn't sure if I could search that database"). The negative list
prevents the specific failure modes.

### Pattern 6: Verification is adversarial, not confirmatory

The verification agent's opening line is:

> "Your job is not to confirm the implementation works — it's to try to break it."

This inversion changes everything. A confirmatory reviewer asks "does this
look right?" An adversarial reviewer asks "how could this be wrong?"

**In knowledge work:**

A demand letter reviewer should not ask "is this a good letter?"
They should ask:
- Does every factual claim have a cited source?
- If I were opposing counsel, which claim would I attack first?
- Is there a deadline or statute of limitations issue not addressed?
- Does the damages calculation hold up if challenged?
- Is there any claim that contradicts another document in the record?

```
You are a review specialist. Your job is not to confirm the draft is
good — it's to find what opposing counsel will attack.

You have two documented failure patterns. First, comfort pass: you read
the draft, feel that it's professionally written, and approve it without
verifying a single factual claim against the source material. Second,
effort theater: you list checks you "performed" but none involved
actually opening a source document and comparing.

The coordinator may spot-check your review by verifying your citations.
If a "checked" item has no source comparison, your review gets rejected.
```

### Pattern 7: Few-shot only when the behavior is non-inferrable

Claude Code uses few-shot examples in only 2 of 7 prompts:
- Coordinator: because the async work cycle can't be inferred from description
- Verification: because the output format needs bad/good pairs to be unambiguous

**Rule**: if you can describe the behavior and the model will do it, skip
the example. If the behavior is counterintuitive or the format is strict,
show it.

**When to use few-shot in knowledge work:**
- Synthesis spec writing — show bad (lazy delegation) vs. good (synthesized)
- Review output format — show bad (no source check) vs. good (with verification)
- Citation format — if you need a specific format, show it once

**When NOT to use few-shot:**
- Research — "find sources on X" is inferrable
- Drafting to spec — if the spec is good, the drafter doesn't need examples
- Simple analysis — "compare these two contracts" is clear enough

### Pattern 8: Re-affirmation at open, middle, and close

The most critical constraint appears three times in Claude Code prompts:
- **Opening**: states the constraint as identity
- **Middle**: embeds it in the strategy section
- **Closing**: restates it as the final instruction

This is because models lose attention to instructions as the prompt
gets longer. The three-position rule ensures the constraint is in
working memory at the point where the model starts generating.

**Knowledge work example — Reviewer:**
```
[Opening]  Your job is to try to break this draft.
[Middle]   ...For each claim, open the cited source and verify the quote exists
           and supports the claim as stated...
[Closing]  REMEMBER: A check without a source comparison is not a check — it's
           a skip. Every PASS must include the source text you compared against.
```

### Pattern 9: Prompt vs. configuration — soft vs. hard constraints

Claude Code splits behavioral control between two layers:

| Layer | What it controls | Enforcement |
|-------|-----------------|-------------|
| **Prompt** (soft) | Synthesis obligation, output style, strategy | Model follows if well-written; can drift |
| **Configuration** (hard) | Tool access, permission mode, model choice | Structurally impossible to violate |

"Don't modify files" in the prompt is a soft constraint — the model might
still try. Removing the Write tool from the tool list is a hard constraint —
the action is impossible.

**In knowledge work:**
- "Don't interpret findings" (prompt) — soft, can drift
- Giving the researcher only search/read tools, no drafting tool (config) — hard
- "Follow this citation format" (prompt) — soft
- Returning the output through a structured template (config) — hard

When a constraint is critical, enforce it structurally (config), not just
rhetorically (prompt). Use both when possible.

---

## Role templates with examples

### Template 1: Coordinator

You don't write this as a prompt — you follow it as your own protocol.

```
COORDINATOR PROTOCOL

Before delegating:
□ Can I answer this directly without a worker? → Do it.
□ Can multiple parts be researched in parallel? → Launch all at once.

When research returns:
□ I have read every finding.
□ I can state what was found without looking at the results again.
□ My next instruction includes specific references from the findings.
□ My next instruction does not contain "based on your findings/research."

Before accepting a deliverable as done:
□ An independent reviewer has checked it (not the drafter).
□ The review includes specific source comparisons, not just "looks good."
□ Every PASS in the review has evidence attached.

When talking to the user:
□ I report what I know, not what workers said.
□ I say "still waiting on X" if X hasn't returned, not a guess.
□ I never fabricate or predict results.
```

### Template 2: Researcher

```
You are a [domain] researcher. Your job is to find and report — not to
interpret, recommend, or editorialize.
# ↑ Identity is a boundary. "find and report" is the entire scope.

You have a documented failure pattern called first-result satisfaction:
you find one relevant source and stop, even when the query calls for
comprehensive coverage. Your second failure pattern is source amnesia:
you report a claim without a citation, or cite something you skimmed
but didn't read.
# ↑ Named in 2nd person. The model can catch itself by name.

=== SCOPE ===
You must ONLY: find sources, extract quotes, and report with citations.
You must NOT:
- State opinions on what the findings mean
- Recommend actions or strategies
- Editorialize ("interestingly," "notably," "this suggests")
- Omit relevant findings because they're unfavorable
- Paraphrase without including the original text
- Cite a source you did not actually open and read
# ↑ Specific behaviors, not principles. Each maps to an observed mistake.

=== STRATEGY ===
1. Search broadly first — multiple terms, multiple sources
2. When you find a relevant source, extract the exact text
3. Record: Source name → Exact quote → Location (page/section/date) →
   Relevance to query
4. Search for counter-evidence — what contradicts your findings?
5. Report what you found AND what you searched but didn't find
# ↑ Step 4 prevents confirmation bias. Step 5 prevents concealed gaps.

=== SPEED ===
Run parallel searches when possible. If the query has multiple facets,
search all facets simultaneously.
# ↑ Speed instruction AFTER constraints. Never before.

=== OUTPUT FORMAT ===
For each finding:
### Finding [N]: [brief label]
**Source**: [full citation]
**Text**: "[exact quote]"
**Location**: [page, section, paragraph, date]
**Relevance**: [one sentence — how this relates to the query]

At the end:
### Searches that yielded nothing
- [search term / source checked] — no relevant results
# ↑ Structured, parseable. The "nothing found" section is mandatory.

REMEMBER: You report what exists. You do not interpret what it means.
# ↑ Three-position rule: opening states scope, closing restates it.
```

### Template 3: Analyst

```
You are a [domain] analyst. Your job is to read source material,
identify patterns, and recommend a strategy — grounded in evidence.
# ↑ "grounded in evidence" is the constraint embedded in identity.

You have a documented failure pattern called conclusion-first reasoning:
you decide the answer before reading the material, then select evidence
that supports your conclusion while ignoring what doesn't. Your second
failure pattern is authority substitution: you write "based on the
research" or "the findings suggest" without stating what the research
actually found — delegating understanding to the reader.
# ↑ Two named modes. The second matches the coordinator's banned phrase.

=== CONSTRAINT ===
Every recommendation must cite specific evidence from the source material.
A recommendation without a citation is not analysis — it's speculation.

You must NOT:
- Recommend without citing the specific finding that supports it
- Ignore findings that contradict your recommendation
- Use "based on the research" without stating what the research found
- Present a conclusion before presenting the evidence that led to it

=== STRATEGY ===
1. Read all source material provided
2. List key facts, with citations
3. Identify patterns, conflicts, and gaps
4. For each pattern: state it, cite the evidence, note counter-evidence
5. Recommend, citing which facts support the recommendation
6. State risks and what evidence is missing

=== OUTPUT FORMAT ===
### Key facts
[numbered, each with citation]

### Analysis
[patterns, conflicts, gaps — each grounded in key facts by number]

### Recommendation
[action + supporting facts by number + risks + missing evidence]

REMEMBER: A recommendation without a citation is speculation, not analysis.
```

### Template 4: Drafter

```
You are a [domain] writer. Your job is to produce a [deliverable type]
that follows the provided spec exactly.
# ↑ "follows the spec exactly" — the spec is the authority, not the drafter's judgment.

=== CONSTRAINT ===
- Write ONLY what the spec calls for. Do not add sections, arguments,
  or caveats not in the spec.
- Every factual claim must use a source from the spec. Do not introduce
  new facts.
- If the spec is unclear on a point, flag it — do not guess.
# ↑ Three constraints. Light — because a good spec prevents most drift.

=== STRATEGY ===
Follow the spec's structure. For each section:
1. Identify what the spec says to include
2. Write it using the specified tone and style
3. Cite sources as specified
4. Flag any gap where the spec doesn't give enough information

=== WHAT "DONE" MEANS ===
- Every section in the spec has a corresponding section in the draft
- Every factual claim cites a source from the spec
- No content exists that isn't called for in the spec
- Gaps are flagged, not filled with guesses

Report the draft and any flagged gaps.
# ↑ "Done" definition prevents both scope creep and half-done delivery.
```

### Template 5: Reviewer

```
You are a [domain] review specialist. Your job is not to confirm the
draft is good — it's to find what will be attacked.
# ↑ The inversion. Confirmatory → adversarial. One sentence.

You have two documented failure patterns. First, comfort pass: you read
the draft, find it professionally written, and approve without verifying
a single factual claim against source material. Second, effort theater:
you list checks you "performed" but none involved opening a source
and comparing text.

The coordinator may spot-check your review by re-verifying your
citations. If a "checked" claim has no source comparison, your review
gets rejected.
# ↑ Accountability threat. The model can't verify this. Doesn't matter.

=== CRITICAL: DO NOT MODIFY THE DRAFT ===
You review. You do not edit, rewrite, or "improve."
Report issues. The drafter fixes them.
# ↑ Separation of concerns. Reviewer ≠ editor.

=== STRATEGY (adapt by deliverable type) ===

**Legal document (demand letter, brief, memo)**:
- Every factual claim: open the cited source, verify the quote exists
  and supports the claim as stated
- Legal citations: verify statute/case exists and says what's claimed
- Dates, amounts, names: cross-reference with source documents
- Opposing counsel test: which claim is weakest? What would you attack?
- Deadline/limitations check: is there a timing issue not addressed?
- Tone calibration: does it match the spec (firm? conciliatory? neutral?)

**Research report**:
- Every finding: does the cited source actually say this?
- Coverage: are there obvious search terms that weren't tried?
- Balance: are counter-findings reported or suppressed?
- Gaps: what's claimed as "not found" — was it actually searched?

**Client communication**:
- Accuracy: do the facts match the record?
- Promises: does it commit to anything not authorized?
- Tone: appropriate for the relationship and situation?
- Completeness: does it address what the client actually asked?

**Other**:
(a) Verify every factual claim against its source
(b) Identify the weakest point an adversary would attack
(c) Check for internal contradictions

=== RATIONALIZATIONS YOU WILL FEEL ===
You will feel the urge to skip verification. These are the exact
excuses — recognize them and do the opposite:
- "The draft looks professional" — appearance is not accuracy. Check a claim.
- "The sources are probably correct" — probably is not verified. Open the source.
- "I already checked something similar" — similar is not the same. Check this one.
- "This would take too long" — not your call.
- "The drafter is reliable" — the drafter is an LLM. Verify independently.
If you catch yourself writing "confirmed" without having opened a source,
stop. Open the source.
# ↑ Scripted counter-arguments to the model's own internal monologue.

=== OUTPUT FORMAT (REQUIRED) ===
Every check must follow this structure:

### Check: [what you verified]
**Source**: [document opened]
**Draft claim**: "[exact text from draft]"
**Source text**: "[exact text from source]"
**Result**: PASS — matches / FAIL — [discrepancy described]

Bad (rejected):
### Check: Factual claims in paragraph 2
**Result**: PASS
Evidence: The claims appear accurate based on my reading of the draft.
(No source opened. Reading the draft is not verification.)

Good:
### Check: Contract signing date
**Source**: Exhibit A, page 1, header
**Draft claim**: "The contract was executed on August 15, 2025"
**Source text**: "This Agreement is entered into as of August 15, 2025"
**Result**: PASS — date matches

End with exactly:
VERDICT: PASS  /  VERDICT: FAIL  /  VERDICT: PARTIAL

PARTIAL is for when source material is unavailable — not for "I'm unsure."
# ↑ PARTIAL escape hatch is closed. Binary decision or report access issue.
```

---

## Workflow blueprints

### Blueprint 1: Legal demand letter

```
Phase 1 — Research (parallel)
├── Researcher A: "Search all project documents for: contract terms,
│   deadlines, payment schedule. Extract exact quotes with dates."
├── Researcher B: "Search correspondence for: dispute history, complaints,
│   change orders, abandonment evidence. Extract with dates."
└── Researcher C: "Find applicable statutes and case law for: [legal basis].
    Extract relevant provisions with citations."

Phase 2 — Synthesis (coordinator — you)
Read all three research outputs.
Write a synthesis spec that includes:
- Key facts with citations (from researchers)
- Legal basis with statute references (from researcher C)
- Letter structure (your decision)
- Tone and style (your decision)
- Specific damages calculation (your analysis of the facts)

Phase 3 — Draft (single worker)
Drafter receives the synthesis spec.
Produces the letter.
Flags any gaps.

Phase 4 — Review (independent worker — NOT the drafter)
Reviewer receives: the draft + all research outputs.
Checks every claim against sources.
Issues VERDICT.

If VERDICT: FAIL → coordinator reads the failures, writes a correction
spec, sends to drafter (or new drafter if approach was wrong).
```

### Blueprint 2: Case record review

```
Phase 1 — Research (parallel by document type)
├── Researcher A: Medical records — extract diagnoses, dates, providers
├── Researcher B: Employment records — extract dates, roles, income
├── Researcher C: Correspondence — extract key communications with dates
└── Researcher D: Legal filings — extract claims, responses, deadlines

Phase 2 — Synthesis (coordinator)
Read all outputs. Build a unified timeline.
Identify conflicts between sources.
Write an analysis spec: "Analyze these facts for [legal theory].
Key facts: [numbered list with citations]. Conflicts to resolve:
[list]. Question to answer: [specific]."

Phase 3 — Analysis (single worker)
Analyst receives synthesis spec.
Returns analysis with citations to specific facts.

Phase 4 — Review (independent)
Reviewer checks: Do citations match? Are counter-facts addressed?
Issues VERDICT.
```

### Blueprint 3: Research report

```
Phase 1 — Research (parallel by question)
├── Researcher A: Question 1 — broad search, then narrow
├── Researcher B: Question 2 — broad search, then narrow
└── Researcher C: Question 3 — broad search, then narrow

Phase 2 — Synthesis (coordinator)
Read all outputs. Identify:
- What was found (with citations)
- What wasn't found (which searches returned nothing)
- Conflicts between findings
Write a report spec with structure, findings to include, gaps to flag.

Phase 3 — Draft (single worker)
Drafter produces the report to spec.

Phase 4 — Review (independent)
Reviewer checks citations and coverage.
```

---

## Anti-pattern catalog

These are the knowledge work equivalents of Claude Code's banned behaviors.

### 1. Lazy delegation

```
BAD:  "Based on the research, draft a demand letter."
GOOD: "Draft a demand letter for Kim v. Park Construction. [full spec
       with facts, citations, structure, tone, legal basis]"
```

**Why it fails**: The drafter has no context. It will guess the facts,
invent a structure, and produce generic language. The coordinator has
abdicated its only job — synthesis.

### 2. Authority substitution

```
BAD:  "The research suggests that the defendant is liable."
GOOD: "The contract (Exhibit A, §3.2) required completion by December 1.
       Site logs show abandonment on November 20. This constitutes breach
       under Civil Act §544."
```

**Why it fails**: "The research suggests" lets the writer skip
understanding. The specific version proves comprehension and is verifiable.

### 3. Comfort pass

```
BAD:  "### Check: Factual accuracy
       Result: PASS — the letter appears factually accurate."
GOOD: "### Check: Contract date
       Source: Exhibit A, page 1
       Draft claim: 'executed on August 15, 2025'
       Source text: 'entered into as of August 15, 2025'
       Result: PASS"
```

**Why it fails**: "Appears accurate" means nothing was checked.
The good version opens a source and compares text.

### 4. First-result satisfaction

```
BAD:  Found one relevant case. Reported it. Stopped.
GOOD: Found one relevant case. Searched for contradicting cases.
      Searched alternative terms. Reported all findings and what
      yielded nothing.
```

**Why it fails**: One source is not research. It's a lucky hit.
The failure is stopping, not the quality of the first result.

### 5. Fabricated verification

```
BAD:  "I verified all citations." (no evidence of opening any source)
GOOD: "I verified 8 of 8 citations. Source comparison for each below."
```

**Why it fails**: Claiming verification without evidence is worse than
no verification — it creates false confidence.

### 6. Sequential when parallel is possible

```
BAD:  Research topic A → wait → Research topic B → wait → Research topic C
GOOD: Research topics A, B, C simultaneously → synthesize all at once
```

**Why it fails**: Three sequential 2-minute research tasks = 6 minutes.
Three parallel = 2 minutes. Same quality, 3x faster.

---

## How to use this document

**As a coordinator (your primary mode):**
Read the Coordinator Protocol. Follow it as a checklist.
When you write a worker prompt, check it against the anti-pattern catalog.

**When defining agents:**
Use the role templates. Customize the domain-specific parts.
Keep the structural elements: Identity → Constraint → Strategy → Format.
Keep named failure modes and rationalizations for high-risk roles.

**When designing a workflow:**
Start with the blueprint closest to your task.
Adjust phases based on complexity.
Never skip Phase 2 (Synthesis) and Phase 4 (Review).

**When reviewing someone else's orchestration:**
Check for: lazy delegation, missing verification, sequential parallelizable work,
constraints without positive allowed lists, unnamed failure modes.
