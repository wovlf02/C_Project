# Next.js 웹 애플리케이션 설계

> 최종 수정: 2026-03-30

---

## 1. 프로젝트 구조

```
web/
├── src/
│   ├── app/                          # App Router (페이지)
│   │   ├── layout.js                 # 루트 레이아웃 (다크 테마, 네비게이션)
│   │   ├── page.js                   # 메인 페이지 (기구 현황)
│   │   ├── login/
│   │   │   └── page.js               # 로그인 페이지
│   │   ├── register/
│   │   │   └── page.js               # 회원가입 (이메일 인증)
│   │   ├── dashboard/
│   │   │   └── page.js               # 사용자 대시보드
│   │   ├── records/
│   │   │   ├── page.js               # 운동 기록 메인
│   │   │   └── [period]/
│   │   │       └── page.js           # 기간별 조회 (daily/weekly/monthly/yearly)
│   │   ├── admin/
│   │   │   ├── page.js               # 관리자 대시보드
│   │   │   ├── equipment/
│   │   │   │   └── page.js           # 기구 관리 (CRUD)
│   │   │   ├── sessions/
│   │   │   │   └── page.js           # 활성 세션 관리
│   │   │   └── logs/
│   │   │       └── page.js           # 접속 기록
│   │   ├── privacy-policy/
│   │   │   └── page.js               # 개인정보 처리방침
│   │   └── api/                      # API Routes
│   │       ├── auth/
│   │       │   ├── login/route.js
│   │       │   ├── logout/route.js
│   │       │   ├── me/route.js
│   │       │   └── email-verify/
│   │       │       ├── send/route.js
│   │       │       └── confirm/route.js
│   │       ├── records/
│   │       │   ├── daily/route.js
│   │       │   ├── weekly/route.js
│   │       │   ├── monthly/route.js
│   │       │   ├── yearly/route.js
│   │       │   ├── custom/route.js
│   │       │   └── summary/route.js
│   │       ├── equipment/
│   │       │   ├── route.js           # GET 전체 목록
│   │       │   ├── [id]/route.js      # GET 상세
│   │       │   └── occupancy/route.js # GET 혼잡도
│   │       ├── user/
│   │       │   ├── profile/route.js
│   │       │   └── account/route.js
│   │       └── admin/
│   │           ├── equipment/
│   │           │   ├── route.js       # GET, POST
│   │           │   └── [id]/
│   │           │       ├── route.js   # PUT, DELETE
│   │           │       └── status/route.js # PUT 상태 변경
│   │           ├── sessions/
│   │           │   ├── active/route.js
│   │           │   └── [id]/
│   │           │       └── force-logout/route.js
│   │           ├── gym/
│   │           │   └── close/route.js
│   │           ├── stats/
│   │           │   └── daily/route.js
│   │           └── logs/route.js
│   │
│   ├── components/                    # 재사용 UI 컴포넌트
│   │   ├── common/
│   │   │   ├── Header.js             # 공통 헤더/네비게이션
│   │   │   ├── Footer.js             # 공통 푸터
│   │   │   ├── LoadingSpinner.js     # 로딩 인디케이터
│   │   │   ├── ErrorMessage.js       # 에러 메시지 표시
│   │   │   └── Modal.js              # 모달 컴포넌트
│   │   ├── auth/
│   │   │   ├── LoginForm.js          # 전화번호 로그인 폼
│   │   │   ├── RegisterForm.js       # 회원가입 폼
│   │   │   └── EmailVerify.js        # 이메일 인증 코드 입력
│   │   ├── equipment/
│   │   │   ├── EquipmentGrid.js      # 기구 카드 그리드
│   │   │   ├── EquipmentCard.js      # 개별 기구 카드
│   │   │   ├── EquipmentDetail.js    # 기구 상세 모달
│   │   │   ├── OccupancyBar.js       # 혼잡도 바
│   │   │   └── TypeFilter.js         # 유형별 필터
│   │   ├── records/
│   │   │   ├── RecordTable.js        # 운동 기록 테이블
│   │   │   ├── PeriodSelector.js     # 기간 선택 UI
│   │   │   ├── SummaryCard.js        # 요약 카드
│   │   │   └── FilterPanel.js        # 기구 유형 필터
│   │   ├── charts/
│   │   │   ├── CalorieChart.js       # 칼로리 추이 차트
│   │   │   ├── DurationChart.js      # 운동 시간 차트
│   │   │   └── ChartWrapper.js       # Recharts 공통 래퍼
│   │   └── admin/
│   │       ├── EquipmentForm.js      # 기구 추가/수정 폼
│   │       ├── SessionTable.js       # 활성 세션 테이블
│   │       └── LogTable.js           # 접속 기록 테이블
│   │
│   ├── lib/                           # 서버 유틸리티
│   │   ├── prisma.js                 # Prisma 싱글턴 클라이언트
│   │   ├── redis.js                  # ioredis 싱글턴 클라이언트
│   │   ├── session.js                # 세션 CRUD (Redis + MySQL 백업)
│   │   ├── auth-middleware.js        # 세션 인증 미들웨어
│   │   ├── admin-middleware.js       # 관리자 권한 미들웨어
│   │   ├── rate-limit.js             # Rate Limit 유틸리티
│   │   ├── email.js                  # nodemailer 설정 + 발송
│   │   ├── crypto.js                 # AES-256 암복호화, SHA-256
│   │   ├── validation.js             # zod 스키마 (입력 검증)
│   │   ├── response.js               # API 공통 응답 헬퍼
│   │   └── ws-server.js              # WebSocket 서버 + Redis 폴링
│   │
│   ├── hooks/                         # 클라이언트 커스텀 훅
│   │   ├── useAuth.js                # 인증 상태 관리
│   │   ├── useEquipment.js           # 기구 실시간 상태 (WS)
│   │   └── useRecords.js             # 운동 기록 조회
│   │
│   └── constants/
│       └── index.js                   # 공통 상수 정의
│
├── prisma/
│   └── schema.prisma                  # Prisma 스키마
├── public/                            # 정적 파일
├── server.js                          # 커스텀 서버 (Next.js + WS)
├── package.json
├── tailwind.config.js
├── next.config.js
└── jsconfig.json
```

