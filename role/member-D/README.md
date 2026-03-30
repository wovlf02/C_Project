# 멤버 D — 웹 개발자

> 역할: Next.js 웹 앱 전반 (인증, API, 프론트엔드, 관리자)  
> 주요 기술: Next.js 16, JavaScript, Prisma, Redis(ioredis), Tailwind CSS, Recharts

---

## 1. 담당 영역

| 영역 | 설명 |
|------|------|
| 프로젝트 초기화 | Next.js App Router 구조, 패키지, Tailwind 설정 |
| Prisma ORM | schema.prisma 정의, 마이그레이션, DB 연결 |
| Redis 클라이언트 | ioredis 연결, 웹 세션 CRUD |
| 인증 시스템 | 회원가입/로그인/로그아웃, 쿠키 세션, 이메일 인증 |
| Rate Limiting | API 요청 제한 미들웨어 |
| REST API | 운동 기록, 사용자, 기구, 관리자 API 라우트 |
| 프론트엔드 | 대시보드, 기록, 차트, 프로필, 관리자 페이지 |
| 차트/시각화 | Recharts 기반 운동 통계 차트 |
| 관리자 기능 | 사용자/기구 관리, 최근 세션, 시스템 상태 |
| 보안 | XSS/CSRF 방어, 보안 헤더, 입력 검증 (zod) |
| 개인정보 | 개인정보처리방침 페이지, 계정 탈퇴, 프로필 마스킹 |

---

## 2. 담당 TODO 목록

### Phase 1 (1~2주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T1-09 | Next.js 프로젝트 초기화 | [docs/web/web-design.md](../../docs/web/web-design.md) |
| T1-10 | Prisma 스키마 + 마이그레이션 | [docs/database/schema-design.md](../../docs/database/schema-design.md) |
| T1-11 | Redis 클라이언트 (ioredis) | [docs/database/redis-design.md](../../docs/database/redis-design.md) |
| T1-12 | 공통 레이아웃, Tailwind 설정 | [docs/web/web-design.md](../../docs/web/web-design.md) |

### Phase 2 (3~5주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T2-17 | 웹 회원가입 (이메일 인증 포함) | [docs/web/web-design.md](../../docs/web/web-design.md) |
| T2-18 | 웹 로그인/로그아웃 | [docs/web/web-design.md](../../docs/web/web-design.md) |
| T2-19 | 웹 세션 관리 (미들웨어) | [docs/c-server/session-design.md](../../docs/c-server/session-design.md) |
| T2-20 | 운동 기록 API + 페이지 | [docs/web/web-design.md](../../docs/web/web-design.md) |
| T2-21 | Rate Limiting 미들웨어 | [docs/security/security-design.md](../../docs/security/security-design.md) |
| T2-22 | 차트 컴포넌트 (Recharts) | [docs/web/web-design.md](../../docs/web/web-design.md) |

### Phase 3 (5~6주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T3-04 | 관리자 대시보드 | [docs/web/web-design.md](../../docs/web/web-design.md) |
| T3-05 | 관리자 사용자/기구 관리 API | [docs/web/web-design.md](../../docs/web/web-design.md) |
| T3-06 | 관리자 최근 세션 조회 | [docs/web/web-design.md](../../docs/web/web-design.md) |
| T3-07 | Web 기구 현황 페이지 (with C) | [docs/web/web-design.md](../../docs/web/web-design.md) |
| T3-08 | WebSocket 실시간 연동 (with C) | [docs/web/web-design.md](../../docs/web/web-design.md) |
| T3-09 | E2E 테스트 | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |
| T3-10 | 프론트 ↔ API 연동 검증 | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |

### Phase 4 (6~7주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T4-01 | 입력값 검증 강화 (웹 측, zod) | [docs/security/security-design.md](../../docs/security/security-design.md) |
| T4-03 | XSS / CSRF 방어 | [docs/security/security-design.md](../../docs/security/security-design.md) |
| T4-04 | HTTPS 설정 + 보안 헤더 | [docs/security/security-design.md](../../docs/security/security-design.md) |
| T4-06 | access_log 기록 (with A) | [docs/security/security-design.md](../../docs/security/security-design.md) |
| T4-07 | 계정 탈퇴 기능 | [docs/legal/legal-compliance.md](../../docs/legal/legal-compliance.md) |
| T4-08 | 프로필 마스킹 처리 | [docs/security/security-design.md](../../docs/security/security-design.md) |
| T4-09 | 개인정보처리방침 페이지 | [docs/legal/legal-compliance.md](../../docs/legal/legal-compliance.md) |
| T4-10 | 보안 감사 (with B) | [docs/security/security-design.md](../../docs/security/security-design.md) |

### Phase 5 (7~8주차)

| TODO | 제목 | 참조 문서 |
|------|------|-----------|
| T5-01 | 단위 테스트 (Web 부분) | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |
| T5-04 | UI 폴리싱 (Web 페이지) | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |
| T5-05 | 최종 버그 수정 | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |
| T5-06 | 시연 환경 구성 (with A) | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |
| T5-08 | README, 발표 자료 작성 | [docs/testing/test-demo-plan.md](../../docs/testing/test-demo-plan.md) |

---

## 3. 협업 시점

| 시점 | 협업 대상 | 내용 |
|------|-----------|------|
| Phase 1, T1-10 완료 후 | 멤버 B | SQL DDL과 Prisma 스키마 컬럼/타입 일치 확인 |
| Phase 1, T1-11 완료 후 | 멤버 B | Redis 키 패턴 합의 (session:, equipment:, rate_limit:) |
| Phase 2, T2-17~T2-19 | 멤버 B | 비밀번호 해시 방식(bcrypt) 일관성, 세션 Redis 구조 합의 |
| Phase 3, T3-03 | 멤버 B | Redis equipment 캐시 읽기 형식 확인 |
| Phase 3, T3-07, T3-08 | 멤버 C | 기구 현황 페이지 분담 (D: 레이아웃/API, C: WebSocket 훅) |
| Phase 3, T3-11 | 멤버 B | daily_stats 갱신 확인 (C 서버 UPSERT → Prisma 조회) |
| Phase 4, T4-06 | 멤버 A | access_log 테이블 기록 (C: INSERT, Next.js: INSERT) |
| Phase 4, T4-10 | 멤버 B | C/Web 양쪽 보안 감사 공동 진행 |
| Phase 5, T5-06 | 멤버 A | Docker Compose, 시연 환경 명령어 정리 |
| Phase 5, T5-07 | 전원 | 시연 리허설 (web 브라우저 시연 담당) |

---

## 4. 주요 산출물

```
web/src/app/layout.js
web/src/app/page.js (대시보드)
web/src/app/login/page.js
web/src/app/register/page.js
web/src/app/records/page.js
web/src/app/equipment/page.js (with C)
web/src/app/profile/page.js
web/src/app/admin/page.js
web/src/app/admin/users/page.js
web/src/app/admin/equipment/page.js
web/src/app/privacy/page.js
web/src/app/api/auth/login/route.js
web/src/app/api/auth/register/route.js
web/src/app/api/auth/logout/route.js
web/src/app/api/auth/verify-email/route.js
web/src/app/api/records/route.js
web/src/app/api/equipment/route.js
web/src/app/api/user/route.js
web/src/app/api/admin/*.js
web/src/lib/prisma.js
web/src/lib/redis.js
web/src/lib/session.js
web/src/lib/email.js
web/src/middleware.js
web/src/components/common/*.js
web/src/components/auth/*.js
web/src/components/records/*.js
web/src/components/charts/*.js
web/src/components/admin/*.js
web/prisma/schema.prisma
```
