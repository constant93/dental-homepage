# DATA_MODEL — Dental AX OS (Entity 초안)

> 상태: 초안 (2026-10-17). 실제 테이블은 Sprint마다 필요한 것만 migration으로 추가한다.
> 아래 "Phase" 열은 그 테이블이 처음 만들어지는 시점.

## 0. 공통 규칙

- 모든 주요 테이블: `id`(uuid), `createdAt`, `updatedAt`, `createdById`(가능한 경우), 삭제 가능한 것은 `deletedAt`(soft delete).
- 금액은 **원 단위 정수**(`Int`)로 저장. 소수점 오차 방지.
- 상태값은 enum. 자유 텍스트 상태 금지 (통계·AI가 쓸 수 없음).
- EMR에서 온 데이터는 `source` + `externalId` 를 가진다. AX OS 내부 `id`와 섞지 않는다.
- **저장하지 않는 것**: 주민등록번호, 진단명·치료 상세의 전체 사본(DentWeb이 원본), 카드번호.
- 단일 병원 시스템이므로 `clinicId`(multi-tenant)는 두지 않는다. (D-005)

## 1. Entity 관계 요약

```
User ─┬─< AuditLog
      ├─< Task (assignee)
      └─< Consultation (counselor)

Patient ─┬─< Appointment
         ├─< Consultation ─< ContactLog
         ├─< Receivable
         └─< (Phase 2) RecallItem, MessageDraft

InventoryItem ─┬─< StockMovement
               ├─< StockLot (유효기간)
               └─< SupplierOffer >─ Supplier
                        └─< PriceRecord
PurchaseOrder ─< PurchaseOrderLine >─ SupplierOffer

Insight (규칙 결과) ─ (relatedEntityType, relatedEntityId)
Task ─ (relatedEntityType, relatedEntityId)   ← Insight가 Task를 만들 수 있음
TaskTemplate ─< Task  (오픈/마감 체크리스트)
ImportBatch (EMR/CSV 가져오기 기록)
```

## 2. 공통 기반 (Phase 1)

### User
| 필드 | 타입 | 비고 |
|---|---|---|
| id | uuid | |
| name | string | 표시 이름 |
| loginId | string unique | |
| role | enum `OWNER, DENTIST, HYGIENIST, COORDINATOR, FRONT_DESK` | 1인 1 role (MVP) |
| isActive | bool | 퇴사 시 false (삭제 안 함) |
| lastLoginAt | datetime? | |

(비밀번호·세션 테이블은 인증 라이브러리 규격을 따른다.)

### AuditLog (append-only)
| 필드 | 타입 | 비고 |
|---|---|---|
| id | uuid | |
| actorId | User? | 시스템 작업이면 null |
| action | string | `consultation.update`, `patient.phone.view`, `auth.login.fail` … |
| entityType / entityId | string? | |
| summary | json | 변경 전/후 요약 (민감 필드는 마스킹) |
| ip | string? | |
| createdAt | datetime | |

### ImportBatch
| 필드 | 타입 | 비고 |
|---|---|---|
| source | enum `MOCK, CSV, MANUAL, DENTWEB` | |
| fileName | string? | |
| rowsTotal / rowsImported / rowsFailed | int | |
| errors | json | 행 번호 + 사유 (환자정보 원문은 넣지 않음) |
| runById | User | |

## 3. Patient / Schedule (Phase 1, 원본: EMR Adapter)

### Patient
| 필드 | 타입 | 비고 |
|---|---|---|
| source | enum | MOCK / CSV / MANUAL / DENTWEB |
| externalId | string? | DentWeb 차트번호 등. `(source, externalId)` unique |
| displayName | string | |
| phone | string? | 화면 기본 마스킹 |
| birthYear | int? | 연령대 분석용. 생년월일 전체 저장 여부는 Phase 2 결정 |
| firstVisitDate / lastVisitDate | date? | |
| referralSource | enum? | `INTERNET, REFERRAL, SIGNBOARD, BLOG, ETC, UNKNOWN` |
| recallDueDate | date? | Phase 1은 이 값으로 Recall 카운트. Phase 2에 RecallItem으로 확장 |
| memo | string? | 운영 메모 (임상 기록 아님) |
| deletedAt | datetime? | |

