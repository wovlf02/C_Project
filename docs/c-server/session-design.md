# 세션 관리 및 자동 로그아웃 설계

> 최종 수정: 2026-03-30

---

## 1. 기기 세션 생명주기

```
[생성] LOGIN_OK 시점
  │
  ├─ sessions INSERT (end_time = NULL)
  ├─ equipment UPDATE (status='in_use', current_user_id)
  ├─ Redis equipment:{id} 갱신
  │
  ▼
[유지] HEARTBEAT 수신 (3초 주기)
  │
  ├─ last_heartbeat 갱신
  ├─ last_sensor_change 갱신 (센서 변화 시)
  ├─ 칼로리 재계산
  │
  ▼
[종료] LOGOUT / FORCE_LOGOUT / 타임아웃
  │
  ├─ sessions UPDATE (end_time, duration_sec, calories)
  ├─ daily_stats UPSERT
  ├─ equipment UPDATE (status='available', current_user_id=NULL)
  ├─ Redis equipment:{id} 갱신
  ├─ access_log INSERT
  └─ 세션 메모리 해제
```

---

## 2. 자동 로그아웃 시나리오

### 2.1 시나리오 5가지

| # | 시나리오 | 감지 | 타임아웃 | auto_logout_reason |
|---|----------|------|----------|-------------------|
| 1 | 사용자 수동 로그아웃 | LOGOUT 메시지 | 즉시 | manual |
| 2 | 다른 기구에 로그인 | 동일 user_id 활성 세션 | 즉시 | duplicate_login |
| 3 | 기기 꺼짐/네트워크 단절 | 하트비트 미수신 10초 | 10초 | heartbeat_timeout |
| 4 | 운동 중단 (비활동) | 센서 무변화 3분 | 3분 | inactivity |
| 5 | 영업 종료 | 스케줄 시각 도달 | 설정 시각 | gym_close |

### 2.2 비활동 경고 흐름

```
센서 무변화 2분 경과
  │
  ├─ WARNING 메시지 전송 (remaining_sec: 60)
  ├─ 기기 화면 우측 상단에 카운트다운 표시
  │
  ├─ [센서 변화 감지] → 경고 해제, 카운트다운 중지
  │
  └─ [3분 경과] → FORCE_LOGOUT (inactivity)
```

---

## 3. 워치독 스레드 설계

### 3.1 점검 주기: 3초

```c
void* session_watchdog(void* arg) {
    while (server_running) {
        sleep(3);
        
        pthread_mutex_lock(&g_sessions_lock);
        time_t now = time(NULL);
        
        for (int i = 0; i < MAX_DEVICES; i++) {
            if (!g_sessions[i].is_logged_in) continue;
            
            DeviceSession *s = &g_sessions[i];
            
            // 1. 하트비트 타임아웃 (10초)
            if (now - s->last_heartbeat > HEARTBEAT_TIMEOUT) {
                force_logout(s, "heartbeat_timeout");
                continue;
            }
            
            // 2. 비활동 타임아웃 (3분)
            int inactive_sec = now - s->last_sensor_change;
            if (inactive_sec > INACTIVITY_TIMEOUT) {
                force_logout(s, "inactivity");
                continue;
            }
            
            // 3. 비활동 경고 (2분)
            if (inactive_sec > WARNING_THRESHOLD) {
                int remaining = INACTIVITY_TIMEOUT - inactive_sec;
                send_warning(s, remaining);
            }
            
            // 4. 영업 종료 체크
            if (is_gym_closed(now)) {
                force_logout(s, "gym_close");
            }
        }
        
        pthread_mutex_unlock(&g_sessions_lock);
    }
    return NULL;
}
```

---

## 4. 중복 로그인 처리

```
LOGIN 수신 시:
  1. user_id로 기존 활성 세션 검색
  2. 기존 세션 발견:
     ├─ FORCE_LOGOUT 전송 (duplicate_login)
     ├─ 기존 세션 칼로리 정산 + 종료
     └─ 기존 기구 상태 → available
  3. 새 세션 생성
```

---

## 5. 웹 세션 관리

### 5.1 세션 생성 (로그인)

```
POST /api/auth/login
  1. 전화번호 SHA-256 → DB 조회 → bcrypt 검증
  2. sessionId = crypto.randomUUID()
  3. Redis SET session:{sessionId} → { user_id, role, created_at } TTL 14일
  4. MySQL web_sessions INSERT (백업)
  5. Set-Cookie: session_id=sessionId (httpOnly, secure, sameSite=strict, 14일)
```

### 5.2 세션 검증 (미들웨어)

```
모든 인증 API:
  1. Cookie에서 session_id 추출
  2. Redis GET session:{session_id}
     ├─ 존재 → user_id, role 반환
     └─ 미존재 → MySQL web_sessions fallback
                  ├─ 존재 + 미만료 → user_id 반환 + Redis 재생성
                  └─ 미존재/만료 → 401 Unauthorized
```

### 5.3 세션 삭제 (로그아웃)

```
POST /api/auth/logout
  1. Redis DEL session:{session_id}
  2. MySQL web_sessions DELETE
  3. Set-Cookie: session_id="" (만료시간=과거)
```
