# 멤버 B — 서버 개발자 B

> 역할: C 서버 칼로리/인증/워치독, DB 최적화, Redis 연동  
> 주요 기술: C, OpenSSL, MySQL, Redis, pthread

---

## 1. 담당 영역

| 영역 | 설명 |
|------|------|
| 암호화 유틸 | SHA-256, bcrypt 래퍼 함수 |
| 사용자 등록 | REGISTER 처리 (해시+암호화+DB) |
| 칼로리 계산 | 기구별 칼로리, 강도 계수 산출 |
| 워치독 스레드 | 자동 로그아웃 감시 (5가지 시나리오) |
| Redis 연동 | 기구 상태 캐시 갱신 |
| DB CRUD | Prepared Statement 기반 쿼리, daily_stats UPSERT |
| 서버 복구 | 시작 시 미완료 세션 정리 |
| 로깅 | 레벨별 로깅 모듈 |
| SQL 스크립트 | 스키마, 인덱스, 트리거, 시드 |

---

## 2. 담당 TODO 목록

### Phase 1 (1~2주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T1-02 | MySQL 스키마 구현 | [docs/database/schema-design.md](../../docs/database/schema-design.md) |
| T1-13 | C 서버 암호화 유틸리티 | [docs/security/security-design.md](../../docs/security/security-design.md) |
| T1-14 | C 서버 로깅 모듈 | [docs/c-server/server-design.md](../../docs/c-server/server-design.md) |
| T1-15 | C 서버 기본 DB CRUD | [docs/database/schema-design.md](../../docs/database/schema-design.md) |

### Phase 2 (3~5주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T2-03 | 신규 사용자 등록 (REGISTER) | [docs/architecture/communication-flow.md](../../docs/architecture/communication-flow.md) |
| T2-06 | 칼로리 계산 로직 | [docs/c-server/calorie-design.md](../../docs/c-server/calorie-design.md) |
| T2-08 | 자동 로그아웃 워치독 | [docs/c-server/session-design.md](../../docs/c-server/session-design.md) |
| T2-09 | C 서버 Redis 연동 | [docs/database/redis-design.md](../../docs/database/redis-design.md) |
| T2-10 | 서버 시작 시 미완료 세션 정리 | [docs/c-server/server-design.md](../../docs/c-server/server-design.md) |

### Phase 3 (5~6주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T3-03 | Redis 기구 상태 캐시 통합 (with D) | [docs/database/redis-design.md](../../docs/database/redis-design.md) |
| T3-11 | daily_stats 정합성 확인 (with D) | [docs/database/schema-design.md](../../docs/database/schema-design.md) |

### Phase 4 (6~7주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T4-01 | 입력값 검증 강화 (C 서버 측) | [docs/security/security-design.md](../../docs/security/security-design.md) |
| T4-02 | SQL Injection 방어 확인 (C 서버 측) | [docs/security/security-design.md](../../docs/security/security-design.md) |

### Phase 5 (7~8주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T5-01 | 단위 테스트 (C 부분) | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |
| T5-03 | 부하 테스트 (with A) | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |

---

## 3. 협업 시점

| 시점 | 협업 대상 | 내용 |
|------|-----------|------|
| Phase 1, T1-02 완료 후 | 멤버 D | Prisma 스키마와 SQL DDL 일치 확인 |
| Phase 2, T2-03 | 멤버 A | REGISTER 핸들러를 tcp_handler에 연결 |
| Phase 2, T2-08 | 멤버 A | 워치독에서 세션 관리자의 force_logout 호출 |
| Phase 2, T2-06 | 멤버 A | 하트비트 처리에서 칼로리 계산 함수 호출 |
| Phase 3, T3-03 | 멤버 D | Redis 키 형식 확인, equipment:all JSON 구조 합의 |
| Phase 3, T3-11 | 멤버 D | daily_stats 갱신 타이밍, 데이터 형식 합의 |
| Phase 4, T4-01~T4-02 | 멤버 D | C/Next.js 양쪽 보안 점검 동시 진행 |
| Phase 5, T5-03 | 멤버 A | 부하 테스트 시뮬레이터 스크립트 공동 작성 |
| Phase 5, T5-07 | 전원 | 시연 리허설 |

---

## 4. 주요 산출물

```
server/src/auth.c (REGISTER 부분)
server/src/calorie.c
server/src/session_watchdog.c
server/src/redis_client.c
server/src/crypto_utils.c
server/src/db.c
server/src/db_pool.c
server/src/logger.c
server/include/calorie.h, crypto_utils.h, redis_client.h, db.h, db_pool.h, logger.h
sql/schema.sql, sql/indexes.sql, sql/triggers.sql, sql/init.sql
```
