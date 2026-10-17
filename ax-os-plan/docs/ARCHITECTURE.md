# ARCHITECTURE — Dental AX OS

> 상태: 초안 (2026-10-17). 구현 전 설계이며, Sprint 진행에 따라 갱신한다.
> `UNKNOWN` = 공식 문서로 확인되지 않은 것. 확인 전에는 이를 전제로 코드를 만들지 않는다.

---

## A. 전체 Architecture

### A-1. 계층 구조

```
┌─────────────────────────────────────────────────────────────────┐
│  사용자: 원장 · 치과의사 · 치위생사 · 상담실장 · 데스크          │
│  (Desktop 우선, Tablet 대응, 한국어 UI)                           │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTPS + 로그인 + Role
┌──────────────────────────────▼──────────────────────────────────┐
│  ⑤ Presentation        Command Center / 모듈 화면 / 승인 버튼    │
├─────────────────────────────────────────────────────────────────┤
│  ④ Application         모듈별 actions · queries                 │
│                        (권한 확인 → 검증 → 저장 → Audit Log)     │
├─────────────────────────────────────────────────────────────────┤
│  ③ Intelligence        Insight Engine (규칙, Phase 1)           │
│                        AI Gateway (LLM, Phase 2+, 비식별화 경유) │
├─────────────────────────────────────────────────────────────────┤
│  ② Dental AX Database  PostgreSQL (환자 최소정보, 상담, 재고,    │
│                        구매, Task, Insight, Audit Log)          │
├─────────────────────────────────────────────────────────────────┤
│  ① Integration Layer   EMR Adapter: Mock │ CSV/Excel │ Manual    │
│                        │ DentWeb (UNKNOWN)                      │
└──────────────────────────────┬──────────────────────────────────┘
                               │ 읽기 전용 (write-back 없음)
                       ┌───────▼───────┐
                       │   DentWeb      │  System of Record (진료기록)
                       └───────────────┘
```

### A-2. 핵심 설계 원칙

| 원칙 | 의미 |
|---|---|
| Modular Monolith | 앱 1개, DB 1개, 배포 1개. 모듈 경계는 폴더로 나눈다. Microservice 없음. |
| DentWeb은 읽기만 | AX OS는 DentWeb 데이터를 **가져와서 해석**만 한다. 진료기록을 확정하는 곳은 DentWeb. |
| AX OS가 원본인 데이터 | 상담 메모, 보류 이유, Follow-up 기록, 재고, 공급처 가격, Task — DentWeb에 없거나 구조화되어 있지 않은 **운영 데이터**는 AX OS가 원본이다. 이것이 AX OS가 새로 만드는 가치. |
| Rules first, AI second | "누가 Follow-up 대상인가"처럼 규칙으로 정확히 계산되는 것은 규칙으로. LLM은 요약·문장화·자연어 질의에만. |
| Human in the loop | 추천 → 사람 확인 → 실행 → 기록. 자동 실행 없음. |
| 데이터 부족은 데이터 부족 | 모든 Insight 규칙은 최소 데이터 조건을 가진다. 조건 미충족 시 결론 대신 "데이터 부족". |

### A-3. 기술 스택 (권장, 버전은 Sprint 1 시작 시 현재 안정 버전 확인 후 확정)

| 영역 | 선택 | 이유 |
|---|---|---|
| Full-stack | Next.js (App Router) + TypeScript | 화면과 서버 로직을 한 프로젝트에서. 자료·예제가 가장 많아 유지보수 쉬움. |
| DB | PostgreSQL | 요구사항. 관계형 데이터(환자-상담-Follow-up)에 적합. |
| ORM / Migration | Prisma | `schema.prisma` 한 파일에서 테이블 구조를 읽기 쉬움. `migrate` 로 migration 관리. |
| 입력 검증 | Zod | 폼 입력과 CSV import 검증을 같은 방식으로. |
| 인증 | Better Auth (후보, Sprint 1에서 확정) | 이메일/비밀번호, 세션, role, 2FA 확장. 인증 암호화를 직접 구현하지 않기 위함. |
| UI | Tailwind CSS + shadcn/ui | 컴포넌트 코드가 저장소 안에 복사되어 읽고 고칠 수 있음. |
| Test | Vitest (unit/integration) + Playwright (e2e) | |
| 로컬 DB | Docker Compose의 PostgreSQL | 개발 DB를 명령 하나로 생성/초기화. |

향후 데이터 분석·ML이 필요하면 `analytics` 작업만 Python 서비스로 분리할 수 있도록, 분석 로직은 DB를 직접 읽는 독립 함수(`insights/rules`)로 유지한다. 지금은 분리하지 않는다.

