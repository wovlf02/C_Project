# C 서버 설계

> 최종 수정: 2026-03-30

---

## 1. 모듈 구조

```
server/
├── src/
│   ├── main.c                  # 진입점, 서버 초기화 및 시작
│   ├── tcp_server.c            # TCP 리스너, 클라이언트 연결 수락
│   ├── tcp_handler.c           # 메시지 수신/파싱/디스패치
│   ├── protocol.c              # JSON 메시지 빌드/파싱 유틸리티
│   ├── db.c                    # MySQL CRUD 함수 (Prepared Statement)
│   ├── db_pool.c               # MySQL Connection Pool 관리
│   ├── auth.c                  # 기기 인증 (API Key) + 사용자 인증 (전화번호)
│   ├── session_manager.c       # 기기 세션 생성/조회/종료
│   ├── session_watchdog.c      # 자동 로그아웃 감시 스레드
│   ├── calorie.c               # 칼로리 계산 로직
│   ├── redis_client.c          # Redis 연동 (hiredis 또는 직접 TCP)
│   ├── crypto_utils.c          # SHA-256, bcrypt 래퍼
│   ├── config.c                # 설정 파일 파싱
│   └── logger.c                # 로깅 유틸리티
├── include/
│   ├── protocol.h              # 메시지 타입 상수, 구조체
│   ├── db.h                    # DB 함수 선언
│   ├── db_pool.h               # Connection Pool 선언
│   ├── session.h               # 세션 관련 구조체/함수
│   ├── calorie.h               # 칼로리 계산 선언
│   ├── redis_client.h          # Redis 함수 선언
│   ├── crypto_utils.h          # 암호화 함수 선언
│   ├── config.h                # 설정 구조체
│   └── logger.h                # 로깅 매크로
├── config/
│   └── server.conf             # 런타임 설정
└── Makefile
```

---

## 2. 스레드 모델

```
[main 스레드]
  │
  ├─ 설정 로드 (config.c)
  ├─ DB Connection Pool 초기화 (db_pool.c)
  ├─ Redis 연결 (redis_client.c)
  ├─ TCP 서버 시작 (tcp_server.c)
  │
  ├─ [워치독 스레드] ← pthread_create
  │     └─ 3초 주기: 활성 세션 점검 (session_watchdog.c)
  │           ├─ 하트비트 타임아웃 체크
  │           ├─ 비활동 타임아웃 체크
  │           └─ 영업 종료 시각 체크
  │
  └─ [메인 루프] accept() → 기기 연결마다 새 스레드
        │
        └─ [클라이언트 스레드] ← pthread_create (기기당 1개)
              ├─ 메시지 수신 (tcp_handler.c)
              ├─ JSON 파싱 (protocol.c)
              ├─ 타입별 핸들러 호출:
              │   ├─ DEVICE_AUTH → auth.c
              │   ├─ LOGIN → auth.c + session_manager.c
              │   ├─ REGISTER → auth.c + db.c
              │   ├─ HEARTBEAT → session_manager.c + calorie.c + redis_client.c
              │   └─ LOGOUT → session_manager.c + calorie.c + db.c
              └─ 연결 종료 시 자원 정리
```

---

## 3. 핵심 자료구조

### 3.1 활성 세션 테이블

```c
#define MAX_DEVICES 100

typedef struct {
    int device_id;
    int socket_fd;
    int user_id;
    int equipment_id;
    time_t session_start;
    time_t last_heartbeat;
    time_t last_sensor_change;
    double current_calories;
    double avg_intensity;
    int is_authenticated;    // 기기 인증 완료 여부
    int is_logged_in;        // 사용자 로그인 여부
    pthread_mutex_t lock;    // 세션별 뮤텍스
} DeviceSession;

// 전역 세션 배열 (뮤텍스 보호)
extern DeviceSession g_sessions[MAX_DEVICES];
extern pthread_mutex_t g_sessions_lock;
```

