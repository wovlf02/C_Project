# 통신 흐름 상세 설계

> 최종 수정: 2026-03-30

---

## 1. IoT 기기 ↔ C 서버 TCP 프로토콜

### 1.1 메시지 포맷

```
[4 bytes: payload length (big-endian)] [JSON payload (UTF-8)]
```

- 최대 페이로드: 8KB
- JSON 인코딩: UTF-8
- 라이브러리: cJSON (파싱/생성)

### 1.2 연결 생명주기

```
[기기 전원 ON]
  │
  ├─ 1. TCP 연결 수립 (서버:9000)
  ├─ 2. DEVICE_AUTH (API Key 인증)
  │     ├─ AUTH_OK → 대기 상태
  │     └─ AUTH_FAIL → 연결 종료
  │
  ├─ 3. 사용자 로그인 대기 (로그인 화면)
  │
  ├─ 4. LOGIN (전화번호 입력)
  │     ├─ LOGIN_OK → 대시보드 진입
  │     ├─ LOGIN_FAIL (user_not_found) → 등록 화면
  │     └─ LOGIN_FAIL (consent_required) → 동의 화면
  │
  ├─ [미등록시] 5. REGISTER (전화번호 + 이메일 + 동의)
  │     ├─ REGISTER_OK → 대시보드 진입
  │     └─ REGISTER_FAIL → 오류 표시
  │
  ├─ 6. HEARTBEAT (3초 주기: 센서 데이터)
  │     └─ DASHBOARD (응답: 칼로리/시간 정보)
  │
  ├─ [서버→기기] WARNING (비활동 2분 경과 시)
  ├─ [서버→기기] FORCE_LOGOUT (자동 로그아웃)
  │
  ├─ 7. LOGOUT (사용자 운동 종료)
  │     └─ LOGOUT_OK → 로그인 화면 복귀
  │
  └─ [기기 전원 OFF / 네트워크 단절]
        └─ 하트비트 미수신 10초 → 서버 측 강제 세션 종료
```

### 1.3 메시지 타입별 상세

#### 기기 → 서버

| 타입 | 필드 | 설명 |
|------|------|------|
| DEVICE_AUTH | device_id, api_key | 기기 최초 인증 |
| LOGIN | device_id, phone | 사용자 로그인 요청 |
| REGISTER | device_id, phone, email, consent | 신규 사용자 등록 |
| CONSENT | device_id, phone, email, agreed | 개인정보 동의 |
| HEARTBEAT | device_id, user_id, elapsed_sec, sensor_data | 주기적 상태 보고 |
| LOGOUT | device_id, user_id, local_data | 운동 종료 + 로컬 데이터 전송 |

#### 서버 → 기기

| 타입 | 필드 | 설명 |
|------|------|------|
| AUTH_OK / AUTH_FAIL | reason | 기기 인증 결과 |
| LOGIN_OK | user_id, today_total_calories, today_total_time_sec | 로그인 성공 |
| LOGIN_FAIL | reason | 로그인 실패 (user_not_found / consent_required) |
| REGISTER_OK | user_id, today_total_calories, today_total_time_sec | 등록 성공 |
| REGISTER_FAIL | reason | 등록 실패 |
| DASHBOARD | today_total_calories, current_calories, today_total_time_sec, current_time_sec | 대시보드 갱신 |
| WARNING | code, message, remaining_sec | 비활동 경고 |
| FORCE_LOGOUT | reason, message | 강제 로그아웃 |

---

## 2. Next.js REST API 설계

### 2.1 공통 응답 형식

```json
// 성공
{ "success": true, "data": { ... }, "error": null, "timestamp": "ISO8601" }

// 실패
{ "success": false, "data": null, "error": { "code": "ERROR_CODE", "message": "..." }, "timestamp": "ISO8601" }
```

### 2.2 인증 API

| Method | Endpoint | Body/Query | 인증 | Rate Limit |
|--------|----------|------------|------|------------|
| POST | /api/auth/login | { phone } | - | IP 5회/분 |
| POST | /api/auth/logout | - | 세션 | - |
| GET | /api/auth/me | - | 세션 | - |
| POST | /api/auth/email-verify/send | { phone, email } | - | IP 3회/분 + 이메일 3회/10분 |
| POST | /api/auth/email-verify/confirm | { phone, email, code, consent } | - | IP 5회/분 |

### 2.3 운동 기록 API

| Method | Endpoint | Query | 인증 | 데이터 소스 |
|--------|----------|-------|------|-------------|
| GET | /api/records/daily | date | 세션 | daily_stats + sessions |
| GET | /api/records/weekly | start | 세션 | daily_stats |
| GET | /api/records/monthly | year, month | 세션 | daily_stats |
| GET | /api/records/yearly | year | 세션 | daily_stats (GROUP BY) |
| GET | /api/records/custom | start, end | 세션 | daily_stats |
| GET | /api/records/summary | - | 세션 | daily_stats (집계) |

### 2.4 기구 현황 API

| Method | Endpoint | 인증 | 데이터 소스 |
|--------|----------|------|-------------|
| GET | /api/equipment | - | Redis (equipment:all) |
| GET | /api/equipment/:id | - | Redis (equipment:{id}) |
| GET | /api/equipment/occupancy | - | Redis 기반 계산 |

### 2.5 사용자/관리자 API

| Method | Endpoint | 인증 | 설명 |
|--------|----------|------|------|
| GET | /api/user/profile | 세션 | 내 정보 (마스킹) |
| DELETE | /api/user/account | 세션 | 회원 탈퇴 (소프트 삭제) |
| GET | /api/user/privacy-policy | - | 개인정보 처리방침 |
| GET | /api/admin/equipment | admin | 기구 CRUD |
| POST | /api/admin/equipment | admin | 기구 추가 |
| PUT | /api/admin/equipment/:id | admin | 기구 수정 |
| DELETE | /api/admin/equipment/:id | admin | 기구 삭제 |
| POST | /api/admin/sessions/:id/force-logout | admin | 강제 로그아웃 |
| POST | /api/admin/gym/close | admin | 영업 종료 일괄 로그아웃 |
| GET | /api/admin/stats/daily | admin | 헬스장 일별 통계 |
| GET | /api/admin/logs | admin | 접속 기록 조회 |

---

## 3. WebSocket 통신 (기구 실시간 현황)

### 3.1 연결 흐름

```
[브라우저] ── ws://host:3000/ws ──▶ [Next.js WS 서버]
                                        │
                                        ├─ 연결 시: Redis equipment:all 캐시 전송
                                        │
                                        └─ 2~3초 주기: Redis equipment:all 폴링
                                              → 변경 감지 시 전체 클라이언트 브로드캐스트
```

### 3.2 WS 메시지 형식

```json
// 서버 → 클라이언트 (전체 기구 상태)
{
  "type": "EQUIPMENT_STATUS",
  "data": [
    { "id": 1, "name": "트레드밀 #1", "type": "cardio", "status": "in_use", "since": "2026-03-29T10:00:00Z" },
    { "id": 2, "name": "트레드밀 #2", "type": "cardio", "status": "available" }
  ],
  "occupancy": { "total": 20, "in_use": 8, "rate": 0.40 },
  "timestamp": "2026-03-29T10:30:00Z"
}
```
