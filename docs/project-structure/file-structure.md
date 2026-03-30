# 프로젝트 파일 구조 설계

> 최종 수정: 2026-03-30  
> 설계 원칙: 도메인별 폴더 분리, 파일당 400라인 미만, 코드 중복 최소화

---

## 1. 전체 구조

```
C_Project/
├── docs/                              # 설계 문서
├── todo/                              # TODO 관리
├── role/                              # 역할 분담
│
├── server/                            # C 백엔드 (TCP 서버)
│   ├── src/                           # 소스 코드 (도메인별)
│   ├── include/                       # 헤더 파일
│   ├── config/                        # 런타임 설정
│   ├── tests/                         # 단위 테스트
│   └── Makefile
│
├── device/                            # IoT 기기 시뮬레이터 (C)
│   ├── src/                           # 소스 코드
│   ├── include/                       # 헤더 파일
│   ├── tests/                         # 단위 테스트
│   └── Makefile
│
├── web/                               # Next.js 웹 애플리케이션
│   ├── src/
│   │   ├── app/                       # 페이지 + API Routes
│   │   ├── components/                # UI 컴포넌트 (도메인별)
│   │   ├── lib/                       # 서버 유틸리티
│   │   ├── hooks/                     # 클라이언트 훅
│   │   └── constants/                 # 상수 정의
│   ├── prisma/                        # Prisma 스키마
│   └── server.js                      # 커스텀 서버 (WS)
│
├── shared/                            # C 서버/기기 공유 코드
│   ├── protocol.h                     # 프로토콜 상수/구조체
│   ├── protocol.c                     # JSON 메시지 빌드/파싱
│   └── common.h                       # 공통 매크로/타입
│
├── sql/                               # 데이터베이스 스크립트
│   ├── schema.sql                     # DDL
│   ├── indexes.sql                    # 인덱스
│   ├── triggers.sql                   # 트리거
│   └── init.sql                       # 시드 데이터
│
├── docker-compose.yml                 # MySQL + Redis
└── README.md
```

---

## 2. 코드 중복 제거 전략

### 2.1 C 코드 공유 (shared/)

| 파일 | 공유 대상 | 내용 |
|------|-----------|------|
| shared/protocol.h | server + device | 메시지 타입 상수, JSON 키 정의 |
| shared/protocol.c | server + device | JSON 메시지 빌드/파싱 함수 |
| shared/common.h | server + device | 에러 코드, 공통 매크로 |

```makefile
# server/Makefile
SHARED_DIR = ../shared
CFLAGS += -I$(SHARED_DIR)
SRCS += $(SHARED_DIR)/protocol.c

# device/Makefile (동일)
```

### 2.2 Next.js 유틸리티 공유

| 파일 | 사용처 | 내용 |
|------|--------|------|
| lib/response.js | 모든 API route | 공통 응답 빌더 |
| lib/auth-middleware.js | 인증 필요 API | 세션 검증 |
| lib/admin-middleware.js | 관리자 API | 관리자 권한 검증 |
| lib/validation.js | 모든 입력 처리 | zod 스키마 모음 |
| lib/rate-limit.js | Rate Limit API | Redis 기반 IP 제한 |

### 2.3 컴포넌트 재사용

| 컴포넌트 | 사용 페이지 |
|----------|-------------|
| ChartWrapper | CalorieChart, DurationChart |
| PeriodSelector | 모든 기록 조회 페이지 |
| TypeFilter | 기구 현황, 운동 기록 |
| LoadingSpinner | 모든 비동기 페이지 |
| ErrorMessage | 모든 에러 상태 |
| Modal | 기구 상세, CRUD 폼 |

---

## 3. 파일당 400라인 제한 전략

### 3.1 C 서버 파일 분리

| 파일 | 예상 라인 | 핵심 역할 |
|------|-----------|-----------|
| main.c | ~100 | 초기화, 시작 |
| tcp_server.c | ~150 | 소켓 바인딩, accept 루프 |
| tcp_handler.c | ~250 | 메시지 수신, 타입별 디스패치 |
| protocol.c | ~200 | JSON 빌드/파싱 (shared) |
| db.c | ~350 | CRUD 쿼리함수 (Prepared Statement) |
| db_pool.c | ~200 | Connection Pool 관리 |
| auth.c | ~250 | 기기 인증 + 사용자 인증 |
| session_manager.c | ~300 | 세션 생성/종료/조회 |
| session_watchdog.c | ~250 | 자동 로그아웃 감시 |
| calorie.c | ~100 | 칼로리 계산 |
| redis_client.c | ~200 | Redis 명령 실행 |
| crypto_utils.c | ~150 | SHA-256, bcrypt |
| config.c | ~100 | 설정 파일 파싱 |
| logger.c | ~80 | 로깅 |

### 3.2 IoT 기기 파일 분리

| 파일 | 예상 라인 | 핵심 역할 |
|------|-----------|-----------|
| main.c | ~150 | 메인 루프, 상태 머신 |
| network.c | ~200 | TCP 연결, 메시지 송수신 |
| protocol.c | ~200 | JSON 빌드/파싱 (shared) |
| display.c | ~300 | ncurses 화면 렌더링 |
| input.c | ~150 | 사용자 입력 처리 |
| sensor.c | ~200 | 센서 시뮬레이션 |
| storage.c | ~200 | 암호화 로컬 저장 |
| config.c | ~80 | 기기 설정 |

### 3.3 Next.js API route 분리

- 각 route.js: 50~150라인 (핸들러만)
- 비즈니스 로직은 lib/ 유틸리티에 위임
- 검증은 lib/validation.js에 zod 스키마로 통합

---

## 4. 도메인별 폴더 분리 원칙

### 4.1 C 프로젝트

```
server/src/
├── [네트워크 도메인] tcp_server.c, tcp_handler.c
├── [프로토콜 도메인] protocol.c (shared)
├── [인증 도메인]     auth.c, crypto_utils.c
├── [세션 도메인]     session_manager.c, session_watchdog.c
├── [데이터 도메인]   db.c, db_pool.c, redis_client.c
├── [비즈니스 도메인] calorie.c
└── [인프라 도메인]   config.c, logger.c, main.c
```

### 4.2 Next.js 컴포넌트

```
components/
├── common/      → 페이지 무관 공통 UI
├── auth/        → 로그인/회원가입 관련
├── equipment/   → 기구 현황 관련
├── records/     → 운동 기록 관련
├── charts/      → 차트 관련
└── admin/       → 관리자 관련
```

### 4.3 Next.js API

```
api/
├── auth/        → 인증 도메인
├── records/     → 운동 기록 도메인
├── equipment/   → 기구 현황 도메인
├── user/        → 사용자 관리 도메인
└── admin/       → 관리자 도메인
```
