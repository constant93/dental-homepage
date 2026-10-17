# ROADMAP — Dental AX OS

> 상태: 초안 (2026-10-17)

---

## C. Phase 1 MVP 기능 범위

원칙: **Mock Data로 끝까지 작동**하고, 실제 치과 직원이 "이거면 매일 쓰겠다"고 말할 수 있는 최소 범위.

### ① Command Center
- 상단: "오늘 확인할 것 N개" — 우선순위 순서의 할 일 목록 (클릭 → 처리 화면)
- 카드: 오늘 예약 · 오늘 신환 · 오늘 예상 매출(권한자만) / 상담 Follow-up 필요 · Recall 필요 · 미수금 확인 필요 / 재고 부족 · 발주 추천 / 오늘 내 Task
- Daily Briefing: 규칙 기반 문장, FACT/INFERENCE 구분, 데이터 부족 표시
- Role별로 다른 카드 (데스크는 매출 카드 없음)
- **제외**: 체어 가동률, 실시간 현재 매출 (실제 수납 데이터 연동 후)

### ② Patient / Consultation CRM
- 환자 목록·검색 (Mock), 환자 상세(예약·상담·연락 기록 타임라인)
- 상담 빠른 입력: 치료 종류 · 견적 · 상태 · 보류 이유(칩 클릭) · 고민 메모 · Follow-up 날짜 → **30초 안에 입력**이 목표
- Follow-up 목록: "오늘 연락할 상담 환자" + 점수 근거
- 연락 결과 기록 2클릭 (부재중 / 다시 연락 / 예약 잡힘 / 고민 중 / 거절) → 다음 Follow-up 날짜 자동 제안
- 미수금 목록 + 연락 기록

### ③ Inventory & Purchase Intelligence
- 품목 등록 (최소/적정 재고, 단위, 리드타임, 유효기간 추적 여부)
- 주간 실사 화면 (태블릿에서 품목별 숫자 입력)
- 입고 처리 (거래명세서 단가 입력 → 가격 이력 자동 기록)
- 재고 부족 · 발주 추천 · 유효기간 임박 목록
- 공급처별 가격 비교 + "이번 주문은 어디서?" 계산 (배송비·최소주문 포함, 계산 근거 표시)
- 발주서 초안 → Owner 확정 → 입고

### ④ Staff Task
- 오늘 내 Task / 역할 Task / 전체(Owner)
- 오픈·마감 체크리스트 (템플릿 → 매일 자동 생성 → 태블릿 체크)
- 규칙이 만든 Task (재고 부족 → 담당자에게 "발주 검토")
- 마감 지난 Task 강조

### 공통 기반
- 로그인 · 5개 Role · 권한 표 · Audit Log · 오류/빈 화면/로딩 처리 · 한국어 UI · Desktop/Tablet 반응형
- Synthetic seed 데이터 (환자 약 200명, 8주치 예약, 상담 40건, 품목 40개, 공급처 4곳)

---

## I. 3단계 Development Roadmap

### Phase 1 — Mock MVP (목표: 직원이 매일 열어보는 화면)
| Sprint | 내용 | 완료 기준 |
|---|---|---|
| 1 | Foundation + Command Center 골격 (아래 J) | 로그인 → Role별 Command Center가 Mock 데이터로 보임 |
| 2 | Consultation CRM + 미수금 + Follow-up 규칙 | 상담 입력 → Follow-up 목록 → 연락 기록 → Command Center 반영 |
| 3 | Inventory: 품목·실사·입고·재고부족·유효기간 | 실사 입력 → 재고 부족 카드 + 담당자 Task 자동 생성 |
| 4 | Purchasing: 공급처·가격 이력·최저 구매처·발주 | 발주 추천 → 가격 비교 → 발주 초안 → 확정 → 입고 |
| 5 | Staff Task + 체크리스트 + Briefing 다듬기 | 오픈/마감 체크리스트 매일 생성, Briefing 규칙 전체 |
| 6 | 안정화: 권한 테스트 전수, e2e, 사용성 점검 | **직원 시연 → 피드백 반영** |