---

## B. Module Map

```
                         ┌──────────────────────┐
                         │   Command Center     │  "오늘 확인할 것 N개"
                         └──────────┬───────────┘
                                    │ 읽기
                         ┌──────────▼───────────┐
                         │   Insight Engine     │  규칙 → Insight(우선순위, FACT/INFERENCE)
                         └──┬─────┬─────┬─────┬─┘
            ┌───────────────┘     │     │     └──────────────┐
   ┌────────▼───────┐  ┌──────────▼──┐ ┌▼────────────┐ ┌─────▼──────┐
   │ Patients /     │  │ Consultation│ │ Inventory & │ │ Staff Task │
   │ Appointments   │  │ CRM         │ │ Purchasing  │ │ & Checklist│
   └────────▲───────┘  └─────────────┘ └─────────────┘ └────────────┘
            │                                   ▲ Insight가 Task를 생성
   ┌────────┴───────┐                           │ (예: 재고부족 → "발주 검토")
   │ EMR Adapter    │
   └────────────────┘
   공통 기반: Auth · RBAC · Audit Log · Error handling · Design System
```

| 모듈 | Phase | 역할 | 원본 데이터 |
|---|---|---|---|
| **Command Center** | 1 | 오늘 주의가 필요한 항목을 우선순위로. Daily Briefing. | 없음 (집계만) |
| **Patients / Appointments** | 1 | 환자 최소정보, 예약, 신환, 예상 매출 | EMR Adapter (Phase 1: Mock) |
| **Consultation CRM** | 1 | 상담·견적·보류 이유·Follow-up·연락 기록 | **AX OS** |
| **Inventory** | 1 | 품목, 최소/적정 재고, 입출고, 실사, 유효기간 | **AX OS** |
| **Purchasing** | 1 | 공급처, 공급처별 가격, 가격 이력, 발주 추천, 최저 구매처 계산 | **AX OS** |
| **Staff Task** | 1 | Task, 오픈/마감 체크리스트, 규칙이 만든 Task | **AX OS** |
| **Insight Engine** | 1 | 규칙 함수 모음. Command Center와 Task 생성의 근거 | 계산 결과 저장 |
| Recall | 2 | Recall 대상 · 우선순위 · 연락 기록 | EMR + AX OS |
| Patient Communication | 2 | 치료별 안내문 템플릿 → 환자 맞춤 초안 → 승인 → 출력/발송 | AX OS |
| AI Gateway | 2 | LLM 호출 단일 통로 (비식별화, 기록, 비용 한도) | — |
| Management Analytics | 3 | 매출·신환·동의율·No-show 추이, "무엇이 달라졌나" | EMR + AX OS |
| AI Copilot | 3 | 자연어 질의 → 허용된 조회 도구 실행 → FACT/INFERENCE 답변 | — |
| SOP / Knowledge Base | 3 | 병원 매뉴얼 + 근거 인용 답변 | AX OS |

---

## E. Folder Structure (비공개 저장소 `dental-ax-os` 기준)

```
dental-ax-os/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── ARCHITECTURE.md   DATA_MODEL.md   DECISIONS.md   ROADMAP.md
├── docker-compose.yml          # 로컬 PostgreSQL
├── .env.example                # 변수 이름만. 실제 값(.env)은 절대 커밋하지 않음
├── prisma/
│   ├── schema.prisma           # 테이블 구조 (DATA_MODEL.md와 일치)
│   ├── migrations/             # 자동 생성. 손으로 수정하지 않음
│   └── seed.ts                 # Synthetic 데이터 생성 (고정 seed → 매번 같은 결과)
├── src/
│   ├── app/                    # 화면(라우트)만. 로직은 modules로
│   │   ├── (auth)/login/
│   │   └── (app)/
│   │       ├── page.tsx            # Command Center
│   │       ├── consultations/
│   │       ├── patients/
│   │       ├── inventory/
│   │       ├── purchasing/
│   │       ├── tasks/
│   │       └── settings/           # 사용자·권한 (Owner만)
│   ├── modules/
│   │   ├── command-center/     # queries.ts, components/
│   │   ├── insights/           # rules/*.ts (규칙 1개 = 파일 1개), priority.ts, *.test.ts
│   │   ├── patients/           # queries.ts, actions.ts, schema.ts, components/
│   │   ├── consultations/
│   │   ├── inventory/
│   │   ├── purchasing/         # price-comparison.ts (최저 구매처 계산, 순수 함수)
│   │   └── tasks/
│   ├── integrations/
│   │   └── emr/
│   │       ├── types.ts            # EmrSource interface, Emr* 데이터 타입
│   │       ├── mock-source.ts      # Phase 1
│   │       ├── csv-source.ts       # Phase 2
│   │       ├── dentweb/README.md   # 확인된 사실 / UNKNOWN 목록. 코드 없음
│   │       └── import-runner.ts    # Source → 검증 → upsert → ImportBatch 기록
│   ├── ai/                     # Phase 2부터
│   ├── lib/
│   │   ├── db.ts               # Prisma client
│   │   ├── auth.ts             # 로그인/세션
│   │   ├── permissions.ts      # Role → 권한 표 (한 파일에서 전체가 보이게)
│   │   ├── audit.ts            # writeAuditLog()
│   │   └── errors.ts           # 사용자용 한국어 오류 메시지 변환
│   └── components/ui/          # 공통 디자인 시스템 (shadcn/ui)
└── tests/
    └── e2e/                    # Playwright
```

