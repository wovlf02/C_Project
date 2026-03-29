# 요구사항 명세서 (Requirements Specification) — 최종 기획안

> **버전**: 3.1  
> **최종 수정일**: 2026-03-30  
> **상태**: 의사결정 반영 완료 — 기초 기획 확정

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 프로젝트명 | IoT 기반 헬스장 기구 관리 및 사용자 운동 기록 시스템 |
| 주요 언어 | C (IoT/백엔드 코어), JavaScript (Next.js 웹) |
| 개발 인원 | 4명 (대학교 2학년 전공자) |
| 개발 기간 | 2개월 |
| 플랫폼 | IoT 기기(C) + C 백엔드 서버 + Next.js 웹 애플리케이션 |

---

## 2. 시스템 아키텍처

### 2.1 전체 구성도

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            사용자 접점                                   │
│                                                                         │
│  ┌──────────────┐                           ┌────────────────────────┐  │
│  │  IoT 기기     │                           │  웹 브라우저 (PC/모바일) │  │
│  │ (기구 탑재)   │                           │  - 운동 기록 조회       │  │
│  │ - 로그인      │                           │  - 기구 현황           │  │
│  │ - 대시보드    │                           │  - 관리자 페이지       │  │
│  └──────┬───────┘                           └──────────┬─────────────┘  │
│         │ TCP                                          │ HTTPS          │
└─────────┼──────────────────────────────────────────────┼────────────────┘
          │                                              │
┌─────────┼──────────────────────────────────────────────┼────────────────┐
│         ▼                 백엔드 레이어                  ▼                │
│  ┌──────────────┐                           ┌────────────────────────┐  │
│  │  TCP 서버(C)  │                           │   Next.js 서버 (SSR)   │  │
│  │ - 기기 통신   │──── 공유 DB/Redis ────────│   - API Routes         │  │
│  │ - 세션 관리   │◀───────────────────────── │   - 세션 인증           │  │
│  │ - 자동로그아웃 │                           │   - 웹 페이지 렌더링    │  │
│  └──────┬───────┘                           └──────────┬─────────────┘  │
│         │                                              │                │
│         └──────────────┬───────────────────────────────┘                │
│                        ▼                                                │
│              ┌──────────────────┐     ┌─────────────┐                   │
│              │   MySQL 8.x      │     │   Redis     │                   │
│              │ (InnoDB, 파티셔닝) │     │ (세션 저장소, │                   │
│              │                  │     │  실시간 캐시) │                   │
│              └──────────────────┘     └─────────────┘                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 구성 요소 상세

| 구성 요소 | 설명 | 기술 스택 |
|-----------|------|-----------|
| IoT 기기 (클라이언트) | 각 기구에 탑재, WiFi LAN으로 C 서버에 연결, 사용자 로그인·대시보드·센서 데이터 수집 | C (POSIX Socket, ncurses UI) |
| C 백엔드 서버 | IoT 기기 TCP 통신(WiFi LAN), 센서 데이터 처리, 세션 관리, 자동 로그아웃 | C (멀티스레드 TCP 서버) |
| Next.js 웹 서버 | 웹 사용자 인증(세션 ID 쿠키), 운동 기록 조회 API, SSR 페이지 렌더링, WebSocket 기구 실시간 상태, 관리자 기능 | Next.js 16 (App Router, JavaScript, ws) |
| MySQL 8.x | 사용자·기구·세션·통계 데이터 영구 저장, InnoDB 엔진, 파티셔닝 적용 | MySQL 8.x |
| Redis | 웹 세션 저장소, 기구 실시간 상태 캐시, Rate Limit 카운터 | Redis 7.x |

### 2.3 서비스 간 통신 흐름

```
[IoT 기기] ──TCP (WiFi LAN)──▶ [C 서버] ──MySQL──▶ [DB]
                                   │
                                   └──Redis──▶ [기구 상태 캐시]
                                                    │
[웹 브라우저] ──HTTPS──▶ [Next.js] ──MySQL──▶ [DB]
                  │          │                       │
                  │          └──Redis 읽기───────────┘
                  │
                  └──WebSocket──▶ [Next.js WS 서버] ──Redis 폴링──▶ 기구 실시간 상태
```

---

## 3. 기능 요구사항 (Functional Requirements)

### 3.1 사용자 관리 (기기)

| ID | 요구사항 | 우선순위 | 설명 |
|----|----------|----------|------|
| FR-U01 | 전화번호 기반 기기 로그인 | 필수 | 사용자는 기구 탑재 기기에서 전화번호를 입력하여 로그인한다 |
| FR-U02 | 신규 사용자 등록 | 필수 | IoT 기기에서 전화번호 미등록 확인 시 전화번호+이메일+동의를 입력받아 사용자를 등록한다. 기기 현장 등록은 이메일 인증 없이 처리된다 (물리적 현장 인증으로 갈음) |
| FR-U03 | 기기 세션 관리 | 필수 | 로그인 후 기구 사용 종료(로그아웃) 시까지 기기 세션을 유지한다 |
| FR-U04 | 수동 로그아웃 | 필수 | 사용자가 기구 사용을 종료하면 기기에서 로그아웃 처리한다 |
| FR-U05 | 개인정보 수집 동의 | 필수 | 최초 등록 시 전화번호·이메일 수집·이용에 대한 동의를 기기 화면에서 받는다 |

### 3.2 사용자 관리 (웹)

| ID | 요구사항 | 우선순위 | 설명 |
|----|----------|----------|------|
| FR-UW01 | 전화번호 기반 웹 로그인 | 필수 | 사용자는 웹사이트에서 전화번호를 입력하여 로그인한다 |
| FR-UW02 | 세션 ID 발급 | 필수 | 로그인 성공 시 서버에서 세션 ID(UUID)를 생성하여 httpOnly 쿠키로 클라이언트에 저장한다 |
| FR-UW03 | 세션 기반 인증 | 필수 | 모든 API 요청 시 쿠키의 세션 ID로 Redis에서 사용자를 식별하고 요청을 처리한다 |
| FR-UW04 | 웹 로그아웃 | 필수 | 로그아웃 시 Redis에서 세션을 삭제하고 클라이언트 쿠키를 제거한다 |
| FR-UW05 | 웹 자동 로그인 | 필수 | 브라우저를 닫았다 다시 열어도 세션 쿠키(httpOnly)가 유효하면 자동으로 로그인 상태를 복원한다 (세션 만료: 14일) |
| FR-UW06 | 기기 자동 로그인 없음 | 필수 | 헬스장 기구 태블릿(IoT 기기)는 매번 전화번호를 입력해야 하며, 이전 사용자 정보를 기억하지 않는다 |
| FR-UW07 | 웹 회원가입 — 이메일 인증 코드 발송 | 필수 | 회원가입 화면에서 전화번호·이메일을 입력하고 인증 버튼을 누르면 입력된 이메일로 6자리 숫자 인증코드를 발송한다. 코드는 Redis에 5분 TTL로 저장된다 |
| FR-UW08 | 웹 회원가입 — 이메일 인증 확인 | 필수 | 수신한 6자리 코드를 웹에 입력하고 인증하기 버튼을 누르면 서버가 Redis 저장값과 비교 검증한다. 검증 성공 시 users 테이블에 신규 계정을 생성(AUTO_INCREMENT ID)하고 세션을 발급하여 자동 로그인 처리한다 |

### 3.3 자동 로그아웃 및 보안 처리

| ID | 요구사항 | 우선순위 | 설명 |
|----|----------|----------|------|
| FR-AL01 | 중복 기기 로그인 시 이전 기기 자동 로그아웃 | 필수 | 사용자가 A 기구에서 로그아웃하지 않고 B 기구에 로그인하면 A 기구 세션을 자동 종료한다 |
| FR-AL02 | 기기 비활동 타임아웃 | 필수 | 센서 데이터가 3분간 무입력(운동 미감지) 상태이면 자동 로그아웃한다 |
| FR-AL03 | 기기 하트비트 기반 로그아웃 | 필수 | IoT 기기에서 10초 이상 하트비트가 수신되지 않으면 비정상 종료로 판단하고 세션을 자동 정리한다 |
| FR-AL04 | 헬스장 영업 종료 시 전체 로그아웃 | 권장 | 영업 종료 시각에 모든 활성 기기 세션을 일괄 종료하는 스케줄 기능을 제공한다 |
| FR-AL05 | 자동 로그아웃 알림 | 권장 | 자동 로그아웃이 발생하면 해당 기기 화면에 안내 메시지를 표시하고, 웹에서도 기록에 해당 사유를 남긴다 |
| FR-AL06 | 자동 로그아웃 시 칼로리 정산 | 필수 | 자동 로그아웃 시에도 마지막 센서 데이터 시점까지의 칼로리를 정확히 계산하여 저장한다 |

### 3.4 기구 상태 관리

| ID | 요구사항 | 우선순위 | 설명 |
|----|----------|----------|------|
| FR-E01 | 기구 사용 상태 표시 | 필수 | 각 기구의 사용 중 / 미사용 상태를 실시간으로 파악한다 |
| FR-E02 | 기구 목록 관리 | 필수 | 헬스장에 등록된 전체 기구 목록을 CRUD로 관리한다 |
| FR-E03 | 기구 상태 실시간 갱신 | 필수 | IoT 기기로부터 주기적(3초)으로 상태를 수신, Redis 캐시를 갱신한다 |
| FR-E04 | 기구 유형 분류 | 필수 | 유산소 / 무산소 / 스트레칭 등 기구 유형별 분류를 지원한다 |
| FR-E05 | 기구 고장·점검 상태 | 권장 | 관리자가 기구를 "점검 중" 상태로 변경하여 사용자 접근을 차단할 수 있다 |

### 3.5 대시보드 (기기 화면)

