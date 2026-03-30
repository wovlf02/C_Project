# 데이터베이스 설계

> 최종 수정: 2026-03-30

---

## 1. 설계 원칙

- **InnoDB 엔진 전용**: 트랜잭션, 외래키, Row-level Lock
- **UTF-8mb4 문자셋**: 한국어 완전 지원
- **월별 RANGE 파티셔닝**: sessions 테이블
- **복합 인덱스**: 실제 쿼리 패턴(user_id + date 범위)에 최적화
- **사전 집계 테이블**: daily_stats로 반복 집계 연산 제거
- **Connection Pool**: C 서버(libmysqlclient) + Next.js(Prisma)

---

## 2. ER 다이어그램

```
┌─────────────┐        ┌──────────────────┐
│   users      │        │   equipment       │
│─────────────│        │──────────────────│
│ PK id        │◀──┐   │ PK id             │
│ phone_hash   │   │   │ name              │
│ phone_bcrypt │   │   │ type              │
│ email_encrypt│   │   │ calorie_per_min   │
│ role         │   │   │ status            │
│ consent_at   │   ├──│ FK current_user_id│
│ is_deleted   │   │   │ api_key_hash      │
└──────┬──────┘   │   └────────┬──────────┘
       │          │            │
       │ 1:N      │     1:N    │
       ▼          │            ▼
┌──────────────────┐  ┌──────────────────┐
│   sessions        │  │   web_sessions    │
│──────────────────│  │──────────────────│
│ PK (id,start_time)│  │ PK id             │
│ FK user_id        │──│ session_id (UUID) │
│ FK equipment_id   │  │ FK user_id        │
│ start_time        │  │ expires_at        │
│ end_time          │  └──────────────────┘
│ calories          │
│ auto_logout       │
│ auto_logout_reason│
└──────┬───────────┘
       │
       │ 집계
       ▼
┌──────────────────┐  ┌──────────────────┐
│   daily_stats     │  │   access_log      │
│──────────────────│  │──────────────────│
│ PK id             │  │ PK id             │
│ FK user_id        │  │ FK user_id        │
│ stat_date         │  │ action            │
│ total_sessions    │  │ detail            │
│ total_calories    │  │ ip_address        │
│ cardio_calories   │  │ created_at        │
│ strength_calories │  └──────────────────┘
└──────────────────┘
```

---

## 3. 테이블 인덱스 전략

### 3.1 users

| 인덱스 | 컬럼 | 용도 |
|--------|------|------|
| idx_phone_hash | phone_hash (UNIQUE) | 전화번호 로그인 조회 |
| idx_role | role | 관리자 필터링 |
| idx_is_deleted | is_deleted | 탈퇴 사용자 제외 |

### 3.2 equipment

| 인덱스 | 컬럼 | 용도 |
|--------|------|------|
| idx_status | status | 상태별 필터링 |
| idx_type_status | (type, status) | 유형+상태 복합 조회 |
| idx_current_user | current_user_id | 현재 사용자 조회 |
| idx_api_key_hash | api_key_hash | 기기 인증 |

### 3.3 sessions (파티셔닝 적용)

| 인덱스 | 컬럼 | 용도 |
|--------|------|------|
| PRIMARY | (id, start_time) | 파티셔닝 키 포함 PK |
| idx_user_start | (user_id, start_time) | 사용자별 기록 조회 |
| idx_user_date | (user_id, start_time, end_time) | 기간별 조회 |
| idx_equipment_start | (equipment_id, start_time) | 기구별 사용 이력 |
| idx_active_sessions | (end_time, user_id) | 진행 중 세션 조회 |

### 3.4 daily_stats

| 인덱스 | 컬럼 | 용도 |
|--------|------|------|
| idx_user_date | (user_id, stat_date) UNIQUE | 일일 통계 조회 (핵심) |
| idx_stat_date | stat_date | 날짜별 전체 통계 |
| idx_user_date_range | (user_id, stat_date) | 주간/월간/연간 범위 조회 |

---

## 4. 파티셔닝 전략

### sessions 테이블 — 월별 RANGE

```sql
PARTITION BY RANGE (YEAR(start_time) * 100 + MONTH(start_time))
-- p202601, p202602, ..., p202612, p_future
```

- **효과**: 월별 조회 시 파티션 프루닝 → 전체 테이블 스캔 방지
- **관리**: 분기마다 새 파티션 추가 (ALTER TABLE ... ADD PARTITION)

---

## 5. daily_stats 갱신 전략

| 갱신 시점 | 트리거 | 처리 |
|-----------|--------|------|
| 세션 종료 | C 서버 (LOGOUT/자동 로그아웃) | UPSERT: INSERT ... ON DUPLICATE KEY UPDATE |
| 데이터 정합성 배치 | cron (매일 02:00) | 전일 daily_stats를 sessions 기반 재계산 |

### UPSERT 쿼리

```sql
INSERT INTO daily_stats (user_id, stat_date, total_sessions, total_duration_sec, total_calories, ...)
VALUES (?, CURDATE(), 1, ?, ?, ...)
ON DUPLICATE KEY UPDATE
  total_sessions = total_sessions + 1,
  total_duration_sec = total_duration_sec + VALUES(total_duration_sec),
  total_calories = total_calories + VALUES(total_calories),
  updated_at = NOW();
```

---

## 6. Connection Pool 설계

| 서버 | 라이브러리 | 최소 | 최대 | 비고 |
|------|-----------|------|------|------|
| C TCP 서버 | libmysqlclient (자체 풀) | 10 | 50 | pthread_mutex로 동기화 |
| Next.js | Prisma | 10 | 50 | connection_limit 설정 |
| **합산** | | **20** | **100** | NFR-P07 충족 |

---

## 7. Prisma 스키마 핵심 모델

```prisma
model User {
  id             BigInt    @id @default(autoincrement()) @db.UnsignedBigInt
  phoneHash      String    @unique @map("phone_hash") @db.Char(64)
  phoneBcrypt    String    @map("phone_bcrypt") @db.Char(60)
  emailEncrypted Bytes     @map("email_encrypted") @db.VarBinary(512)
  role           UserRole  @default(user)
  consentAt      DateTime  @map("consent_at")
  isDeleted      Int       @default(0) @map("is_deleted") @db.TinyInt
  // ... relations
  @@map("users")
}
```

> 전체 Prisma 스키마는 `web/prisma/schema.prisma`에 구현
