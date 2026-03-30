# Phase 1: 환경 구축 및 기반 코드 (1~2주차)

> 담당: 전원 참여  
> 목표: 개발 환경 세팅, DB 스키마 구현, 기본 통신 골격 완성

---

## TODO 목록

### T1-01: Docker 환경 구성 (docker-compose.yml)

- **담당**: 멤버 A (서버 개발자)
- **설명**: MySQL 8.x + Redis 7.x 로컬 개발 환경 구성
- **참조 문서**: [docs/architecture/system-overview.md](../docs/architecture/system-overview.md) §4
- **완료 조건**:
  - [ ] docker-compose.yml 작성 완료
  - [ ] `docker-compose up -d`로 MySQL, Redis 정상 기동
  - [ ] MySQL 포트 3306, Redis 포트 6379 접속 확인
  - [ ] 환경 변수 파일(.env.example) 작성

---

### T1-02: MySQL 스키마 구현 (sql/)

- **담당**: 멤버 B (서버 개발자)
- **설명**: 6개 테이블 DDL, 인덱스, 파티셔닝 구현
- **참조 문서**: [docs/database/schema-design.md](../docs/database/schema-design.md)
- **완료 조건**:
  - [ ] sql/schema.sql — users, equipment, sessions, daily_stats, web_sessions, access_log 생성
  - [ ] sql/indexes.sql — 모든 복합 인덱스 정의
  - [ ] sql/triggers.sql — daily_stats 자동 갱신 트리거
  - [ ] sql/init.sql — 기구 20대 + 관리자 계정 시드 데이터
  - [ ] sessions 테이블 월별 파티셔닝 정상 동작 확인

---

### T1-03: C 서버 프로젝트 초기화

- **담당**: 멤버 A
- **설명**: 디렉토리 구조 생성, Makefile, 설정 파싱
- **참조 문서**: [docs/c-server/server-design.md](../docs/c-server/server-design.md) §1, [docs/project-structure/file-structure.md](../docs/project-structure/file-structure.md) §3.1
- **완료 조건**:
  - [ ] server/src/, server/include/, server/config/ 디렉토리 생성
  - [ ] Makefile 작성 (GCC, 필요 라이브러리 링크)
  - [ ] config.c/config.h — server.conf 파싱 구현
  - [ ] main.c — 기본 구조 (초기화 → 서버 시작)
  - [ ] `make` 명령으로 빌드 성공

---

### T1-04: C 서버 MySQL Connection Pool 구현

- **담당**: 멤버 A
- **설명**: libmysqlclient 기반 Connection Pool
- **참조 문서**: [docs/database/schema-design.md](../docs/database/schema-design.md) §6, [docs/c-server/server-design.md](../docs/c-server/server-design.md) §3.2
- **완료 조건**:
  - [ ] db_pool.c/db_pool.h — Pool 초기화/획득/반환/종료
  - [ ] pthread_mutex 동기화 구현
  - [ ] 최소 10, 최대 50 커넥션 설정
  - [ ] 풀에서 커넥션 획득 → 쿼리 실행 → 반환 테스트 통과

---

### T1-05: C 서버 기본 TCP 소켓 구현

- **담당**: 멤버 A
- **설명**: TCP 리스너, 클라이언트 수락, 스레드 생성
- **참조 문서**: [docs/c-server/server-design.md](../docs/c-server/server-design.md) §2
- **완료 조건**:
  - [ ] tcp_server.c — socket, bind, listen, accept 구현
  - [ ] 클라이언트 연결 시 pthread_create로 핸들러 스레드 생성
  - [ ] tcp_handler.c — 4바이트 길이 헤더 + JSON 페이로드 수신
  - [ ] 기본 연결/수신 테스트 통과 (telnet 또는 간단한 클라이언트)

---

### T1-06: 공유 프로토콜 모듈 구현 (shared/)

