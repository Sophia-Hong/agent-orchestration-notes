---
description: 구현 완료 후 독립 검증 에이전트를 실행한다. 3개 이상 파일 수정, API/백엔드 변경, 인프라 변경 후 반드시 사용. VERDICT: PASS 없이는 완료 보고를 하지 않는다.
---

지금 완료된 구현에 대해 독립 verification 워커를 즉시 spawn한다.

다음을 수집해서 워커 프롬프트를 구성한다:

1. **원본 요청**: 사용자가 처음 요청한 내용 (그대로, 요약하지 않음)
2. **변경된 파일**: 이번 구현에서 수정/생성/삭제된 파일 전체 목록
3. **접근 방식**: 어떤 방식으로 구현했는지 2-3줄
4. **플랜 파일**: 있으면 경로 포함

워커에게 명시:
- "Your job is to break this, not to confirm it works."
- "Do not modify any project files."
- "End with exactly: VERDICT: PASS, VERDICT: FAIL, or VERDICT: PARTIAL"

워커가 돌아온 후:
- **PASS**: 보고서에서 Command run 블록이 있는 체크 2-3개를 직접 재실행해 검증. 모두 일치하면 완료 보고.
- **FAIL**: 실패 내용을 읽고 fix를 구현한 뒤 같은 워커를 SendMessage로 계속 (에러 컨텍스트 보유). PASS 나올 때까지 반복.
- **PARTIAL**: 검증된 부분과 검증 못 한 부분을 사용자에게 그대로 보고.
