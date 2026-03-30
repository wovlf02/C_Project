# 설계 문서 인덱스

> 최종 수정: 2026-03-30

---

## 문서 구조

```
docs/
├── README.md                              ← 현재 파일 (인덱스)
├── architecture/
│   ├── system-overview.md                 # 시스템 아키텍처 전체 구성
│   └── communication-flow.md              # 통신 흐름 (TCP, REST, WebSocket)
├── database/
│   ├── schema-design.md                   # DB 테이블/인덱스/파티셔닝 설계
│   └── redis-design.md                    # Redis 키 설계 및 접근 패턴
├── c-server/
│   ├── server-design.md                   # C TCP 서버 모듈/스레드/흐름 설계
│   ├── session-design.md                  # 세션 관리 및 자동 로그아웃 설계
│   └── calorie-design.md                  # 칼로리 계산 및 센서 설계
├── iot-device/
│   └── device-design.md                   # IoT 기기 시뮬레이터 설계
├── web/
│   └── web-design.md                      # Next.js 웹 앱 설계
├── security/
│   └── security-design.md                 # 보안 설계
├── legal/
│   └── legal-compliance.md                # 법률 준수 설계
├── project-structure/
│   └── file-structure.md                  # 파일 구조 및 모듈화 설계
└── testing/
    └── test-demo-plan.md                  # 테스트 및 시연 계획
```

---

## 문서별 요약

| 문서 | 핵심 내용 |
|------|-----------|
| system-overview | 3-Tier 아키텍처, 구성 요소, 데이터 흐름, 배포 구조 |
| communication-flow | TCP 프로토콜, REST API, WebSocket 상세 |
| schema-design | 6개 테이블, 인덱스 전략, 파티셔닝, Prisma 스키마 |
| redis-design | 7가지 키 패턴, R/W 패턴, Fallback 전략 |
| server-design | 14개 모듈, 스레드 모델, 핵심 처리 흐름 |
| session-design | 세션 생명주기, 자동 로그아웃 5가지, 워치독 설계 |
| calorie-design | 칼로리 공식, 강도 계수, 실시간 갱신 |
| device-design | 8개 모듈, ncurses UI, 센서 시뮬레이션, Encrypted Storage |
| web-design | 페이지 구조, 인증 흐름, WebSocket, 컴포넌트 트리 |
| security-design | 개인정보 보호, 인증/인가, 공격 방어, Rate Limit |
| legal-compliance | 개인정보 보호법, 정보통신망법, 특허 요소 |
| file-structure | 전체 디렉토리 구조, 코드 중복 제거, 400라인 제한 |
| test-demo-plan | 테스트 전략, 시연 시나리오, 환경 준비 |
