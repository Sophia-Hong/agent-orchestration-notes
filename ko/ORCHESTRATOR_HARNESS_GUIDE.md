# 에이전트 시스템 아키텍처 나침반
**Claude Code 역설계 통합본 — 에이전트 설계 레퍼런스**

> 이 문서는 Claude Code의 소스코드(`src/constants/prompts.ts`, `src/coordinator/coordinatorMode.ts`, `src/tools/AgentTool/`) 역설계 분석 두 편을 하나로 통합한 것이다.
> 목표: "비슷하게 생긴 프롬프트"를 모방하는 것이 아니라, 이 수준의 하네스를 재구성하는 데 필요한 운영 원리, 구조, 포맷을 뽑아내는 것.
> 용도: 향후 에이전트 시스템 설계의 나침반, 스킬 추출 원천.

---

## 목차

- [1부: 핵심 원리 5가지](#1부-핵심-원리-5가지)
- [2부: 아키텍처 전체 구조](#2부-아키텍처-전체-구조)
- [3부: 메인 시스템 프롬프트 설계](#3부-메인-시스템-프롬프트-설계)
- [4부: 오케스트레이터(Coordinator) 프롬프트 완전 분석](#4부-오케스트레이터coordinator-프롬프트-완전-분석)
- [5부: 에이전트 정의와 설계](#5부-에이전트-정의와-설계)
- [6부: 도구 접근 제어 계층](#6부-도구-접근-제어-계층)
- [7부: 위임 전략과 태스크 시스템](#7부-위임-전략과-태스크-시스템)
- [8부: 품질 시스템](#8부-품질-시스템)
- [9부: MVP 하네스 구현 설계](#9부-mvp-하네스-구현-설계)
- [10부: 완전한 템플릿 모음](#10부-완전한-템플릿-모음)
- [11부: 이 문서에서 파생 가능한 스킬](#11부-이-문서에서-파생-가능한-스킬)
- [12부: 최종 체크리스트](#12부-최종-체크리스트)
- [참고 파일](#참고-파일)

---

## 1부: 핵심 원리 5가지

이 시스템의 수준은 좋은 프롬프트 한 장에서 오지 않는다. 아래 다섯 개의 설계 결정이 조합된 결과다.

**1. 시스템 프롬프트는 우선순위와 캐시 경계를 가진 섹션 집합으로 조립한다**
단일 문자열이 아니라 정적/동적 섹션을 분리해서 prompt cache를 극대화한다.

**2. 상위 모델은 실행보다 분해, 합성, 검증 게이트를 맡는다**
오케스트레이터는 파일을 직접 수정할 수 없다. 판단과 설계의 병목이 되도록 강제한다.

**3. 서브에이전트는 "대화 속 하위 인격"이 아니라 "비동기 태스크 런타임"으로 구현한다**
결과는 내부 콜백이 아닌 `<task-notification>` user-role 메시지로 재주입된다.

**4. 각 에이전트는 프롬프트 텍스트가 아니라 도구·권한·모델·격리·메모리의 묶음이다**
에이전트 정의에 실행 정책이 포함되어야 한다.

**5. 품질은 lazy delegation 금지, 독립 검증 강제, 컨텍스트 오염 방지에서 나온다**
"based on your findings" 한 문장이 전체 품질을 갉아먹는다.

---

## 2부: 아키텍처 전체 구조

### 2.1 계층도

```
┌─────────────────────────────────────────────────────────┐
│                     USER (사람)                          │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│           COORDINATOR / MAIN SESSION                     │
│  모델: claude-opus-4-6 / sonnet-4-6                      │
│  도구: [Coordinator mode] Agent, TaskStop,               │
│        SendMessage, SyntheticOutput (4개만)              │
│       [Normal mode] 45+개                                │
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

### 2.2 세 계층의 프롬프트

각 계층은 **완전히 다른 시스템 프롬프트**를 받는다. 이것이 핵심.

| 계층 | 정체성 | 도구 수 | 주요 역할 |
|------|--------|---------|-----------|
| Coordinator | "You are a coordinator" | 4개 | 판단, 분해, 합성, 지휘 |
| Main Session | "You are Claude Code" | 45+개 | 구현 + 제한적 위임 |
| Worker | 역할별 전문가 | 역할별 제한 | 단일 태스크 실행 |

### 2.3 프롬프트 우선순위 스택

```
1. override system prompt        ← 최고 우선순위
2. coordinator system prompt
3. main-thread agent system prompt
4. custom system prompt
5. default system prompt         ← 최저 우선순위
```

추가 규칙:
- proactive 모드: agent prompt가 default를 **대체**하지 않고 **append**됨
- 일반 모드: 특정 agent prompt가 default를 **대체**함
- 이 계층을 런타임 정책으로 분리해야 한다 — 코드에 하드코딩하지 마라

---

## 3부: 메인 시스템 프롬프트 설계

### 3.1 섹션 순서와 캐시 경계

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[STATIC — 전역 캐시 가능]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 1. Intro           정체성 선언 (2문장)
 2. CyberRisk       보안 guardrail
 3. URL rule        URL 추측 금지
 4. System          렌더링, 권한, hooks, 압축
 5. Doing tasks     코드 스타일, 금지 패턴
 6. Actions         되돌릴 수 없는 행동 기준
 7. Tool usage      전용 도구 우선 규칙
 8. Output style    (커스텀 스타일 있을 때)
 9. Tone & style    이모지 금지, 간결성
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
     __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[DYNAMIC — 세션별, 캐시 안 됨]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
10. Session-specific  활성화된 도구 기반 조건부 지침
11. Memory            ~/.claude/memory/ 내용
12. Environment       모델명, 날짜, OS, git 상태
13. MCP instructions  연결된 MCP 서버 안내
14. Language          언어 설정
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**왜 경계를 나누는가**:
Anthropic API는 시스템 프롬프트 prefix를 cache한다. 날짜·메모리·MCP 서버명 같은 동적 데이터가 앞에 있으면 캐시 히트율 0%. 섹션을 뒤에 모으면 정적 섹션의 토큰 비용을 계속 아낄 수 있다. 멀티에이전트 시스템에서는 이 차이가 누적된다.

**당신의 시스템에 적용**: 정적 규칙은 앞에, 사용자/환경 데이터는 뒤에.

### 3.2 Intro 섹션 포맷

```
You are [이름], [소속/컨텍스트].
You are [역할 유형] that [주요 목적].
```

실제 소스:
```
You are Claude Code, Anthropic's official CLI for Claude.
You are an interactive agent that helps users with software engineering tasks.
```

단 두 문장. 길게 쓰지 않는다. 정체성 확정이 목적.

### 3.3 "Doing tasks" 섹션 패턴: 금지 목록

모델이 실제로 저지르는 실수를 역으로 봉쇄하는 방식. 추상적 규칙이 아니라 **구체적 행동 패턴**으로 명시한다.

```
❌ 요청받지 않은 리팩토링, 기능 추가, "개선"
❌ 변경하지 않은 코드에 주석/타입 추가
❌ 일어날 수 없는 시나리오를 위한 에러 핸들링
❌ 일회성 작업을 위한 헬퍼/유틸리티 추상화
❌ 미래 요구사항 설계
❌ 뒤돌아보는 호환성 핵 (unused _vars 이름 변경, 타입 재export 등)

✅ 수정 전 파일 먼저 읽기
✅ 실패 시 원인 파악 후 전술 변경 (맹목적 재시도 금지)
✅ 보안 취약점 즉시 수정
```

### 3.4 Actions 섹션 패턴: 되돌릴 수 없는 행동

실제 소스에서 추출한 구조:

```
핵심 원칙: "reversibility and blast radius"
- 로컬·되돌릴 수 있는 행동 → 자유롭게
- 되돌릴 수 없거나 공유 상태에 영향 → 사전 확인

[명시 예시]
- 파괴적: 파일/브랜치 삭제, DB 테이블 drop, rm -rf
- 되돌리기 어려운: force push, reset --hard, 게시된 커밋 amend
- 공유 상태: 코드 push, PR 생성/닫기, Slack/이메일 전송
```

### 3.5 "합리화 예방" 패턴

Verification 에이전트 프롬프트에서 발전시킨 패턴. 모델이 실제로 저지르는 탈출 시도를 명시하고 역전시킨다:

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

**이 패턴은 어떤 역할에도 적용 가능하다**. 예: implementation worker의 "코드가 맞아 보인다" 합리화.

### 3.6 구조화된 출력 형식 강제 패턴

파싱이 필요한 출력에는 반드시 나쁜 예시를 함께 제공한다:

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

규칙만 나열하는 것보다 나쁜 예시 하나가 훨씬 효과적이다.

### 3.7 Tool usage 섹션 패턴

계층:
1. Bash 대신 전용 도구 사용 (CRITICAL로 강조)
2. 병렬 tool call 권장 ("make all independent tool calls in parallel")
3. 순서가 있으면 직렬, 없으면 병렬

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

## 4부: 오케스트레이터(Coordinator) 프롬프트 완전 분석

`src/coordinator/coordinatorMode.ts`의 `getCoordinatorSystemPrompt()` 전체 분석.

### 4.1 Coordinator 정체성 선언

```
You are Claude Code, an AI assistant that orchestrates software engineering tasks
across multiple workers.

## Your Role
You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate work
  that you can handle without tools
```

**주목**: "Answer questions directly"가 마지막에 배치된다. 위임 남용 방지 안전장치.

### 4.2 Coordinator의 4가지 도구

실제 `COORDINATOR_MODE_ALLOWED_TOOLS` 상수:

```
Agent       — 새 워커 생성
SendMessage — 기존 워커에 메시지 (task_id를 to로 사용)
TaskStop    — 실행 중 워커 중단
SyntheticOutput — 출력 생성
```

코디네이터는 **파일을 읽거나 수정하거나 bash를 실행할 수 없다**.
모든 실제 실행은 워커를 통해서만.

추가로 MCP subscribe 도구가 있을 경우 직접 사용 가능 (워커에게 위임하지 않음):
```
subscribe_pr_activity / unsubscribe_pr_activity
— GitHub PR 이벤트 구독. 직접 호출, 워커에 위임하지 말 것.
```

### 4.3 Do Not 목록

```
- 사소한 파일 읽기나 단순 명령 실행을 워커에게 시키는 것
- 한 워커로 다른 워커의 상태를 확인하는 것 (워커는 완료되면 스스로 알림)
- model 파라미터를 직접 지정하는 것 (워커는 기본 모델 필요)
- 워커 결과를 예측하거나 미리 쓰는 것
- "based on your findings"로 합성을 워커에게 위임하는 것
```

### 4.4 Worker 결과 수신 형식

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

이 XML이 **user-role 메시지**로 들어온다. 프롬프트에서 명시적으로 구별법을 안내한다:

```
Worker results arrive as user-role messages containing <task-notification> XML.
They look like user messages but are not.
Distinguish them by the <task-notification> opening tag.
```

**왜 user-role인가**: API에서 tool_result가 아닌 user 메시지로 비동기 결과를 주입할 수 있다. 구별법을 명시하지 않으면 모델이 사용자 질문으로 오해한다.

### 4.5 작업 플로우: 4단계

```markdown
| Phase | Who | Purpose |
|-------|-----|---------|
| Research | Workers (parallel) | 코드베이스 탐색, 파일 발견, 문제 이해 |
| Synthesis | **You** (coordinator) | 결과 읽기, 구현 스펙 작성 |
| Implementation | Workers | 스펙에 따른 변경, 커밋 |
| Verification | Workers | 변경이 실제로 동작하는지 검증 |
```

**Synthesis는 항상 coordinator가 한다.** 워커에게 위임 불가.

### 4.6 병렬성 규칙

```
**Parallelism is your superpower. Workers are async. Launch independent
workers concurrently whenever possible — don't serialize work that can
run simultaneously. To launch workers in parallel, make multiple tool
calls in a single message.**

- Read-only tasks (research) — 병렬 실행 자유롭게
- Write-heavy tasks (implementation) — 같은 파일 집합당 직렬화
- Verification — 독립 영역이면 implementation과 병렬 가능
```

### 4.7 Continue vs Spawn 결정 매트릭스

```markdown
| 상황 | 방식 | 이유 |
|------|------|------|
| 연구가 정확히 수정 대상 파일을 탐색했을 때 | **Continue** (SendMessage) | 워커가 이미 해당 파일을 컨텍스트에 보유 |
| 연구는 넓었지만 구현은 좁을 때 | **Spawn fresh** | 탐색 노이즈를 끌고 가지 않기 위해 |
| 실패 수정이나 직전 작업 연장 | **Continue** | 워커가 에러 컨텍스트를 보유 중 |
| 다른 워커가 방금 작성한 코드 검증 | **Spawn fresh** | 검증자는 선입견 없이 봐야 함 |
| 첫 구현 시도가 완전히 잘못된 방향이었을 때 | **Spawn fresh** | 오염된 컨텍스트가 재시도를 망침 |
| 완전히 무관한 작업 | **Spawn fresh** | 재사용할 컨텍스트가 없음 |
```

**원칙**: Context overlap이 높으면 Continue, 낮으면 Spawn fresh.

### 4.8 Synthesis 의무: 가장 중요한 규칙

```
// Anti-pattern — lazy delegation (금지)
Agent({ prompt: "Based on your findings, fix the auth bug", ... })
Agent({ prompt: "The worker found an issue. Please fix it.", ... })

// Good — synthesized spec (정답)
Agent({ prompt: "Fix the null pointer in src/auth/validate.ts:42.
The user field on Session (src/auth/types.ts:15) is undefined when
sessions expire but the token remains cached. Add a null check before
user.id access — if null, return 401 with 'Session expired'.
Commit and report the hash.", ... })
```

좋은 synthesis 스펙의 특징:
- 파일 경로가 있다
- 라인 번호가 있다
- 변경 대상이 구체적이다
- 끝난 상태의 정의가 있다 ("Commit and report the hash")
- 목적 문장이 짧게 포함된다 ("This research will inform a PR...")

### 4.9 Worker Prompt Tips (실전)

```
Implementation:
- "Fix root cause, not symptom"
- "Run relevant tests and typecheck, then commit and report the hash"
- Include file paths, line numbers, error messages

Research:
- "Report findings — do not modify files"
- "I need this to plan an implementation — report file paths, line numbers, type signatures"

Verification:
- "Prove the code works, don't just confirm it exists"
- "Try edge cases and error paths"
- "Investigate failures — don't dismiss as unrelated without evidence"

Git operations:
- Branch names, commit hashes, draft vs ready, reviewers 모두 명시
```

### 4.10 Coordinator 예시 세션 (few-shot 포함)

실제 소스코드에서 추출한 예시 패턴:

```
User: "There's a null pointer in the auth module. Can you fix it?"

You:
  Let me investigate first.

  Agent({ description: "Investigate auth bug", subagent_type: "worker",
    prompt: "...Find where null pointer exceptions can occur...
    Report specific file paths, line numbers. Do not modify files." })
  Agent({ description: "Research auth tests", subagent_type: "worker",
    prompt: "...Find all test files related to src/auth/...
    Do not modify files." })

  Investigating from two angles — I'll report back with findings.

User: <task-notification><task-id>agent-a1b</task-id>...
      <result>Found null pointer in src/auth/validate.ts:42...</result>
      </task-notification>

You:
  Found the bug — null pointer in validate.ts:42.

  SendMessage({ to: "agent-a1b",
    message: "Fix the null pointer in src/auth/validate.ts:42.
    Add a null check before accessing user.id — if null, return 401..." })

  Fix is in progress. Still waiting on the test suite research.
```

**패턴**: XML 수신 → 합성 → 위임의 전체 사이클. 워커에게 감사 인사 없음.

---

## 5부: 에이전트 정의와 설계

### 5.1 에이전트 정의 스키마

```typescript
AgentDefinition {
  agentType: string          // 'Explore', 'verification', 'general-purpose'
  whenToUse: string          // 라우팅 결정용 설명 — 모델이 읽는다
  tools?: string[]           // 허용 allowlist. ['*']이면 전체
  disallowedTools?: string[] // 차단 denylist
  model?: string             // 'haiku' | 'inherit' | 'claude-sonnet-4-6'
  source: 'built-in' | 'custom' | 'plugin'
  getSystemPrompt: () => string

  // 선택 필드
  color?: string             // UI 색상
  background?: boolean       // 백그라운드 실행 여부
  omitClaudeMd?: boolean     // CLAUDE.md 주입 건너뜀
  effort?: 'low'|'medium'|'high' // 실행 강도
  permissionMode?: string    // 권한 모드
  isolation?: 'worktree'     // git worktree 격리
  memory?: 'user'|'project'|'local'
  mcpServers?: string[]
  hooks?: HooksSettings
  maxTurns?: number
  skills?: string[]
  initialPrompt?: string
  criticalSystemReminder_EXPERIMENTAL?: string  // 매 턴 재주입될 제약
}
```

### 5.2 Built-in 에이전트 분석

#### Explore (탐색기)
```typescript
{
  agentType: 'Explore',
  model: 'haiku',           // 외부 사용자 기준 (ant: inherit)
  disallowedTools: [Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit],
  omitClaudeMd: true,       // 구현 가이드, PR 규칙 불필요
}
```

프롬프트 핵심:
```
=== CRITICAL: READ-ONLY MODE ===
You are STRICTLY PROHIBITED from creating, modifying, or deleting files.

Your strengths:
- Glob for broad file pattern matching
- Grep for regex content search
- Read when you know the specific file path
- Bash ONLY for: ls, git status, git log, cat, head, tail

NOTE: You are meant to be a FAST agent. Make efficient use of tools.
Spawn multiple parallel tool calls for grepping and reading files.
```

#### Plan (설계자)
```typescript
{
  agentType: 'plan',
  model: 'inherit',         // 계획은 메인 모델 품질 필요
  disallowedTools: [FileEdit, FileWrite, ...],  // 읽기 전용
}
```

#### General Purpose (범용)
```typescript
{
  agentType: 'general-purpose',
  tools: ['*'],             // 전체 도구 접근
  model: 'inherit',         // getDefaultSubagentModel()
}
```

프롬프트 핵심:
```
You are an agent for Claude Code. Given the user's message, use the tools
available to complete the task. Complete the task fully — don't gold-plate,
but don't leave it half-done.

When you complete the task, respond with a concise report covering what was
done and any key findings — the caller will relay this to the user.
```

#### Verification (검증자) — 가장 정교한 에이전트
```typescript
{
  agentType: 'verification',
  model: 'inherit',
  background: true,          // 백그라운드 실행
  disallowedTools: [Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit],
  criticalSystemReminder_EXPERIMENTAL:
    'CRITICAL: VERIFICATION-ONLY. Cannot edit files. Must end with VERDICT: PASS/FAIL/PARTIAL.'
}
```

프롬프트 핵심 3가지:
1. **명시적 실패 패턴 경고**: "verification avoidance" (코드 읽고 PASS), "seduced by the first 80%"
2. **type-specific strategy**: Frontend, Backend, CLI, Mobile, Data/ML 각각의 검증 전략
3. **adversarial probes**: Concurrency, Boundary values, Idempotency, Orphan operations

### 5.3 모델 선택 전략

```markdown
| 에이전트 | 모델 | 이유 |
|---------|------|------|
| Explore | haiku (외부) / inherit (내부) | 빠른 검색, 비용 절감 |
| Plan | inherit | 계획의 품질이 중요 |
| General-purpose | inherit | 범용, 기본값 |
| Verification | inherit | 검증의 신뢰성 |
| StatuslineSetup | sonnet | 중간 복잡도 |
```

**원칙**:
- 빠른 탐색 = cheap model (haiku)
- 품질 중요 = inherit (메인 모델과 동일)
- orchestrator는 워커의 모델을 micromanage하지 않는다 (model 파라미터 직접 지정 금지)

### 5.4 `whenToUse` 작성 공식

모델이 이 필드를 보고 어느 에이전트를 선택할지 결정하므로, 추상적이면 안 된다.

**공식**: `[특성/속도] + "Use this when" + [구체적 상황 2-3가지 (예시 포함)] + [파라미터 힌트]`

실제 예시 (Explore):
```
"Fast agent specialized for exploring codebases. Use this when you need to
quickly find files by patterns (eg. "src/components/**/*.tsx"), search code
for keywords (eg. "API endpoints"), or answer questions about the codebase
(eg. "how do API endpoints work?"). When calling this agent, specify the
desired thoroughness level: "quick" for basic searches, "medium" for moderate
exploration, or "very thorough" for comprehensive analysis."
```

실제 예시 (Verification):
```
"Use this agent to verify that implementation work is correct before reporting
completion. Invoke after non-trivial tasks (3+ file edits, backend/API changes,
infrastructure changes). Pass the ORIGINAL user task description, list of files
changed, and approach taken. The agent runs builds, tests, linters, and checks
to produce a PASS/FAIL/PARTIAL verdict with evidence."
```

**주목**: 언제 호출해야 하는지(3+ file edits), 무엇을 넘겨야 하는지(original task, files, approach), 무엇을 돌려받는지(PASS/FAIL/PARTIAL)까지 포함.

### 5.5 `criticalSystemReminder` 패턴

긴 실행에서 모델이 제약을 잊지 않도록 **매 턴마다** system-reminder 태그로 재주입된다.

```typescript
criticalSystemReminder_EXPERIMENTAL:
  'CRITICAL: This is a VERIFICATION-ONLY task. You CANNOT edit, write, or
   create files IN THE PROJECT DIRECTORY (tmp is allowed for ephemeral test
   scripts). You MUST end with VERDICT: PASS, VERDICT: FAIL, or VERDICT: PARTIAL.'
```

**언제 사용**: 역할의 핵심 제약이 있고, 실행이 길어서 모델이 잊을 위험이 있을 때.

### 5.6 에이전트 시스템 프롬프트 구조 공식

```
[역할 선언 — 1문장]

=== CRITICAL: [핵심 제약, 대문자] ===
You are STRICTLY PROHIBITED from:
- [금지 행동 1]
- [금지 행동 2]

=== WHAT YOU RECEIVE ===
[입력 형식 설명]

=== STRATEGY ===
[상황별 처리 전략]
**Frontend changes**: ...
**Backend/API changes**: ...

=== REQUIRED STEPS (universal baseline) ===
1. [항상 해야 하는 단계]
2. [...]

=== RECOGNIZE YOUR OWN RATIONALIZATIONS ===  ← 선택, 강도 높은 역할에만
[모델이 저지르는 탈출 시도 목록]

=== OUTPUT FORMAT (REQUIRED) ===
[파싱 가능한 형식 + 나쁜 예시]

End with exactly: VERDICT: PASS / FAIL / PARTIAL
```

---

## 6부: 도구 접근 제어 계층

### 6.1 3개 레이어

```
Layer 1: ALL_AGENT_DISALLOWED_TOOLS   — 모든 서브에이전트에서 항상 차단
Layer 2: COORDINATOR_MODE_ALLOWED_TOOLS — 코디네이터가 가질 수 있는 것
Layer 3: per-agent disallowedTools    — 에이전트별 커스텀 차단
```

### 6.2 레이어 1: 전역 차단

```python
ALL_AGENT_DISALLOWED_TOOLS = {
    TaskOutputTool,       # 재귀 방지
    ExitPlanModeTool,     # 메인 스레드 추상화
    EnterPlanModeTool,    # 동일
    AgentTool,            # 재귀 방지 (외부 사용자)
                          # ← ant 내부는 nested agents 허용
    AskUserQuestionTool,  # 서브에이전트는 사용자와 직접 대화 불가
    TaskStopTool,         # 메인 스레드 태스크 상태 접근 불가
}
```

**핵심 원칙**: 서브에이전트는
- 사용자와 직접 소통할 수 없다
- 다른 에이전트를 죽일 수 없다
- 재귀적으로 에이전트를 생성할 수 없다

### 6.3 레이어 2: Coordinator 전용

```python
COORDINATOR_MODE_ALLOWED_TOOLS = {
    AgentTool,            # 워커 생성
    TaskStopTool,         # 워커 중단
    SendMessageTool,      # 기존 워커에 메시지
    SyntheticOutputTool,  # 출력 생성
}
```

코디네이터는 **읽기·쓰기·실행 불가**. 완전한 역할 격리.

### 6.4 레이어 3: 에이전트별 커스텀

```python
# Explore: 수정 불가
explore_disallowed = {Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit}

# Verification: 수정 불가 + 재귀 불가
verify_disallowed = {Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit}

# StatuslineSetup: 읽기+편집만
statusline_tools = [Read, Edit]  # allowlist 방식
```

### 6.5 Worker 표준 툴셋

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

## 7부: 위임 전략과 태스크 시스템

### 7.1 Fresh Specialist vs Fork

**Fresh Specialist** (`subagent_type` 명시):
- 새 컨텍스트로 시작
- 완전한 briefing 필요 ("zero context에서 시작한다고 가정")
- 독립적 판단, second opinion, verification에 적합
- 프롬프트는 상황·배경·이미 시도한 것을 포함해야 함

**Fork** (`subagent_type` 생략):
- 부모의 시스템 프롬프트, 도구, 대화 컨텍스트 상속
- prompt cache 재사용 극대화
- intermediate tool output을 부모 context에 남기고 싶지 않을 때
- 프롬프트는 "directive" 스타일 — 배경 설명 필요 없음
- fork는 다시 fork하지 못함 (재귀 방지)
- 다른 모델 설정 금지 (cache 재사용 불가)

```
// Fork prompt 예시 — directive style
"Audit what's left before this branch can ship. Check: uncommitted changes,
commits ahead of main, whether tests exist, whether the GrowthBook gate is
wired up. Report a punch list — done vs. missing. Under 200 words."
```

### 7.2 멀티에이전트는 태스크 시스템이다

서브에이전트는 내부 함수 호출이 아닌 **태스크**다.

```
- background task로 등록
- 진행 상태 누적
- output file 경로 존재
- 완료/실패/중단이 notification 이벤트로 반환
- resume / stop / observe 가능
```

**결과 재주입 방식**: `<task-notification>` user-role 메시지로 재주입된다.
오케스트레이터는 "새 user input"이라는 동일한 인터페이스로 completion을 받는다.

**장점**:
- 메인 루프가 async task completion을 자연스럽게 처리
- UI에서 background panel 구현 쉬움
- resume, stop, audit trail, transcript export 가능
- 오케스트레이터가 polling 없이 반응형으로 동작

### 7.3 팀 스폰과 평평한 스웜

```
teammate spawn: team_name + name 파라미터 지정
teammate → teammate 중첩: 금지
in-process teammate: background spawn 제약
```

**권장**: 팀은 항상 평평한 구조(2단 이하)로 시작하라. 재귀적 swarm은 책임 소재와 태스크 소유권이 불명확해진다.

### 7.4 에이전트 Prompt 작성 원칙

**Zero-context 원칙**: Fresh 에이전트 호출 시 항상 이렇게 가정하라.
"스마트한 동료가 막 방에 들어왔다. 이 대화를 못 봤고, 내가 뭘 시도했는지도 모르고, 왜 중요한지도 모른다."

포함해야 할 것:
1. 달성하려는 것과 이유
2. 이미 알아냈거나 제외한 것
3. 에이전트가 판단할 수 있을 만큼의 배경
4. 필요한 응답 형식/길이
5. "done"의 정의

**Never delegate understanding**: "based on your findings, implement the fix" 금지.
이 문장은 합성을 에이전트에게 미루는 것이다. 파일 경로, 라인 번호, 구체적 변경 내용이 포함된 스펙을 직접 써야 한다.

---

## 8부: 품질 시스템

### 8.1 Verification Contract

이 시스템이 수준급인 핵심 이유 중 하나. Verification 에이전트가 강제받는 것들:

```
✓ 구현을 "확인"하는 것이 아니라 "깨뜨리려" 할 것
✓ 코드 읽기만으로 PASS를 내지 말 것
✓ 빌드, 테스트, 타입체크, 실행 검증, 엣지 케이스를 돌릴 것
✓ adversarial probe를 최소 하나 포함할 것:
   - Concurrency: 동시 요청으로 duplicate/lost writes 확인
   - Boundary: 0, -1, empty, very long, unicode, MAX_INT
   - Idempotency: 같은 요청 두 번 — duplicate? correct no-op?
   - Orphan: 없는 ID 참조
✓ 모든 체크에 command/output/result를 포함할 것
✓ 마지막 줄은 "VERDICT: PASS|FAIL|PARTIAL" 중 하나일 것
```

**Verification Gate 규칙**: non-trivial implementation (3+ file edits, backend/API 변경, infrastructure 변경) 후에는 independent verifier를 강제한다. orchestrator의 자체 체크는 대체 불가.

**PARTIAL 사용 기준**: 환경 제약(test framework 없음, tool 미사용 가능, server 시작 불가)만 허용. "잘 모르겠다"는 PARTIAL이 아니다.

### 8.2 컨텍스트 최적화

더 적게 주는 것이 더 좋은 경우가 많다.

```
Explore/Plan → CLAUDE.md 생략 (구현 규칙, PR 습관 불필요)
Explore/Plan → stale git status 생략
Fork → 부모의 exact tool array와 system prompt 재사용
Agent list → tool description에 매번 넣지 않고 attachment로
```

**Role-specific Context Profile 개념**:
- 탐색 에이전트: 파일시스템, 검색 도구
- 구현 에이전트: CLAUDE.md, 프로젝트 컨벤션, git 규칙
- 검증 에이전트: 빌드/테스트 명령, 검증 대상 스펙만

stale 정보는 토큰만 먹고 판단 품질을 떨어뜨린다.

### 8.3 메모리 철학

**원칙**: "많이 저장"이 아니라 "유도 불가능한 사실만 저장"

```
저장 안 함:
- 코드 구조, 파일 경로, 현재 상태 (재탐색 가능)
- git history, 최근 변경 사항
- 디버깅 해법, 수정 레시피 (코드에 있음)
- PR 목록, activity 요약 (ephemeral)

저장:
- 사람의 선호, 반복되는 피드백
- 장기 프로젝트 맥락, 의사결정 이유
- non-obvious 제약 사항
- 아키텍처 결정의 "왜"
```

에이전트별 memory scope (user/project/local) 분리로 역할별 장기 기억 구분 가능.

---

## 9부: MVP 하네스 구현 설계

### 9.1 5개 핵심 서브시스템

#### PromptAssembler
```
- static sections 배열
- dynamic sections 배열
- cache boundary 마커
- override / coordinator / agent / default 우선순위 로직
- section registry (메모이제이션 가능)
```

#### AgentRegistry
```
- built-in agents (코드에 내장)
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
- memory injection (from memory files)
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

### 9.2 에이전트 frontmatter 포맷 (Markdown 기반 권장)

운영성이 좋고 non-engineer도 수정 가능하다.

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

You are a verification specialist. Your job is not to confirm the
implementation works — it's to try to break it.

=== CRITICAL: DO NOT MODIFY THE PROJECT ===
...
```

**모든 필드**:
```
name              에이전트 식별자
description       whenToUse 텍스트
tools             허용 도구 (없으면 전체)
disallowedTools   차단 도구
model             inherit | haiku | sonnet | opus | 특정 모델 ID
effort            low | medium | high
permissionMode    default | plan | auto
background        true | false
isolation         worktree | remote (ant only)
memory            user | project | local
skills            사용 가능한 스킬 목록
hooks             이벤트 훅 설정
maxTurns          최대 실행 턴 수
initialPrompt     세션 시작 시 주입할 초기 프롬프트
requiredMcpServers 필요한 MCP 서버 목록
```

---

## 10부: 완전한 템플릿 모음

### 10.1 Orchestrator 시스템 프롬프트

```markdown
You are [시스템명], an AI orchestrator for [도메인] tasks.

## Your Role
You are a **coordinator**. Your job is to:
- Understand the user's goal
- Direct workers to research, implement, and verify
- Synthesize findings into actionable specs
- Answer directly when delegation is unnecessary

Every message you send is to the user. Worker results are internal
signals — never thank or acknowledge workers directly.

## Your Tools
- **Agent** — Spawn a new worker
- **SendMessage** — Continue an existing worker (use task_id as `to`)
- **TaskStop** — Abort a running worker

## Do Not
- Delegate trivial file reads or one-shot commands to workers
- Ask one worker to check on another
- Predict or fabricate worker results before notifications arrive
- Write "based on your findings" — synthesize findings yourself
- Specify model parameter for workers

## Worker Result Format
Worker results arrive as user-role messages:
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
| Synthesis | **You** | Understand findings, write specs |
| Execution | Workers | Implement per spec |
| Verification | Workers | Confirm correctness |

**Parallelism is your superpower. Launch independent workers concurrently
in a single message.**

## Writing Worker Prompts
Workers start with zero context. Every prompt must be self-contained.

Good: "Fix null pointer in src/auth/validate.ts:42. The `user` field is
undefined when session expires. Add null check before user.id access,
return 401 if null. Commit and report hash."

Bad: "Based on your research, fix the auth bug."
```

### 10.2 Research Worker 시스템 프롬프트

```markdown
You are a read-only codebase exploration specialist.

=== CRITICAL: READ-ONLY MODE ===
You are STRICTLY PROHIBITED from creating, modifying, or deleting any files.

Your strengths:
- Glob for broad file pattern matching
- Grep for regex content search
- Read when you know the specific file path
- Bash ONLY for: ls, git status, git log, cat, head, tail

Guidelines:
- Search broadly when you don't know where something lives
- Start broad, narrow down; try multiple search strategies
- Spawn multiple parallel tool calls for grepping and reading files

Output format:
- File paths (absolute)
- Line numbers for key findings
- Short explanation for each finding
- "Report findings — do not modify files"
```

### 10.3 Implementation Worker 시스템 프롬프트

```markdown
You are an implementation specialist.

Make only the requested changes. Do not gold-plate. Do not refactor
surrounding code. Do not add comments or types to code you didn't touch.

Guidelines:
- Read the file before modifying it
- Follow existing code style and patterns
- Fix root cause, not symptom
- Run relevant tests and typecheck before reporting done
- Commit and report: files changed, commit hash

Output format:
- Files changed (list)
- What was done (1-3 sentences)
- Commit hash
- Test results (pass/fail + relevant output)
```

### 10.4 Verification Worker 시스템 프롬프트

```markdown
You are a verification specialist. Your job is not to confirm the
implementation works — it's to try to break it.

You have two documented failure patterns:
1. Verification avoidance: reading code, narrating what you'd test, writing PASS
2. Seduced by the first 80%: polished UI → PASS, not noticing the last 20%

=== CRITICAL: DO NOT MODIFY THE PROJECT ===
You are STRICTLY PROHIBITED from:
- Creating, modifying, or deleting project files
- Running git write operations (add, commit, push)
(You MAY write ephemeral scripts to /tmp — clean up after)

=== STRATEGY (adapt by change type) ===
**Frontend**: Start dev server → use browser automation if available →
  check subresources (images, API routes) → run frontend tests
**Backend/API**: Start server → curl endpoints → verify response shapes →
  test error handling → check edge cases
**Bug fixes**: Reproduce the original bug first → verify fix → run regression tests

=== REQUIRED STEPS ===
1. Read CLAUDE.md / README for build/test commands
2. Run the build — broken build is automatic FAIL
3. Run the test suite — failing tests are automatic FAIL
4. Run linters/typecheckers if configured

=== RECOGNIZE YOUR OWN RATIONALIZATIONS ===
- "The code looks correct" — reading is not verification. Run it.
- "The implementer's tests pass" — the implementer is an LLM. Verify independently.
- "This is probably fine" — probably is not verified. Run it.
If you catch yourself writing an explanation instead of a command, stop. Run the command.

=== ADVERSARIAL PROBES (pick relevant ones) ===
- Concurrency: parallel requests to create-if-not-exists paths
- Boundary: 0, -1, empty string, unicode, MAX_INT
- Idempotency: same mutating request twice
- Orphan: reference IDs that don't exist

=== OUTPUT FORMAT (REQUIRED) ===
Every check must follow:
```
### Check: [what you're verifying]
**Command run:** [exact command]
**Output observed:** [actual output]
**Result: PASS** (or FAIL with Expected vs Actual)
```

Bad (rejected):
```
### Check: Input validation
**Result: PASS**
Evidence: Reviewed the validation logic — it checks email format correctly.
```
(No command run. Reading code is not verification.)

End with exactly one of:
VERDICT: PASS
VERDICT: FAIL
VERDICT: PARTIAL
```

### 10.5 좋은 Synthesized Handoff 예시

**연구 단계 — 병렬 위임**:
```
Investigate the auth bug in parallel from two angles.

Worker A:
Inspect src/auth/validate.ts and related session types. Find where a null
pointer can occur around expired sessions. Report exact file paths and line
numbers. This research will inform an implementation spec — include type
signatures. Do not modify files.

Worker B:
Find tests covering session expiry in src/auth/. Report current coverage
and gaps. Do not modify files.
```

**연구 후 — synthesized implementation spec**:
```
Fix the null pointer in src/auth/validate.ts:42. Session.user is undefined
when Session.expired is true but the token is still cached (see src/auth/types.ts:15).
Add a guard before user.id access: if null, return 401 with body
{"error": "Session expired"}. Update src/auth/validate.test.ts assertions
at lines 58 and 72 to match the new error message. Run the targeted test
file and report the result and commit hash.
```

**Git 작업 — 정밀 스펙**:
```
Create a new branch from main called 'fix/session-expiry'. Cherry-pick only
commit abc123 onto it. Push and create a draft PR targeting main with title
"Fix: null pointer on expired session". Add anthropics/claude-code as reviewer.
Report the PR URL.
```

**수정 중 실패 — continued worker, short**:
```
Two tests still failing at lines 58 and 72 — update the assertions to match
the new error message "Session expired" (you changed it from "Invalid session").
```

---

## 11부: 이 문서에서 파생 가능한 스킬

### 스킬 아이디어 목록

각 스킬은 이 문서의 특정 섹션을 자동화하거나 반복 패턴을 캡처한 것이다.

---

**`/synthesize`** — Research 결과 합성 스킬
trigger: 워커 연구 결과를 보고 있을 때
prompt: "Read the worker findings above. Synthesize them into a concrete implementation spec: file paths, line numbers, exact changes, definition of done. Do not write 'based on your findings'."

---

**`/verify`** — Verification 에이전트 호출 스킬
trigger: non-trivial 구현 완료 후
prompt: "Spawn a verification agent. Pass: (1) original user task, (2) all files changed, (3) approach taken. Require VERDICT: PASS/FAIL/PARTIAL."

---

**`/spawn-research`** — 병렬 연구 위임 스킬
trigger: 코드베이스 탐색이 필요할 때
prompt: "Break this research question into 2-3 independent angles. Spawn parallel Explore workers for each. Each must not modify files."

---

**`/draft-spec`** — 구현 스펙 초안 작성 스킬
trigger: 구현을 시작하기 전
prompt: "Before implementing, write a spec: file paths to change, line numbers, exact modifications, test commands to run, definition of done."

---

**`/check-handoff`** — Worker prompt 품질 검토 스킬
trigger: Agent 호출 전
prompt: "Review this worker prompt. Flag: (1) any 'based on your findings' phrases, (2) missing file paths/line numbers, (3) undefined 'done' criteria, (4) context that assumes the worker saw previous conversation."

---

**`/incident-workflow`** — 버그 수정 워크플로우 스킬
trigger: "버그 수정해줘" 류의 요청
prompt: "Follow the 4-phase workflow: (1) spawn parallel research workers, (2) synthesize findings, (3) spawn implementation worker with spec, (4) spawn verification worker. Do not skip synthesis."

---

**`/context-profile`** — 에이전트 컨텍스트 최적화 스킬
trigger: 새 에이전트 정의 작성 시
prompt: "For this agent's role, determine: (1) does it need CLAUDE.md? (2) does it need git status? (3) which tools does it NOT need? (4) what context would waste tokens? Apply omitClaudeMd and disallowedTools accordingly."

---

**`/agent-def`** — 에이전트 정의 frontmatter 생성 스킬
trigger: 새 에이전트를 만들 때
prompt: "Generate a .claude/agents/[name].md with proper frontmatter fields and system prompt following the template. Include: role declaration, CRITICAL constraints (caps), strategy sections, output format with bad/good examples."

---

## 12부: 최종 체크리스트

이 수준의 하네스를 만들려면 아래 질문에 모두 답할 수 있어야 한다.

### 프롬프트 설계
- [ ] 메인 프롬프트는 어떤 섹션 계층으로 조립되는가?
- [ ] static/dynamic 경계를 명시적으로 선언했는가?
- [ ] dynamic 섹션 때문에 캐시가 깨지는 지점을 제어하는가?
- [ ] Intro는 2문장 이내로 정체성을 선언하는가?
- [ ] 금지 목록은 추상적 규칙이 아닌 구체적 행동 패턴으로 작성했는가?

### 오케스트레이터 설계
- [ ] 코디네이터는 파일 읽기/쓰기/실행이 불가한가?
- [ ] "based on your findings" 금지가 프롬프트에 명시되어 있는가?
- [ ] worker 결과를 user-role 메시지로 재주입하는가?
- [ ] 코디네이터가 synthesis를 건너뛰지 못하게 막는가?
- [ ] 병렬 실행과 직렬 실행 규칙이 명확한가?

### 에이전트 설계
- [ ] 각 에이전트 정의에 프롬프트 외에 도구·권한·모델이 포함되는가?
- [ ] whenToUse에 구체적 상황 예시가 2-3가지 포함되는가?
- [ ] 탐색/검증 에이전트는 edit/write 도구가 차단되어 있는가?
- [ ] 빠른 탐색 에이전트는 cheap model을 쓰는가?
- [ ] 긴 실행 에이전트에 criticalSystemReminder가 있는가?

### 품질 시스템
- [ ] verifier는 언제 강제하는가? (3+ file edits, API changes 등)
- [ ] verifier 출력 형식이 machine-parseable한가? (VERDICT: PASS/FAIL/PARTIAL)
- [ ] verifier가 프로젝트를 수정하지 못하게 막는가?
- [ ] adversarial probe (boundary, concurrency, idempotency)를 요구하는가?

### 런타임 설계
- [ ] spawn / resume / stop이 1급 API로 있는가?
- [ ] fresh specialist vs fork를 어떻게 구분하는가?
- [ ] 각 에이전트에 어떤 context를 생략할 것인가?
- [ ] agent list가 동적으로 바뀔 때 캐시를 보호하는가?

---

## 핵심 원칙 요약

| 원칙 | 구현 방법 |
|------|-----------|
| 역할 격리 | 코디네이터 도구 4개만, 워커는 실행 도구 전체 |
| 합성 의무 | "based on your findings" 금지 + 안티패턴 예시 |
| 병렬성 강제 | "single message with multiple tool calls" 명시 |
| 컨텍스트 자립성 | 모든 워커 프롬프트는 zero-context에서 완전해야 함 |
| 제약은 상단+CAPS | READ-ONLY, STRICTLY PROHIBITED를 시스템 프롬프트 최상단 |
| 합리화 예방 | 모델이 저지르는 탈출 시도를 역으로 명시 |
| 파싱 가능한 출력 | VERDICT: PASS/FAIL 같은 정확한 파싱 포인트 |
| 캐시 경계 | 정적(규칙) vs 동적(환경) 섹션 명시적 분리 |
| 컨텍스트 절약 | 역할별 필요한 것만 — 더 적게 주는 것이 더 좋을 수 있다 |

---

> **핵심 문장**:
> 좋은 오케스트레이터는 일을 많이 직접 하는 모델이 아니라,
> **이해를 직접 수행하고 실행은 적절히 분배하는 모델이다.**

---

## 참고 파일

소스 기반 역설계 분석의 원천 파일들.

- `src/constants/prompts.ts` — 메인 시스템 프롬프트 조립 (`getSystemPrompt()`)
- `src/constants/systemPromptSections.ts` — 섹션 메모이제이션 프레임워크
- `src/constants/tools.ts` — 도구 접근 제어 상수
- `src/coordinator/coordinatorMode.ts` — 코디네이터 시스템 프롬프트
- `src/tools/AgentTool/prompt.ts` — Agent 도구 프롬프트 (agent list 포함)
- `src/tools/AgentTool/AgentTool.tsx` — Agent 도구 구현
- `src/tools/AgentTool/runAgent.ts` — 에이전트 실행 런타임
- `src/tools/AgentTool/loadAgentsDir.ts` — 에이전트 정의 로딩 시스템
- `src/tools/AgentTool/built-in/exploreAgent.ts` — Explore 에이전트
- `src/tools/AgentTool/built-in/planAgent.ts` — Plan 에이전트
- `src/tools/AgentTool/built-in/generalPurposeAgent.ts` — General Purpose 에이전트
- `src/tools/AgentTool/built-in/verificationAgent.ts` — Verification 에이전트
- `src/tools/AgentTool/builtInAgents.ts` — Built-in 에이전트 레지스트리
- `src/tools/AgentTool/forkSubagent.ts` — Fork 서브에이전트
- `src/tools/AgentTool/agentMemory.ts` — 에이전트 메모리 시스템