규칙: 화면(`app/`)은 얇게, 판단은 `rules.ts`(테스트 가능한 순수 함수)에, DB 접근은 `queries.ts`/`actions.ts`에만.

---

## F. Security Architecture

### F-1. 단계별 보안 수준

| 단계 | 데이터 | 필요한 보안 |
|---|---|---|
| Phase 1 | Synthetic만 | 로그인, RBAC, Audit Log, 비밀값 관리, 비공개 저장소. **실제 데이터 입력 금지** |
| Phase 2 진입 조건 | 실제 데이터 연결 직전 | 아래 F-5 체크리스트 전부 완료 + 원장 승인 |

### F-2. 인증 (Authentication)

- 직원 1인 1계정. **공용 계정 금지** (Audit Log가 의미를 잃음).
- 비밀번호 해시·세션은 검증된 인증 라이브러리에 맡긴다. 직접 구현하지 않는다.
- 세션 만료(업무용 PC 방치 대비): 무활동 시 자동 로그아웃.
- Owner / Dentist 계정은 Phase 2부터 2단계 인증(2FA) 필수.
- 퇴사자 계정은 삭제하지 않고 비활성화 (기록 보존).

### F-3. 권한 (RBAC, Least Privilege)

- 권한은 `src/lib/permissions.ts` 한 파일의 표로 정의 (`DATA_MODEL.md`의 RBAC 표와 일치).
- 모든 서버 action/query 첫 줄에서 `requirePermission()`. 화면에서 버튼을 숨기는 것은 편의일 뿐 보안이 아니다.
- 필드 단위 최소 노출:
  - 전화번호는 기본 마스킹(`010-****-1234`), "번호 보기" 클릭 시에만 전체 표시 + Audit Log.
  - 매출·단가 금액은 Owner(및 지정 role)에게만.

### F-4. Audit Log

- 기록 대상: 모든 쓰기(생성·수정·삭제·상태변경), 로그인/로그아웃/실패, 권한 변경, 환자 연락처 열람, 데이터 import/export, AI 초안 승인·거절.
- 내용: 누가 · 언제 · 무엇을(entity, id) · 어떻게(action) · 변경 요약 · 접속 IP.
- Audit Log는 수정·삭제 API를 만들지 않는다 (append-only).
- 보관 기간 등 법적 요건은 실제 데이터 연결 전 확인 (개인정보 보호법 및 하위 고시 — 전문가 확인 필요).

### F-5. 실제 데이터 연결 전 필수 체크리스트 (Phase 2 Gate)

- [ ] 서버 위치 결정 (병원 내부 서버 vs 국내 리전 클라우드) — D-006
- [ ] HTTPS 전 구간, DB 접속 암호화, 백업 암호화
- [ ] 자동 백업 + **복구 테스트 1회 이상 성공**
- [ ] 운영 비밀값은 저장소가 아닌 환경변수/비밀 관리 도구에
- [ ] 저장 항목 최소화 검토 (주민등록번호 저장 안 함 확인, 생년월일 필요성 검토)
- [ ] 개인정보 처리 관련 법적 검토 (의료기관 개인정보 · 민감정보 · 외부 위탁/국외 이전 여부)
- [ ] 직원 계정·권한 배정 원장 확인
- [ ] DentWeb 데이터 접근 방법의 공식 확인 (H 참조)

### F-6. 비밀값 · 저장소

- `.env`는 `.gitignore`. `.env.example`에는 변수 이름만.
- 코드 저장소는 **비공개**. CI 로그에 비밀값/데이터 출력 금지.
- Synthetic seed 외의 데이터 파일(CSV 등)은 저장소에 커밋하지 않는다 (`/data-imports/`는 gitignore).

---

## G. AI Architecture