### Appointment
| 필드 | 타입 | 비고 |
|---|---|---|
| patientId | Patient | |
| source / externalId | | |
| startsAt / endsAt | datetime | |
| dentistId | User? | |
| chair | string? | 체어 가동률은 Phase 3 |
| visitType | enum | `NEW_PATIENT, CONSULTATION, TREATMENT, CHECKUP, EMERGENCY, ETC` |
| treatmentCategory | enum? | 아래 TreatmentCategory |
| status | enum | `BOOKED, ARRIVED, COMPLETED, CANCELLED, NO_SHOW` |
| expectedAmount | int? | 오늘 예상 매출 계산용 |
| cancelReason | string? | |

### Receivable (미수금)
| 필드 | 타입 | 비고 |
|---|---|---|
| patientId | Patient | |
| amount | int | |
| sinceDate | date | 발생일 |
| status | enum | `OPEN, CONTACTED, PROMISED, PAID, WRITTEN_OFF` |
| lastContactAt | datetime? | |

## 4. Consultation CRM (Phase 1, 원본: AX OS)

### TreatmentCategory (enum)
`IMPLANT, BONE_GRAFT, PROSTHESIS_CROWN, AESTHETIC_PROSTHESIS, RESIN, ENDO, PERIO, PERIO_SURGERY, EXTRACTION, ORTHO, DENTURE, SCALING, ETC`
(병원 실제 진료 항목에 맞춰 Sprint 2에서 원장 확인)

### Consultation
| 필드 | 타입 | 비고 |
|---|---|---|
| patientId | Patient | |
| consultedAt | date | |
| counselorId | User | 상담 담당 (동의율 분석용) |
| dentistId | User? | |
| treatmentCategory | enum | 주 치료 |
| quotedAmount | int | 견적 총액 |
| status | enum | `UNDECIDED, AGREED, PARTIALLY_AGREED, ON_HOLD, DECLINED, LOST` |
| holdReasons | enum[] | `PRICE, FEAR_PAIN, TIME_SCHEDULE, COMPARING_CLINICS, FAMILY_DISCUSSION, INSURANCE_QUESTION, NOT_URGENT, ETC` — 클릭 선택 |
| patientConcern | string? | 환자의 고민 (짧은 메모) |
| nextFollowUpDate | date? | 기본값: 상담일 + 3일 (설정 가능) |
| agreedAt | date? | |
| deletedAt | datetime? | |

> 치료계획 상세(치아 번호별 항목)는 DentWeb이 원본. MVP에서는 **치료 종류 + 견적 총액**만 기록한다.
> 항목별 견적이 실제로 필요해지면 `ConsultationItem`을 추가한다.

### ContactLog (Follow-up 기록)
| 필드 | 타입 | 비고 |
|---|---|---|
| consultationId | Consultation? | 상담 Follow-up |
| patientId | Patient | Recall·미수금 연락도 같은 테이블 |
| purpose | enum | `CONSULT_FOLLOW_UP, RECALL, RECEIVABLE, APPOINTMENT_REMINDER, ETC` |
| channel | enum | `PHONE, SMS, KAKAO, IN_PERSON` |
| outcome | enum | `NO_ANSWER, CALL_BACK_LATER, BOOKED, THINKING, DECLINED, WRONG_NUMBER` |
| note | string? | |
| contactedById | User | |
| contactedAt | datetime | |

