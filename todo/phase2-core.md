# Phase 2: 핵심 기능 구현 (3~5주차)

> 목표: 각 컴포넌트(C 서버, IoT 기기, 웹)의 핵심 비즈니스 로직 완성

---

## TODO 목록

### T2-01: C 서버 기기 인증 (DEVICE_AUTH)

- **담당**: 멤버 A
- **설명**: IoT 기기의 API Key 인증 처리
- **참조 문서**: [docs/c-server/server-design.md](../docs/c-server/server-design.md) §4, [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §1.3
- **완료 조건**:
  - [ ] auth.c — DEVICE_AUTH 메시지 처리
  - [ ] API Key SHA-256 해시 비교 (equipment.api_key_hash)
  - [ ] 인증 성공 → AUTH_OK, 실패 → AUTH_FAIL 응답
  - [ ] tcp_handler.c에서 DEVICE_AUTH 디스패치 연결

---

### T2-02: C 서버 사용자 로그인 (LOGIN)

- **담당**: 멤버 A
- **설명**: 전화번호 기반 사용자 로그인 처리
- **참조 문서**: [docs/c-server/server-design.md](../docs/c-server/server-design.md) §4.1, [docs/c-server/session-design.md](../docs/c-server/session-design.md)
- **완료 조건**:
  - [ ] auth.c — LOGIN 메시지 처리
  - [ ] 전화번호 → SHA-256 → DB 조회 → bcrypt 검증
  - [ ] 기존 활성 세션 확인 → 중복 로그인 처리 (FORCE_LOGOUT)
  - [ ] 새 세션 생성 (sessions INSERT, equipment UPDATE)
  - [ ] LOGIN_OK 응답 (user_id, today 통계)
  - [ ] LOGIN_FAIL 응답 (user_not_found, consent_required)

---

### T2-03: C 서버 신규 사용자 등록 (REGISTER)

- **담당**: 멤버 B
- **설명**: 기기에서의 신규 사용자 등록
- **참조 문서**: [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §1.3, [docs/legal/legal-compliance.md](../docs/legal/legal-compliance.md) §1.1
- **완료 조건**:
  - [ ] auth.c — REGISTER 메시지 처리
  - [ ] 전화번호 중복 확인
  - [ ] 전화번호 bcrypt 해시 + SHA-256 인덱스 해시 생성
  - [ ] 이메일 AES-256 암호화
  - [ ] users INSERT (consent_at 기록)
  - [ ] REGISTER_OK / REGISTER_FAIL 응답

---

### T2-04: C 서버 세션 관리자 구현

- **담당**: 멤버 A
- **설명**: 기기 세션 생성/조회/종료 관리
- **참조 문서**: [docs/c-server/session-design.md](../docs/c-server/session-design.md) §1
- **완료 조건**:
  - [ ] session_manager.c — 세션 생성 (g_sessions 배열 등록)
  - [ ] session_manager.c — 세션 조회 (device_id, user_id 기반)
  - [ ] session_manager.c — 세션 종료 (칼로리 정산 + DB 기록 + 자원 해제)
  - [ ] 중복 로그인 감지 및 기존 세션 강제 종료

---

### T2-05: C 서버 하트비트 처리

- **담당**: 멤버 A
- **설명**: 3초 주기 하트비트 수신/응답, 칼로리 실시간 갱신
- **참조 문서**: [docs/c-server/server-design.md](../docs/c-server/server-design.md) §4.2, [docs/c-server/calorie-design.md](../docs/c-server/calorie-design.md)
- **완료 조건**:
  - [ ] tcp_handler.c — HEARTBEAT 메시지 처리
  - [ ] last_heartbeat, last_sensor_change 갱신
  - [ ] 칼로리 재계산 (calorie.c 호출)
  - [ ] DASHBOARD 응답 (today_total, current 칼로리/시간)

---

### T2-06: C 서버 칼로리 계산 로직

- **담당**: 멤버 B
- **설명**: 기구별 칼로리 계산, 강도 계수 산출
- **참조 문서**: [docs/c-server/calorie-design.md](../docs/c-server/calorie-design.md)
- **완료 조건**:
  - [ ] calorie.c — calc_calories(calorie_per_min, elapsed_sec, avg_intensity)
  - [ ] calorie.c — calc_intensity(sensor_value, base_value) (0.5~2.0)
  - [ ] 기구 유형별 base_value 설정
  - [ ] 자동 로그아웃 시 마지막 센서 시점까지만 계산
  - [ ] 단위 테스트: 경계값, 기구별 계산 정확성

---

### T2-07: C 서버 로그아웃 처리 (LOGOUT)

- **담당**: 멤버 A
- **설명**: 사용자 수동 로그아웃, 칼로리 정산, DB 기록
- **참조 문서**: [docs/c-server/session-design.md](../docs/c-server/session-design.md) §1, [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §1.3
- **완료 조건**:
  - [ ] sessions UPDATE (end_time, duration_sec, calories)
  - [ ] daily_stats UPSERT
  - [ ] equipment 상태 → available
  - [ ] access_log INSERT (logout_device)
  - [ ] LOGOUT_OK 응답 (session_calories, session_duration_sec)

---

### T2-08: C 서버 자동 로그아웃 워치독

- **담당**: 멤버 B
- **설명**: 워치독 스레드 — 하트비트/비활동/영업종료 감시
- **참조 문서**: [docs/c-server/session-design.md](../docs/c-server/session-design.md) §2, §3
- **완료 조건**:
  - [ ] session_watchdog.c — 3초 주기 점검 스레드
  - [ ] 하트비트 타임아웃 (10초) → FORCE_LOGOUT (heartbeat_timeout)
  - [ ] 비활동 타임아웃 (3분) → FORCE_LOGOUT (inactivity)
  - [ ] 비활동 경고 (2분) → WARNING (remaining_sec)
  - [ ] 영업 종료 시각 체크 → FORCE_LOGOUT (gym_close)
  - [ ] 강제 종료 시 칼로리 정산 + DB 기록 + 기구 상태 변경

---

### T2-09: C 서버 Redis 연동

- **담당**: 멤버 B
- **설명**: 기구 실시간 상태를 Redis에 캐시
- **참조 문서**: [docs/database/redis-design.md](../docs/database/redis-design.md) §3.1
- **완료 조건**:
  - [ ] redis_client.c — Redis 연결/명령 실행
  - [ ] equipment:{id} Hash 갱신 (HSET + EXPIRE)
  - [ ] equipment:all JSON 갱신 (SET + EXPIRE)
  - [ ] user:{id}:today Hash 갱신

---

### T2-10: C 서버 시작 시 미완료 세션 정리

- **담당**: 멤버 B
- **설명**: 서버 재시작 시 end_time IS NULL 세션 자동 정리
- **참조 문서**: [docs/c-server/server-design.md](../docs/c-server/server-design.md) §5
- **완료 조건**:
  - [ ] main.c 초기화에서 미완료 세션 조회
  - [ ] 각 세션: end_time = last_heartbeat, 칼로리 정산
  - [ ] auto_logout_reason = 'server_cleanup'
  - [ ] equipment.status = 'available'

---

### T2-11: IoT 기기 DEVICE_AUTH + LOGIN 구현

- **담당**: 멤버 C
- **설명**: 기기 인증 및 로그인 메시지 송수신
- **참조 문서**: [docs/iot-device/device-design.md](../docs/iot-device/device-design.md) §5, [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §1.2
- **완료 조건**:
  - [ ] network.c — DEVICE_AUTH 전송 + AUTH_OK/FAIL 수신
  - [ ] network.c — LOGIN 전송 + LOGIN_OK/FAIL 수신
  - [ ] 화면 전환: AUTH_OK → 로그인 화면, LOGIN_OK → 대시보드

---

### T2-12: IoT 기기 등록 화면 (REGISTER)

- **담당**: 멤버 C
- **설명**: 미등록 사용자 등록 + 동의 화면
- **참조 문서**: [docs/iot-device/device-design.md](../docs/iot-device/device-design.md) §2.1, [docs/legal/legal-compliance.md](../docs/legal/legal-compliance.md) §1.1
- **완료 조건**:
  - [ ] display.c — 등록 화면 (전화번호+이메일+동의)
  - [ ] display.c — 동의 화면 (개인정보 수집 동의 문구)
  - [ ] REGISTER 메시지 전송 + REGISTER_OK/FAIL 수신

---

### T2-13: IoT 기기 대시보드 화면

- **담당**: 멤버 C
- **설명**: 오늘 총 칼로리, 현재 칼로리, 시간 표시
- **참조 문서**: [docs/iot-device/device-design.md](../docs/iot-device/device-design.md) §2.3
- **완료 조건**:
  - [ ] display.c — 대시보드 레이아웃 (칼로리, 시간, 센서 정보)
  - [ ] DASHBOARD 응답 수신 → 화면 갱신
  - [ ] 3초 주기 자동 갱신

---

### T2-14: IoT 기기 센서 시뮬레이션

- **담당**: 멤버 C
- **설명**: 센서 데이터 생성 + HEARTBEAT 전송
- **참조 문서**: [docs/iot-device/device-design.md](../docs/iot-device/device-design.md) §3
- **완료 조건**:
  - [ ] sensor.c — 4가지 모드 (NORMAL, INCREASE, DECREASE, IDLE)
  - [ ] 기구 유형별 센서 범위 프리셋
  - [ ] 3초 주기 HEARTBEAT 전송 (sensor_data 포함)
  - [ ] 모드 전환 키 바인딩 (시연용)

---

### T2-15: IoT 기기 로그아웃 + 비활동 경고

- **담당**: 멤버 C
- **설명**: 운동 종료 버튼 + 비활동 경고 오버레이
- **참조 문서**: [docs/iot-device/device-design.md](../docs/iot-device/device-design.md) §2.4, [docs/c-server/session-design.md](../docs/c-server/session-design.md) §2.2
- **완료 조건**:
  - [ ] 운동 종료 버튼 → LOGOUT 메시지 전송 → 로그인 화면 복귀
  - [ ] WARNING 수신 → 경고 오버레이 (카운트다운)
  - [ ] FORCE_LOGOUT 수신 → 로그인 화면 복귀 + 안내 메시지

---

### T2-16: IoT 기기 Encrypted Storage

- **담당**: 멤버 C
- **설명**: 운동 중 데이터 로컬 암호화 저장
- **참조 문서**: [docs/iot-device/device-design.md](../docs/iot-device/device-design.md) §4
- **완료 조건**:
  - [ ] storage.c — 세션 데이터 암호화 저장/읽기
  - [ ] LOGOUT 시 로컬 데이터를 서버 전송 후 삭제
  - [ ] FORCE_LOGOUT 시 동일 처리

---

### T2-17: 웹 로그인 API + 페이지

- **담당**: 멤버 D
- **설명**: 전화번호 웹 로그인, 세션 쿠키 발급
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §3.1, [docs/c-server/session-design.md](../docs/c-server/session-design.md) §5
- **완료 조건**:
  - [ ] api/auth/login/route.js — SHA-256 → DB 조회 → bcrypt 검증 → 세션 발급
  - [ ] lib/session.js — Redis 세션 CRUD (생성, 조회, 삭제)
  - [ ] lib/crypto.js — SHA-256, AES-256 유틸리티
  - [ ] login/page.js — 로그인 폼 UI (전화번호 입력)
  - [ ] components/auth/LoginForm.js

---

### T2-18: 웹 회원가입 (이메일 인증)

- **담당**: 멤버 D
- **설명**: 이메일 인증 코드 발송/검증 + 회원가입
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §3.2, [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §2.1
- **완료 조건**:
  - [ ] api/auth/email-verify/send/route.js — 코드 생성, Redis 저장, 이메일 발송
  - [ ] api/auth/email-verify/confirm/route.js — 코드 검증, users INSERT, 세션 발급
  - [ ] lib/email.js — nodemailer (네이버 SMTP) 설정
  - [ ] register/page.js — 회원가입 폼 UI
  - [ ] components/auth/RegisterForm.js, EmailVerify.js

---

### T2-19: 웹 세션 미들웨어 + 로그아웃

- **담당**: 멤버 D
- **설명**: 인증 미들웨어 + 로그아웃 API
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §3.3, [docs/c-server/session-design.md](../docs/c-server/session-design.md) §5
- **완료 조건**:
  - [ ] lib/auth-middleware.js — 쿠키 → Redis → 사용자 식별
  - [ ] Redis 장애 시 MySQL fallback
  - [ ] api/auth/logout/route.js — Redis/MySQL 세션 삭제 + 쿠키 제거
  - [ ] api/auth/me/route.js — 현재 로그인 상태 확인

---

### T2-20: 웹 운동 기록 API (일별/주간/월별/연도별)

- **담당**: 멤버 D
- **설명**: daily_stats 기반 운동 기록 조회 API
- **참조 문서**: [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §2.3
- **완료 조건**:
  - [ ] api/records/daily/route.js — 일별 조회 (세션 목록 포함)
  - [ ] api/records/weekly/route.js — 주간 조회 (일별 합산)
  - [ ] api/records/monthly/route.js — 월별 조회
  - [ ] api/records/yearly/route.js — 연도별 조회
  - [ ] Prisma 쿼리 최적화 (daily_stats 인덱스 활용)

---

### T2-21: 웹 운동 기록 페이지 + 차트

- **담당**: 멤버 D
- **설명**: 운동 기록 조회 UI + Recharts 시각화
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §2
- **완료 조건**:
  - [ ] records/page.js — 기간 선택 + 기록 표시
  - [ ] components/records/RecordTable.js — 기록 테이블
  - [ ] components/records/PeriodSelector.js — 기간 선택 UI
  - [ ] components/charts/CalorieChart.js — Recharts 칼로리 차트
  - [ ] components/charts/DurationChart.js — 운동 시간 차트
  - [ ] components/charts/ChartWrapper.js — 공통 차트 래퍼

---

### T2-22: 웹 Rate Limit 구현

- **담당**: 멤버 D
- **설명**: Redis 기반 API Rate Limiting
- **참조 문서**: [docs/security/security-design.md](../docs/security/security-design.md) §3.4
- **완료 조건**:
  - [ ] lib/rate-limit.js — Redis INCR + TTL 기반 제한
  - [ ] 로그인 API: IP당 5회/분
  - [ ] 이메일 코드: IP 3회/분 + 이메일 3회/10분
  - [ ] 전체 API: IP당 100회/분
  - [ ] 제한 초과 시 429 Too Many Requests 응답
