# 보안 설계

> 최종 수정: 2026-03-30

---

## 1. 개인정보 보호

### 1.1 전화번호 처리

| 단계 | 처리 |
|------|------|
| 저장 | bcrypt(cost=12)로 해시 저장 (phone_bcrypt) |
| 조회 | SHA-256 해시 인덱스 (phone_hash)로 검색 |
| 표시 | 마스킹 처리: `010-****-1234` |
| 검증 | bcrypt_verify(입력값, phone_bcrypt) |

### 1.2 이메일 처리

| 단계 | 처리 |
|------|------|
| 저장 | AES-256 암호화 (email_encrypted) |
| 조회 | 복호화 후 사용 (발송 시에만) |
| 키 관리 | 환경 변수 EMAIL_ENCRYPTION_KEY (32바이트 hex) |

### 1.3 최소 수집 원칙

- 수집 항목: 전화번호, 이메일 (2개만)
- 용도: 사용자 식별, 운동 기록 관리, 계정 만료 사전 안내
- 보유 기간: 마지막 이용일로부터 1년

---

## 2. 인증/인가

### 2.1 웹 세션 보안

| 항목 | 설정 |
|------|------|
| 세션 ID 생성 | crypto.randomUUID() (UUID v4) |
| 쿠키 속성 | httpOnly, secure, sameSite=strict |
| 만료 | 14일 |
| 저장 | Redis (메인) + MySQL web_sessions (백업) |
| 무효화 | 로그아웃 시 Redis 즉시 삭제 |

### 2.2 기기 인증

| 항목 | 설정 |
|------|------|
| 인증 방식 | 기기별 사전 발급 API Key (SHA-256 해시 저장) |
| 통신 범위 | 헬스장 내부 WiFi LAN 전용 |
| 세션 저장 | 기기에 인증 정보 영구 저장 금지 |

### 2.3 관리자 인가

- `users.role = 'admin'` 확인
- `/api/admin/*` 경로에 admin-middleware 적용
- 세션 미들웨어 → role 검사 → admin 아니면 403

---

## 3. 공격 방어

### 3.1 SQL Injection

| 대상 | 방어 |
|------|------|
| C 서버 | mysql_stmt_prepare / mysql_stmt_bind_param (Prepared Statement) |
| Next.js | Prisma (parameterized query 자동 적용) |

### 3.2 XSS

| 대상 | 방어 |
|------|------|
| Next.js | React JSX 자동 이스케이핑 |
| CSP 헤더 | Content-Security-Policy 설정 |

### 3.3 CSRF

| 방어 | 설명 |
|------|------|
| SameSite 쿠키 | sameSite=strict로 크로스사이트 요청 차단 |
| CSRF 토큰 | 상태 변경 요청에 이중 방어 |

### 3.4 Rate Limiting

| 대상 | 제한 | Redis 키 |
|------|------|----------|
| 로그인 API | IP당 5회/분 | rate_limit:login:{ip} |
| 이메일 코드 발송 | IP당 3회/분 + 이메일당 3회/10분 | email_verify_rate:* |
| 전체 API | IP당 100회/분 | rate_limit:api:{ip} |

### 3.5 보안 헤더

```javascript
// next.config.js headers
{
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
  'Content-Security-Policy': "default-src 'self'; ..."
}
```

---

## 4. 입력값 검증

### 4.1 전화번호

```
정규식: /^01[016789]\d{7,8}$/
서버 측 검증 필수 (클라이언트 검증은 UX 목적)
```

### 4.2 이메일

```
zod: z.string().email().max(254)
```

### 4.3 날짜 파라미터

```
zod: z.string().regex(/^\d{4}-\d{2}-\d{2}$/)
범위 제한: 과거 2년 ~ 현재
```

---

## 5. HTTPS

| 환경 | 설정 |
|------|------|
| 프로덕션 | TLS 1.2+ 필수, HTTP → 301 → HTTPS |
| 로컬 개발 | HTTP 허용 (secure 쿠키 비활성) |

---

## 6. 접속 기록 (법률 준수)

### access_log 테이블

- 모든 인증 관련 이벤트 기록
- 최소 1년 보관 (정보통신망법 제49조의2)
- 기록 항목: user_id, action, ip_address, user_agent, created_at

### 기록 대상 이벤트

| action | 설명 |
|--------|------|
| login_device | 기기 로그인 |
| logout_device | 기기 로그아웃 |
| auto_logout | 자동 로그아웃 (사유 포함) |
| login_web | 웹 로그인 |
| logout_web | 웹 로그아웃 |
| delete_account | 회원 탈퇴 |
| view_records | 운동 기록 조회 |