"지금 다시 연락하면 좋은 환자" 규칙 (Phase 1, 설명 가능한 점수):
- 미동의 상태 + Follow-up 날짜 도래 → 대상
- 점수 ↑: 견적 금액, 상담 후 경과일(7–30일 구간), 보류 이유가 해결 가능한 유형(TIME_SCHEDULE, FAMILY_DISCUSSION, INSURANCE_QUESTION)
- 제외: 최근 3일 내 연락, 연락 시도 3회 이상 무응답, DECLINED

## 5. Inventory & Purchasing (Phase 1, 원본: AX OS)

### InventoryItem
| 필드 | 타입 | 비고 |
|---|---|---|
| name | string | 예: 리도카인 카트리지 |
| category | enum | `MEDICINE, ANESTHETIC, DISINFECTANT, IMPLANT, GBR, SUTURE, DISPOSABLE, IMPRESSION, BONDING, CEMENT, ETC` |
| baseUnit | string | 재고를 세는 최소 단위 (예: 카트리지, 개) |
| minQty | int | 이 수량 이하 → 재고 부족 |
| targetQty | int | 발주 시 채울 적정 수량 |
| currentQty | int | StockMovement 합계 캐시 (실사로 보정) |
| leadTimeDays | int? | 주문→도착 일수 |
| tracksExpiry | bool | |
| responsibleUserId | User? | 발주 검토 Task 담당 |
| isActive | bool | |

### StockMovement (재고 원장)
| 필드 | 타입 | 비고 |
|---|---|---|
| itemId | InventoryItem | |
| type | enum | `RECEIVE, USE, COUNT_ADJUST, DISCARD_EXPIRED, RETURN` |
| qty | int | baseUnit 기준 (+/-) |
| reason | string? | |
| purchaseOrderLineId | ? | 입고 시 연결 |
| createdById | User | |

> **현장 현실 반영**: 사용할 때마다 기록하는 방식은 바쁜 진료 중 지켜지지 않는다.
> 기본 방식은 **주 1회 실사(재고 세기)** — 실사 수량과 장부 수량의 차이 = 사용량으로 계산.
> 고가 품목(임플란트 픽스처, GBR 재료)만 사용 시 기록을 권장한다.

### StockLot (유효기간 추적 품목만)
| itemId | lotNo? | qty | expiryDate |

### Supplier
| 필드 | 타입 | 비고 |
|---|---|---|
| name | string | 예: 블루팜, ○○치과재료 |
| type | enum | `PHARMA_MALL, DENTAL_SUPPLY, MANUFACTURER_MALL, B2B_MALL, LOCAL_DEALER` |
| website / contact | string? | |
| freeShippingThreshold | int? | 이 금액 이상 무료배송 |
| shippingFee | int? | |
| minOrderAmount | int? | |
| memo | string? | 결제조건 등 |

### SupplierOffer (공급처별 상품)
| 필드 | 타입 | 비고 |
|---|---|---|
| itemId | InventoryItem | |
| supplierId | Supplier | |
| productName | string | 공급처에서 부르는 이름 |
| packSize | int | 구매 1단위 = baseUnit 몇 개 (예: 1박스 = N 카트리지) |
| currentPrice | int | 구매 1단위 가격 |
| productUrl | string? | 사람이 열어 확인하는 링크. 자동 접속 없음 |
| isPreferred | bool | |

### PriceRecord (가격 이력)
| offerId | price | observedAt | source: `MANUAL, INVOICE, CSV` | recordedById |

### PurchaseOrder / PurchaseOrderLine
| PurchaseOrder | supplierId, status `DRAFT, ORDERED, PARTIALLY_RECEIVED, RECEIVED, CANCELLED`, orderedAt, totalAmount, shippingFee |
|---|---|
| PurchaseOrderLine | orderId, offerId, packQty, unitPrice(주문 시점 가격 고정), receivedPackQty |

최저 구매처 계산 (순수 함수, AI 아님):
`실구매비용 = Σ(packQty × price) + (총액 < 무료배송 기준 ? 배송비 : 0)`, `baseUnit당 단가 = 실구매비용 / (packQty × packSize)`.
여러 품목을 한 공급처에 모으면 배송비가 사라지는 경우까지 비교하고, 결과에 계산 근거를 함께 보여준다.

