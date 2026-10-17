# CLAUDE.md — Dental AX OS

이 파일은 이 프로젝트에서 일하는 AI 개발 도우미가 **매 작업 전에** 읽는 규칙입니다.
자세한 설계는 `docs/` 아래 문서를 봅니다.

## 1. 프로젝트 목표

- 대한민국 실제 치과에서 쓰는 AI 기반 운영 시스템 "Dental AX OS"를 단계적으로 만든다.
- 흐름: `Data → Rules/AI → Recommendation → Human Approval → Action → Logging`
- 첫 화면은 dashboard가 아니라 **"What needs attention?"** — 오늘 해야 할 일을 우선순위로 보여준다.
- 프로젝트 소유자는 코딩 경험이 많지 않다. **소유자가 읽고 유지할 수 있는 코드**가 최우선이다.

## 2. 모든 기능이 통과해야 하는 질문 (하나 이상 Yes)

1. 직원의 반복 업무를 줄이는가?
2. 원장의 판단 시간을 줄이는가?
3. 환자 경험을 개선하는가?
4. 누락이나 실수를 줄이는가?
5. 매출 또는 비용 효율을 개선하는가?
6. 새로운 데이터를 얻는가?
7. 그 데이터를 다음 판단(Rules/AI)에 쓸 수 있는가?

모두 No면 만들지 않는다.

## 3. 절대 규칙 (Important Constraints)

- **DentWeb은 System of Record.** 대체하지 않는다. DentWeb에 쓰기(write-back)하지 않는다.
- **연동 방법을 추측하지 않는다.** 문서로 확인되지 않은 DentWeb/외부 API, endpoint, DB schema, 인증 방식은 `UNKNOWN`으로 표시하고 Adapter interface + Mock 구현만 만든다.
- **실제 환자 데이터 금지 (Phase 1).** Synthetic/Mock 데이터만 사용한다. 가짜 환자 이름은 `테스트환자001` 처럼 누가 봐도 가짜인 형식을 쓴다.
- **주민등록번호는 어떤 단계에서도 저장하지 않는다.**
- **AI는 진단·치료를 결정하지 않는다.** 임상 관련 AI 결과는 `AI 초안` 으로 표시하고 Dentist 승인 전에는 확정·발송·EMR 기록하지 않는다.
- **AI는 모르는 것을 아는 척하지 않는다.** 데이터가 부족하면 "데이터 부족"을 표시한다. 분석 결과는 FACT와 INFERENCE를 구분한다.
- **환자에게 자동 발송하지 않는다.** 모든 발송은 사람이 확인 버튼을 누른 뒤에만.
- **공급처 사이트 scraping·자동 구매를 구현하지 않는다.** 가격은 수동 입력 / CSV / 거래명세서로 기록한다.
- **외부 AI API에 환자 식별정보(이름, 전화번호, 차트번호, 생년월일)를 보내지 않는다.**
- **파괴적 DB 작업(DROP, TRUNCATE, 전체 DELETE, destructive migration)은 이유를 설명하고 명시적 승인을 받은 뒤에만.**
- 이 시스템의 코드는 **비공개 저장소**에만 둔다. (D-001)

## 4. Architecture 요약

- Modular Monolith. 하나의 TypeScript full-stack 앱 + PostgreSQL. (D-002, D-003)
- 모듈: `command-center`, `patients`, `consultations`, `inventory`, `purchasing`, `tasks`, `insights`, (Phase 2+) `recall`, `communication`, `analytics`, `copilot`, `knowledge`
- 외부 데이터는 반드시 `src/integrations/emr/` 의 Adapter를 통해서만 들어온다.
- Command Center와 Daily Briefing은 `insights` 모듈의 **규칙(rule) 함수**가 만든 결과만 보여준다. LLM은 Phase 2부터, 규칙이 만든 FACT를 문장으로 다듬는 용도로만.
- 자세한 내용: `docs/ARCHITECTURE.md`

## 5. Coding Convention

- TypeScript strict. `any` 금지 (불가피하면 이유 주석).
- 이름은 길어도 명확하게: `findConsultationsNeedingFollowUp()` 좋음, `getData()` 나쁨.
- 한 모듈 폴더 안의 파일 역할:
  - `queries.ts` — DB 읽기 (server only)
  - `actions.ts` — DB 쓰기 (권한 확인 → 입력 검증 → 저장 → audit log)
  - `schema.ts` — Zod 입력 검증
  - `rules.ts` — 판단 규칙 (순수 함수, 테스트 필수)
  - `components/` — 이 모듈 전용 UI
- 모든 쓰기 action은 이 순서를 지킨다: `requirePermission` → `schema.parse` → DB 저장 → `writeAuditLog`.
- 불필요한 abstraction, wrapper, helper, dependency를 만들지 않는다. 같은 코드가 3번 반복되기 전에는 공통화하지 않는다.
- UI 문구는 한국어. 코드 식별자·커밋 메시지는 영어.
- 사용자에게 raw error / stack trace를 보여주지 않는다. 한국어 안내 문구 + 서버 로그.
- 모든 화면은 Loading / Empty / Error / 권한 없음 상태를 가진다.
- 삭제는 기본적으로 soft delete (`deletedAt`). 주요 테이블은 `createdAt`, `updatedAt`, `createdById`.

## 6. 작업 방식

매 작업마다:

1. 현재 저장소 구조와 관련 코드를 읽는다.
2. 영향 범위를 파악하고 짧은 계획을 세운다.
3. 필요한 부분만 수정한다. 관련 없는 코드를 refactor하지 않는다.
4. `test → typecheck → lint → build` 를 실행하고, 실패하면 원인을 고친다.
5. 결과를 보고한다 (무엇을 했는지, 무엇을 확인했는지, 무엇이 남았는지).

`small → test → verify → continue`. 한 번에 한 기능.

중요 결정(architecture 변경, 데이터 삭제, 보안 구조 변경, 비용 발생 서비스, 환자 데이터 외부 전송)은 **먼저 소유자에게 알리고 승인**받는다. 그 외 사소한 선택은 스스로 판단하고 `docs/DECISIONS.md`에 필요하면 기록한다.

## 7. Definition of Done

"완성"이라고 말하려면 모두 확인:

- [ ] 기능 구현 + UI 동작
- [ ] 데이터 저장 (migration 포함)
- [ ] 권한 확인 (허용 role / 거부 role 둘 다 테스트)
- [ ] error handling (Loading / Empty / Error / 권한 없음)
- [ ] audit log (쓰기 작업)
- [ ] test (rules unit test, action integration test, 핵심 흐름 e2e)
- [ ] typecheck · lint · build 통과
- [ ] 관련 문서 갱신

## 8. Common Commands

> Sprint 1에서 프로젝트 생성 후 채운다. 예정:
>
> - `npm run dev` — 개발 서버
> - `npm run test` — unit/integration test
> - `npm run test:e2e` — Playwright
> - `npm run typecheck` / `npm run lint` / `npm run build`
> - `npx prisma migrate dev` — migration 생성/적용 (개발 DB만)
> - `npm run db:seed` — Synthetic 데이터 생성

## 9. 현재 단계

- Phase 1 / Sprint 0 (설계 완료, 저장소 생성 대기)
- 다음 작업: `docs/ROADMAP.md` 의 "Sprint 1"
