# TODO 문서 인덱스

> 최종 수정: 2026-03-30

---

## Phase 구성

| Phase | 문서 | 기간 | 핵심 목표 |
|-------|------|------|-----------|
| Phase 1 | [phase1-setup.md](phase1-setup.md) | 1~2주차 | 환경 구축 + 기반 코드 |
| Phase 2 | [phase2-core.md](phase2-core.md) | 3~5주차 | 핵심 기능 구현 |
| Phase 3 | [phase3-integration.md](phase3-integration.md) | 5~6주차 | 시스템 통합 + 웹 완성 |
| Phase 4 | [phase4-security-legal.md](phase4-security-legal.md) | 6~7주차 | 보안 + 법률 준수 |
| Phase 5 | [phase5-testing-demo.md](phase5-testing-demo.md) | 7~8주차 | 테스트 + 시연 준비 |

---

## 진행 상태 범례

- `[ ]` 미시작
- `[→]` 진행 중
- `[✓]` 완료

---

## 전체 요약

- **총 TODO 수**: 68개
- **Phase 1**: 16개 (환경 + 기반)
- **Phase 2**: 22개 (핵심 기능)
- **Phase 3**: 12개 (통합 + 웹)
- **Phase 4**: 10개 (보안 + 법률)
- **Phase 5**: 8개 (테스트 + 시연)

---

## 의존성 그래프 (Phase 간)

```
Phase 1 ──▶ Phase 2 ──▶ Phase 3 ──▶ Phase 4 ──▶ Phase 5
(기반)       (핵심)       (통합)       (보안)       (시연)
```

- Phase 2는 Phase 1의 환경/DB/기본 통신이 완료되어야 시작 가능
- Phase 3는 Phase 2의 각 컴포넌트 개별 기능이 완료되어야 시작 가능
- Phase 4는 Phase 3 통합이 완료되어야 보안 점검 가능
- Phase 5는 Phase 4까지 완료 후 최종 테스트/시연
