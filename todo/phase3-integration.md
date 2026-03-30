# Phase 3: 시스템 통합 및 웹 완성 (5~6주차)

> 목표: C 서버 ↔ IoT 기기 ↔ 웹 전체 통합, 기구 현황 실시간 기능 완성

---

## TODO 목록

### T3-01: C 서버 ↔ IoT 기기 통합 테스트

- **담당**: 멤버 A + 멤버 C (협업)
- **설명**: 전체 TCP 통신 흐름 통합 검증
- **참조 문서**: [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §1, [docs/c-server/server-design.md](../docs/c-server/server-design.md), [docs/iot-device/device-design.md](../docs/iot-device/device-design.md)
- **완료 조건**:
  - [ ] 기기 인증 → 로그인 → 하트비트 → 로그아웃 전체 흐름 정상 동작
  - [ ] 미등록 사용자 등록 흐름 정상 동작
  - [ ] 중복 로그인 시 이전 기기 FORCE_LOGOUT 동작 확인
  - [ ] 비활동 경고 → 자동 로그아웃 동작 확인
  - [ ] 하트비트 타임아웃 자동 로그아웃 동작 확인

---

### T3-02: 다중 기기 동시 접속 테스트

- **담당**: 멤버 A + 멤버 C (협업)
- **설명**: 여러 기기 시뮬레이터 동시 실행
- **참조 문서**: [docs/c-server/server-design.md](../docs/c-server/server-design.md) §2, [docs/testing/test-demo-plan.md](../docs/testing/test-demo-plan.md)
- **완료 조건**:
  - [ ] 최소 5대 기기 동시 접속 + 하트비트 정상
  - [ ] 각 기기 독립적 세션 관리 확인
  - [ ] 동일 사용자 기구 변경 시 세션 전이 동작

---

### T3-03: Redis 기구 상태 캐시 통합

- **담당**: 멤버 B + 멤버 D (협업)
- **설명**: C 서버가 Redis에 쓴 기구 상태를 Next.js가 읽는 통합
- **참조 문서**: [docs/database/redis-design.md](../docs/database/redis-design.md) §3
- **완료 조건**:
  - [ ] C 서버 하트비트 처리 시 Redis equipment:{id}, equipment:all 갱신 확인
  - [ ] Next.js에서 Redis equipment:all 읽기 + API 응답 확인
  - [ ] 기구 상태 변경 (로그인/로그아웃) 시 Redis 반영 확인

---

### T3-04: 웹 기구 현황 페이지 (메인)

- **담당**: 멤버 D (+ 멤버 C 협업)
- **설명**: 전체 기구 현황 조회 + 혼잡도 표시
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §2, [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §2.4
- **완료 조건**:
  - [ ] page.js (메인) — 기구 카드 그리드 렌더링
  - [ ] components/equipment/EquipmentGrid.js
  - [ ] components/equipment/EquipmentCard.js — 상태별 색상 구분
  - [ ] components/equipment/OccupancyBar.js — 혼잡도 바
  - [ ] api/equipment/route.js — Redis 캐시 기반 응답
  - [ ] api/equipment/occupancy/route.js — 혼잡도 계산

---

### T3-05: 웹 WebSocket 실시간 기구 현황

- **담당**: 멤버 D (+ 멤버 C 협업)
- **설명**: WebSocket으로 기구 상태 실시간 브로드캐스트
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §4, [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §3
- **완료 조건**:
  - [ ] server.js — 커스텀 서버 (Next.js + ws 통합)
  - [ ] lib/ws-server.js — Redis 폴링 + 브로드캐스트 로직
  - [ ] hooks/useEquipment.js — WebSocket 클라이언트 훅
  - [ ] 기구 상태 변경 시 2~3초 내 웹 화면 갱신 확인
  - [ ] 연결 끊김 시 자동 재연결

---

### T3-06: 웹 사용자 대시보드 페이지

- **담당**: 멤버 D
- **설명**: 사용자 운동 요약 대시보드
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §2
- **완료 조건**:
  - [ ] dashboard/page.js — 오늘 통계 + 최근 기록 요약
  - [ ] api/records/summary/route.js — 종합 요약 API
  - [ ] components/records/SummaryCard.js — 요약 카드
  - [ ] hooks/useAuth.js — 인증 상태 관리 훅

---

### T3-07: 웹 기구 유형별 필터

- **담당**: 멤버 C
- **설명**: 유산소/무산소/스트레칭 필터링
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §1
- **완료 조건**:
  - [ ] components/equipment/TypeFilter.js — 유형 토글 UI
  - [ ] 기구 현황 + 운동 기록 페이지에서 공유 사용
  - [ ] 필터 상태에 따른 데이터 필터링

---

### T3-08: 웹 기구 상세 정보 모달

- **담당**: 멤버 C
- **설명**: 기구 클릭 시 상세 정보 표시
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §1
- **완료 조건**:
  - [ ] components/equipment/EquipmentDetail.js — 상세 모달
  - [ ] 현재 사용자 마스킹 표시 (010-****-1234)
  - [ ] 사용 시간, 기구 정보 표시
  - [ ] api/equipment/[id]/route.js — 상세 API

---

### T3-09: 웹 관리자 대시보드

- **담당**: 멤버 D
- **설명**: 관리자 전용 페이지 구현
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §1, [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §2.5
- **완료 조건**:
  - [ ] lib/admin-middleware.js — 관리자 권한 검증
  - [ ] admin/page.js — 관리자 대시보드
  - [ ] admin/equipment/page.js — 기구 CRUD
  - [ ] components/admin/EquipmentForm.js — 기구 추가/수정 폼
  - [ ] 관리자 API routes 전체 구현 (GET, POST, PUT, DELETE)

---

### T3-10: 웹 활성 세션 관리 (관리자)

- **담당**: 멤버 D
- **설명**: 관리자 활성 세션 조회 + 강제 로그아웃
- **참조 문서**: [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §2.5
- **완료 조건**:
  - [ ] admin/sessions/page.js — 활성 세션 목록
  - [ ] components/admin/SessionTable.js — 세션 테이블
  - [ ] api/admin/sessions/active/route.js — 활성 세션 조회
  - [ ] api/admin/sessions/[id]/force-logout/route.js
  - [ ] api/admin/gym/close/route.js — 영업 종료 일괄 로그아웃

---

### T3-11: 웹 ↔ C 서버 daily_stats 정합성 확인

- **담당**: 멤버 B + 멤버 D (협업)
- **설명**: C 서버가 기록한 daily_stats를 웹이 정확히 조회하는지 검증
- **참조 문서**: [docs/database/schema-design.md](../docs/database/schema-design.md) §5, [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §2.3
- **완료 조건**:
  - [ ] C 서버 세션 종료 시 daily_stats UPSERT 확인
  - [ ] 웹 API records/daily 조회 → daily_stats 데이터 일치 확인
  - [ ] 복수 세션 종료 후 누적 합계 정확성 확인

---

### T3-12: 전체 시스템 End-to-End 테스트

- **담당**: 전원
- **설명**: 기기 → C서버 → Redis/MySQL → Next.js → 웹 브라우저 전체 흐름
- **참조 문서**: [docs/testing/test-demo-plan.md](../docs/testing/test-demo-plan.md) §2
- **완료 조건**:
  - [ ] 시연 시나리오 2.1 (핵심 흐름) 전체 정상 동작
  - [ ] 기기 운동 → 웹 기록 조회 데이터 일치
  - [ ] 기구 상태 실시간 반영 확인
  - [ ] 관리자 기능 전체 동작 확인