| ID | 요구사항 | 우선순위 | 설명 |
|----|----------|----------|------|
| FR-D01 | 오늘 총 소모 칼로리 | 필수 | 당일 해당 사용자가 헬스장에서 소모한 총 칼로리를 표시한다 |
| FR-D02 | 현재 기구 소모 칼로리 | 필수 | 현재 사용 중인 기구에서 소모 중인 칼로리를 실시간 표시한다 |
| FR-D03 | 오늘 총 운동 시간 | 필수 | 당일 해당 사용자의 누적 운동 시간을 표시한다 |
| FR-D04 | 현재 기구 사용 시간 | 필수 | 현재 기구 사용 시작 이후 경과 시간을 표시한다 |
| FR-D05 | 대시보드 자동 전환 | 필수 | 로그인 성공 시 자동으로 대시보드 화면으로 전환한다 |
| FR-D06 | 운동 종목별 기록 | 권장 | 당일 사용한 기구별 운동 기록을 구분하여 표시한다 |
| FR-D07 | 비활동 경고 표시 | 필수 | 비활동 상태가 2분 지속 시 화면 우측 상단에 "자동 로그아웃 1분 전 (남은 시간: 60초)" 카운트다운 경고를 표시한다 |
| FR-D08 | 로컬 암호화 저장 | 필수 | 로그인 후 운동 중 발생하는 모든 세션 데이터(시작 시각, 칼로리, 센서 값)를 기기 Encrypted Storage에 실시간 저장한다 |
| FR-D09 | 운동 종료 데이터 전송 | 필수 | 사용자가 "운동 종료" 버튼을 누르면 Encrypted Storage의 데이터를 서버로 전송하고, 전송 성공 후 로컬 데이터를 완전 삭제한다 |
| FR-D10 | 자동 로그아웃 로컬 정리 | 필수 | 자동 로그아웃 시에도 Encrypted Storage 데이터를 서버로 전송하고, 전송 후 로컬 데이터를 삭제한다 |

### 3.6 웹 대시보드 — 사용자 운동 기록 조회

| ID | 요구사항 | 우선순위 | 설명 |
|----|----------|----------|------|
| FR-WR01 | 일별 운동 기록 조회 | 필수 | 특정 날짜의 운동 세션 목록(기구명, 시간, 칼로리)을 조회한다 |
| FR-WR02 | 주간 운동 기록 조회 | 필수 | 선택한 주의 일별 총 운동 시간·칼로리를 차트와 표로 조회한다 |
| FR-WR03 | 월별 운동 기록 조회 | 필수 | 선택한 월의 일별 요약(총 시간, 총 칼로리, 운동 횟수)을 조회한다 |
| FR-WR04 | 연도별 운동 기록 조회 | 필수 | 선택한 연도의 월별 요약 통계를 조회한다 |
| FR-WR05 | 기간 커스텀 조회 | 권장 | 사용자가 시작일~종료일을 직접 지정하여 기록을 조회할 수 있다 |
| FR-WR06 | 기구 유형별 필터링 | 권장 | 유산소/무산소 등 기구 유형별로 기록을 필터링하여 조회한다 |
| FR-WR07 | 운동 통계 시각화 | 필수 | 칼로리·운동시간 추이를 차트(막대/라인)로 시각화한다 |
| FR-WR08 | 개인 운동 요약 | 권장 | 총 운동 일수, 누적 칼로리, 가장 많이 사용한 기구 등 종합 요약을 제공한다 |

### 3.7 웹 대시보드 — 기구 현황 (관리자/공용)

| ID | 요구사항 | 우선순위 | 설명 |
|----|----------|----------|------|
| FR-W01 | 전체 기구 현황 조회 | 필수 | 웹에서 전체 기구의 사용/미사용/점검 상태를 한눈에 파악한다 |
| FR-W02 | 실시간 현황 갱신 (WebSocket) | 필수 | WebSocket으로 기구 상태를 실시간 수신한다 (서버가 Redis에서 2~3초 주기로 읽어 전체 클라이언트에 브로드캐스트) |
| FR-W03 | 기구별 상세 정보 | 권장 | 기구 클릭 시 현재 사용자(마스킹), 사용 시간 등 상세 정보를 확인한다 |
| FR-W04 | 혼잡도 표시 | 필수 | 전체 기구 대비 사용 중인 기구 비율을 시각적으로 표시한다 |
| FR-W05 | 기구 유형별 필터 | 권장 | 유산소/무산소 등 유형별로 기구 현황을 필터링한다 |

---

## 4. 비기능 요구사항 (Non-Functional Requirements)

### 4.1 성능

| ID | 요구사항 | 기준 |
|----|----------|------|
| NFR-P01 | 기기 → C 서버 데이터 전송 지연 | 500ms 이내 |
| NFR-P02 | 웹 기구 상태 갱신 주기 | 3~5초 (Redis 캐시 활용) |
| NFR-P03 | 동시 접속 기기 수 | 최소 50대 이상 |
| NFR-P04 | Next.js API 응답 시간 | 200ms 이내 (캐시 hit), 500ms 이내 (캐시 miss) |
| NFR-P05 | MySQL 쿼리 응답 시간 | 일별 조회 50ms 이내, 연도별 집계 500ms 이내 |
| NFR-P06 | 웹 페이지 초기 로딩 | 2초 이내 (SSR + Code Splitting) |
| NFR-P07 | DB Connection Pool | 최소 20, 최대 100 커넥션 (C 서버 + Next.js 합산) |

### 4.2 안정성

| ID | 요구사항 | 기준 |
|----|----------|------|
| NFR-R01 | 기기 연결 끊김 감지 | 10초 이상 하트비트 미수신 시 기구 상태를 "미사용"으로 전환 |
| NFR-R02 | 서버 장애 복구 | MySQL 서버 상시 운영, C 서버·Next.js 서버 재시작 시 자동 재연결 |
| NFR-R03 | 비정상 종료 처리 | C 서버 시작 시 `end_time IS NULL`인 미완료 세션을 자동 정리 |
| NFR-R04 | DB 트랜잭션 보장 | 세션 시작/종료·칼로리 기록은 트랜잭션으로 원자성을 보장한다 |
| NFR-R05 | Redis 장애 시 Fallback | Redis 다운 시 MySQL 직접 조회로 서비스 지속 가능 |

### 4.3 확장성

| ID | 요구사항 | 기준 |
|----|----------|------|
| NFR-SC01 | DB 파티셔닝 | sessions 테이블을 월별 RANGE 파티셔닝하여 쿼리 성능 유지 |
| NFR-SC02 | 인덱스 최적화 | 조회 패턴에 맞는 복합 인덱스 설계로 Full Table Scan 방지 |
| NFR-SC03 | 통계 사전 집계 | daily_stats 테이블로 일별 통계를 사전 집계하여 중복 연산 제거 |

### 4.4 보안

| ID | 요구사항 | 기준 |
|----|----------|------|
| NFR-S01 | 전화번호 해시 저장 | 전화번호를 bcrypt(cost=12)로 해시하여 저장. 조회용 SHA-256 인덱스 해시를 별도 보관 |
| NFR-S02 | 개인정보 최소 수집 | 전화번호·이메일 외 불필요한 개인정보를 수집하지 않음. 이메일은 AES-256 암호화 저장 |
| NFR-S03 | 세션 ID 보안 | crypto.randomUUID()로 추측 불가능한 세션 ID 생성, 서버 측 Redis에서만 유효성 검증 |
| NFR-S04 | 세션 쿠키 (웹) | httpOnly + secure + sameSite=strict 쿠키로 세션 ID 저장, 만료 14일 |
| NFR-S05 | 세션 서버 저장 | Redis에 세션 데이터 저장 (TTL 14일), MySQL web_sessions 테이블에 백업 |
| NFR-S05-D | 기기 인증 정책 | IoT 기기(태블릿)는 인증 정보를 영구 저장하지 않음. 기구 사용 종료 또는 자동 로그아웃 시 모든 인증 정보 즉시 삭제 |
| NFR-S06 | 세션 무효화 | 로그아웃 시 Redis에서 세션 즉시 삭제, 쿠키 제거 |
| NFR-S07 | SQL Injection 방지 | Prepared Statement(MySQL C API: `mysql_stmt_*`, Next.js: Prisma parameterized query) |
| NFR-S08 | XSS 방지 | Next.js의 기본 이스케이핑 + Content-Security-Policy 헤더 설정 |
| NFR-S09 | CSRF 방지 | SameSite 쿠키 + CSRF 토큰 이중 방어 |
| NFR-S10 | 입력값 검증 | 전화번호: `/^01[016789]-?\d{3,4}-?\d{4}$/` 서버 측 검증 |
| NFR-S11 | Rate Limiting | 로그인 API: IP당 5회/분, 이메일 인증 코드 발송: IP당 3회/분 + 이메일당 3회/10분, 전체 API: IP당 100회/분 |
| NFR-S12 | HTTPS 강제 | 웹 서버는 TLS 1.2 이상 필수, HTTP 접속 시 301 리다이렉트 (로컬 개발 시 HTTP 허용) |
| NFR-S13 | 전화번호 마스킹 | 웹 화면에서 전화번호를 `010-****-1234` 형태로 마스킹하여 표시 |
| NFR-S14 | 기기-서버 통신 인증 | IoT 기기별 고유 API Key를 사전 발급하여 TCP 연결 시 인증 (헬스장 내부 WiFi LAN 전용) |
| NFR-S15 | 보안 헤더 설정 | X-Content-Type-Options, X-Frame-Options, Strict-Transport-Security 등 적용 |

---

## 5. 인증/인가 상세 설계

### 5.1 웹 인증 흐름 (세션 ID 쿠키)

