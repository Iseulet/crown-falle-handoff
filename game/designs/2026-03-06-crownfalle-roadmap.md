# CrownFalle Development Roadmap
> 🇰🇷 크라운폴 개발 로드맵

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Status: Active Reference Document
- Purpose: Development sequence guide and action plan reference
> 🇰🇷 생성일: 2026-03-06
> 🇰🇷 상태: 활성 참조 문서
> 🇰🇷 목적: 개발 순서 가이드 및 액션 플랜 참조

---

## Phase 0 — Environment Setup
> 🇰🇷 Phase 0 — 환경 설정

### Completed ✅
- Windows Terminal + Git Bash
- Claude Code (Sonnet 4.6, Max subscription)
- Gemini CLI (API key configured)
- Godot 4.6.1
- Global/Project rules (CLAUDE.md / GEMINI.md)
- handoff/ system (Global + Project)
- Agent architecture (PLANNER / IMPLEMENTOR / REVIEWER)
- Git + GitHub private repositories
- Godot project folder structure

> 🇰🇷 완료 항목:
> 🇰🇷 Windows Terminal + Git Bash 연동
> 🇰🇷 Claude Code (Sonnet 4.6, Max 구독)
> 🇰🇷 Gemini CLI (API 키 설정)
> 🇰🇷 Godot 4.6.1 설치
> 🇰🇷 글로벌/프로젝트 규칙 정비
> 🇰🇷 handoff/ 시스템 (글로벌 + 프로젝트)
> 🇰🇷 에이전트 아키텍처 완성
> 🇰🇷 Git + GitHub Private 저장소 연결
> 🇰🇷 Godot 프로젝트 폴더 구조 생성

---

## Phase 1 — Combat System Prototype
> 🇰🇷 Phase 1 — 전투 시스템 프로토타입

### 1-1. Grid System ✅
> 🇰🇷 격자 시스템

- [x] TileMap-based grid generation
- [x] Unit placement on grid
> 🇰🇷 TileMap 기반 격자 생성
> 🇰🇷 유닛 배치 및 이동

### 1-2. Movement System ✅
> 🇰🇷 이동 시스템

- [x] BFS movement range calculation
- [x] Highlight movable tiles
- [x] Click-to-move unit
> 🇰🇷 BFS 이동 범위 계산
> 🇰🇷 이동 가능 타일 하이라이트
> 🇰🇷 클릭으로 유닛 이동

### 1-3. Turn System ✅
> 🇰🇷 턴 시스템

- [x] Turn queue (ally → enemy)
- [x] End turn button
- [x] Turn indicator UI
> 🇰🇷 턴 순서 큐 (아군 → 적군)
> 🇰🇷 턴 종료 버튼
> 🇰🇷 턴 표시 UI

### 1-4. Combat Action ✅
> 🇰🇷 전투 액션

- [x] Attack adjacent tile enemy
- [x] Damage calculation
- [x] HP display UI
- [x] Unit death handling
> 🇰🇷 인접 타일 적 공격 (6방향 Staggered Grid)
> 🇰🇷 데미지 계산
> 🇰🇷 HP 표시 UI
> 🇰🇷 유닛 사망 처리

---

## Phase 2 — Character System
> 🇰🇷 Phase 2 — 캐릭터 시스템

### 2-1. Character Data ✅
> 🇰🇷 캐릭터 데이터

- [x] MercenaryData Resource design
- [x] Attributes: Name / Class / HP / STR / MV
> 🇰🇷 MercenaryData Resource 설계
> 🇰🇷 이름/직업/HP/STR/MV 속성

### 2-2. Warband Management ✅
> 🇰🇷 용병단 관리

- [x] Warband inventory system
- [x] Character list UI
> 🇰🇷 Warband 인벤토리 시스템
> 🇰🇷 캐릭터 목록 UI

### 2-3. Level Up System ✅
> 🇰🇷 레벨업 시스템

- [x] Experience gain
- [x] Stat selection on level up
> 🇰🇷 경험치 획득
> 🇰🇷 레벨업 시 능력치 선택 (HP/STR/MV)

---

## Phase 3 — World Map System
> 🇰🇷 Phase 3 — 월드맵 시스템

### 3-1. World Map Movement ✅
> 🇰🇷 월드맵 이동

- [x] Click-to-move character icon
- [x] Smooth movement animation
> 🇰🇷 캐릭터 아이콘 클릭 이동
> 🇰🇷 부드러운 이동 애니메이션 (Tween, 거리 비례 속도)

