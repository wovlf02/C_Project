# Phase 4: 보안 강화 및 법률 준수 (6~7주차)

> 목표: OWASP 보안 점검, 개인정보 보호법 준수 기능 완성, 접속 기록

---

## TODO 목록

### T4-01: 입력값 검증 강화 (전체)

- **담당**: 멤버 D (웹) + 멤버 B (C 서버)
- **설명**: 모든 사용자 입력에 대한 서버 측 검증 강화
- **참조 문서**: [docs/security/security-design.md](../docs/security/security-design.md) §4
- **완료 조건**:
  - [ ] C 서버: 전화번호 정규식 검증, 이메일 형식 검증
  - [ ] C 서버: JSON 페이로드 크기 제한 (8KB)
  - [ ] Next.js: 모든 API에 zod 스키마 적용
  - [ ] 날짜 파라미터 범위 제한 (과거 2년 ~ 현재)

---

### T4-02: SQL Injection 방어 확인

- **담당**: 멤버 B (C 서버) + 멤버 D (Next.js)
- **설명**: 모든 DB 쿼리가 Prepared Statement/Prisma 활용 확인
- **참조 문서**: [docs/security/security-design.md](../docs/security/security-design.md) §3.1
- **완료 조건**:
  - [ ] C 서버: 모든 쿼리 mysql_stmt_* 함수 사용 확인
  - [ ] Next.js: Prisma parameterized query 사용 확인
  - [ ] 비정상 입력(single quote, UNION 등) 주입 테스트 통과

---

### T4-03: XSS/CSRF 방어

- **담당**: 멤버 D
- **설명**: 보안 헤더 설정 + CSRF 토큰
- **참조 문서**: [docs/security/security-design.md](../docs/security/security-design.md) §3.2, §3.3, §3.5
- **완료 조건**:
  - [ ] next.config.js — 보안 헤더 설정 (CSP, X-Frame-Options 등)
  - [ ] SameSite=strict 쿠키 확인
  - [ ] 스크립트 태그 입력 테스트 → 자동 이스케이핑 확인

---

### T4-04: HTTPS 설정

- **담당**: 멤버 D
- **설명**: TLS 설정 및 HTTP 리다이렉트
- **참조 문서**: [docs/security/security-design.md](../docs/security/security-design.md) §5
- **완료 조건**:
  - [ ] 시연 환경: HTTP 허용 (로컬 개발)
  - [ ] next.config.js: Strict-Transport-Security 헤더
  - [ ] 쿠키 secure 플래그: 프로덕션에서만 활성화

---

### T4-05: 개인정보 처리방침 페이지

- **담당**: 멤버 C
- **설명**: 개인정보 처리방침 전문 웹 페이지
- **참조 문서**: [docs/legal/legal-compliance.md](../docs/legal/legal-compliance.md) §1.2
- **완료 조건**:
  - [ ] privacy-policy/page.js — 처리방침 전문 (8개 필수 항목)
  - [ ] Header에 개인정보 처리방침 링크 추가
  - [ ] 회원가입 페이지에서 처리방침 링크 표시

---

### T4-06: 접속 기록 (access_log) 전체 적용

- **담당**: 멤버 A (C 서버) + 멤버 D (Next.js)
- **설명**: 모든 인증 이벤트를 access_log에 기록
- **참조 문서**: [docs/security/security-design.md](../docs/security/security-design.md) §6, [docs/legal/legal-compliance.md](../docs/legal/legal-compliance.md) §2.1
- **완료 조건**:
  - [ ] C 서버: login_device, logout_device, auto_logout 기록
  - [ ] Next.js: login_web, logout_web, view_records, delete_account 기록
  - [ ] access_log INSERT 시 ip_address, user_agent 포함
  - [ ] 관리자 접속 기록 조회 API/페이지 동작 확인

---

### T4-07: 회원 탈퇴 기능

- **담당**: 멤버 D
- **설명**: 소프트 삭제 + 30일 후 하드 삭제
- **참조 문서**: [docs/legal/legal-compliance.md](../docs/legal/legal-compliance.md) §1.5, §1.6
- **완료 조건**:
  - [ ] api/user/account/route.js — DELETE: is_deleted=1, 세션 삭제
  - [ ] 탈퇴 사용자 로그인 차단
  - [ ] access_log에 delete_account 기록

---

### T4-08: 사용자 프로필 조회 (마스킹)

- **담당**: 멤버 D
- **설명**: 본인 정보 열람 (전화번호 마스킹)
- **참조 문서**: [docs/security/security-design.md](../docs/security/security-design.md) §1
- **완료 조건**:
  - [ ] api/user/profile/route.js — 전화번호 마스킹 (010-****-1234)
  - [ ] 이메일 복호화 후 부분 마스킹 (u***@example.com)
  - [ ] 계정 생성일, 마지막 로그인 표시

---

### T4-09: 관리자 접속 기록 조회 페이지

- **담당**: 멤버 D
- **설명**: access_log 테이블 조회 관리자 페이지
- **참조 문서**: [docs/architecture/communication-flow.md](../docs/architecture/communication-flow.md) §2.5
- **완료 조건**:
  - [ ] admin/logs/page.js — 접속 기록 테이블
  - [ ] components/admin/LogTable.js — 페이지네이션 포함
  - [ ] api/admin/logs/route.js — 날짜/액션 필터링

---

### T4-10: 보안 통합 점검

- **담당**: 전원
- **설명**: OWASP Top 10 기반 전체 보안 점검
- **참조 문서**: [docs/security/security-design.md](../docs/security/security-design.md), [docs/testing/test-demo-plan.md](../docs/testing/test-demo-plan.md) §1.3
- **완료 조건**:
  - [ ] SQL Injection 테스트 통과
  - [ ] XSS 테스트 통과
  - [ ] Rate Limit 동작 확인 (429 응답)
  - [ ] 세션 탈취 방어 확인 (만료 세션 재사용 차단)
  - [ ] 보안 헤더 전체 적용 확인