```
┌─────────┐          ┌─────────────┐          ┌───────┐  ┌───────┐
│  브라우저  │          │  Next.js API │         │ MySQL │  │ Redis │
└────┬────┘          └──────┬──────┘          └───┬───┘  └───┬───┘
     │  POST /api/auth/login│                     │          │
     │  { phone: "010..." } │                     │          │
     ├─────────────────────▶│                     │          │
     │                      │  SHA-256 → 조회      │          │
     │                      ├────────────────────▶│          │
     │                      │  bcrypt 검증         │          │
     │                      │◀────────────────────┤          │
     │                      │                     │          │
     │                      │  세션 ID 생성 (UUID) │          │
     │                      │  session:{id} 저장   │          │
     │                      ├─────────────────────┼────────▶│
     │                      │                     │          │
     │  Set-Cookie:         │                     │          │
     │  session_id (httpOnly)│                     │          │
     │◀─────────────────────┤                     │          │
     │                      │                     │          │
     │  GET /api/records    │                     │          │
     │  Cookie: session_id  │                     │          │
     ├─────────────────────▶│                     │          │
     │                      │  Redis 세션 조회     │          │
     │                      ├─────────────────────┼────────▶│
     │                      │  user_id 확인        │          │
     │                      │◀─────────────────────┼────────│
     │                      │  DB 조회             │          │
     │                      ├────────────────────▶│          │
     │  200 { records }     │                     │          │
     │◀─────────────────────┤                     │          │
     │                      │                     │          │
     │  POST /api/auth/logout                     │          │
     │  Cookie: session_id  │                     │          │
     ├─────────────────────▶│                     │          │
     │                      │  Redis 세션 삭제     │          │
     │                      ├─────────────────────┼────────▶│
     │  Set-Cookie: 삭제    │                     │          │
     │◀─────────────────────┤                     │          │
```

### 5.2 웹 세션 정책

| 항목 | 세션 ID 쿠키 |
|------|-------------|
| 세션 ID 생성 | `crypto.randomUUID()` (UUID v4) |
| 만료 시간 | 14일 |
| 저장 위치 (웹 클라이언트) | httpOnly + secure + sameSite=strict 쿠키 |
| 저장 위치 (IoT 기기) | 해당 없음 (기기 세션은 C 서버가 TCP로 관리) |
| 서버 저장 | Redis: `session:{session_id}` → `{user_id, role, created_at}` (TTL 14일) + MySQL `web_sessions` 백업 |
| 로그아웃 처리 | Redis 세션 삭제 + 쿠키 제거 |
| 자동 로그인 (웹) | **지원** — 유효한 세션 쿠키 존재 시 Redis 조회로 로그인 상태 자동 복원 |
| 자동 로그인 (IoT 기기) | **미지원** — 기구별 전용 태블릿은 사용자 정보를 저장하지 않음 |

### 5.3 세션 미들웨어 처리 흐름

```
모든 인증 필요 API 요청:
1. 쿠키에서 session_id 추출
2. Redis에서 session:{session_id} 조회
3. 세션 존재 → user_id, role 추출 → 요청 처리
4. 세션 미존재 → 401 Unauthorized 응답
5. Redis 장애 시 → MySQL web_sessions 테이블 Fallback 조회
```

### 5.4 기기 인증 흐름

```
[IoT 기기] ──TCP CONNECT (WiFi LAN)──▶ [C 서버]
           ──DEVICE_AUTH { device_id, api_key }──▶
           ◀──AUTH_OK──
           ──LOGIN { phone }──▶ (SHA-256 변환 후 DB 조회)
           ◀──LOGIN_OK { user_id, today_stats }──
           (미등록 사용자인 경우)
           ◀──LOGIN_FAIL { reason: "user_not_found" }──
           ──REGISTER { phone, email, consent: true }──▶
           ◀──REGISTER_OK { user_id }──
```

- 기기는 서버에서 사전 발급된 `api_key`로 TCP 연결 시 최초 인증
- 전화번호는 기기에서 평문으로 전송하되, **헬스장 내부 WiFi LAN 전용**으로 통신
- 미등록 사용자는 이메일 입력 + 개인정보 동의 후 REGISTER 메시지로 등록
- **기기 현장 등록은 이메일 인증 없이 처리** (물리적 현장 인증으로 갈음)

### 5.5 웹 회원가입 흐름 (이메일 인증 포함)

```
┌─────────┐          ┌─────────────┐          ┌───────┐  ┌───────┐
│  브라우저  │          │  Next.js API │         │ MySQL │  │ Redis │
└────┬────┘          └──────┬──────┘          └───┬───┘  └───┬───┘
     │                      │                     │          │
     │  --- [1단계: 이메일 인증 코드 발송] ---        │          │
     │                      │                     │          │
     │  POST /api/auth/      │                     │          │
     │  email-verify/send   │                     │          │
     │  { phone, email }    │                     │          │
     ├─────────────────────▶│                     │          │
     │                      │  전화번호 중복 확인   │          │
     │                      ├────────────────────▶│          │
     │                      │  이메일 형식 검증     │          │
     │                      │  6자리 코드 생성      │          │
     │                      │  (crypto.randomInt)  │          │
     │                      │  email_verify:{hash} │          │
     │                      │  저장 (TTL 5분)      │          │
     │                      ├─────────────────────┼────────▶│
     │                      │  nodemailer로        │          │
     │                      │  인증코드 이메일 발송 │          │
     │  200 { sent: true }  │                     │          │
     │◀─────────────────────┤                     │          │
     │                      │                     │          │
     │  --- [2단계: 인증 코드 검증 + 회원가입] ---    │          │
     │                      │                     │          │
     │  POST /api/auth/      │                     │          │
     │  email-verify/confirm│                     │          │
     │  { phone, email,     │                     │          │
     │    code, consent }   │                     │          │
     ├─────────────────────▶│                     │          │
     │                      │  Redis에서 코드 조회 │          │
     │                      ├─────────────────────┼────────▶│
     │                      │  코드 일치 여부 검증  │          │
     │                      │  Redis 키 즉시 삭제   │          │
     │                      ├─────────────────────┼────────▶│
     │                      │  users 테이블에      │          │
     │                      │  INSERT (AUTO_INCREMENT)       │
     │                      ├────────────────────▶│          │
     │                      │  세션 ID 생성 + 저장 │          │
     │                      ├─────────────────────┼────────▶│
     │  Set-Cookie:         │                     │          │
     │  session_id (httpOnly)│                     │          │
     │  200 { success: true }│                     │          │
     │◀─────────────────────┤                     │          │
```

#### 인증 코드 상세 규칙

| 항목 | 값 |
|------|-----|
| 코드 형식 | 6자리 숫자 (000000~999999) |
| 생성 방법 | `crypto.randomInt(0, 1000000).toString().padStart(6, '0')` |
| Redis 키 | `email_verify:{SHA-256(email)}` |
| TTL | 5분 (300초) |
| 검증 실패 허용 횟수 | 5회 초과 시 Redis 키 삭제 (재발송 필요) |
| 재발송 제한 | IP당 3회/분, 이메일당 3회/10분 (Rate Limit) |
| 코드 일치 후 | Redis 키 즉시 삭제 (1회용) |

---

## 6. 자동 로그아웃 상세 설계

### 6.1 시나리오별 처리

| 시나리오 | 감지 방법 | 처리 방식 | 타임아웃 |
|----------|-----------|-----------|----------|
| **사용자가 로그아웃을 누름** | 기기에서 LOGOUT 메시지 전송 | 세션 정상 종료, 칼로리 정산 | 즉시 |
| **다른 기구에 로그인** | C 서버가 동일 user_id의 활성 세션 감지 | 이전 기구 세션 강제 종료 + 새 기구 세션 시작 | 즉시 |
| **기기 전원 꺼짐/네트워크 단절** | 하트비트 미수신 | 기구 상태 "미사용" 전환 + 세션 강제 종료 | 10초 |
| **운동 중단 (기구 위에서 장시간 정지)** | 센서 데이터 무변화 (RPM=0 등) | 2분 경과 시 화면 우측 상단 경고(카운트다운), 3분 경과 시 자동 로그아웃 | 3분 |
| **헬스장 영업 종료** | 관리자 스케줄 설정 | 전체 활성 세션 일괄 종료 | 스케줄 |

### 6.2 자동 로그아웃 처리 흐름

```
[C 서버: 세션 워치독 스레드]
  │
  ├── 3초마다 모든 활성 기기 세션 점검
  │     │
  │     ├── 하트비트 체크 → last_heartbeat + 10초 < now → 강제 종료
  │     │
  │     ├── 비활동 체크 → last_sensor_change + 3분 < now → 강제 종료
  │     │     └── last_sensor_change + 2분 < now → WARNING 메시지 전송 (remaining_sec: 60)
  │     │
  │     └── 중복 로그인 체크 → 새 로그인 시 기존 세션 즉시 종료
  │
  └── 강제 종료 시:
        ├── 1. end_time = last_sensor_change (마지막 활동 시점)
        ├── 2. 칼로리 정산 (start_time ~ last_sensor_change 기준)
        ├── 3. 기기에 FORCE_LOGOUT 메시지 전송 → 기기가 Encrypted Storage 데이터 서버 전송 후 로컬 삭제
        ├── 4. equipment.status = 'available' 로 변경
        ├── 5. session.auto_logout_reason 기록
        └── 6. 전송 실패 시: 서버는 자체 보유 데이터로 세션 마감 처리 (데이터 유실 방지)
```

---

## 7. 법률 및 규정 준수 요구사항

### 7.1 개인정보 보호법 준수