### 3-2. Fatigue System ✅
> 🇰🇷 피로도 시스템

- [x] Fatigue accumulation on movement
- [x] Fatigue UI display
> 🇰🇷 이동 시 피로도 누적
> 🇰🇷 피로도 UI 표시

### 3-3. Camp System ✅
> 🇰🇷 캠프 시스템

- [x] Scene transition to camp
- [x] Food consumption to recover fatigue
> 🇰🇷 캠프 씬 전환
> 🇰🇷 식량 소모 피로도 회복

---

## Phase 4 — NPC / Quest System
> 🇰🇷 Phase 4 — NPC/퀘스트 시스템

### 4-1. Interaction System ✅
> 🇰🇷 상호작용 시스템

- [x] Distance-based proximity detection
- [x] E key interaction
- [x] Popup UI
> 🇰🇷 거리 기반 근접 감지
> 🇰🇷 E키 상호작용
> 🇰🇷 팝업 UI

### 4-2. Dialogue System ✅
> 🇰🇷 대화 시스템

- [x] Dialogue window UI
- [x] Dialogue data structure (Array[Dictionary] in NPC.gd)
> 🇰🇷 대화창 UI
> 🇰🇷 대화 데이터 구조

### 4-3. Quest System ✅
> 🇰🇷 퀘스트 시스템

- [x] JSON-based quest data
- [x] Quest accept/complete status
- [x] Quest log UI
> 🇰🇷 JSON 기반 퀘스트 데이터
> 🇰🇷 퀘스트 수락/완료 상태
> 🇰🇷 퀘스트 로그 UI

---

## Phase 5 — Integration & Polish
> 🇰🇷 Phase 5 — 통합 및 polish

### 5-1. Scene Transitions ✅
> 🇰🇷 씬 전환

- [x] World map ↔ Combat ↔ Camp connections
- [x] Player position restored on combat return
> 🇰🇷 월드맵 ↔ 전투 ↔ 캠프 연결
> 🇰🇷 전투 복귀 시 진입 위치 복원

### 5-2. Data Integration ✅
> 🇰🇷 데이터 연동

- [x] Combat results → Quest completion + food reward
- [x] Defeat/turn_limit → status message on world map
> 🇰🇷 전투 승리 → 퀘스트 완료 + 식량 보상
> 🇰🇷 패배/턴초과 → 월드맵 상태 메시지

### 5-3. UI/UX Polish ✅
> 🇰🇷 기본 UI/UX 정리

- [x] World map HUD: fatigue + food combined display
- [ ] Basic sound effects (skipped — no assets)
> 🇰🇷 월드맵 HUD 피로도 + 식량 통합 표시
> 🇰🇷 사운드: 에셋 없어 스킵

### 5-4. Prototype Validation ✅
> 🇰🇷 프로토타입 검증

- [x] Full flow test (world map → quest → combat → reward → camp)
- [x] Balance: spawn positions spread (y=2 / y=7), fatigue/food tuning
> 🇰🇷 전체 플로우 테스트 완료
> 🇰🇷 스폰 위치 분리, 피로도/식량 수치 조정

---

## Phase A — 3D 전투 씬 재구축 (진행 중)
> 🇰🇷 Phase A — 3D 씬 전환 + 애니메이션 시스템

- [x] A-1: 지형 타입 시스템
- [x] 3D 전환 (TileRenderer3D, UnitRenderer3D, 조명)
- [x] 카메라 시스템 (orbit/zoom/pan + WASD + 시네마틱 4종)
- [x] 전투 레벨 4종 (small/medium/large)
- [x] AnimState state machine + Tween 이동 + 공격 50% 피해 타이밍
- [ ] A-2: 데이터 기반 애니메이션 시스템 (Doc 1~6) ← **현재 위치**
- [ ] A-3: 조명 시스템

---

## 전투 시스템 구현 로드맵 — Tier 1~4 [2026-03-13 변경]
> 🇰🇷 전투 시스템 통합 설계 기반 구현 우선순위
> 참조: `handoff/plans/design/2026-03-13-combat-system-proposal-final.md` §11

### Tier 1 — 전투의 뼈대 (Phase B)
> 🇰🇷 모든 전투의 기반. 이것 없이는 다른 시스템 테스트 불가.

