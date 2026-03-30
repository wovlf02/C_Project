# 멤버 A — 서버 개발자 A

> 역할: C TCP 서버 코어, 네트워크, 세션 관리  
> 주요 기술: C, POSIX Socket, pthread, libmysqlclient, cJSON

---

## 1. 담당 영역

| 영역 | 설명 |
|------|------|
| TCP 서버 | 소켓 바인딩, 연결 수락, 스레드 관리 |
| 메시지 핸들러 | TCP 메시지 수신/파싱/디스패치 |
| 기기 인증 | DEVICE_AUTH 처리 |
| 사용자 인증 | LOGIN 처리 (로그인/중복 감지) |
| 세션 관리 | 세션 생성/조회/종료 |
| 로그아웃 | LOGOUT + FORCE_LOGOUT 처리 |
| 접속 기록 | access_log 기록 (C 서버 측) |
| 시연 환경 | Docker + 시작 스크립트 |

---

## 2. 담당 TODO 목록

### Phase 1 (1~2주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T1-01 | Docker 환경 구성 | [docs/architecture/system-overview.md](../../docs/architecture/system-overview.md) |
| T1-03 | C 서버 프로젝트 초기화 | [docs/c-server/server-design.md](../../docs/c-server/server-design.md) |
| T1-04 | MySQL Connection Pool | [docs/database/schema-design.md](../../docs/database/schema-design.md) |
| T1-05 | 기본 TCP 소켓 구현 | [docs/c-server/server-design.md](../../docs/c-server/server-design.md) |

### Phase 2 (3~5주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T2-01 | 기기 인증 (DEVICE_AUTH) | [docs/architecture/communication-flow.md](../../docs/architecture/communication-flow.md) |
| T2-02 | 사용자 로그인 (LOGIN) | [docs/c-server/session-design.md](../../docs/c-server/session-design.md) |
| T2-04 | 세션 관리자 구현 | [docs/c-server/session-design.md](../../docs/c-server/session-design.md) |
| T2-05 | 하트비트 처리 | [docs/c-server/calorie-design.md](../../docs/c-server/calorie-design.md) |
| T2-07 | 로그아웃 처리 | [docs/c-server/session-design.md](../../docs/c-server/session-design.md) |

### Phase 3 (5~6주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T3-01 | C서버 ↔ IoT 기기 통합 (with C) | [docs/architecture/communication-flow.md](../../docs/architecture/communication-flow.md) |
| T3-02 | 다중 기기 동시 접속 테스트 (with C) | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |

### Phase 4 (6~7주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T4-06 | access_log 전체 적용 (C서버 측) | [docs/security/security-design.md](../../docs/security/security-design.md) |

### Phase 5 (7~8주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T5-02 | 자동 로그아웃 5가지 시나리오 (with C) | [docs/c-server/session-design.md](../../docs/c-server/session-design.md) |
| T5-03 | 부하 테스트 (with B) | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |
| T5-06 | 시연 환경 구성 | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |

---

## 3. 협업 시점

| 시점 | 협업 대상 | 내용 |
|------|-----------|------|
| Phase 1, T1-05 완료 후 | 멤버 C | TCP 연결 테스트, 프로토콜 인터페이스 합의 |
| Phase 2, T2-01~T2-02 | 멤버 B | 기기 인증 + 사용자 등록 연계 |
| Phase 2, T2-04~T2-05 | 멤버 B | 세션 관리 + 워치독 연계, 칼로리 정산 호출 |
| Phase 3, T3-01~T3-02 | 멤버 C | 기기 시뮬레이터 통합 테스트 |
| Phase 4, T4-06 | 멤버 D | access_log 포맷 합의 (C서버↔웹 일관성) |
| Phase 5, T5-03 | 멤버 B | 부하 테스트 스크립트 공동 작성 |
| Phase 5, T5-07 | 전원 | 시연 리허설 |

---

## 4. 주요 산출물

```
server/src/main.c
server/src/tcp_server.c
server/src/tcp_handler.c
server/src/session_manager.c
server/src/auth.c (LOGIN/DEVICE_AUTH 부분)
server/include/session.h
server/config/server.conf
docker-compose.yml
```