### 3.2 DB Connection Pool

```c
#define POOL_MIN 10
#define POOL_MAX 50

typedef struct {
    MYSQL *connections[POOL_MAX];
    int available[POOL_MAX];     // 0: 사용 중, 1: 가용
    int pool_size;
    pthread_mutex_t lock;
    pthread_cond_t available_cond;
} DBPool;
```

---

## 4. 주요 처리 흐름

### 4.1 로그인 처리

```
LOGIN 메시지 수신
  │
  ├─ 1. 전화번호 형식 검증 (정규식)
  ├─ 2. SHA-256 해시 생성
  ├─ 3. DB 조회 (phone_hash 인덱스)
  │     ├─ 미발견 → LOGIN_FAIL (user_not_found) 응답
  │     └─ 발견 → bcrypt 검증
  ├─ 4. 기존 활성 세션 확인 (동일 user_id)
  │     └─ 있으면 → 이전 기기에 FORCE_LOGOUT + 세션 종료 + 칼로리 정산
  ├─ 5. 새 세션 생성 (sessions INSERT)
  ├─ 6. equipment 상태 갱신 (in_use)
  ├─ 7. Redis equipment:{id} 갱신
  ├─ 8. 오늘 통계 조회 (daily_stats)
  └─ 9. LOGIN_OK 응답 (user_id, today 통계)
```

### 4.2 하트비트 처리

```
HEARTBEAT 메시지 수신 (3초 주기)
  │
  ├─ 1. last_heartbeat 갱신
  ├─ 2. 센서 데이터 변화 확인
  │     └─ 변화 있으면 → last_sensor_change 갱신
  ├─ 3. 칼로리 재계산 (calorie.c)
  │     └─ calories = calorie_per_min × elapsed_min × intensity
  ├─ 4. Redis equipment:{id} 갱신
  └─ 5. DASHBOARD 응답 (칼로리/시간 정보)
```

### 4.3 자동 로그아웃 처리

```
워치독 스레드 (3초 주기)
  │
  ├─ 각 활성 세션 순회:
  │   ├─ 하트비트 타임아웃 (10초) → FORCE_LOGOUT (heartbeat_timeout)
  │   ├─ 비활동 타임아웃 (3분) → FORCE_LOGOUT (inactivity)
  │   ├─ 비활동 경고 (2분) → WARNING (remaining_sec: 60)
  │   └─ 영업 종료 시각 → FORCE_LOGOUT (gym_close)
  │
  └─ 강제 종료 시:
        ├─ 칼로리 정산 (마지막 센서 변화 시점까지)
        ├─ sessions UPDATE (end_time, calories, auto_logout_reason)
        ├─ daily_stats UPSERT
        ├─ equipment 상태 → available
        ├─ Redis equipment:{id} 갱신
        ├─ FORCE_LOGOUT 메시지 전송
        └─ access_log INSERT
```

---

## 5. 서버 시작 시 복구 처리

```
main() 초기화 단계:
  │
  ├─ end_time IS NULL인 미완료 세션 조회
  ├─ 각 미완료 세션에 대해:
  │   ├─ end_time = last_heartbeat 또는 start_time
  │   ├─ 칼로리 정산
  │   ├─ auto_logout_reason = 'server_cleanup'
  │   └─ equipment.status = 'available'
  └─ 로그 기록: "N개 미완료 세션 정리 완료"
```

---

## 6. 에러 처리 전략

| 상황 | 처리 |
|------|------|
| DB 연결 실패 | 재시도 3회 → 실패 시 서버 종료 |
| DB 쿼리 실패 | 로깅 + 클라이언트에 에러 응답 |
| Redis 연결 실패 | 경고 로깅 + Redis 없이 운영 (기구 캐시 비활성) |
| 소켓 에러 | 해당 클라이언트 세션 정리 |
| 메모리 할당 실패 | 로깅 + 해당 요청 거부 |