### G-1. 3단 구조

```
 ① Insight Engine (Rules)          ② AI Gateway (LLM)             ③ Human
 ─────────────────────────         ──────────────────────         ───────────
 DB 조회 → 규칙 계산               FACT 묶음(비식별) 입력         검토·수정
 → FACT + 최소데이터 판정    ──▶   → 문장화/요약/분류 초안   ──▶   → 승인/거절
 → 우선순위 점수                   → 결과 + 메타데이터 저장        → 실행 + Log
 (Phase 1부터)                     (Phase 2부터)
```

- **Phase 1의 "AI Daily Briefing"은 LLM을 쓰지 않는다.** 규칙이 계산한 FACT를 한국어 템플릿 문장으로 보여준다. → 비용 0, 환자정보 외부 전송 0, 지어내기 0.
  - 예: 규칙 `consultation.followUpOverdue` → FACT `{ 대상: 3명, 조건: 임플란트 상담 후 7일 이상 미동의 }` → "임플란트 상담 후 7일 이상 경과한 미동의 환자가 3명 있습니다."
- Phase 2에서 LLM을 붙여도 **LLM은 FACT를 만들지 않는다.** 규칙이 만든 FACT를 다듬고, 추론(INFERENCE)을 제안할 뿐이며, 화면에서 두 가지를 구분 표시한다.

### G-2. Insight 형식 (모든 규칙 공통)

```ts
type Insight = {
  ruleId: string;              // 'inventory.belowMinimum'
  category: 'schedule' | 'consultation' | 'recall' | 'receivable' | 'inventory' | 'task' | 'trend';
  severity: 'critical' | 'warning' | 'info';
  facts: { label: string; value: string | number; source: string }[];
  inference?: string;          // 있을 때만. 화면에 'INFERENCE' 라벨
  dataSufficient: boolean;     // false면 결론 대신 "데이터 부족"
  actionHref?: string;         // 클릭 한 번으로 처리 화면 이동
  relatedEntities: { type: string; id: string }[];
};
```

우선순위 점수는 `severity × 긴급도(오늘/지남) × 금액 영향` 의 단순 공식으로 계산하고, 화면에서 "왜 이 순서인지" 펼쳐 볼 수 있게 한다. (설명 가능한 규칙 > 블랙박스 점수)

### G-3. Phase 1 규칙 후보 (최소 데이터 조건 포함)

| ruleId | 내용 | 최소 데이터 조건 |
|---|---|---|
| `schedule.todaySummary` | 오늘 예약 N, 신환 N, 예상 매출 | 오늘 예약 데이터 존재 |
| `consultation.followUpDue` | Follow-up 날짜 도래/경과한 미동의 상담 | — |
| `consultation.staleUndecided` | 상담 후 7일 이상 미동의 & 최근 연락 없음 | — |
| `receivable.needsCheck` | 미수금 N건, 오래된 순 | — |
| `recall.due` | Recall 예정일 경과 환자 수 (Phase 1은 카운트만) | — |
| `inventory.belowMinimum` | 현재 재고 ≤ 최소 재고 | 최근 실사 기록 존재 |
| `inventory.reorderSuggestion` | 사용 속도 기반 소진 예상일 < 리드타임 | 4주 이상 사용/실사 이력, 없으면 최소재고 기준으로 대체 + "데이터 부족" 표기 |
| `inventory.expiringSoon` | 유효기간 30일 이내 로트 | 유효기간 입력된 품목만 |
| `trend.cancellationRateUp` | 이번 주 취소율 vs 지난 4주 평균 | 지난 4주 예약 데이터, 주당 최소 건수 |
| `task.overdue` | 마감 지난 Task | — |

### G-4. AI Gateway (Phase 2)

모든 LLM 호출은 `src/ai/gateway.ts` 한 곳을 통과한다.

1. **비식별화**: 이름·전화·차트번호·생년월일 제거 → `환자A`, `50대`, `임플란트 상담` 같은 최소 속성만. 매핑표는 서버 메모리에만.
2. **목적 제한**: 호출마다 `purpose`(briefing / draft_patient_guide / consult_summary …) 필수. 목적별 허용 필드 화이트리스트.
3. **기록**: `AiSuggestion` 테이블에 목적, 프롬프트 버전, 입력 참조, 출력, 상태(DRAFT → APPROVED/REJECTED), 승인자 저장.
4. **실패 대비**: LLM 실패·지연 시 규칙 템플릿 문장으로 대체 표시 (화면이 비지 않게).
5. **비용 한도**: 일일 호출 수/비용 상한. 초과 시 템플릿 모드.
6. **Provider 교체 가능**: `LlmProvider` interface 하나. 특정 회사 SDK를 모듈 곳곳에서 직접 부르지 않는다.