| ID | 요구사항 | 관련 법령 | 설명 |
|----|----------|-----------|------|
| LR-01 | 개인정보 수집·이용 동의 | 개인정보 보호법 제15조 | 전화번호·이메일 수집 전 이용 목적, 항목, 보유기간을 안내하고 동의를 받는다 |
| LR-02 | 개인정보 처리방침 게시 | 개인정보 보호법 제30조 | 개인정보 처리방침을 작성하고 웹에 공개한다 |
| LR-03 | 개인정보 최소 수집 원칙 | 개인정보 보호법 제16조 | 서비스에 필요한 최소한의 정보(전화번호, 이메일)만 수집한다 |
| LR-04 | 개인정보 안전성 확보 | 개인정보 보호법 제29조 | 전화번호를 bcrypt 해시 처리하고 접근 통제를 적용한다 |
| LR-05 | 개인정보 파기 | 개인정보 보호법 제21조 | 보유기간 경과 또는 목적 달성 시 지체 없이 파기한다 |
| LR-06 | 개인정보 열람·삭제 요청 | 개인정보 보호법 제35·36조 | 사용자가 웹에서 자신의 정보 열람 및 삭제(탈퇴)를 요청할 수 있다 |
| LR-07 | 개인정보 유출 통지 | 개인정보 보호법 제34조 | 개인정보 유출 인지 시 72시간 내 통지 절차를 수립한다 |

### 7.2 정보통신망법 준수

| ID | 요구사항 | 관련 법령 | 설명 |
|----|----------|-----------|------|
| LR-08 | 접속 기록 보관 | 정보통신망법 제49조의2 | 개인정보 처리 시스템 접속 기록을 최소 1년 보관한다 |
| LR-09 | 기술적 보호조치 | 정보통신망법 제28조 | 개인정보에 대한 기술적·관리적 보호조치를 이행한다 |

### 7.3 전기통신사업법 관련

| ID | 요구사항 | 설명 |
|----|----------|------|
| LR-10 | 전화번호 수집 목적 제한 | 전화번호를 마케팅, 제3자 제공 등 본래 목적 외에 사용하지 않는다 |

---

## 8. 특허 관련 요구사항

### 8.1 특허 출원 가능 요소

| ID | 요소 | 설명 |
|----|------|------|
| PR-01 | IoT 기반 헬스기구 실시간 모니터링 방법 | 기구에 탑재된 IoT 기기가 사용 상태를 자동 감지하여 중앙 서버에 보고하는 방법 |
| PR-02 | 전화번호 기반 간편 인증을 활용한 헬스기구 사용자 식별 시스템 | 별도 회원가입 없이 전화번호만으로 사용자를 식별하고 운동 기록을 연동하는 시스템 |
| PR-03 | 실시간 칼로리 소모량 산출 및 기구 간 누적 계산 방법 | 복수 기구 사용 시 기구별 칼로리를 개별 산출하고 일일 총합을 제공하는 방법 |
| PR-04 | 멀티 기구 환경에서의 자동 로그아웃 및 세션 전이 방법 | 사용자가 기구를 변경할 때 이전 기구 세션을 자동 종료하고 칼로리를 정산하는 방법 |

### 8.2 선행기술 회피를 위한 차별점

- 기구 자체에 탑재된 IoT 기기에서 직접 사용자 인증 및 대시보드를 제공하는 점
- 전화번호 해시 기반 경량 인증으로 별도 앱 설치 없이 기구에서 바로 사용 가능한 점
- C언어 기반 경량 IoT 서버와 Next.js 웹 서버의 하이브리드 아키텍처
- 센서 비활동 감지 기반 지능형 자동 로그아웃 및 멀티 기구 세션 전이 기술

---

## 9. 기술 요구사항

### 9.1 개발 환경

| 항목 | 사양 |
|------|------|
| IoT/백엔드 코어 언어 | C (C99 이상) |
| 웹 프레임워크 | Next.js 16 (App Router, JavaScript) |
| 빌드 도구 (C) | GCC / Makefile |
| 빌드 도구 (Web) | Node.js 24.14.1 / npm |
| DB | MySQL 8.x (InnoDB) |
| 캐시/토큰 저장소 | Redis 7.x |
| ORM | Prisma (Next.js ↔ MySQL) |
| 네트워크 (IoT) | POSIX Socket API (TCP) |
| 버전 관리 | Git / GitHub |

### 9.2 외부 라이브러리 / 패키지

#### C 서버

| 라이브러리 | 용도 | 라이선스 |
|------------|------|----------|
| MySQL Connector/C (libmysqlclient) | 데이터베이스 연동 | GPL 2.0 |
| cJSON | JSON 파싱/생성 (프로토콜 메시지) | MIT |
| OpenSSL (libcrypto) | SHA-256 해시, bcrypt 지원 | Apache 2.0 |
| pthread | 멀티스레드 (세션 워치독, 동시 기기 처리) | POSIX 표준 |

#### Next.js 웹

| 패키지 | 용도 |
|--------|------|
| next | SSR/SSG 프레임워크 |
| prisma / @prisma/client | MySQL ORM |
| ioredis | Redis 클라이언트 (세션 저장소, 캐시) |
| bcryptjs | 전화번호 bcrypt 해시 검증 |
| recharts | 운동 기록 차트 시각화 |
| tailwindcss | UI 스타일링 (다크 테마) |
| zod | 입력값 스키마 검증 |
| ws | WebSocket 서버 (기구 실시간 상태 브로드캐스트) |
| nodemailer | 이메일 발송 (네이버 SMTP) — 회원가입 이메일 인증 코드 발송, 계정 만료 사전 안내 |
| cookie | 쿠키 파싱/생성 (세션 관리) |
| uuid | 세션 ID 생성 (crypto.randomUUID 대체용) |

### 9.3 프로젝트 디렉토리 구조

```
C_Project/
├── docs/
│   ├── overview.md
│   └── requirements.md
│
├── server/                          # C 백엔드 (IoT + 코어 로직)
│   ├── src/
│   │   ├── main.c                   # 서버 진입점
│   │   ├── tcp_server.c             # TCP 서버 (IoT 기기 통신)
│   │   ├── tcp_handler.c            # TCP 메시지 핸들러
│   │   ├── db.c                     # MySQL 연동 CRUD
│   │   ├── db_pool.c                # MySQL Connection Pool
│   │   ├── auth.c                   # 기기 인증 (API Key + 전화번호)
│   │   ├── session_manager.c        # 기기 세션 관리
│   │   ├── session_watchdog.c       # 자동 로그아웃 워치독 스레드
│   │   ├── calorie.c                # 칼로리 계산 로직
│   │   └── config.c                 # 설정 파일 파싱
│   ├── include/
│   │   ├── protocol.h               # 통신 프로토콜 정의
│   │   ├── db.h
│   │   ├── session.h
│   │   └── config.h
│   ├── config/
│   │   └── server.conf              # DB 접속 정보, 포트, 타임아웃 등
│   └── Makefile
│
├── device/                          # IoT 기기 시뮬레이터 (C)
│   ├── src/
│   │   ├── main.c                   # 기기 진입점
│   │   ├── sensor.c                 # 센서 데이터 생성
│   │   ├── display.c                # 대시보드 화면 (ncurses)
│   │   ├── network.c                # 서버 통신 + 하트비트
│   │   └── input.c                  # 전화번호 입력 UI
│   ├── include/
│   │   └── device.h
│   └── Makefile
│
├── web/                             # Next.js 웹 애플리케이션
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.js            # 루트 레이아웃
│   │   │   ├── page.js              # 메인 페이지 (기구 현황)
│   │   │   ├── login/
│   │   │   │   └── page.js          # 로그인 페이지
│   │   │   ├── dashboard/
│   │   │   │   └── page.js          # 사용자 대시보드 (운동 기록)
│   │   │   ├── records/
│   │   │   │   ├── page.js          # 운동 기록 조회 (일별/주간/월별/연도별)
│   │   │   │   └── [period]/
│   │   │   │       └── page.js      # 기간별 상세 조회
│   │   │   ├── admin/
│   │   │   │   └── page.js          # 관리자 페이지
│   │   │   └── api/
│   │   │       ├── auth/
│   │   │       │   ├── login/route.js
│   │   │       │   ├── logout/route.js
│   │   │       │   ├── me/route.js
│   │   │       │   └── email-verify/
│   │   │       │       ├── send/route.js   # 코드 발송
│   │   │       │       └── confirm/route.js # 코드 검증 + 회원가입 완료
│   │   │       ├── records/
│   │   │       │   ├── daily/route.js
│   │   │       │   ├── weekly/route.js
│   │   │       │   ├── monthly/route.js
│   │   │       │   └── yearly/route.js
│   │   │       ├── equipment/
│   │   │       │   └── route.js
│   │   │       └── stats/
│   │   │           └── route.js
│   │   ├── components/
│   │   │   ├── charts/              # 차트 컴포넌트
│   │   │   ├── equipment/           # 기구 현황 컴포넌트
│   │   │   └── common/              # 공통 UI 컴포넌트
│   │   ├── lib/
│   │   │   ├── prisma.js            # Prisma 클라이언트
│   │   │   ├── redis.js             # Redis 클라이언트
│   │   │   ├── session.js           # 세션 관리 유틸리티 (ID 생성, Redis CRUD)
│   │   │   ├── auth-middleware.js   # 세션 인증 미들웨어
│   │   │   ├── email.js             # 이메일 발송 (네이버 SMTP)
│   │   │   ├── crypto.js            # 이메일 AES-256 암독호화
│   │   │   └── ws-server.js         # WebSocket 서버 (기구 상태 브로드캐스트)
│   │   └── constants/
│   │       └── index.js             # 공통 상수 정의
│   ├── prisma/
│   │   └── schema.prisma            # Prisma 스키마 (MySQL)
│   ├── package.json
│   ├── tailwind.config.js
│   ├── next.config.js
│   ├── server.js                    # 커스텀 서버 (Next.js + WebSocket 통합)
│   └── jsconfig.json
│
├── sql/
│   ├── schema.sql                   # MySQL DDL (테이블, 인덱스, 파티션)
│   ├── indexes.sql                  # 인덱스 상세 정의
│   ├── triggers.sql                 # 트리거 (통계 자동 갱신)
│   └── init.sql                     # 초기 데이터 (기구 목록 등)
│
├── docker-compose.yml               # MySQL + Redis 로컬 개발 환경
└── README.md
```

---

## 10. 데이터베이스 설계 (MySQL 최적화)

