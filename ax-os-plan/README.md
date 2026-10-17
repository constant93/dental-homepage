# Dental AX OS — 설계 문서 (Planning Stage)

> 상태: **설계 단계 · 코드 없음** (2026-10-17 작성)
>
> 이 폴더는 임시 보관 위치입니다. 이 저장소(`dental-homepage`)는 **공개(public) 저장소**이고,
> `main` 브랜치의 전체 내용이 GitHub Pages로 인터넷에 게시됩니다.
> 따라서 Dental AX OS의 **코드는 이 저장소에 만들지 않습니다.**
> 별도의 **비공개(private) 저장소**를 만든 뒤, 이 폴더의 내용을 그 저장소의 루트로 그대로 옮깁니다.
> (자세한 이유: `docs/DECISIONS.md`의 D-001)
>
> 이 브랜치는 `main`에 merge하지 마세요. merge하면 이 문서들이 홈페이지 주소로 공개됩니다.

## Dental AX OS란

DentWeb(기존 EMR)을 대체하지 않고, 그 **주변에** 붙는 치과 운영 시스템입니다.

```
Data → Rules/AI → Recommendation → Human Approval → Action → Logging
```

원장과 직원이 시스템을 열었을 때 **"오늘 내가 무엇을 해야 하는가?"** 를 먼저 보여주는 것이 목표입니다.

## 문서 지도

| 문서 | 내용 |
|---|---|
| [`CLAUDE.md`](./CLAUDE.md) | AI 개발 도우미가 매번 읽는 프로젝트 규칙 · 원칙 · 명령어 |
| [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md) | 전체 구조, 모듈 지도, 폴더 구조, 보안, AI 계층, DentWeb Adapter |
| [`docs/DATA_MODEL.md`](./docs/DATA_MODEL.md) | Database Entity 초안, 권한(RBAC) 표 |
| [`docs/DECISIONS.md`](./docs/DECISIONS.md) | 중요한 설계 결정 기록 (ADR) 과 아직 결정되지 않은 것 |
| [`docs/ROADMAP.md`](./docs/ROADMAP.md) | MVP 범위, 3단계 로드맵, 첫 Sprint, 지금/나중/안 함 구분 |

## Phase 1 MVP (Mock Data로 완전히 작동)

1. Command Center — 오늘 확인할 것 + Daily Briefing
2. Patient / Consultation CRM — 상담 후 Follow-up 관리
3. Inventory & Purchase Intelligence — 최소재고 · 발주 추천 · 공급처 가격 비교
4. Staff Task — 오늘 할 일 · 오픈/마감 체크리스트

## 개발 명령어

아직 코드가 없습니다. Sprint 1에서 프로젝트를 만든 뒤 이 섹션과 `CLAUDE.md`의 명령어를 채웁니다.