외부 LLM 사용 자체(어떤 회사, 어떤 데이터, 국외 전송 여부)는 **도입 전 원장 승인 + 법적 확인** 대상 (D-007).

### G-5. 임상 안전 (Clinical Safety)

- 임상 관련 출력(환자 안내문, 상담 요약, 치료 설명)은 상태 `DRAFT`로만 생성.
- 화면에 `AI 초안 · 원장 확인 필요` 배지. Dentist role만 `APPROVED` 가능.
- 승인 전에는 인쇄/발송/복사 버튼 비활성화. EMR 자동 기록 기능은 만들지 않는다.

### G-6. AI Copilot (Phase 3) 설계 방향

- LLM에게 **임의 SQL을 쓰게 하지 않는다.** 미리 만든 조회 도구(예: `getRevenueByMonth`, `findConsultationsNeedingFollowUp`)만 호출 가능 (tool use).
- 도구는 호출한 사용자의 권한으로 실행 → Copilot도 RBAC를 우회하지 못함.
- 답변은 FACT(도구 결과, 숫자 출처 포함)와 INFERENCE를 분리해서 표시.

---

## H. DentWeb Integration Adapter

### H-1. 확인된 사실 / UNKNOWN

| 항목 | 상태 |
|---|---|
| DentWeb이 병원의 EMR이며 진료기록 원본이다 | 확인 (원장 진술) |
| 공식 API 존재 여부 | **UNKNOWN** |
| 데이터 export 기능(엑셀/CSV 등) 존재 여부·항목 | **UNKNOWN** |
| DB 종류·스키마·직접 접근 허용 여부 | **UNKNOWN** |
| 외부 연동에 대한 라이선스·약관 조건 | **UNKNOWN** |
| 연동 가능한 파트너/공식 연동 프로그램 | **UNKNOWN** |

→ 위 항목이 확인되기 전에는 DentWeb 전용 코드를 만들지 않는다.

### H-2. DentWeb 측에 확인할 질문 (원장님이 공급사에 문의)

1. 외부 프로그램이 예약·환자·수납·미수금 데이터를 **읽을 수 있는 공식 방법**(API, export, 연동 파트너)이 있는가?
2. 화면/통계 메뉴에서 **엑셀(CSV) 내보내기**가 가능한 항목은 무엇인가? (예약 목록, 신환 목록, 수납/미수 목록, 치료 내역)
3. DB에 직접 읽기 접근을 하는 것이 **라이선스·약관상 허용**되는가?
4. 연동 시 개인정보 처리 관련 공급사의 요구 조건은?

### H-3. Adapter 구조

```ts
// src/integrations/emr/types.ts  — AX OS가 "필요로 하는" 데이터 모양 (DentWeb 모양 아님)
export interface EmrSource {
  readonly kind: 'mock' | 'csv' | 'manual' | 'dentweb';
  fetchPatients(changedSince?: Date): Promise<EmrPatient[]>;
  fetchAppointments(range: { from: Date; to: Date }): Promise<EmrAppointment[]>;
  fetchReceivables(): Promise<EmrReceivable[]>;
}

export type EmrPatient = {
  externalId: string;          // EMR 차트번호 등. AX OS 내부 id와 분리
  displayName: string;
  phone?: string;
  firstVisitDate?: string;     // YYYY-MM-DD
  lastVisitDate?: string;
  referralSource?: string;
};
// EmrAppointment, EmrReceivable 동일한 방식
```

```
EmrSource (mock | csv | dentweb)
   │  fetch*
   ▼
import-runner.ts
   ├─ Zod로 행 단위 검증 (잘못된 행은 건너뛰고 사유 기록)
   ├─ externalId 기준 upsert (중복 생성 방지)
   ├─ 삭제는 하지 않음 (EMR에서 사라진 행 → 표시만)
   └─ ImportBatch 저장 (출처, 시각, 성공/실패 건수, 실행자)
```

- 방향은 **EMR → AX OS 단방향 읽기**만. AX OS → DentWeb 쓰기는 설계하지 않는다.
- Phase 1: `mock-source.ts` (seed와 같은 Synthetic 생성기).
- Phase 2: `csv-source.ts` — DentWeb export 항목 확인 후, 그 파일의 열 이름 ↔ `EmrPatient` 필드 매핑표 작성. 직원이 "파일 올리기" → 미리보기 → 확인 후 반영.
- 공식 API 등이 확인되면 `dentweb/` 에 해당 Source만 추가. 나머지 시스템은 변경 없음.
