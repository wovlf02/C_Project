# 시스템 아키텍처 개요

> 최종 수정: 2026-03-30

---

## 1. 시스템 전체 구성

### 1.1 3-Tier 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                         │
│  ┌──────────────┐                  ┌──────────────────────┐  │
│  │  IoT 기기     │                  │  웹 브라우저          │  │
│  │  (ncurses UI) │                  │  (Next.js SSR)       │  │
│  └──────┬───────┘                  └──────────┬───────────┘  │
│         │ TCP (WiFi LAN)                      │ HTTPS        │
└─────────┼─────────────────────────────────────┼──────────────┘
          │                                     │
┌─────────┼─────────────────────────────────────┼──────────────┐
│         ▼           Application Layer          ▼              │
│  ┌──────────────┐                  ┌──────────────────────┐  │
│  │  C TCP 서버   │                  │  Next.js API Server  │  │
│  │  (멀티스레드)  │                  │  (App Router + WS)   │  │
│  └──────┬───────┘                  └──────────┬───────────┘  │
│         │                                     │              │
└─────────┼─────────────────────────────────────┼──────────────┘
          │                                     │
┌─────────┼─────────────────────────────────────┼──────────────┐
│         ▼            Data Layer                ▼              │
│  ┌──────────────────────┐     ┌──────────────────────────┐  │
│  │   MySQL 8.x (InnoDB) │     │   Redis 7.x              │  │
│  │   영구 데이터 저장      │     │  세션 + 캐시 + Rate Limit │  │
│  └──────────────────────┘     └──────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 1.2 구성 요소 역할

| 구성 요소 | 역할 | 기술 스택 |
|-----------|------|-----------|
| IoT 기기 (C) | 기구 탑재 디바이스. 사용자 로그인, 대시보드, 센서 수집 | C, POSIX Socket, ncurses |
| C TCP 서버 | IoT 기기와 TCP 통신, 세션 관리, 자동 로그아웃, 칼로리 정산 | C, pthread, libmysqlclient, cJSON, OpenSSL |
| Next.js 웹 서버 | 웹 인증(세션 쿠키), 운동 기록 API, SSR, WebSocket 기구 현황 | Next.js 16, Prisma, ioredis, ws |
| MySQL 8.x | users, equipment, sessions, daily_stats, access_log 영구 저장 | InnoDB, 파티셔닝 |
| Redis 7.x | 웹 세션, 기구 실시간 캐시, Rate Limit, 이메일 인증 코드 | Redis 7.x |

---

## 2. 데이터 흐름

### 2.1 IoT 기기 → C 서버 → DB

```
[IoT 기기]
  │ TCP 연결 (WiFi LAN)
  ├─ DEVICE_AUTH (기기별 API Key 인증)
  ├─ LOGIN / REGISTER (전화번호 기반)
  ├─ HEARTBEAT (3초 주기: 센서 데이터 + 하트비트)
  └─ LOGOUT (운동 종료 + 로컬 데이터 전송)
        │
        ▼
[C TCP 서버]
  ├─ MySQL: sessions, daily_stats INSERT/UPDATE
  ├─ Redis: equipment:{id} 상태 갱신 (3초 TTL)
  └─ Redis: user:{id}:today 칼로리 캐시 갱신
```

### 2.2 웹 브라우저 → Next.js → DB

```
[웹 브라우저]
  │ HTTPS
  ├─ POST /api/auth/login → 세션 ID 쿠키 발급
  ├─ GET /api/records/* → 운동 기록 조회 (daily_stats)
  ├─ GET /api/equipment → 기구 현황 (Redis 캐시)
  └─ WebSocket → 기구 실시간 상태 수신
        │
        ▼
[Next.js Server]
  ├─ Redis: 세션 조회/생성/삭제
  ├─ MySQL (Prisma): 사용자/기록/통계 CRUD
  └─ Redis: equipment:all 캐시 읽기 → WS 브로드캐스트
```

### 2.3 C 서버 ↔ Next.js 공유 방식

- **직접 API 호출 없음** — 두 서버는 DB/Redis를 통해 간접 통신
- C 서버가 Redis에 기구 상태를 기록 → Next.js가 Redis에서 읽어 브로드캐스트
- C 서버가 MySQL에 세션/통계 기록 → Next.js가 MySQL에서 읽어 API 응답

---

## 3. 네트워크 토폴로지

```
┌─────────────────── 헬스장 내부 WiFi LAN ───────────────────┐
│                                                              │
│  [기구1 IoT] ──┐                                            │
│  [기구2 IoT] ──┼── TCP ──▶ [C TCP 서버 :9000]              │
│  [기구3 IoT] ──┘              │                             │
│       ...                     ├── MySQL :3306               │
│                               ├── Redis :6379               │
│                               │                             │
│  [관리자 PC] ── HTTPS ──▶ [Next.js :3000]                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
          │
          │ 인터넷 (HTTPS)
          ▼
[외부 사용자 브라우저] ── HTTPS ──▶ [Next.js :3000]
```

---

## 4. 배포 아키텍처 (개발 환경)

```yaml
# docker-compose.yml 기반
services:
  mysql:     MySQL 8.x, port 3306
  redis:     Redis 7.x, port 6379
  c-server:  C TCP 서버 (바이너리 직접 실행), port 9000
  nextjs:    Next.js dev server, port 3000
```

| 환경 | MySQL | Redis | C 서버 | Next.js |
|------|-------|-------|--------|---------|
| 로컬 개발 | Docker | Docker | 네이티브 빌드 | npm run dev |
| 시연 | Docker | Docker | 네이티브 빌드 | npm run build + start |