Phase 1 종료 조건: Definition of Done 충족 + 원장·직원이 Mock으로 하루 업무를 시연해 보고 "실데이터면 쓰겠다" 확인.

### Phase 2 — Real Data Pilot (목표: 실제 데이터로 한 병원에서 매일 사용)
1. DentWeb 데이터 접근 방법 확인 (ARCHITECTURE.md H-2 질문)
2. 서버 위치·보안 Gate 통과 (ARCHITECTURE.md F-5)
3. CSV/Excel import (확인된 export 항목 기준) — 예약·신환·미수금
4. Recall 모듈 (대상 계산 → 우선순위 → 연락 기록)
5. Patient Communication: 치료별 안내문 템플릿 → 환자 맞춤 초안 → **원장 승인** → 인쇄/PDF
6. AI Gateway + LLM 도입(승인 시): Briefing 문장화, 상담 메모 요약
7. 2FA, 백업·복구 리허설

### Phase 3 — Intelligence (목표: 원장의 판단 시간 단축)
1. Management Analytics: 매출·신환·유입경로·동의율(담당자별)·No-show·Recall 회수율·재료비
2. "이번 달 무엇이 달라졌나" — 변화 감지 규칙 + LLM 설명 (FACT/INFERENCE)
3. AI Copilot: 자연어 질의 → 허용된 조회 도구 → 근거 포함 답변
4. SOP / Knowledge Base: 병원 매뉴얼 + 출처 인용 답변
5. SMS/카카오 발송 연동 (승인 후 발송, 수신동의·발송 규정 확인 후)
6. DentWeb 공식 연동 (공식 방법이 확인된 경우에만)

---

## J. 첫 번째 구현 Sprint (Sprint 1 — Walking Skeleton)

**목표**: 기능은 적지만 **처음부터 끝까지 연결된** 시스템. 이후 모든 Sprint는 이 골격에 붙인다.

### 사전 조건 (원장님 결정)
- [ ] 비공개 저장소 생성 (D-001) — 이름 예: `dental-ax-os`
- [ ] 이 문서들을 새 저장소 루트로 이동

### 작업 순서 (작은 단위 → 매번 test/typecheck/lint/build)
1. **프로젝트 생성**: Next.js + TypeScript strict + Tailwind + shadcn/ui, ESLint, Vitest, Playwright. 버전 확인 후 DECISIONS 기록.
2. **로컬 DB**: `docker-compose.yml`(PostgreSQL), `.env.example`, Prisma 연결.
3. **첫 migration**: `User`, `AuditLog`, `Patient`, `Appointment`, `Consultation`, `InventoryItem`, `Task`, `Insight` (Sprint 1에 필요한 필드만).
4. **Synthetic seed**: 고정 seed, 이름은 `테스트환자001` 형식, 오늘 기준 예약이 항상 생기도록 날짜 상대 생성.
5. **인증 + RBAC**: 로그인/로그아웃, 5개 Role 테스트 계정, `permissions.ts` 권한 표, `requirePermission()`, 로그인 기록 Audit.
6. **EMR Adapter 골격**: `EmrSource` interface + `mock-source.ts`, `dentweb/README.md` (UNKNOWN 목록).
7. **Insight Engine 4개 규칙**: `schedule.todaySummary`, `consultation.followUpDue`, `inventory.belowMinimum`, `task.overdue` — 각각 unit test (데이터 부족 케이스 포함).
8. **Command Center 화면**: 상단 "오늘 확인할 것 N개" + 카드 + Daily Briefing. Loading/Empty/Error/권한 없음 상태. Role별 카드 차이.
9. **테스트**: 규칙 unit test, 권한 integration test(허용/거부), e2e 1개(데스크 로그인 → 매출 카드 안 보임 / Owner 로그인 → 보임).
10. **CI**: GitHub Actions에서 lint · typecheck · test · build.
11. **문서 갱신**: CLAUDE.md 명령어, README 실행 방법.