### 10.1 설계 원칙

- **InnoDB 엔진 전용**: 트랜잭션, 외래키, Row-level Lock 지원
- **UTF-8mb4 문자셋**: 한국어 및 이모지 완전 지원
- **월별 RANGE 파티셔닝**: sessions 테이블에 적용하여 대량 데이터 조회 성능 확보
- **복합 인덱스 설계**: 실제 쿼리 패턴(user_id + date 범위)에 최적화
- **사전 집계 테이블**: daily_stats로 반복 집계 연산 제거
- **Connection Pool**: C 서버(libmysqlclient), Next.js(Prisma)에서 각각 풀링

### 10.2 테이블 정의

#### users (사용자)

```sql
CREATE TABLE users (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY COMMENT 'AUTO_INCREMENT PK',
    phone_hash      CHAR(64)        NOT NULL COMMENT 'SHA-256 (조회용 인덱스)',
    phone_bcrypt    CHAR(60)        NOT NULL COMMENT 'bcrypt (검증용)',
    nickname        VARCHAR(30)     DEFAULT NULL COMMENT '선택적 닉네임',
    email_encrypted VARBINARY(512)  NOT NULL COMMENT '이메일 (AES-256 암호화 저장)',
    role            ENUM('user', 'admin') NOT NULL DEFAULT 'user',
    consent_at      DATETIME        NOT NULL COMMENT '개인정보 동의 일시',
    last_login_at   DATETIME        DEFAULT NULL COMMENT '마지막 로그인 일시',
    is_deleted      TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '탈퇴 여부 (소프트 삭제)',
    created_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE INDEX idx_phone_hash (phone_hash),
    INDEX idx_role (role),
    INDEX idx_is_deleted (is_deleted)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### equipment (기구)

```sql
CREATE TABLE equipment (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100)    NOT NULL COMMENT '기구 이름',
    type            ENUM('cardio', 'strength', 'stretching', 'other') NOT NULL COMMENT '기구 유형',
    calorie_per_min DECIMAL(5,2)    NOT NULL DEFAULT 5.00 COMMENT '분당 기본 칼로리',
    status          ENUM('available', 'in_use', 'maintenance') NOT NULL DEFAULT 'available',
    current_user_id BIGINT UNSIGNED DEFAULT NULL,
    current_session_start DATETIME  DEFAULT NULL COMMENT '현재 세션 시작 시각',
    api_key_hash    CHAR(64)        NOT NULL COMMENT '기기 인증용 API Key SHA-256',
    last_heartbeat  DATETIME        DEFAULT NULL COMMENT '마지막 하트비트 수신 시각',
    created_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_status (status),
    INDEX idx_type_status (type, status),
    INDEX idx_current_user (current_user_id),
    INDEX idx_api_key_hash (api_key_hash),
    CONSTRAINT fk_equip_user FOREIGN KEY (current_user_id) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### sessions (운동 세션) — 월별 파티셔닝

