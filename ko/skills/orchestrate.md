---
description: 복잡한 작업을 멀티에이전트 오케스트레이션 워크플로우로 실행한다. Research → Synthesis → Implementation → Verification 4단계를 강제한다. 큰 기능 추가, 버그 수정, 리팩토링 등 3개 이상 파일이 바뀔 작업에 사용.
---

지금부터 다음 오케스트레이션 워크플로우를 따른다. 단계를 건너뛰지 않는다.

## Phase 1: Research (병렬)

작업을 2-3개의 독립적인 조사 각도로 분해한다. 각각 별도의 Explore 또는 general-purpose 에이전트로 병렬 위임한다. 단일 메시지에 여러 Agent 호출을 담아 동시 실행한다.

각 연구 워커 프롬프트에 반드시 포함:
- 조사할 구체적인 범위
- "Report file paths, line numbers, type signatures. Do not modify files."
- 이 연구가 무엇을 위한 것인지 한 문장 ("This research will inform an implementation spec")

## Phase 2: Synthesis (나 혼자)

워커 결과가 돌아오면 직접 읽고 이해한다. 절대 "based on your findings"를 쓰지 않는다.

합성 결과에 반드시 포함:
- 정확한 파일 경로
- 라인 번호
- 구체적으로 무엇을 바꿔야 하는지
- "done"의 정의 (커밋 해시, 테스트 통과 등)

## Phase 3: Implementation

합성된 스펙을 가지고 implementation 워커를 생성한다. 연구 워커와 같은 파일을 탐색했다면 SendMessage로 계속 진행하고, 아니면 새로 Spawn한다.

워커에게 전달: "Run relevant tests and typecheck, then commit and report the hash."

## Phase 4: Verification

구현이 완료되면 반드시 독립적인 verification 워커를 spawnn한다.

전달 내용:
1. 원본 사용자 요청 (그대로)
2. 변경된 파일 목록
3. 구현 접근 방식
4. (있으면) 플랜 파일 경로

VERDICT: PASS가 나올 때까지 완료를 보고하지 않는다.

---

지금 바로 Phase 1부터 시작한다.
