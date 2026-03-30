# Redis 설계

> 최종 수정: 2026-03-30

---

## 1. Redis 역할

| 역할 | 설명 |
|------|------|
| 웹 세션 저장소 | 세션 ID → 사용자 정보 매핑 (메인 저장소) |
| 기구 실시간 캐시 | C 서버가 갱신, Next.js가 읽기 |
| 사용자 오늘 통계 캐시 | 빈번한 조회를 위한 캐싱 |
| Rate Limit 카운터 | API 호출 횟수 제한 |
| 이메일 인증 코드 | 회원가입 인증 코드 임시 저장 |

---

## 2. 키 설계

### 2.1 웹 세션

```
키:    session:{session_id}          (UUID v4)
타입:  Hash
TTL:   14일 (1,209,600초)
필드:  user_id, role, created_at
```

### 2.2 이메일 인증 코드

```
키:    email_verify:{SHA-256(email)}
타입:  Hash
TTL:   5분 (300초)
필드:  code, phone, attempts
```

### 2.3 이메일 Rate Limit

```
키:    email_verify_rate:{ip}
타입:  String (count)
TTL:   1분

키:    email_verify_rate_mail:{SHA-256(email)}
타입:  String (count)
TTL:   10분
```

### 2.4 기구 실시간 상태

```
키:    equipment:{id}
타입:  Hash
TTL:   5초
필드:  status, current_user_id, current_session_start, last_heartbeat

키:    equipment:all
타입:  String (JSON 배열)
TTL:   3초
```

### 2.5 사용자 오늘 통계 캐시

```
키:    user:{id}:today
타입:  Hash
TTL:   60초
필드:  total_calories, total_duration_sec, session_count
```

### 2.6 Rate Limit

```
키:    rate_limit:login:{ip}
타입:  String (count)
TTL:   60초

키:    rate_limit:api:{ip}
타입:  String (count)
TTL:   60초
```

---

## 3. Redis 접근 패턴

### 3.1 C 서버 (쓰기 주체)

- HEARTBEAT 수신 시 → `equipment:{id}` Hash 갱신 + TTL 5초 리셋
- 세션 시작/종료 시 → `equipment:{id}` 상태 변경
- 오늘 칼로리 변경 시 → `user:{id}:today` 갱신

### 3.2 Next.js 서버 (읽기 주체)

- API 요청 시 → `session:{session_id}` 조회 (인증)
- 기구 현황 API → `equipment:all` 조회
- WebSocket 폴링 → `equipment:all` 2~3초 주기 읽기
- Rate Limit 체크 → `rate_limit:*` INCR + TTL

### 3.3 Fallback 전략 (NFR-R05)

```
Redis 조회 실패 시:
  ├─ 세션 → MySQL web_sessions 테이블 조회
  ├─ 기구 상태 → MySQL equipment 테이블 직접 조회
  └─ Rate Limit → 임시 비활성화 (로그 경고)
```