| # | 시스템 | 의존성 |
|---|--------|--------|
| T1-1 | 스탯 시스템 풀구현 (5차 + 2차 + ARM/RES 게이지) | 없음 |
| T1-2 | ARM/RES 게이지 구현 (데미지 흐름 + CC 게이트) | T1-1 |
| T1-3 | 전투 공식 풀구현 (HIT / CRIT / 데미지 + 게이지 차감) | T1-1, T1-2 |
| T1-4 | STA 시스템 구현 | T1-1 |
| T1-5 | 교전 시스템 보강 (기회 공격 + 강제 이탈) | T1-1, T1-4 |

- [ ] T1-1: 스탯 풀구현
- [ ] T1-2: ARM/RES 게이지
- [ ] T1-3: 전투 공식
- [ ] T1-4: STA 시스템
- [ ] T1-5: 교전 보강

### Tier 2 — 전술적 깊이 (Phase B 후반)
> 🇰🇷 Tier 1 완성 후 전투에 전략적 레이어 추가.

| # | 시스템 | 의존성 |
|---|--------|--------|
| T2-1 | 상태이상 시스템 (CC + DoT) | T1-2, T1-3 |
| T2-2 | 액티브 스킬 프레임워크 | T1-4 |
| T2-3 | 패시브 스킬 구현 (PassiveManager) | T1-1 |
| T2-4 | 원소 속성 시스템 (스킬/장비 레이어) | T2-1, T2-2 |
| T2-5 | VP 시스템 (파티 공유, 사기) | T1-5 |
| T2-6 | 조건부 이탈 (VP/스킬/아군 지원) | T2-5, T2-2 |

- [ ] T2-1 ~ T2-6

### Tier 3 — 깊이 확장 (Phase C)
> 🇰🇷 전투 재미 검증 후 장기 리스크와 유틸리티 추가.

| # | 시스템 | 의존성 |
|---|--------|--------|
| T3-1 | 유틸리티 스킬 (밀치기/끌어오기/연막) | T2-2, T1-5 |
| T3-2 | 궁극기 (VP 소모 스킬) | T2-5, T2-2 |
| T3-3 | 부상(Injury) 시스템 | T1-1 |
| T3-4 | 어둠(Dark) 지형 효과 | T1-3 |
| T3-5 | 전투 UI (STA/MP/VP/ARM/RES 표시, 스킬 선택, 전투 로그) | T2-1~T2-5 |

- [ ] T3-1 ~ T3-5

### Tier 4 — 확장 (Phase D+)
> 🇰🇷 기본 전투 완성 후 환경 전술 레이어 추가.

| # | 시스템 |
|---|--------|
| T4-1 | 지형 표면 레이어 (화염/독/얼음 표면) |
| T4-2 | 원소 표면 상호작용 (연쇄 반응) |
| T4-3 | 대형 유닛 (2×2) |
| T4-4 | 배치 구역 시스템 |
| T4-5 | AI 전술 패턴 다양화 |

- [ ] T4-1 ~ T4-5

---

## Agent Role Assignment
> 🇰🇷 에이전트 역할 분담

```
@Planner (Gemini CLI preferred)
→ Feature design before each phase
→ Output: handoff/plans/design/YYYY-MM-DD-feature_name.md

@Implementor (Claude Code preferred)
→ Implementation based on Planner output
→ Output: Godot project code files

@Reviewer (Either tool)
→ Code review after implementation
→ Output: handoff/plans/review/YYYY-MM-DD-feature_name.md
```

> 🇰🇷 @Planner: 각 Phase 시작 전 기능 설계 (Gemini CLI 우선)
> 🇰🇷 @Implementor: 설계 기반 구현 (Claude Code 우선)
> 🇰🇷 @Reviewer: 구현 후 코드 검수 (어느 툴이든 가능)

---

## Development Sequence Per Feature
> 🇰🇷 기능별 개발 순서

```
1. @Planner → Design document
2. User approval
3. @Implementor → Implementation
4. User test in Godot
5. @Reviewer → Code review
6. Fix if needed (max 2 cycles)
7. User confirms → Commit
```

> 🇰🇷 1. @Planner → 설계 문서 작성
> 🇰🇷 2. 사용자 승인
> 🇰🇷 3. @Implementor → 구현
> 🇰🇷 4. 사용자 Godot 테스트
> 🇰🇷 5. @Reviewer → 코드 검수
> 🇰🇷 6. 필요 시 수정 (최대 2회)
> 🇰🇷 7. 사용자 확인 → 커밋