## 6. Staff Task (Phase 1, 원본: AX OS)

### Task
| 필드 | 타입 | 비고 |
|---|---|---|
| title | string | |
| description | string? | |
| assigneeId | User? | 사람 지정 또는 |
| assigneeRole | Role? | 역할 지정 (예: 데스크 누구든) |
| dueAt | datetime? | |
| priority | enum | `HIGH, NORMAL, LOW` |
| status | enum | `TODO, IN_PROGRESS, DONE, CANCELLED` |
| source | enum | `MANUAL, CHECKLIST, RULE` |
| ruleId | string? | RULE이 만든 Task면 규칙 id (중복 생성 방지 키와 함께) |
| relatedEntityType / relatedEntityId | string? | 환자, 상담, 품목, 발주 등 |
| templateId | TaskTemplate? | |
| completedAt / completedById | | |

### TaskTemplate (오픈/마감 체크리스트, 반복 업무)
| name | kind: `OPENING, CLOSING, WEEKLY, MONTHLY` | items: json (체크 항목 목록) | assigneeRole | isActive |

## 7. Insight (Phase 1)

### Insight (규칙 실행 결과 스냅샷)
| 필드 | 타입 | 비고 |
|---|---|---|
| ruleId | string | |
| date | date | 생성 기준일 |
| category / severity | enum | ARCHITECTURE.md G-2 |
| facts | json | FACT 목록 (환자 이름 대신 id 참조) |
| inference | string? | |
| dataSufficient | bool | |
| priorityScore | int | |
| status | enum | `OPEN, ACKNOWLEDGED, RESOLVED, DISMISSED` |
| acknowledgedById | User? | 누가 확인했는지 |

> 저장하는 이유: "어제 경고했는데 처리되었나?"를 추적하고, 나중에 어떤 Insight가 실제 행동으로 이어졌는지 분석하기 위해.

## 8. Phase 2+ (지금 만들지 않음, 방향만)

- `RecallItem` (patientId, reason, dueDate, priorityScore, status)
- `MessageTemplate`, `MessageDraft` (status DRAFT→APPROVED→SENT, approvedById)
- `AiSuggestion` (purpose, promptVersion, inputRefs, output, status, reviewedById)
- `Payment` (실제 수납 — 일/월 매출 분석용, 출처 확인 후)
- `SopDocument`, `SopChunk` (Knowledge Base)

---

## 9. RBAC 권한 표 (기본값 제안 — 원장 확인 필요)

R = 읽기, W = 쓰기, — = 접근 불가, (M) = 마스킹/요약만

| 영역 | Owner | Dentist | Hygienist | Coordinator | Front Desk |
|---|---|---|---|---|---|
| Command Center | 전체 | 임상·상담·Task | 재고·Task·예약 | 상담·미수·Recall·Task | 예약·미수·Task |
| 예약·신환 | R | R | R | R | R/W |
| 환자 기본정보 | R/W | R | R | R/W | R/W |
| 환자 전화번호 | R | R(M) | R(M) | R | R |
| 상담·견적·Follow-up | R/W | R/W | R | R/W | R |
| 미수금 | R/W | — | — | R/W | R/W |
| 오늘 예상 매출 | R | R | — | — | — |
| 재고·실사 | R/W | R | R/W | R | R/W |
| 공급처·단가·가격 이력 | R/W | R | R | — | R |
| 발주 생성·확정 | R/W (확정) | — | W (초안) | — | W (초안) |
| Task | 전체 R/W | 본인·배정 | 본인·배정 | 본인·배정 | 본인·배정 |
| 체크리스트 템플릿 | R/W | — | — | — | — |
| 사용자·권한 관리 | R/W | — | — | — | — |
| Audit Log | R | — | — | — | — |
| AI 임상 초안 승인 (Phase 2) | ✓ | ✓ | — | — | — |