- **담당**: 멤버 C (기기/풀스택)
- **설명**: C 서버와 IoT 기기가 공유하는 프로토콜 코드
- **참조 문서**: [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §1, [docs/project-structure/file-structure.md](../docs/project-structure/file-structure.md) §2.1
- **완료 조건**:
  - [ ] shared/protocol.h — 메시지 타입 상수, JSON 키 매크로
  - [ ] shared/protocol.c — JSON 빌드/파싱 함수 (cJSON 기반)
  - [ ] shared/common.h — 에러 코드, 공통 타입
  - [ ] server/Makefile, device/Makefile에서 shared/ 참조 가능

---

### T1-07: IoT 기기 시뮬레이터 프로젝트 초기화

- **담당**: 멤버 C
- **설명**: 기기 프로젝트 구조 생성, Makefile, 기본 TCP 연결
- **참조 문서**: [docs/iot-device/device-design.md](../docs/iot-device/device-design.md) §1, [docs/project-structure/file-structure.md](../docs/project-structure/file-structure.md) §3.2
- **완료 조건**:
  - [ ] device/src/, device/include/ 디렉토리 생성
  - [ ] Makefile 작성 (ncurses, cJSON 링크)
  - [ ] config.c — 서버 IP, 포트, device_id 설정
  - [ ] network.c — TCP 연결, 메시지 송수신 기본 구현
  - [ ] 서버에 TCP 연결 + 메시지 송수신 테스트 통과

---

### T1-08: IoT 기기 ncurses 기본 UI

- **담당**: 멤버 C
- **설명**: 로그인 화면 기본 레이아웃 구현
- **참조 문서**: [docs/iot-device/device-design.md](../docs/iot-device/device-design.md) §2
- **완료 조건**:
  - [ ] display.c — ncurses 초기화/종료
  - [ ] 로그인 화면 레이아웃 (전화번호 입력 필드)
  - [ ] input.c — 전화번호 입력 처리 (숫자만, 형식 검증)
  - [ ] 화면 전환 프레임워크 (상태 머신 기반)

---

### T1-09: Next.js 프로젝트 초기화

- **담당**: 멤버 D (웹 개발자)
- **설명**: Next.js 16 프로젝트 생성, Tailwind, Prisma 설정
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §1
- **완료 조건**:
  - [ ] `npx create-next-app` 실행, App Router 설정
  - [ ] tailwind.config.js — 다크 테마 팔레트 설정
  - [ ] jsconfig.json — 경로 alias 설정 (`@/` → `src/`)
  - [ ] 기본 layout.js — Header/Footer 컴포넌트 포함
  - [ ] `npm run dev`로 정상 기동 확인

---

### T1-10: Prisma 스키마 정의 및 마이그레이션

- **담당**: 멤버 D
- **설명**: Prisma 스키마 정의, DB 연동 확인
- **참조 문서**: [docs/database/schema-design.md](../docs/database/schema-design.md) §7
- **완료 조건**:
  - [ ] prisma/schema.prisma — 6개 모델 정의
  - [ ] .env에 DATABASE_URL 설정
  - [ ] `npx prisma db pull` 또는 `npx prisma db push` 정상 동작
  - [ ] Prisma Client 생성 확인

---

### T1-11: Redis 클라이언트 설정 (Next.js)

- **담당**: 멤버 D
- **설명**: ioredis 싱글턴 클라이언트, 기본 연동
- **참조 문서**: [docs/database/redis-design.md](../docs/database/redis-design.md)
- **완료 조건**:
  - [ ] lib/redis.js — ioredis 싱글턴 인스턴스
  - [ ] lib/prisma.js — Prisma 싱글턴 인스턴스
  - [ ] Redis 연결 테스트 (SET/GET)

---

### T1-12: 공통 유틸리티 (Next.js)

- **담당**: 멤버 D
- **설명**: API 응답 헬퍼, 입력 검증 스키마
- **참조 문서**: [docs/web/web-design.md](../docs/web/web-design.md) §1, [docs/security/security-design.md](../docs/security/security-design.md) §4
- **완료 조건**:
  - [ ] lib/response.js — success/error 응답 빌더 함수
  - [ ] lib/validation.js — zod 스키마 (전화번호, 이메일, 날짜)
  - [ ] constants/index.js — 공통 상수 (세션 TTL, Rate Limit 값 등)

---

### T1-13: C 서버 암호화 유틸리티

- **담당**: 멤버 B
- **설명**: SHA-256, bcrypt 래퍼 함수
- **참조 문서**: [docs/security/security-design.md](../docs/security/security-design.md) §1
- **완료 조건**:
  - [ ] crypto_utils.c/h — SHA-256 해시 함수 (OpenSSL)
  - [ ] crypto_utils.c/h — bcrypt 해시 생성/검증
  - [ ] 단위 테스트: 해시 생성 → 검증 통과

---

### T1-14: C 서버 로깅 모듈

- **담당**: 멤버 B
- **설명**: 레벨별 로깅 (INFO, WARN, ERROR)
- **참조 문서**: [docs/c-server/server-design.md](../docs/c-server/server-design.md) §1
- **완료 조건**:
  - [ ] logger.c/h — LOG_INFO, LOG_WARN, LOG_ERROR 매크로
  - [ ] 타임스탬프 + 파일명 + 라인번호 포함
  - [ ] stdout/파일 출력 지원

---

### T1-15: C 서버 기본 DB CRUD 구현

- **담당**: 멤버 B
- **설명**: users, equipment 테이블 기본 조회/삽입
- **참조 문서**: [docs/database/schema-design.md](../docs/database/schema-design.md) §3, [docs/c-server/server-design.md](../docs/c-server/server-design.md) §4
- **완료 조건**:
  - [ ] db.c/h — Prepared Statement 기반 CRUD 함수
  - [ ] users: phone_hash로 조회, 신규 INSERT
  - [ ] equipment: 전체 조회, 상태 UPDATE
  - [ ] 빌드 + DB 연동 테스트 통과

---

### T1-16: Git 저장소 + 브랜치 전략 확정

- **담당**: 전원
- **설명**: GitHub 저장소 세팅, 브랜치 전략 합의
- **참조 문서**: 없음 (팀 합의 사항)
- **완료 조건**:
  - [ ] GitHub 저장소 생성
  - [ ] .gitignore 작성 (node_modules, .env, *.o, 빌드 산출물)
  - [ ] 브랜치 전략 확정 (main, develop, feature/* 등)
  - [ ] 전원 clone + push 테스트 통과
