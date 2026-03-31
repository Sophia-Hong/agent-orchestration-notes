# agent-orchestration-notes

Claude Code 소스코드 역설계 분석을 바탕으로 작성한 멀티에이전트 오케스트레이션 아키텍처 레퍼런스.

**버전**: v1.0.0 · **작성일**: 2026-03-31

---

## 문서

- [`ORCHESTRATOR_HARNESS_GUIDE.md`](ORCHESTRATOR_HARNESS_GUIDE.md) — 핵심 아키텍처 나침반. 시스템 프롬프트 구조, 오케스트레이터 설계, 에이전트 정의, 품질 시스템, 템플릿 모음.
- [`skills/`](skills/) — Claude Code에서 바로 사용할 수 있는 오케스트레이션 스킬 파일들.

## 스킬

`~/.claude/skills/`에 복사하면 `/skill-name`으로 호출 가능.

| 스킬 | 파일 | 언제 |
|------|------|------|
| `/orchestrate` | [skills/orchestrate.md](skills/orchestrate.md) | 4단계 멀티에이전트 플로우 전체 |
| `/research` | [skills/research.md](skills/research.md) | 병렬 탐색 에이전트 실행 |
| `/synthesize` | [skills/synthesize.md](skills/synthesize.md) | 연구 결과 → 구현 스펙 변환 |
| `/spec` | [skills/spec.md](skills/spec.md) | 코딩 전 스펙 작성 |
| `/verify` | [skills/verify.md](skills/verify.md) | 독립 검증 에이전트 실행 |
| `/handoff` | [skills/handoff.md](skills/handoff.md) | 워커 프롬프트 품질 점검 |

---

English → [`../`](../)
