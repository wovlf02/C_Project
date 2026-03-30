# 멤버 C — IoT 기기 / 풀스택 개발자

> 역할: IoT 기기 시뮬레이터 전체, Web 기구 현황 UI, 공유 프로토콜 구현  
> 주요 기술: C, ncurses, POSIX 소켓, Next.js (부분), WebSocket

---

## 1. 담당 영역

| 영역 | 설명 |
|------|------|
| 공유 프로토콜 | shared/ 폴더의 protocol.h/c, message_builder, message_parser |
| IoT 네트워크 | TCP 소켓 연결, 4바이트 헤더 + JSON 송수신 |
| IoT UI (ncurses) | 로그인/대시보드/경고/회원가입 화면 |
| 센서 시뮬레이션 | RPM, 속도, 심박수, 무게 시뮬레이션 4가지 모드 |
| IoT 상태 머신 | 메인 루프, 상태 전이 관리 |
| 암호화 저장소 | EncryptedStorage (JSON 파일 암호화) |
| Web 기구 현황 | /equipment 페이지 (WebSocket 실시간) |
| WebSocket 클라이언트 | 프론트에서 ws 연결 훅 |

---

## 2. 담당 TODO 목록

### Phase 1 (1~2주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T1-06 | 공유 프로토콜 구현 (shared/) | [docs/architecture/communication-flow.md](../../docs/architecture/communication-flow.md) |
| T1-07 | IoT 기기 초기 프로젝트 구조 | [docs/iot-device/device-design.md](../../docs/iot-device/device-design.md) |
| T1-08 | IoT ncurses 기본 화면 | [docs/iot-device/device-design.md](../../docs/iot-device/device-design.md) |

### Phase 2 (3~5주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T2-11 | IoT 로그인/인증 화면 | [docs/iot-device/device-design.md](../../docs/iot-device/device-design.md) |
| T2-12 | IoT 회원가입 화면 | [docs/iot-device/device-design.md](../../docs/iot-device/device-design.md) |
| T2-13 | IoT 센서 시뮬레이션 | [docs/iot-device/device-design.md](../../docs/iot-device/device-design.md) |
| T2-14 | IoT 하트비트 전송 | [docs/architecture/communication-flow.md](../../docs/architecture/communication-flow.md) |
| T2-15 | IoT 대시보드 (운동 현황) | [docs/iot-device/device-design.md](../../docs/iot-device/device-design.md) |
| T2-16 | IoT 경고 오버레이 + 자동 로그아웃 대응 | [docs/c-server/session-design.md](../../docs/c-server/session-design.md) |

### Phase 3 (5~6주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T3-01 | C 서버 ↔ IoT 통합 테스트 (with A) | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |
| T3-02 | 다중 기기 동시 접속 (with A) | [docs/c-server/server-design.md](../../docs/c-server/server-design.md) |
| T3-07 | Web 기구 현황 페이지 (with D) | [docs/web/web-design.md](../../docs/web/web-design.md) |
| T3-08 | WebSocket 실시간 연동 (with D) | [docs/web/web-design.md](../../docs/web/web-design.md) |

### Phase 4 (6~7주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T4-05 | IoT 기기 입력값 검증 | [docs/security/security-design.md](../../docs/security/security-design.md) |

### Phase 5 (7~8주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T5-02 | 자동 로그아웃 시나리오 테스트 (with A) | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |
| T5-04 | UI 폴리싱 (IoT ncurses + 일부 Web) | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |

---

## 3. 협업 시점

| 시점 | 협업 대상 | 내용 |
|------|-----------|------|
| Phase 1, T1-06 완료 후 | 멤버 A, B | 프로토콜 코드를 서버/기기 양쪽에서 사용 가능 확인 |
| Phase 2, T2-11~T2-14 | 멤버 A | 로그인/인증/하트비트 메시지가 서버에 정상 수신되는지 확인 |
| Phase 2, T2-16 | 멤버 B | 워치독 FORCE_LOGOUT 수신 후 IoT 동작 확인 |
| Phase 3, T3-01, T3-02 | 멤버 A | 서버-IoT 통합 테스트 공동 진행 |
| Phase 3, T3-07, T3-08 | 멤버 D | 기구 현황 페이지 컴포넌트 분담 (C: WebSocket 훅, D: UI 레이아웃) |
| Phase 4, T4-05 | 멤버 B | IoT 입력값 검증과 서버 측 검증 일관성 확인 |
| Phase 5, T5-02 | 멤버 A, B | 5가지 자동 로그아웃 시나리오 IoT 측 동작 검증 |
| Phase 5, T5-07 | 전원 | 시연 리허설 (IoT 터미널 담당) |

---

## 4. 주요 산출물

```
shared/protocol.h, shared/protocol.c
shared/message_builder.c, shared/message_parser.c
device/src/main.c
device/src/network.c
device/src/ui_login.c
device/src/ui_dashboard.c
device/src/ui_register.c
device/src/ui_warning.c
device/src/sensor.c
device/src/encrypted_storage.c
device/include/*.h (각 모듈 헤더)
web/src/app/equipment/page.js
web/src/components/equipment/EquipmentGrid.js
web/src/components/equipment/EquipmentCard.js
web/src/hooks/useWebSocket.js
```
