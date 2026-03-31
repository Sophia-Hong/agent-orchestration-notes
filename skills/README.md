# 오케스트레이션 스킬 모음

**버전**: v1.0.0
**작성일**: 2026-03-31
**작성자**: Sophia (with Claude Code)

---

## 작성 배경

2026년 3월 31일, Claude Code 소스코드가 npm 패키지 소스맵을 통해 유출되었다.
이 레포(`~/projects/Claude Code/claude-code/`)가 그 스냅샷이다.

소스코드를 역설계 분석하면서 Anthropic이 내부적으로 어떻게 오케스트레이터 프롬프트를 설계하고, 에이전트를 정의하고, 품질 게이트를 강제하는지를 파악했다. 분석을 마치고 나서 "이걸 매번 직접 할 수는 없다"는 생각에 반복 가능한 패턴을 스킬로 추출했다.

**핵심 동기**: 멀티에이전트 작업을 할 때 내가 자꾸 저지르는 실수들을 막기 위해.
- 연구 결과를 읽지 않고 그대로 다음 워커에 넘기는 lazy delegation
- 구현이 끝나도 검증을 생략하고 "됐을 것 같다"고 넘어가는 것
- 워커 프롬프트가 zero-context에서 불완전한 채로 나가는 것
- 병렬로 할 수 있는 것을 순서대로 처리하는 것

---

## 참조 문서

### 1차 분석 원천
**`~/projects/Claude Code/claude-code/`** — Claude Code 소스코드 스냅샷 (2026-03-31 유출본)

핵심 분석 파일:
- `src/constants/prompts.ts` — 메인 시스템 프롬프트 조립 로직
- `src/coordinator/coordinatorMode.ts` — 코디네이터 시스템 프롬프트 전문
- `src/tools/AgentTool/built-in/verificationAgent.ts` — 검증 에이전트 프롬프트
- `src/tools/AgentTool/built-in/exploreAgent.ts` — 탐색 에이전트 프롬프트
- `src/constants/tools.ts` — 도구 접근 제어 상수

### 정리 문서
**[ORCHESTRATOR_HARNESS_GUIDE.ko.md](~/projects/Claude%20Code/claude-code/ORCHESTRATOR_HARNESS_GUIDE.ko.md)**
역설계 분석 두 편을 통합한 아키텍처 나침반 문서.
이 스킬들의 모든 설계 근거가 여기에 있다. 스킬을 수정하거나 새로 만들 때 먼저 참조.

---

## 내 의도

이 스킬들은 "AI한테 시키기 위한 것"이 아니라 **내가 오케스트레이터로 일할 때 따르기 위한 프로토콜**이다.

Claude Code를 쓸 때 나 자신이 코디네이터가 된다. 코디네이터의 역할은:
- 직접 실행하는 것이 아니라 **판단하고 분해하고 합성하는 것**
- 연구 결과를 **직접 이해한 뒤** 스펙으로 변환하는 것
- 모든 non-trivial 구현은 **독립 검증을 통과한 뒤** 완료로 보고하는 것

이 스킬들은 그 원칙을 실제 워크플로우로 굳혀놓은 것이다.

---

## 스킬 목록

### 워크플로우 스킬

| 스킬 | 파일 | 언제 |
|------|------|------|
| `/orchestrate` | [orchestrate.md](orchestrate.md) | 큰 작업을 처음부터 끝까지 — 4단계 플로우 강제 |
| `/research` | [research.md](research.md) | 구현 전 코드베이스 탐색이 필요할 때 |
| `/synthesize` | [synthesize.md](synthesize.md) | 연구 결과가 돌아온 직후 — 스펙으로 변환 |
| `/spec` | [spec.md](spec.md) | 코딩 시작 전 계획을 명확히 할 때 |
| `/verify` | [verify.md](verify.md) | 구현 완료 후 — PASS 없이 완료 보고 금지 |
| `/handoff` | [handoff.md](handoff.md) | 워커 프롬프트를 보내기 전 품질 점검 |

### 일반 사용 흐름

```
처음부터 끝까지:
  /orchestrate

단계별로:
  /research          ← 탐색 에이전트 병렬 실행
      ↓ 결과 도착
  /synthesize        ← 직접 읽고 스펙 작성
      ↓
  (구현)
      ↓
  /verify            ← VERDICT: PASS 게이트

코딩 전에 계획만:
  /spec

프롬프트 점검:
  /handoff           ← 워커에게 보내기 전
```

---

## 설계 원칙 요약

이 스킬들이 강제하는 것:

1. **합성은 위임하지 않는다** — "based on your findings" 금지. `/synthesize`가 이걸 막는다.
2. **검증은 독립적이어야 한다** — `/verify`는 구현 워커와 다른 에이전트를 쓴다.
3. **병렬로 할 수 있으면 병렬** — `/research`는 단일 메시지에 다중 호출을 담는다.
4. **프롬프트는 zero-context에서 완결** — `/handoff`가 이걸 점검한다.
5. **done의 정의가 있어야 완료** — `/spec`이 완료 기준을 먼저 쓰게 만든다.

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0.0 | 2026-03-31 | 초기 작성 — Claude Code 소스 역설계 기반 6개 스킬 |