---

## 2. 페이지 설계

### 2.1 라우트 맵

| 경로 | 페이지 | 인증 | SSR |
|------|--------|------|-----|
| / | 기구 현황 (메인) | 불필요 | SSR + WS |
| /login | 로그인 | 불필요 | SSR |
| /register | 회원가입 | 불필요 | SSR |
| /dashboard | 사용자 대시보드 | 필수 | SSR |
| /records | 운동 기록 조회 | 필수 | SSR |
| /records/[period] | 기간별 상세 | 필수 | SSR |
| /admin | 관리자 대시보드 | admin | SSR |
| /admin/equipment | 기구 관리 | admin | SSR |
| /admin/sessions | 세션 관리 | admin | SSR |
| /admin/logs | 접속 기록 | admin | SSR |
| /privacy-policy | 개인정보 처리방침 | 불필요 | SSG |

### 2.2 레이아웃 구조

```
┌─────────────────────────────────────┐
│  Header (로고, 네비게이션, 로그인)    │
├─────────────────────────────────────┤
│                                     │
│  Main Content (페이지별)             │
│                                     │
├─────────────────────────────────────┤
│  Footer (저작권, 개인정보 처리방침)   │
└─────────────────────────────────────┘
```

---

## 3. 인증 흐름

### 3.1 로그인

```
사용자: 전화번호 입력 → POST /api/auth/login
서버: SHA-256 해시 → DB 조회 → bcrypt 검증
      → Redis 세션 생성 (session:{uuid})
      → MySQL web_sessions 백업
      → Set-Cookie: session_id (httpOnly, secure, sameSite=strict, 14일)
클라이언트: 리다이렉트 /dashboard
```

### 3.2 회원가입 (이메일 인증)

```
1단계: 전화번호+이메일 입력 → POST /api/auth/email-verify/send
       → 6자리 코드 이메일 발송, Redis 저장 (5분 TTL)

2단계: 코드 입력 → POST /api/auth/email-verify/confirm
       → Redis 코드 검증 → users INSERT → 세션 발급 → 자동 로그인
```

### 3.3 세션 미들웨어 (auth-middleware.js)

```javascript
// 모든 인증 필요 API에서 호출
export async function withAuth(request) {
  // 1. 쿠키에서 session_id 추출
  // 2. Redis 조회 (session:{session_id})
  // 3. 세션 존재 → { user_id, role } 반환
  // 4. 미존재 → MySQL web_sessions fallback
  // 5. 둘 다 실패 → 401 Unauthorized
}
```

---

## 4. WebSocket 설계

### 4.1 커스텀 서버 (server.js)

```javascript
// Next.js + WebSocket 통합 서버
const server = createServer(handler);
const wss = new WebSocketServer({ server });

// 2~3초 주기로 Redis equipment:all 폴링
setInterval(async () => {
  const data = await redis.get('equipment:all');
  if (data) {
    wss.clients.forEach(client => {
      client.send(JSON.stringify({
        type: 'EQUIPMENT_STATUS',
        data: JSON.parse(data),
        timestamp: new Date().toISOString()
      }));
    });
  }
}, 3000);
```

### 4.2 클라이언트 훅 (useEquipment.js)

```javascript
export function useEquipment() {
  // WebSocket 연결 관리
  // 실시간 기구 상태 수신
  // 연결 해제 시 자동 재연결
  // 상태: equipment 목록, occupancy
}
```

---

## 5. 스타일링

- **Tailwind CSS** + 다크 테마 기본
- **반응형**: 모바일/태블릿/데스크톱
- **차트**: Recharts (막대/라인 차트)
- **컬러 팔레트**: 다크 배경 (#1a1a2e) + 포인트 (#e94560)