```sql
CREATE TABLE sessions (
    id              BIGINT UNSIGNED AUTO_INCREMENT,
    user_id         BIGINT UNSIGNED NOT NULL,
    equipment_id    BIGINT UNSIGNED NOT NULL,
    equipment_type  ENUM('cardio', 'strength', 'stretching', 'other') NOT NULL,
    start_time      DATETIME        NOT NULL,
    end_time        DATETIME        DEFAULT NULL COMMENT 'NULL이면 진행 중',
    duration_sec    INT UNSIGNED    DEFAULT NULL COMMENT '자동 계산 (end - start)',
    calories        DECIMAL(8,2)    DEFAULT NULL COMMENT '소모 칼로리',
    avg_intensity   DECIMAL(3,2)    DEFAULT 1.00 COMMENT '평균 강도 계수',
    auto_logout     TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '자동 로그아웃 여부',
    auto_logout_reason ENUM('manual', 'duplicate_login', 'heartbeat_timeout', 'inactivity', 'gym_close', 'server_cleanup')
                    NOT NULL DEFAULT 'manual',
    created_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (id, start_time),
    INDEX idx_user_start (user_id, start_time),
    INDEX idx_user_date (user_id, start_time, end_time),
    INDEX idx_equipment_start (equipment_id, start_time),
    INDEX idx_active_sessions (end_time, user_id),
    CONSTRAINT fk_session_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_session_equip FOREIGN KEY (equipment_id) REFERENCES equipment(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
PARTITION BY RANGE (YEAR(start_time) * 100 + MONTH(start_time)) (
    PARTITION p202601 VALUES LESS THAN (202602),
    PARTITION p202602 VALUES LESS THAN (202603),
    PARTITION p202603 VALUES LESS THAN (202604),
    PARTITION p202604 VALUES LESS THAN (202605),
    PARTITION p202605 VALUES LESS THAN (202606),
    PARTITION p202606 VALUES LESS THAN (202607),
    PARTITION p202607 VALUES LESS THAN (202608),
    PARTITION p202608 VALUES LESS THAN (202609),
    PARTITION p202609 VALUES LESS THAN (202610),
    PARTITION p202610 VALUES LESS THAN (202611),
    PARTITION p202611 VALUES LESS THAN (202612),
    PARTITION p202612 VALUES LESS THAN (202701),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

#### daily_stats (일별 사전 집계)

```sql
CREATE TABLE daily_stats (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED NOT NULL,
    stat_date       DATE            NOT NULL,
    total_sessions  INT UNSIGNED    NOT NULL DEFAULT 0,
    total_duration_sec INT UNSIGNED NOT NULL DEFAULT 0,
    total_calories  DECIMAL(10,2)   NOT NULL DEFAULT 0.00,
    cardio_calories DECIMAL(10,2)   NOT NULL DEFAULT 0.00,
    strength_calories DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    equipment_count INT UNSIGNED    NOT NULL DEFAULT 0 COMMENT '사용한 기구 종류 수',
    created_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE INDEX idx_user_date (user_id, stat_date),
    INDEX idx_stat_date (stat_date),
    INDEX idx_user_date_range (user_id, stat_date),
    CONSTRAINT fk_daily_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### web_sessions (웹 세션 관리 — 메인은 Redis, MySQL은 백업)

```sql
CREATE TABLE web_sessions (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    session_id      CHAR(36)        NOT NULL COMMENT 'UUID v4 세션 ID',
    user_id         BIGINT UNSIGNED NOT NULL,
    expires_at      DATETIME        NOT NULL,
    ip_address      VARCHAR(45)     DEFAULT NULL,
    user_agent      VARCHAR(255)    DEFAULT NULL,
    created_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,

    UNIQUE INDEX idx_session_id (session_id),
    INDEX idx_user_id (user_id),
    INDEX idx_expires (expires_at),
    CONSTRAINT fk_ws_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### access_log (접속 기록 — 법률 준수용)

```sql
CREATE TABLE access_log (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED DEFAULT NULL,
    action          ENUM('login_device', 'logout_device', 'auto_logout', 'login_web', 'logout_web', 'token_refresh', 'delete_account', 'view_records') NOT NULL,
    detail          VARCHAR(255)    DEFAULT NULL COMMENT '부가 정보 (기구ID, 사유 등)',
    ip_address      VARCHAR(45)     NOT NULL COMMENT 'IPv4/IPv6',
    user_agent      VARCHAR(255)    DEFAULT NULL,
    created_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_action (user_id, action, created_at),
    INDEX idx_created (created_at),
    CONSTRAINT fk_log_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 10.3 쿼리 성능 최적화 전략

| 조회 유형 | 쿼리 대상 | 최적화 방법 |
|-----------|-----------|-------------|
| 일별 조회 | daily_stats 단일 행 | `idx_user_date` → O(1) 수준 |
| 주간 조회 | daily_stats 7행 | `idx_user_date_range` WHERE BETWEEN → 인덱스 범위 스캔 |
| 월별 조회 | daily_stats 28~31행 | 동일 인덱스 범위 스캔 |
| 연도별 조회 | daily_stats 365행 GROUP BY MONTH | 인덱스 커버링 쿼리 |
| 실시간 기구 상태 | Redis 캐시 | `equipment:{id}` 키로 3초 TTL 캐싱 |
| 활성 세션 조회 | sessions WHERE end_time IS NULL | `idx_active_sessions` 활용 |
| 사용자 오늘 총 칼로리 | daily_stats 또는 Redis | 세션 종료 시 daily_stats UPDATE + Redis 갱신 |

### 10.4 daily_stats 갱신 타이밍

- **세션 종료 시**: C 서버가 sessions INSERT 후 daily_stats UPSERT 수행
- **자동 로그아웃 시**: 동일하게 daily_stats 갱신
- **데이터 정합성 배치**: 매일 02:00 전일 daily_stats를 sessions 기반으로 재계산 (cron)

---

## 11. 통신 프로토콜 설계

### 11.1 기기 ↔ C 서버 (TCP)

JSON 기반 메시지를 TCP 소켓으로 전송한다. 메시지 앞에 4바이트 길이 헤더를 붙인다.

```
[4 bytes: payload length (big-endian)] [JSON payload]
```

#### 기기 인증

```json
{ "type": "DEVICE_AUTH", "device_id": 3, "api_key": "pre-shared-key" }
→ { "type": "AUTH_OK" }
→ { "type": "AUTH_FAIL", "reason": "invalid_key" }
```

#### 로그인 요청

```json
{ "type": "LOGIN", "device_id": 3, "phone": "01012345678" }
→ { "type": "LOGIN_OK", "user_id": 12, "today_total_calories": 320.5, "today_total_time_sec": 3600 }
→ { "type": "LOGIN_FAIL", "reason": "consent_required" }
→ { "type": "LOGIN_FAIL", "reason": "user_not_found" }
```

#### 신규 사용자 등록 (리눅에서 미등록 사용자 발견 시)

```json
{ "type": "REGISTER", "device_id": 3, "phone": "01012345678", "email": "user@example.com", "consent": true }
→ { "type": "REGISTER_OK", "user_id": 13, "today_total_calories": 0, "today_total_time_sec": 0 }
→ { "type": "REGISTER_FAIL", "reason": "invalid_email" }
```

#### 개인정보 동의

```json
{ "type": "CONSENT", "device_id": 3, "phone": "01012345678", "email": "user@example.com", "agreed": true }
→ { "type": "CONSENT_OK", "user_id": 12 }
```

#### 하트비트 + 센서 데이터 (3초 주기)

```json
{
  "type": "HEARTBEAT",
  "device_id": 3,
  "user_id": 12,
  "elapsed_sec": 600,
  "sensor_data": { "rpm": 80, "speed": 12.5, "resistance": 5 }
}
→ {
  "type": "DASHBOARD",
  "today_total_calories": 405.7,
  "current_calories": 85.2,
  "today_total_time_sec": 4200,
  "current_time_sec": 600
}
```

#### 로그아웃

```json
{ "type": "LOGOUT", "device_id": 3, "user_id": 12, "local_data": { "start_time": "2026-03-29T10:00:00Z", "sensor_log": [...], "total_calories": 85.2 } }
→ { "type": "LOGOUT_OK", "session_calories": 85.2, "session_duration_sec": 600 }
```

> **운동 종료 흐름**: 사용자가 "운동 종료" 버튼 클릭 → 기기가 Encrypted Storage에서 세션 데이터 읽기 → LOGOUT 메시지에 포함하여 서버 전송 → LOGOUT_OK 수신 후 로컬 암호화 저장소 완전 삭제 → 로그인 화면으로 복귀
```

#### 서버 → 기기 (강제 로그아웃)

```json
{ "type": "FORCE_LOGOUT", "reason": "duplicate_login", "message": "다른 기구에서 로그인되어 자동 로그아웃됩니다." }
```

#### 서버 → 기기 (비활동 경고)

```json
{ "type": "WARNING", "code": "inactivity", "message": "자동 로그아웃 1분 전입니다. 운동을 계속하려면 기구를 사용해 주세요.", "remaining_sec": 60 }
```

### 11.2 Next.js API Routes (웹 REST API)

#### 인증

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/auth/login` | 전화번호 로그인 → 세션 ID 쿠키 발급 | 없음 |
| POST | `/api/auth/logout` | 로그아웃 → Redis 세션 삭제 + 쿠키 제거 | 세션 필요 |
| GET | `/api/auth/me` | 현재 로그인 상태 확인 (세션 유효성 검증) | 세션 필요 |
| POST | `/api/auth/email-verify/send` | 이메일로 6자리 인증 코드 발송 (TTL 5분, Redis 저장) | 없음 |
| POST | `/api/auth/email-verify/confirm` | 코드 검증 + 회원가입 완료 (users AUTO_INCREMENT INSERT + 세션 발급) | 없음 |

#### 운동 기록 조회

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| GET | `/api/records/daily?date=2026-03-29` | 일별 운동 기록 (세션 목록) | 세션 필요 |
| GET | `/api/records/weekly?start=2026-03-23` | 주간 운동 기록 (일별 합산) | 세션 필요 |
| GET | `/api/records/monthly?year=2026&month=3` | 월별 운동 기록 (일별 합산) | 세션 필요 |
| GET | `/api/records/yearly?year=2026` | 연도별 운동 기록 (월별 합산) | 세션 필요 |
| GET | `/api/records/custom?start=...&end=...` | 커스텀 기간 조회 | 세션 필요 |
| GET | `/api/records/summary` | 종합 요약 (총 운동일, 누적 칼로리 등) | 세션 필요 |

#### 기구 현황

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| GET | `/api/equipment` | 전체 기구 상태 목록 (Redis 캐시) | 없음 (공개) |
| GET | `/api/equipment/:id` | 특정 기구 상세 | 없음 |
| GET | `/api/equipment/occupancy` | 혼잡도 (유형별 사용률) | 없음 |

#### 사용자 관리

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| GET | `/api/user/profile` | 내 정보 조회 (마스킹된 전화번호) | 세션 필요 |
| DELETE | `/api/user/account` | 회원 탈퇴 (소프트 삭제 → 30일 후 하드 삭제) | 세션 필요 |
| GET | `/api/user/privacy-policy` | 개인정보 처리방침 | 없음 |

#### API 응답 형식 (공통)

```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "timestamp": "2026-03-29T10:30:00Z"
}
```

```json
{
  "success": false,
  "data": null,
  "error": { "code": "INVALID_TOKEN", "message": "Access Token이 만료되었습니다." },
  "timestamp": "2026-03-29T10:30:00Z"
}
```

---

## 12. 칼로리 계산 방식

```
소모 칼로리 = 기구별 분당 기본 칼로리 × 사용 시간(분) × 강도 계수
```

- **기구별 분당 기본 칼로리**: DB의 `calorie_per_min` 값 활용
- **강도 계수**: 센서 데이터(RPM, 속도, 저항 등)로부터 산출 (1.0 기준, 0.5~2.0 범위)
- **강도 계수 산출 공식**: `intensity = min(2.0, max(0.5, sensor_value / base_value))`
- 센서가 없는 기구는 강도 계수 1.0(기본값) 적용
- **자동 로그아웃 시**: 마지막 유효 센서 데이터 시점까지만 칼로리 계산 (이후 구간은 제외)
- **실시간 칼로리 갱신**: 하트비트(3초)마다 현재까지의 칼로리를 재계산하여 기기에 전송

---

## 13. 개발 일정 (안)

### 1단계: 기반 구축 (1~2주차)

| 주차 | 작업 내용 | 담당 |
|------|-----------|------|
| 1주 | 요구사항 확정, DB 스키마 최종 확정, 프로토콜 확정, 개발환경 셋업 (docker-compose) | 전원 |
| 1주 | Next.js 프로젝트 초기화, Prisma 스키마 정의, Tailwind 설정 | 웹팀 |
| 2주 | TCP 서버 기본 구현, MySQL Connection Pool, 기기 인증 | 서버팀 (2명) |
| 2주 | IoT 기기 시뮬레이터 기본 구현, TCP 통신 모듈 | 기기팀 (1명) |
| 2주 | Next.js 로그인 페이지, JWT 인증 모듈, Redis 연동 | 웹팀 (1명) |

### 2단계: 핵심 기능 구현 (3~5주차)

| 주차 | 작업 내용 | 담당 |
|------|-----------|------|
| 3주 | 기기 로그인/로그아웃, 세션 관리, 칼로리 계산 | 서버팀 |
| 3주 | 기기 UI (ncurses: 로그인 화면, 대시보드, 비활동 경고), 센서 시뮬레이션 | 기기팀 |
| 3주 | 웹 로그인/로그아웃, JWT 토큰 갱신, 사용자 페이지 레이아웃 | 웹팀 |
| 4주 | 자동 로그아웃 워치독 (중복 로그인, 하트비트, 비활동), daily_stats 갱신 | 서버팀 |
| 4주 | 운동 기록 API (일별/주간/월별/연도별), 차트 컴포넌트 | 웹팀 |
| 5주 | 기기↔C 서버 통합, Next.js↔MySQL 통합, Redis 캐시 연동 | 전원 |

### 3단계: 완성 및 보안·법률 (6~8주차)

| 주차 | 작업 내용 | 담당 |
|------|-----------|------|
| 6주 | 개인정보 동의 절차 (기기+웹), 접속 기록 로깅, 회원 탈퇴 | 서버팀 + 웹팀 |
| 6주 | 기구 현황 페이지, 혼잡도 표시, 관리자 기능 | 웹팀 |
| 6주 | 비정상 종료 처리, 영업 종료 일괄 로그아웃, 안정성 보강 | 서버팀 + 기기팀 |
| 7주 | 보안 점검 (Rate Limiting, CSRF, XSS, SQL Injection 테스트), 통합 테스트 | 전원 |
| 8주 | 부하 테스트, 최종 버그 수정, 문서 정리, 특허 명세서 초안 | 전원 |

---

## 14. 역할 분담 (안)

| 역할 | 인원 | 담당 범위 |
|------|------|-----------|
| 서버 개발자 A | 1명 | C TCP 서버, 세션 관리, 자동 로그아웃 워치독, MySQL 연동 |
| 서버 개발자 B | 1명 | C 서버 칼로리 로직, 기기 인증, daily_stats 배치, DB 최적화 |
| 기기/풀스택 C | 1명 | IoT 기기 시뮬레이터 (C), 센서/네트워크/UI, Next.js 기구 현황 페이지 |
| 웹 개발자 D | 1명 | Next.js 인증(세션), 운동 기록 조회 (API + 프론트), 차트, 관리자 |

---

## 15. 테스트 계획

| 테스트 유형 | 대상 | 방법 |
|-------------|------|------|
| 단위 테스트 | 칼로리 계산, 해시 함수, 세션 검증, 자동 로그아웃 판정 | C: assert, Next.js: Jest |
| 통합 테스트 | 기기→C서버→MySQL→daily_stats 흐름 | 시뮬레이터로 다수 기기 동시 접속 |
| API 테스트 | Next.js REST API 전체 | Jest + Supertest (응답 스키마 검증) |
| 인증 테스트 | 세션 발급/만료/삭제/Redis Fallback | 만료 세션, 탈취 세션 재사용 시나리오 |
| 자동 로그아웃 | 5가지 시나리오별 | 기기 시뮬레이터로 각 시나리오 재현 |
| 보안 테스트 | SQL Injection, XSS, CSRF, Rate Limit | 비정상 입력 주입, 도구(sqlmap 등) 활용 |
| 부하 테스트 | 50대 기기 동시 접속 + 웹 100명 접속 | 시뮬레이터 + k6/ab 부하 테스트 도구 |
| 법률 검증 | 개인정보 처리 흐름 전체 | 체크리스트 기반 점검 |

---

## 부록 A: 개인정보 처리방침 항목 (필수 포함 사항)

1. 개인정보의 처리 목적
2. 수집하는 개인정보 항목 (전화번호, 이메일)
3. 개인정보의 처리 및 보유 기간 (마지막 로그인일로부터 1년)
4. 개인정보의 제3자 제공 여부 (해당 없음)
5. 개인정보의 파기 절차 및 방법 (자동 배치 파기 + 1주일 전 이메일 안내)
6. 이용자의 권리·의무 및 행사 방법
7. 개인정보 보호 책임자 연락처
8. 개인정보의 안전성 확보 조치 (해시 처리, 접근 통제, 암호화 통신)

## 부록 B: 개인정보 수집·이용 동의 화면 문구 (안)

### 기기 화면 (간소화 버전)
```
[개인정보 수집·이용 동의]

1. 수집 항목: 전화번호, 이메일
2. 수집 목적: 사용자 식별, 운동 기록 관리, 계정 만기 안내
3. 보유 기간: 마지막 이용일로부터 1년 (만료 1주일 전 이메일 안내)
4. 동의를 거부할 수 있으며, 거부 시 서비스 이용이 제한됩니다.
5. 전체 개인정보 처리방침: https://[도메인]/privacy-policy

[동의함] [동의하지 않음]
```

### 웹 화면 (상세 버전)
```
웹 로그인 페이지에 개인정보 처리방침 전문 링크를 배치하며,
최초 로그인 시 동의 여부를 확인한다. (기기에서 이미 동의한 사용자는 자동 통과)
미등록 사용자는 전화번호+이메일+동의 입력 후 회원가입 처리
```

## 부록 C: Redis 키 설계

| 키 패턴 | 타입 | TTL | 설명 |
|---------|------|-----|------|
| `session:{session_id}` | Hash | 14일 | 웹 세션 데이터 (user_id, role, created_at) |
| `email_verify:{SHA-256(email)}` | Hash | 5분 | 회원가입 이메일 인증 코드 + 실패횟수 `{ code, phone, attempts }` |
| `email_verify_rate:{ip}` | String | 1분 | IP별 이메일 코드 발송 횟수 (Rate Limit) |
| `email_verify_rate_mail:{SHA-256(email)}` | String | 10분 | 이메일별 코드 발송 횟수 (Rate Limit) |
| `equipment:{id}` | Hash | 5초 | 기구 실시간 상태 캐시 |
| `equipment:all` | String (JSON) | 3초 | 전체 기구 목록 캐시 |
| `user:{id}:today` | Hash | 60초 | 오늘 총 칼로리/시간 캐시 |
| `rate_limit:login:{ip}` | String (count) | 60초 | 로그인 Rate Limit |
| `rate_limit:api:{ip}` | String (count) | 60초 | API Rate Limit |

## 부록 D: 환경 변수 목록

| 변수 | 대상 | 설명 |
|------|------|------|
| `MYSQL_HOST` | C + Next.js | MySQL 호스트 |
| `MYSQL_PORT` | C + Next.js | MySQL 포트 (기본 3306) |
| `MYSQL_USER` | C + Next.js | MySQL 사용자 |
| `MYSQL_PASSWORD` | C + Next.js | MySQL 비밀번호 |
| `MYSQL_DATABASE` | C + Next.js | MySQL 데이터베이스명 |
| `REDIS_URL` | Next.js | Redis 접속 URL |
| `SESSION_SECRET` | Next.js | 세션 쿠키 서명용 비밀키 |
| `SESSION_MAX_AGE` | Next.js | 세션 만료 시간 (기본 14일, 초 단위) |
| `EMAIL_ENCRYPTION_KEY` | C + Next.js | 이메일 AES-256 암호화 키 (32바이트 hex) |
| `SMTP_HOST` | Next.js | 네이버 SMTP 호스트 (smtp.naver.com) |
| `SMTP_PORT` | Next.js | SMTP 포트 (기본 465) |
| `SMTP_USER` | Next.js | 네이버 이메일 계정 |
| `SMTP_PASS` | Next.js | 네이버 이메일 비밀번호 |
| `TCP_PORT` | C 서버 | IoT 기기 TCP 통신 포트 |
| `INACTIVITY_TIMEOUT_SEC` | C 서버 | 비활동 타임아웃 (기본 180초) |
| `HEARTBEAT_TIMEOUT_SEC` | C 서버 | 하트비트 타임아웃 (기본 10초) |
| `GYM_CLOSE_HOUR` | C 서버 | 영업 종료 시각 (기본 23) |

---

## 부록 E: 초기 기구 데이터 (init.sql 시드)

> 총 20대 — 유산소 8, 무산소 8, 스트레칭 3, 기타 1

```sql
INSERT INTO equipment (name, type, calorie_per_min, status, api_key_hash) VALUES
-- 유산소 (cardio) 8대
('트레드밀 #1',    'cardio',     10.00, 'available', SHA2('treadmill-1-key', 256)),
('트레드밀 #2',    'cardio',     10.00, 'available', SHA2('treadmill-2-key', 256)),
('트레드밀 #3',    'cardio',     10.00, 'available', SHA2('treadmill-3-key', 256)),
('실내자전거 #1',  'cardio',      8.00, 'available', SHA2('bike-1-key', 256)),
('실내자전거 #2',  'cardio',      8.00, 'available', SHA2('bike-2-key', 256)),
('일립티컬 #1',    'cardio',      9.00, 'available', SHA2('elliptical-1-key', 256)),
('일립티컬 #2',    'cardio',      9.00, 'available', SHA2('elliptical-2-key', 256)),
('로잉머신 #1',    'cardio',      7.00, 'available', SHA2('rowing-1-key', 256)),
-- 무산소 (strength) 8대
('체스트프레스',    'strength',    5.00, 'available', SHA2('chest-press-key', 256)),
('레그프레스',      'strength',    6.00, 'available', SHA2('leg-press-key', 256)),
('랫풀다운',        'strength',    5.00, 'available', SHA2('lat-pulldown-key', 256)),
('숄더프레스',      'strength',    5.00, 'available', SHA2('shoulder-press-key', 256)),
('레그컬',          'strength',    4.50, 'available', SHA2('leg-curl-key', 256)),
('레그익스텐션',    'strength',    4.50, 'available', SHA2('leg-extension-key', 256)),
('케이블머신',      'strength',    5.00, 'available', SHA2('cable-machine-key', 256)),
('스미스머신',      'strength',    6.00, 'available', SHA2('smith-machine-key', 256)),
-- 스트레칭 (stretching) 3대
('폼롤러존',        'stretching',  3.00, 'available', SHA2('foam-roller-key', 256)),
('스트레칭매트',    'stretching',  3.00, 'available', SHA2('stretching-mat-key', 256)),
('요가존',          'stretching',  3.00, 'available', SHA2('yoga-zone-key', 256)),
-- 기타 (other) 1대
('행잉레그레이즈',  'other',       4.00, 'available', SHA2('hanging-leg-raise-key', 256));
```

```sql
-- 관리자 계정 시드 (비밀번호는 배포 전 변경 필수)
-- 전화번호: 01000000000 / API에서 직접 생성하지 않고 DB에 직접 삽입
INSERT INTO users (phone_hash, phone_bcrypt, email_encrypted, role, consent_at)
VALUES (
    SHA2('01000000000', 256),
    '$2b$12$ADMIN_BCRYPT_HASH_HERE',
    AES_ENCRYPT('admin@gym.com', 'ENCRYPTION_KEY_HERE'),
    'admin',
    NOW()
);
```

---

## 부록 F: 관리자 기능 상세

### 관리자 API

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| GET | `/api/admin/equipment` | 기구 목록 + CRUD | admin 세션 |
| POST | `/api/admin/equipment` | 기구 추가 | admin 세션 |
| PUT | `/api/admin/equipment/:id` | 기구 수정 (이름, 유형, 칼로리, 상태) | admin 세션 |
| DELETE | `/api/admin/equipment/:id` | 기구 삭제 | admin 세션 |
| PUT | `/api/admin/equipment/:id/status` | 기구 상태 변경 (점검 중 등) | admin 세션 |
| GET | `/api/admin/sessions/active` | 현재 활성 세션 목록 | admin 세션 |
| POST | `/api/admin/sessions/:id/force-logout` | 특정 기기 강제 로그아웃 | admin 세션 |
| POST | `/api/admin/gym/close` | 영업 종료 일괄 로그아웃 (수동 트리거) | admin 세션 |
| GET | `/api/admin/stats/daily?date=...` | 일별 헬스장 통계 (이용자수, 가동률) | admin 세션 |
| GET | `/api/admin/logs` | 접속 기록 조회 (access_log) | admin 세션 |

### 관리자 인증

- `role='admin'`인 사용자만 `/admin` 경로 및 `/api/admin/*` API 접근 가능
- 세션 미들웨어에서 role 검사 → admin 아니면 403 Forbidden
- 관리자 계정은 `init.sql`로 DB에 직접 삽입 (D-01 결정사항)

---

## 부록 G: 에러 코드 체계

| 코드 | 설명 |
|------|------|
| `AUTH_001` | 전화번호 형식 오류 |
| `AUTH_002` | 사용자 미등록 (웹에서 회원가입 필요) |
| `AUTH_003` | 세션 만료 |
| `AUTH_004` | 세션 무효 (Redis에 존재하지 않음) |
| `AUTH_005` | 이메일 형식 오류 |
| `AUTH_006` | 동의 미완료 (consent_required) |
| `AUTH_007` | Rate Limit 초과 |
| `AUTH_008` | 권한 부족 (admin 전용 API) |
| `AUTH_009` | 이웸일 인증 코드 오류 |
| `AUTH_010` | 이메일 인증 코드 만료 (5분 TTL 초과) |
| `AUTH_011` | 이메일 인증 코드 실패 횟수 초과 (5회 이상) |
| `AUTH_012` | 신규 가입 시 이맸 존재하는 전화번호 |
| `EQUIP_001` | 기구 없음 |
| `EQUIP_002` | 기구 점검 중 |
| `EQUIP_003` | 기구 이미 사용 중 |
| `SESSION_001` | 활성 세션 없음 |
| `SESSION_002` | 세션 종료 실패 |
| `RECORD_001` | 조회 기간 오류 |
| `RECORD_002` | 데이터 없음 |
| `USER_001` | 탈퇴 처리 실패 |
| `USER_002` | 이미 탈퇴한 계정 |
| `SYSTEM_001` | DB 연결 오류 |
| `SYSTEM_002` | Redis 연결 오류 |
| `SYSTEM_003` | 내부 서버 오류 |

---

## 부록 H: 이메일 알림 설계 (네이버 SMTP)

### SMTP 설정

| 항목 | 값 |
|------|-----|
| 호스트 | `smtp.naver.com` |
| 포트 | 465 (SSL) |
| 인증 | 네이버 계정 아이디/비밀번호 |
| 패키지 | `nodemailer` |

### 이메일 발송 시나리오

| 시나리오 | 발송 시점 | 제목 | 내용 |
|----------|-----------|------|------|
| **회원가입 이메일 인증** | `/api/auth/email-verify/send` 호출 시 | `[헬스장 IoT] 이메일 인증번호 안내` | 6자리 인증코드, 유효시간 5분 |
| 계정 만료 사전 안내 | 마지막 로그인 + 358일 (만료 7일 전) | `[헬스장 IoT] 계정 만료 7일 전 안내` | 1년간 미이용 시 계정 자동 삭제 안내, 로그인 유도 |
| 계정 삭제 완료 | 마지막 로그인 + 365일 (만료 당일) | `[헬스장 IoT] 계정이 삭제되었습니다` | 개인정보 파기 완료 안내 |

### 자동 배치 (cron)

```
# 매일 새벽 2시 실행 (Next.js API Route 또는 별도 Node.js 스크립트)

1. 만료 7일 전 사용자 조회:
   SELECT id, email_encrypted FROM users
   WHERE is_deleted = 0
   AND last_login_at <= NOW() - INTERVAL 358 DAY
   AND last_login_at > NOW() - INTERVAL 359 DAY

2. 이메일 발송 (AES 복호화 후 nodemailer로 전송)

3. 만료 당일 사용자 조회 + 소프트 삭제:
   UPDATE users SET is_deleted = 1
   WHERE is_deleted = 0
   AND last_login_at <= NOW() - INTERVAL 365 DAY

4. 소프트 삭제 30일 경과 사용자 하드 삭제:
   DELETE FROM users
   WHERE is_deleted = 1
   AND updated_at <= NOW() - INTERVAL 30 DAY
```

### 이메일 템플릿 (이메일 인증 코드)

```
안녕하세요.

[헬스장 IoT 기구 관리 시스템]에 가입하신 것을 환영합니다.

아래 인증번호를 입력하여 회원가입을 완료해 주세요.

인증번호: {CODE}

유효시간: 5분이내

본인이 요청하지 않았다면 이 메일을 무시하시기 바랍니다.

감사합니다.
헬스장 IoT 기구 관리 시스템
```

### 이메일 템플릿 (만료 7일 전 안내)

```
안녕하세요.

[헬스장 IoT 기구 관리 시스템]을 이용해 주셔서 감사합니다.

귀하의 계정이 1년간 미이용으로 인해 {만료_예정일}에 자동 삭제될
예정입니다.

계정 유지를 원하시면 아래 웹사이트에서 로그인해 주세요:
{로그인_URL}

삭제 시 모든 운동 기록이 영구적으로 삭제되며 복구할 수 없습니다.

감사합니다.
헬스장 IoT 기구 관리 시스템
```

---

## 부록 I: WebSocket 프로토콜 설계

### 연결 방식

```
클라이언트: ws://localhost:3000  (개발 환경)
           wss://[도메인]        (운영 환경)
```

### 서버 구현 (Next.js 커스텀 서버)

```javascript
// web/server.js 개념
const { createServer } = require('http');
const { parse } = require('url');
const next = require('next');
const { WebSocketServer } = require('ws');

const app = next({ dev: true });
const handle = app.getRequestHandler();

app.prepare().then(() => {
  const server = createServer((req, res) => handle(req, res, parse(req.url, true)));
  const wss = new WebSocketServer({ server, path: '/ws/equipment' });

  // 2~3초마다 Redis에서 기구 상태 읽어 브로드캐스트
  setInterval(async () => {
    const equipmentData = await redis.get('equipment:all');
    const message = JSON.stringify({ type: 'EQUIPMENT_UPDATE', data: JSON.parse(equipmentData) });
    wss.clients.forEach(client => {
      if (client.readyState === 1) client.send(message);
    });
  }, 2000);

  server.listen(3000);
});
```

### 메시지 형식

#### 서버 → 클라이언트 (기구 상태 브로드캐스트)

```json
{
  "type": "EQUIPMENT_UPDATE",
  "data": [
    { "id": 1, "name": "트레드밀 #1", "type": "cardio", "status": "in_use", "current_user_masked": "010-****-5678", "session_start": "2026-03-29T10:00:00Z" },
    { "id": 2, "name": "트레드밀 #2", "type": "cardio", "status": "available", "current_user_masked": null, "session_start": null }
  ],
  "occupancy": { "total": 20, "in_use": 8, "available": 11, "maintenance": 1, "rate": 0.40 },
  "timestamp": "2026-03-29T10:30:00Z"
}
```

### 흐름

```
1. 사용자가 IoT 기기에서 로그인
2. C 서버 → MySQL equipment.status='in_use' + Redis equipment:{id} 갱신
3. Next.js WebSocket 서버 → Redis 폴링 (2초) → equipment:all 갱신 감지
4. WebSocket 서버 → 연결된 모든 웹 브라우저에 EQUIPMENT_UPDATE 브로드캐스트
5. 웹 브라우저 → UI 즉시 갱신 (사용 중 → 빨간색, 미사용 → 초록색)
6. 사용자 로그아웃/자동 로그아웃 → C 서버 → equipment.status='available' + Redis 갱신
7. WebSocket 브로드캐스트 → 웹 UI 즉시 갱신
```

---

## 부록 J: 회원 탈퇴 및 계정 삭제 흐름

### 사용자 직접 탈퇴 (D-10)

```
1. 사용자가 웹에서 DELETE /api/user/account 요청
2. 서버: users.is_deleted = 1, updated_at = NOW() (소프트 삭제)
3. 서버: Redis 세션 삭제, 쿠키 제거 (즉시 로그아웃)
4. 30일간 복구 가능 (같은 전화번호로 로그인 시 복구 안내)
5. 30일 후: 자동 배치에서 하드 삭제 (users + sessions + daily_stats + access_log 연쇄 삭제)
```

### 자동 만료 삭제 (D-06)

```
1. 마지막 로그인(last_login_at) 기준 358일 경과: 이메일 안내 발송
2. 365일 경과: is_deleted = 1 (소프트 삭제)
3. 395일 경과 (소프트 삭제 + 30일): 하드 삭제
```

---

## 부록 K: 센서 데이터 유형별 정의

| 기구 유형 | 센서 항목 | base_value (강도 1.0 기준) | 칼로리/분 기본값 |
|-----------|-----------|---------------------------|-----------------|
| **cardio (유산소)** | rpm, speed(km/h), resistance(1-20) | rpm=60, speed=10, resistance=5 | 8.0~10.0 kcal |
| **strength (무산소)** | reps(반복횟수), weight(kg) | reps=10, weight=20 | 4.5~6.0 kcal |
| **stretching** | duration_hold(초) | hold=30 | 3.0 kcal |
| **other** | 없음 (기본값 사용) | N/A | 4.0 kcal |

### 강도 계수 산출 공식

```
cardio:     intensity = avg(rpm/60, speed/10, resistance/5) → clamp(0.5, 2.0)
strength:   intensity = avg(reps/10, weight/20) → clamp(0.5, 2.0)
stretching: intensity = 1.0 (고정)
other:      intensity = 1.0 (고정)
```

---

## 부록 L: 의사결정 요약

> 아래는 v2.0 → v3.0 업데이트 시 확정된 12개 의사결정 사항입니다. v3.1에서 이메일 인증 확정 사항이 추가되었습니다.

| ID | 항목 | 결정 |
|----|------|------|
| D-01 | 관리자 계정 생성 방식 | DB 직접 삽입 + init.sql 시드 |
| D-02 | TCP 통신 보안 | 헬스장 내부 WiFi LAN 전용 (TLS 불필요) |
| D-03 | 웹 실시간 갱신 방식 | WebSocket (ws 라이브러리, 2~3초 Redis 폴링 후 브로드캐스트) |
| D-04 | 인증 방식 | 세션 ID 쿠키 기반 (JWT 제거, Redis 세션 저장소) |
| D-05 | 배포 환경 | 로컬 전용 (docker-compose로 MySQL+Redis) |
| D-06 | 개인정보 만료 처리 | 자동 배치 (마지막 로그인 1년, 7일 전 이메일 안내, 네이버 SMTP) |
| D-07 | 웹 최초 로그인 동의 | 동의 여부 확인 후 통과/표시, 미등록 시 회원가입 |
| D-08 | 초기 기구 데이터 | 중규모 20대 (유산소8+무산소8+스트레칭3+기타1) |
| D-09 | 영업시간 설정 | 종료 시간만 (GYM_CLOSE_HOUR) |
| D-10 | 회원 탈퇴 처리 | 소프트 삭제 → 30일 후 하드 삭제 |
| D-11 | 웹 UI 테마 | 다크 테마 (Tailwind dark mode) |
| D-12 | C 서버 ↔ Next.js 통신 | 공유 DB + Redis (HTTP 내부 API 없음) |
