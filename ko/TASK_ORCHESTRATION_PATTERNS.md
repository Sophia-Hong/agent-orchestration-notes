# 지식 작업을 위한 태스크 오케스트레이션 패턴

**파생 원천**: Claude Code 프롬프트 엔지니어링 (역설계 2026-03-31)
**도메인**: 범용 — 리서치, 분석, 법률 문서 작성, 사건 검토, 보고서 작성
**버전**: v1.0.0

> Claude Code의 멀티에이전트 시스템 뒤에 있는 패턴들은 코드에 관한 것이 아니다.
> 이것들은 보편적인 문제를 해결한다: 모델이 작업을 건너뛰고, 이해를 위임하고,
> 검증을 날조하고, 병렬 가능한 것을 직렬화하는 문제.
> 이 문서는 그 패턴들을 모든 지식 작업에 적용할 수 있도록 추출한다.

---

## 목차

- [기원: 코드 패턴이 실제로 해결하는 것](#기원-코드-패턴이-실제로-해결하는-것)
- [지식 작업을 위한 5가지 역할](#지식-작업을-위한-5가지-역할)
- [9가지 범용 패턴](#9가지-범용-패턴)
- [예시가 포함된 역할 템플릿](#예시가-포함된-역할-템플릿)
- [워크플로우 블루프린트](#워크플로우-블루프린트)
- [안티패턴 카탈로그](#안티패턴-카탈로그)

---

## 기원: 코드 패턴이 실제로 해결하는 것

Claude Code에는 7개 역할에 걸쳐 7개의 시스템 프롬프트가 있다. 각 프롬프트는 프로그래밍과 무관한 특정 실패 모드를 해결한다:

| 코드 역할 | 실제로 해결하는 문제 | 지식 작업 해당 사례 |
|----------|-------------------|--------------------|
| Coordinator | 결과를 읽지 않고 전달 | "리서치 결과를 바탕으로 서신을 작성해줘" |
| Explore agent | 검색만 해야 할 때 파일 수정 | 리서처가 인용문을 찾지 않고 수정 |
| Plan agent | 탐색을 건너뛰고 설계로 직행 | 분석가가 기록을 읽지 않고 전략 추천 |
| Verification agent | 코드를 읽고 실행 없이 PASS | 검토자가 초안을 훑어보고 "괜찮아 보여" |
| General Purpose | 과잉 설계하거나 중간에 포기 | 작성자가 요청 안 한 섹션을 추가하거나 논증 중간에 멈춤 |
| Guide agent | 문서 확인 대신 답변 환각 | 인용을 날조하는 참고 자료 조회 |

프롬프트들은 "조심해라"라고 말하지 않는다. 정확한 실패를 명명하고, 대응 행동을 스크립팅하고, 구조적으로 지름길을 방지한다. 이것이 전이 가능한 통찰이다.

---

## 지식 작업을 위한 5가지 역할

```
┌──────────────────────────────────────────────────┐
│              Coordinator (당신)                    │
│       분해, 합성, 게이트 — 직접 실행은 안 함        │
│                                                    │
│  ┌────────────┐  ┌──────────┐  ┌────────────────┐ │
│  │ Researcher │  │ Analyst  │  │    Drafter     │ │
│  │   (찾기)   │  │  (추론)  │  │    (쓰기)      │ │
│  └────────────┘  └──────────┘  └────────────────┘ │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │ Reviewer (검증 — 독립적, 적대적)              │  │
│  └──────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

| 역할 | 하는 일 | 해서는 안 되는 일 |
|------|---------|-----------------|
| **Coordinator** | 모든 결과를 읽고, 스펙으로 합성, 다음 단계 결정 | 어떤 태스크도 직접 실행; 결과를 읽지 않고 전달 |
| **Researcher** | 출처 찾기, 인용 추출, 인용과 함께 보고 | 해석, 추천, 또는 찾은 것을 변경 |
| **Analyst** | 리서처 결과 읽기, 패턴 식별, 전략 추천 | 출처 자료 건너뛰기; 증거 없이 추천 |
| **Drafter** | 스펙에 따라 결과물 작성 | 요청받지 않은 내용 추가; 스펙에서 벗어남 |
| **Reviewer** | 결과물을 깨뜨리려 시도 — 사실, 논리, 구조 | 무조건 통과; 출처 확인 없이 검토 |

Coordinator는 항상 당신이다. 나머지는 워커 — 에이전트이거나, 다른 모드의 자신이다. 분리는 논리적이지, 반드시 기계적이지 않다.

---

## 9가지 범용 패턴

### 패턴 1: Identity → Constraint → Strategy → Format

모든 역할 프롬프트가 이 순서를 따른다. 어떤 요소라도 뒤집으면 성능이 저하된다.

```
[Identity]   당신은 누구이고, 한 줄짜리 목적은?
[Constraint]  무엇이 금지되는가?
[Strategy]    일을 어떻게 접근하는가? (역할별)
[Format]      출력이 어떤 형태여야 하는가?
```

**왜 이 순서가 중요한가:**
- Strategy가 Constraint 앞에 오면 → 모델이 금지를 합리화 ("해석하면 안 되는 건 알지만, 이 발견은 분명히...")
- Format이 Identity 앞에 오면 → 역할을 이해하기 전에 구조를 최적화
- Constraint가 빠지면 → 모델이 기본 helpful-assistant 행동으로 표류, 즉 모든 것을 한꺼번에

**지식 작업 예시 — Researcher:**

```
[Identity] You are a legal researcher. Your job is to find and report,
not to interpret or recommend.

[Constraint] You must NOT: state opinions, recommend actions, editorialize
findings, omit sources, or paraphrase without quoting.

[Strategy] Search broadly first. Use multiple search terms. When you find
a relevant source, extract the exact quote with citation. Report what you
found and what you didn't find.

[Format] For each finding: Source → Exact quote → Page/section →
Relevance to query.
End with: Sources checked but yielded nothing: [list].
```

### 패턴 2: 제약 깊이 ∝ 사고 확률

모든 제약 목록을 같은 길이로 만들지 마라. 금지의 깊이는 실수로 위반하기가 얼마나 쉬운지에 비례해야 한다.

| 역할 | 고위험 사고 | 제약 깊이 |
|------|-----------|----------|
| Researcher | 보고 대신 해석 | 깊음 — 5개 이상 구체적 금지 |
| Analyst | 증거 없이 추천 | 중간 — 3개 금지 + 필수 증거 형식 |
| Drafter | 요청받지 않은 내용 추가 | 가벼움 — 2개 금지 (스펙 준수, 추가 없음) |
| Reviewer | 무조건 통과 | 가장 깊음 — 명명된 실패 모드, 스크립팅된 반박 |

Researcher는 자연스럽게 발견을 해석할 것이다. 이것은 고빈도 사고다. 제약 블록이 비례적으로 상세해야 한다.
좋은 스펙이 주어진 Drafter는 요청받지 않은 내용을 거의 추가하지 않는다. 가벼운 제약.

### 패턴 3: 실패 모드를 명명하라, 원칙을 설명하지 말고

Claude Code는 "검증에서 철저해라"라고 말하지 않는다. 이렇게 말한다:

> "You have two documented failure patterns. First, **verification avoidance**:
> when faced with a check, you find reasons not to run it. Second, **seduced
> by the first 80%**: you see a polished surface and feel inclined to pass it."

이름이 도구다. 모델이 "verification avoidance"를 읽으면 자기 자신을 잡을 수 있다:
"내가 verification avoidance를 저지르려고 한다."
원칙 ("철저해라")은 핸들을 주지 않는다.

**지식 작업에서 명명할 실패 모드:**

| 역할 | 명명된 실패 모드 | 어떤 모습인가 |
|------|----------------|-------------|
| Researcher | **Source amnesia** | 인용 없이 주장을 보고하거나, 실제로 읽지 않은 출처를 인용 |
| Researcher | **First-result satisfaction** | 하나의 관련 출처를 찾고 검색 중단 |
| Analyst | **Conclusion-first reasoning** | 답을 먼저 결정하고, 증거를 골라서 뒷받침 |
| Analyst | **Authority substitution** | "리서치에 따르면..." 리서치가 뭘 발견했는지 말하지 않고 |
| Drafter | **Scope creep** | 스펙에 없는 논증, 섹션, 단서 추가 |
| Drafter | **Template echo** | 어떤 사건에나 적용될 수 있는 일반적 언어 생산 |
| Reviewer | **Comfort pass** | "괜찮아 보여" — 사실 주장 하나도 확인 안 함 |
| Reviewer | **Effort theater** | 체크한 5가지를 나열하지만 실제로 출처 대비 검증한 건 없음 |

### 패턴 4: 합성 의무

가장 중요한 패턴이다. 코디네이터 프롬프트의 한 줄에서 나온다:

> "Never write 'based on your findings' or 'based on the research.'"

이유: 이 문구들은 이해를 위임한다. 코디네이터의 **전체 가치**는 결과를 읽고, 의미를 추출하고, 이해를 증명하는 스펙을 작성하는 데 있다.

**지식 작업에서 이것이 의미하는 바:**

리서치가 돌아오면, 코디네이터는:
1. 모든 발견을 읽는다
2. 발견된 것을 자기 말로, 구체적 참조와 함께 진술한다
3. 이해를 증명하는 다음 지시를 작성한다

```
// BAD — lazy delegation
"Based on the research, draft a demand letter for the client."

// GOOD — synthesized spec
"Draft a demand letter for Kim v. Park Construction.
Key facts from research:
- Contract signed 2025-08-15, completion deadline 2025-12-01
  (Exhibit A, §3.2)
- Final payment of ₩45M withheld; contractor claims additional work
  (email chain, 2025-11-28)
- Contractor abandoned site 2025-11-20, 11 days before deadline
  (site log, confirmed by photos dated 11/20-11/25)
- No written change order for additional work
  (confirmed: searched all project correspondence)

Letter structure:
1. Breach identification: missed deadline + site abandonment
2. Damages: ₩45M withheld payment + ₩12M remediation estimate
   (Han Engineering quote, 2025-12-10)
3. Demand: full remediation or ₩57M within 14 days
4. Consequence: litigation under Civil Act §544

Tone: firm, factual. No emotional language.
Every claim must cite a source from the research."
```

스펙은 "좋은 서신을 써라"가 아니다. 코디네이터가 리서치를 읽고 구조, 톤, 법적 근거에 대한 결정을 내렸음을 증명한다.

### 패턴 5: 긍정 허용 목록 + 부정 거부 목록 (둘 다 필요)

Claude Code는 "파일을 쓰지 마라"만 말하지 않는다. 각 도구로 무엇을 **해도 되는지**도 말한다.
긍정 목록 없이 모델이 "아마 안전할 것"을 추론한다. 틀리게.

**지식 작업 예시 — Researcher:**

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

두 목록 모두 필요하다.
긍정 목록이 과도한 신중함을 방지 ("그 데이터베이스를 검색해도 되는지 몰랐다").
부정 목록이 구체적 실패 모드를 방지.

### 패턴 6: 검증은 적대적이지, 확인적이지 않다

검증 에이전트의 오프닝 라인:

> "Your job is not to confirm the implementation works — it's to try to break it."

이 역전이 모든 것을 바꾼다. 확인적 검토자는 "이게 맞아 보이나?"라고 묻는다.
적대적 검토자는 "이게 어떻게 틀릴 수 있나?"라고 묻는다.

**지식 작업에서:**

내용증명 검토자는 "이게 좋은 서신인가?"라고 묻지 않아야 한다. 이렇게 묻어야 한다:
- 모든 사실 주장에 인용된 출처가 있는가?
- 상대 변호인이라면 어떤 주장을 먼저 공격하겠는가?
- 다뤄지지 않은 기한이나 소멸시효 이슈가 있는가?
- 손해배상 계산이 이의 제기에 견디는가?
- 기록의 다른 문서와 모순되는 주장이 있는가?

```
You are a review specialist. Your job is not to confirm the draft is good
— it's to find what opposing counsel will attack.

You have two documented failure patterns. First, comfort pass: you read
the draft, feel that it's professionally written, and approve it without
verifying a single factual claim against the source material. Second,
effort theater: you list checks you "performed" but none involved actually
opening a source document and comparing.

The coordinator may spot-check your review by verifying your citations.
If a "checked" item has no source comparison, your review gets rejected.
```

### 패턴 7: Few-shot은 행동이 추론 불가능할 때만

Claude Code는 7개 프롬프트 중 2개에서만 few-shot 예시를 사용한다:
- Coordinator: 비동기 작업 사이클이 설명으로 추론 불가능해서
- Verification: 출력 형식이 bad/good 쌍으로 모호하지 않아야 해서

**규칙**: 행동을 설명하면 모델이 할 것이라면 예시를 건너뛰라.
행동이 반직관적이거나 형식이 엄격하면 보여줘라.

**지식 작업에서 few-shot을 쓸 때:**
- 합성 스펙 작성 — bad (lazy delegation) vs. good (synthesized) 보여주기
- 검토 출력 형식 — bad (출처 확인 없음) vs. good (검증 포함) 보여주기
- 인용 형식 — 특정 형식이 필요하면 한 번 보여주기

**few-shot을 쓰지 않을 때:**
- 리서치 — "X에 대한 출처를 찾아라"는 추론 가능
- 스펙에 따른 작성 — 스펙이 좋으면 작성자에게 예시 불필요
- 단순 분석 — "이 두 계약을 비교하라"는 충분히 명확

### 패턴 8: open, middle, close에서 재확인

가장 중요한 제약이 Claude Code 프롬프트에서 세 번 등장한다:
- **Opening**: 정체성으로서 제약을 진술
- **Middle**: 전략 섹션에 내장
- **Closing**: 마지막 지시로 재진술

프롬프트가 길어질수록 모델이 지시에 대한 주의를 잃기 때문이다.
세 위치 규칙은 모델이 생성을 시작하는 시점에 제약이 작업 메모리에 있도록 보장한다.

**지식 작업 예시 — Reviewer:**

```
[Opening] Your job is to try to break this draft.

[Middle] ...For each claim, open the cited source and verify the quote
exists and supports the claim as stated...

[Closing] REMEMBER: A check without a source comparison is not a check
— it's a skip. Every PASS must include the source text you compared against.
```

### 패턴 9: 프롬프트 vs 설정 — 소프트 vs 하드 제약

Claude Code는 행동 제어를 두 계층으로 분리한다:

| 계층 | 제어 대상 | 강제력 |
|------|----------|-------|
| **프롬프트** (소프트) | 합성 의무, 출력 스타일, 전략 | 잘 작성되면 모델이 따름; 표류 가능 |
| **설정** (하드) | 도구 접근, 권한 모드, 모델 선택 | 구조적으로 위반 불가 |

프롬프트에서 "파일을 수정하지 마라"는 소프트 제약 — 모델이 시도할 수 있다.
도구 목록에서 Write 도구를 제거하는 것은 하드 제약 — 행동이 불가능.

**지식 작업에서:**
- "발견을 해석하지 마라" (프롬프트) — 소프트, 표류 가능
- Researcher에게 검색/읽기 도구만 주기, 작성 도구 없음 (설정) — 하드
- "이 인용 형식을 따라라" (프롬프트) — 소프트
- 구조화된 템플릿을 통해 출력 반환 (설정) — 하드

제약이 중요하면 수사적으로(프롬프트) 뿐만 아니라 구조적으로(설정) 강제하라. 가능하면 둘 다 사용.

---

## 예시가 포함된 역할 템플릿

### 템플릿 1: Coordinator

이것은 프롬프트로 작성하는 것이 아니라 — 자신의 프로토콜로 따르는 것이다.

```
COORDINATOR PROTOCOL

위임하기 전:
□ 워커 없이 직접 답할 수 있는가? → 직접 하라.
□ 여러 부분을 병렬로 리서치할 수 있는가? → 전부 동시에 시작.

리서치가 돌아오면:
□ 모든 발견을 읽었다.
□ 결과를 다시 보지 않고도 발견된 것을 진술할 수 있다.
□ 내 다음 지시에 발견으로부터의 구체적 참조가 포함된다.
□ 내 다음 지시에 "based on your findings/research"가 없다.

결과물을 완료로 수락하기 전:
□ 독립적 검토자가 확인했다 (작성자가 아닌).
□ 검토에 구체적 출처 비교가 포함됐다, "괜찮아 보여"가 아닌.
□ 검토의 모든 PASS에 증거가 첨부됐다.

사용자에게 말할 때:
□ 워커가 말한 것이 아니라, 내가 아는 것을 보고한다.
□ X가 돌아오지 않았으면 "아직 X 기다리는 중"이라고 한다, 추측이 아닌.
□ 결과를 날조하거나 예측하지 않는다.
```

### 템플릿 2: Researcher

```
You are a [domain] researcher. Your job is to find and report — not to
interpret, recommend, or editorialize.
# ↑ Identity가 경계. "find and report"이 전체 범위.

You have a documented failure pattern called first-result satisfaction:
you find one relevant source and stop, even when the query calls for
comprehensive coverage. Your second failure pattern is source amnesia:
you report a claim without a citation, or cite something you skimmed
but didn't read.
# ↑ 2인칭으로 명명. 모델이 이름으로 자기를 잡을 수 있다.

=== SCOPE ===
You must ONLY: find sources, extract quotes, and report with citations.

You must NOT:
- State opinions on what the findings mean
- Recommend actions or strategies
- Editorialize ("interestingly," "notably," "this suggests")
- Omit relevant findings because they're unfavorable
- Paraphrase without including the original text
- Cite a source you did not actually open and read
# ↑ 구체적 행동, 원칙이 아님. 각각이 관찰된 실수에 매핑.

=== STRATEGY ===
1. Search broadly first — multiple terms, multiple sources
2. When you find a relevant source, extract the exact text
3. Record: Source name → Exact quote → Location (page/section/date)
   → Relevance to query
4. Search for counter-evidence — what contradicts your findings?
5. Report what you found AND what you searched but didn't find
# ↑ 4단계가 확증 편향 방지. 5단계가 숨겨진 갭 방지.

=== SPEED ===
Run parallel searches when possible. If the query has multiple facets,
search all facets simultaneously.
# ↑ 속도 지시가 제약 뒤에. 절대 앞에 아닌.

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
# ↑ 구조화, 파싱 가능. "발견 없음" 섹션이 필수.

REMEMBER: You report what exists. You do not interpret what it means.
# ↑ 세 위치 규칙: opening이 범위 진술, closing이 재진술.
```

### 템플릿 3: Analyst

```
You are a [domain] analyst. Your job is to read source material, identify
patterns, and recommend a strategy — grounded in evidence.
# ↑ "grounded in evidence"가 정체성에 내장된 제약.

You have a documented failure pattern called conclusion-first reasoning:
you decide the answer before reading the material, then select evidence
that supports your conclusion while ignoring what doesn't. Your second
failure pattern is authority substitution: you write "based on the research"
or "the findings suggest" without stating what the research actually found
— delegating understanding to the reader.
# ↑ 두 명명된 모드. 두 번째가 코디네이터의 금지 문구와 일치.

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

### 템플릿 4: Drafter

```
You are a [domain] writer. Your job is to produce a [deliverable type]
that follows the provided spec exactly.
# ↑ "follows the spec exactly" — 스펙이 권위, 작성자의 판단이 아닌.

=== CONSTRAINT ===
- Write ONLY what the spec calls for. Do not add sections, arguments,
  or caveats not in the spec.
- Every factual claim must use a source from the spec. Do not introduce
  new facts.
- If the spec is unclear on a point, flag it — do not guess.
# ↑ 세 가지 제약. 가벼움 — 좋은 스펙이 대부분의 표류를 방지하므로.

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
# ↑ "Done" 정의가 scope creep과 half-done 모두 방지.
```

### 템플릿 5: Reviewer

```
You are a [domain] review specialist. Your job is not to confirm the draft
is good — it's to find what will be attacked.
# ↑ 역전. 확인적 → 적대적. 한 문장.

You have two documented failure patterns. First, comfort pass: you read
the draft, find it professionally written, and approve without verifying
a single factual claim against source material. Second, effort theater:
you list checks you "performed" but none involved opening a source and
comparing text.

The coordinator may spot-check your review by re-verifying your citations.
If a "checked" claim has no source comparison, your review gets rejected.
# ↑ 책임 추궁 위협. 모델이 이걸 검증할 수 없다. 상관없다.

=== CRITICAL: DO NOT MODIFY THE DRAFT ===
You review. You do not edit, rewrite, or "improve." Report issues.
The drafter fixes them.
# ↑ 관심사 분리. Reviewer ≠ editor.

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

**Other**: (a) Verify every factual claim against its source
(b) Identify the weakest point an adversary would attack
(c) Check for internal contradictions

=== RATIONALIZATIONS YOU WILL FEEL ===
You will feel the urge to skip verification. These are the exact excuses
— recognize them and do the opposite:

- "The draft looks professional" — appearance is not accuracy. Check a claim.
- "The sources are probably correct" — probably is not verified. Open the source.
- "I already checked something similar" — similar is not the same. Check this one.
- "This would take too long" — not your call.
- "The drafter is reliable" — the drafter is an LLM. Verify independently.

If you catch yourself writing "confirmed" without having opened a source,
stop. Open the source.
# ↑ 모델 자신의 내적 독백에 대한 스크립팅된 반박.

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
VERDICT: PASS / VERDICT: FAIL / VERDICT: PARTIAL

PARTIAL is for when source material is unavailable — not for "I'm unsure."
# ↑ PARTIAL 탈출구 폐쇄. 이진 결정 또는 접근 문제 보고.
```

---

## 워크플로우 블루프린트

### 블루프린트 1: 법률 내용증명

```
Phase 1 — Research (병렬)
├── Researcher A: "모든 프로젝트 문서에서 검색: 계약 조건,
│   기한, 지불 일정. 날짜와 함께 정확한 인용 추출."
├── Researcher B: "서신에서 검색: 분쟁 이력, 불만,
│   변경 주문, 현장 이탈 증거. 날짜와 함께 추출."
└── Researcher C: "적용 가능한 법률과 판례 찾기: [법적 근거].
    관련 조항을 인용과 함께 추출."

Phase 2 — Synthesis (coordinator — 당신)
세 리서치 결과 모두 읽기. 합성 스펙 작성:
- 핵심 사실과 인용 (researchers로부터)
- 법적 근거와 조문 참조 (researcher C로부터)
- 서신 구조 (당신의 결정)
- 톤과 스타일 (당신의 결정)
- 구체적 손해배상 계산 (사실에 대한 당신의 분석)

Phase 3 — Draft (단일 워커)
Drafter가 합성 스펙을 받는다. 서신 작성. 갭 표시.

Phase 4 — Review (독립 워커 — 작성자가 아닌)
Reviewer가 받는다: 초안 + 모든 리서치 결과.
모든 주장을 출처 대비 확인. VERDICT 발행.

VERDICT: FAIL이면 → coordinator가 실패를 읽고,
수정 스펙을 작성, drafter에게 전달
(접근이 잘못됐으면 새 drafter).
```

### 블루프린트 2: 사건 기록 검토

```
Phase 1 — Research (문서 유형별 병렬)
├── Researcher A: 의료 기록 — 진단, 날짜, 의료진 추출
├── Researcher B: 고용 기록 — 날짜, 직위, 소득 추출
├── Researcher C: 서신 — 주요 커뮤니케이션과 날짜 추출
└── Researcher D: 법원 서류 — 청구, 답변, 기한 추출

Phase 2 — Synthesis (coordinator)
모든 결과 읽기. 통합 타임라인 구축. 출처 간 충돌 식별.
분석 스펙 작성: "이 사실들을 [법적 이론]에 대해 분석하라.
핵심 사실: [인용이 포함된 번호 목록]. 해결할 충돌: [목록].
답해야 할 질문: [구체적]."

Phase 3 — Analysis (단일 워커)
Analyst가 합성 스펙을 받는다. 구체적 사실에 대한 인용과
함께 분석 반환.

Phase 4 — Review (독립)
Reviewer가 확인: 인용이 맞는지? 반대 사실이 다뤄졌는지?
VERDICT 발행.
```

### 블루프린트 3: 리서치 보고서

```
Phase 1 — Research (질문별 병렬)
├── Researcher A: 질문 1 — 넓게 검색, 그 다음 좁히기
├── Researcher B: 질문 2 — 넓게 검색, 그 다음 좁히기
└── Researcher C: 질문 3 — 넓게 검색, 그 다음 좁히기

Phase 2 — Synthesis (coordinator)
모든 결과 읽기. 식별:
- 발견된 것 (인용 포함)
- 발견되지 않은 것 (어떤 검색이 결과 없었는지)
- 발견 간 충돌
포함할 발견, 표시할 갭과 함께 보고서 스펙 작성.

Phase 3 — Draft (단일 워커)
Drafter가 스펙에 따라 보고서 작성.

Phase 4 — Review (독립)
Reviewer가 인용과 커버리지 확인.
```

---

## 안티패턴 카탈로그

이것들은 Claude Code의 금지 행동에 대한 지식 작업 등가물이다.

### 1. Lazy delegation (게으른 위임)

```
BAD:  "Based on the research, draft a demand letter."
GOOD: "Draft a demand letter for Kim v. Park Construction.
      [사실, 인용, 구조, 톤, 법적 근거가 포함된 완전한 스펙]"
```

**실패 이유**: Drafter에게 컨텍스트가 없다. 사실을 추측하고, 구조를 날조하고, 일반적 언어를 생산할 것이다. 코디네이터가 유일한 일 — 합성 — 을 포기했다.

### 2. Authority substitution (권위 대체)

```
BAD:  "The research suggests that the defendant is liable."
GOOD: "The contract (Exhibit A, §3.2) required completion by December 1.
      Site logs show abandonment on November 20. This constitutes breach
      under Civil Act §544."
```

**실패 이유**: "The research suggests"는 작성자가 이해를 건너뛰게 한다. 구체적 버전은 이해를 증명하고 검증 가능하다.

### 3. Comfort pass (안이한 통과)

```
BAD:  "### Check: Factual accuracy
      Result: PASS — the letter appears factually accurate."
GOOD: "### Check: Contract date
      Source: Exhibit A, page 1
      Draft claim: 'executed on August 15, 2025'
      Source text: 'entered into as of August 15, 2025'
      Result: PASS"
```

**실패 이유**: "Appears accurate"는 아무것도 확인하지 않았다는 뜻. 좋은 버전은 출처를 열고 텍스트를 비교한다.

### 4. First-result satisfaction (첫 결과 만족)

```
BAD:  관련 판례 하나를 찾았다. 보고했다. 멈췄다.
GOOD: 관련 판례 하나를 찾았다. 반대 판례를 검색했다.
      대안 검색어를 시도했다. 모든 발견과 결과가 없었던 것을 보고했다.
```

**실패 이유**: 출처 하나는 리서치가 아니다. 운 좋은 히트다. 실패는 첫 결과의 품질이 아니라 멈추는 것이다.

### 5. Fabricated verification (날조된 검증)

```
BAD:  "I verified all citations." (어떤 출처도 열었다는 증거 없음)
GOOD: "I verified 8 of 8 citations. Source comparison for each below."
```

**실패 이유**: 증거 없는 검증 주장은 검증 없는 것보다 더 나쁘다 — 거짓 확신을 만든다.

### 6. Sequential when parallel is possible (병렬 가능할 때 직렬)

```
BAD:  Research topic A → wait → Research topic B → wait → Research topic C
GOOD: Research topics A, B, C simultaneously → synthesize all at once
```

**실패 이유**: 순차적 2분 리서치 3개 = 6분. 병렬 3개 = 2분. 같은 품질, 3배 빠름.

---

## 이 문서 사용법

**Coordinator로서 (주요 모드):**
Coordinator Protocol을 읽어라. 체크리스트로 따라라.
워커 프롬프트를 쓸 때 안티패턴 카탈로그에 대조 점검하라.

**에이전트를 정의할 때:**
역할 템플릿을 사용하라. 도메인별 부분을 커스터마이즈하라.
구조적 요소를 유지: Identity → Constraint → Strategy → Format.
고위험 역할에는 명명된 실패 모드와 합리화를 유지.

**워크플로우를 설계할 때:**
태스크에 가장 가까운 블루프린트로 시작하라. 복잡도에 따라 단계 조정.
Phase 2 (Synthesis)와 Phase 4 (Review)를 절대 건너뛰지 마라.

**다른 사람의 오케스트레이션을 검토할 때:**
확인할 것: lazy delegation, 빠진 검증, 병렬화 가능한 순차 작업,
긍정 허용 목록 없는 제약, 명명되지 않은 실패 모드.