### Sprint 1에 하지 않는 것
상담 입력 폼, 재고 입력, 발주, Task 생성 UI — 화면은 읽기 전용. (Sprint 2부터)

---

## 지금 / 나중 / 안 함

### 🟢 지금 당장 구현 (Phase 1)
| 항목 | 이유 |
|---|---|
| 비공개 저장소 · 인증 · RBAC · Audit Log | 나중에 붙이면 모든 코드를 다시 고쳐야 함 |
| Synthetic 데이터 생성기 | 실제 데이터 없이 끝까지 검증 |
| EMR Adapter interface + Mock | DentWeb 확인 전에도 개발 진행, 나중에 교체만 |
| Command Center + 규칙 기반 Briefing | 제품의 얼굴. "오늘 무엇을 해야 하나" |
| 상담 Follow-up CRM + 연락 기록 | DentWeb에 없는 데이터 → AX OS 고유 가치, 매출 직결 |
| 미수금 확인 목록 | 누락 방지, 바로 돈과 연결 |
| 재고 최소수량 · 주간 실사 · 입고 · 유효기간 | 누락(수술 중 재료 부족) 방지 |
| 공급처 가격 비교 · 가격 이력 · 발주 추천 | 비용 절감이 숫자로 보임 |
| Task · 오픈/마감 체크리스트 | 반복 업무의 시스템화, 직원 습관 형성 |

### 🟡 나중에 구현 (Phase 2–3)
| 항목 | 시점 · 조건 |
|---|---|
| DentWeb 연동 (CSV → 공식 방법) | 접근 방법 공식 확인 후 |
| 실제 환자 데이터 | 보안 Gate 통과 + 원장 승인 후 |
| Recall 모듈 (우선순위 추천) | Phase 2, 실데이터 필요 |
| Patient Communication (안내문 초안) | Phase 2, 원장 승인 흐름과 함께 |
| LLM (Briefing 문장화, 요약) | Phase 2, D-007 결정 후 |
| SMS · 카카오톡 발송 | Phase 3, 발송 규정·비용 확인 후 |
| Management Analytics · "무엇이 달라졌나" | Phase 3, 수개월 데이터 축적 후 |
| AI Copilot (자연어 질의) | Phase 3, 조회 도구가 충분히 쌓인 후 |
| SOP / Knowledge Base | Phase 3, 매뉴얼 문서화가 먼저 |
| 체어 가동률 · 시간당 매출 | 체어·시간 데이터 확보 후 |
| 재고 사용량 예측 고도화 | 실사 데이터 3개월 이상 후 (그전엔 이동평균) |
| 2FA · 백업 · 운영 모니터링 | Phase 2 Gate |

### 🔴 구현하지 않아도 되는 것 (또는 하지 말아야 할 것)
| 항목 | 이유 |
|---|---|
| DentWeb 대체 · 진료기록(차트) 기능 | DentWeb이 원본. 중복은 혼란과 법적 부담 |
| AX OS → DentWeb 자동 기록 (write-back) | 원본 훼손 위험 |
| AI 진단 · 치료 결정 | 임상 안전 원칙 |
| 환자 메시지 자동 발송 | 사람 승인 원칙 |
| 공급처 사이트 scraping · 자동 구매 | 약관 위반 · 보안 위험 (D-009) |
| Microservices · Kubernetes · 메시지 큐 | 1개 병원 규모에 불필요 |
| Multi-clinic SaaS 구조 | 목표가 아님 (D-005) |
| 네이티브 모바일 앱 | 반응형 웹으로 태블릿 대응 충분 |
| LLM이 직접 SQL을 작성·실행하는 Copilot | 권한 우회·오류 위험. 허용된 조회 도구만 |
| 자체 ML 수요예측 모델 | 단순 이동평균으로 충분한 규모 |
| 자체 인증·암호화 구현 | 검증된 라이브러리 사용 |
| 주민등록번호 저장 | 불필요한 고위험 데이터 |
| 직원 개인 성과 감시용 지표 | 상담 담당자별 동의율은 **코칭 용도**로만, 공개 순위표 만들지 않음 |
