# CrownFalle — 전투 HUD & 스킬 UI 설계

- Date: 2026-03-14
- Agent: @Planner
- Status: 사용자 승인 대기
- References:
  - 워테일즈 전투 HUD 스킬 UI 심층 분석 보고서 (첨부)
  - `2026-03-13-combat-system-proposal-final.md` — 전투 시스템 전체
  - `2026-03-13-tier2-active-skills.md` — 스킬 매니저
  - `2026-03-13-tier2-status-effects.md` — 상태이상 11종
  - `2026-03-13-tier2-element-system.md` — 원소 상성

---

## 0. 설계 원칙

| 원칙 | 워테일즈 참조 | CrownFalle 적용 |
|------|-------------|-----------------|
| 시각적 위계 | 하단 중앙 스킬 바 최우선 접근 | 동일 — 하단 3분할 (유닛 상태 / 스킬 바 / 타겟 상태) |
| 자원 시각화 | VP(용기 점수) 다이아몬드 아이콘 | VP + ARM/RES 게이지 + STA/MP 바 — 4종 자원 동시 표시 |
| 정보 밀도 | 핵심은 크게, 보조는 툴팁에 | 스킬 바에 비용만 표시, 상세는 호버 툴팁 |
| 인터랙션 | 단축키 + 타겟팅 범위 시각화 | 1~8 단축키 + 헥스 사거리/LOS/AoE 하이라이트 |
| 세계관 톤 | Gritty & Tactical (어두운 중세) | 동일 — 다크 UI, 채도 낮은 팔레트, 게이지형 자원 |

---

## 1. 전체 레이아웃

### 1-1. 영역 배치

| 영역 | 위치 | 표시 조건 |
|------|------|----------|
| 턴 정보 | 좌상단 | 항상 |
| VP 카운터 | 상단 중앙 | 항상 |
| 턴 종료 버튼 | 우상단 | 아군 페이즈만 활성 |
| 선택 유닛 패널 | 좌하단 | 아군 유닛 선택 시 |
| 스킬 바 | 하단 중앙 | 아군 유닛 선택 시 |
| 타겟 유닛 패널 | 우하단 | 적 유닛 호버/타겟 시 |
| 예상 결과 팝업 | 타겟 유닛 근처 (3D 월드) | 스킬 선택 + 타겟 호버 시 |
| 사거리/AoE 오버레이 | 3D 전장 위 | 스킬 선택 시 |

### 1-2. 정보 우선순위

1. 선택 유닛 HP/ARM/RES — "이 유닛이 지금 얼마나 위험한가?"
2. 스킬 바 활성 상태 — "지금 뭘 할 수 있는가?"
3. VP 카운터 — "강력한 스킬을 쓸 여유가 있는가?"
4. 타겟 유닛 ARM/RES — "이 적에게 CC를 걸 수 있는가?"
5. 타겟팅 범위 — "어디까지 닿는가?"
6. 예상 결과 — "이 행동의 결과는?"

---

## 2. 유닛 상태 패널

### 2-1. 게이지 색상 코딩

| 게이지 | 색상 | 비율별 전환 | 소진 시 |
|--------|------|-----------|--------|
| HP | #97C459 (녹색) | >50% 녹 / 25~50% 노랑 / <25% 빨강 | 사망 |
| ARM | #B4B2A9 (회색/은색) | 단색 유지 | 빨간 ⚠ + "물리 CC 가능" |
| RES | #AFA9EC (보라) | 단색 유지 | 빨간 ⚠ + "마법 CC 가능" |
| STA | #F2A623 (노랑/주황) | >30% 노랑 / ≤30% 주황 | 행동 불가 경고 |
| MP | #85B7EB (파랑) | 단색 유지 | — |

### 2-2. ARM/RES 소진 표시 (핵심 전술 정보)

ARM 또는 RES가 0이면:
- 바: 빈 바 + 빨간 테두리
- 텍스트: "0/24 ⚠" (빨간색)
- 타겟 패널: CC 가능 태그가 강조 배경으로 표시

### 2-3. 상태이상 태그

| 카테고리 | 배경색 | 예시 |
|---------|--------|------|
| CC (행동 차단) | 빨간 20% | [기절 1t] [빙결 1t] |
| DoT | 주황 20% | [출혈 3t] [연소 2t] |
| 디버프 | 파랑 20% | [감전 2t] [냉기 2t] |
| 원소 상태 | 원소 고유색 | [💧wet] [🔥burning] |

---

## 3. 스킬 바 (8슬롯 고정)

### 3-1. 슬롯 배치

[1 이동][2 공격] | [3 스킬][4 스킬][5 스킬][6 스킬] | [7 궁극][8 VP이탈]

공통 영역 / 구분선 / 클래스 스킬 영역 / 구분선 / 특수 영역

### 3-2. 클래스별 슬롯

| 슬롯 | Fighter | Archer | Rogue | Mage |
|------|---------|--------|-------|------|
| 1 | 이동 | 이동 | 이동 | 이동 |
| 2 | 기본 공격 (melee) | 기본 공격 (ranged) | 기본 공격 (melee) | 기본 공격 (instant) |
| 3 | 방패 강타 (S5) | 조준 사격 (S4) | 연막 (S4) | 화염구 (M8) |
| 4 | 도발 (S3) | 🔒 | 🔒 | 번개 볼트 (M7) |
| 5 | 🔒 | 🔒 | 🔒 | 물 화살 (M5) |
| 6 | 🔒 | 🔒 | 🔒 | 🔒 |
| 7 | 전선 돌파 (VP2) | 화살비 (VP3) | 그림자 일격 (VP2) | 원소 폭풍 (VP3) |
| 8 | VP 이탈 (VP1) | VP 이탈 (VP1) | VP 이탈 (VP1) | VP 이탈 (VP1) |

### 3-3. 슬롯 비주얼 상태

| 상태 | 비주얼 |
|------|--------|
| 활성 | 밝은 배경 + 색상 테두리 |
| 자원 부족 | 어두운 배경 + 비용 빨간색 |
| 쿨다운 중 | 회색 오버레이 + "CD N턴" |
| 조건 미충족 | 반투명 + 회색 아이콘 |
| 잠금 | 거의 투명 + 🔒 |
| 선택됨 | 밝은 테두리 + 펄스 |

### 3-4. 비용 표시 (슬롯 우하단)

S5 = STA 5 (노랑), M8 = MP 8 (파랑), VP2 = VP 2 (금색)

### 3-5. VP 이탈 (슬롯 8) 특수 규칙

교전 중: 활성 (빨간 배경). 비교전: 비활성. VP=0: 비활성 + "VP 부족".

---

## 4. 툴팁

### 4-1. 구조

스킬 호버 0.3초 후 표시:
- 헤더: 스킬 이름 + 타입 뱃지 + 원소 뱃지
- 설명: 1~2줄
- 수치 그리드: 데미지/사거리/비용/쿨다운/(전달)/(원소)
- 효과: CC (게이트 조건 표시) / DoT / 원소 상성 힌트
- 하단: 단축키 + HIT 판정 여부

### 4-2. 원소 상성 힌트

마법 스킬 툴팁에 포함 (element_table.json 동적 로드):
- 💧wet 대상: 데미지 ×0.5, wet 제거 (fire + wet)
- 💧wet 대상: 데미지 ×1.5, shocked 적용 (lightning + wet)
- ❄chilled 대상: frozen 전환 [RES=0 필요] (ice + chilled)

### 4-3. 비활성 사유 표시

스킬 비활성 시 호버하면 사유 표시:
- STA/MP 부족 (필요 N / 현재 N)
- 쿨다운 N턴 남음
- 사거리 밖
- 교전 중 아님 (VP 이탈)

---

## 5. VP 카운터

상단 중앙, 다이아몬드 아이콘:
- 채워진 ◆: 금색 (#F2A623)
- 빈 ◇: 투명 + 금색 테두리
- 최대값: 파티 인원 수

VP 변동 피드백:
- +1 (처치/치명타): ◇→◆ 펄스 + "+1 VP" 플로팅
- -1 (이탈): ◆→◇ 어두워짐 + "-1 VP" 플로팅
- -N (궁극기): 순차 비워짐

---

## 6. 턴 정보

좌상단: "Turn 3/10 — 아군 페이즈"
- 아군: teal (#5DCAA5), 적: coral (#D85A30), 번호: amber (#F2A623)
- 턴 종료 버튼: 아군 페이즈만 활성

---

## 7. 타겟팅 시각화

### 7-1. 3D 오버레이

| 요소 | 색상 | 조건 |
|------|------|------|
| 이동 가능 | 파란 30% | 이동 선택 |
| 공격 가능 | 빨간 30% | 공격/스킬 선택 |
| AoE 범위 | 주황 40% | 광역 스킬 호버 |
| 사각지대/LOS 차단 | 하이라이트 제외 | min_range 내 / 장애물 뒤 |

### 7-2. 예상 결과 팝업

타겟 위 플로팅 (3D→2D 변환):
- 데미지 범위, 적중률(물리만), 치명타%, ARM 흡수량, CC 가능 여부

### 7-3. Archer 사각지대

인접 적 타겟 시: 빨간 X + 기본 공격 비활성 + "사정거리 밖" 툴팁

---

## 8. 교전 이탈 UI

교전 중 이동 시도 시 모달:
- [VP 이탈] VP 1 소모, 기회 공격 없음 (teal, VP≥1 시 활성)
- [강제 이탈] 기회 공격 받음 (빨간)
- [취소] (회색)

---

## 9. 인터랙션 규칙

### 단축키

1~8: 스킬 슬롯 / ESC: 취소 / Space/Enter: 턴 종료 / Tab: 다음 유닛 / F3: 디버그

### 스킬 사용 흐름

유닛 선택 → 스킬 선택 → 사거리 하이라이트 → 타겟 호버(예상 결과) → 타겟 클릭(실행) → 자원 소모 + 쿨다운 + 모션 + 데미지 + 효과 → UI 갱신

---

## 10. JSON 데이터

| 파일 | 내용 |
|------|------|
| data/ui/skill_bar_layout.json | 8슬롯 구조, 타입, 구분선 위치 |
| data/ui/gauge_colors.json | 게이지 색상 + 임계값 |
| data/ui/status_icons.json | 상태이상 11종 아이콘/색상/카테고리 |

---

## 11. Godot 씬 구조

```
CombatUI (CanvasLayer)
├── TopBar
│   ├── TurnInfo
│   ├── VPCounter
│   └── EndTurnButton
├── BottomHUD
│   ├── UnitStatusPanel      ← scripts/ui/UnitStatusPanel.gd
│   ├── SkillBar             ← scripts/ui/SkillBar.gd
│   │   └── SkillSlot × 8   ← scripts/ui/SkillSlot.gd
│   └── TargetStatusPanel
├── SkillTooltip             ← scripts/ui/SkillTooltip.gd
├── DamagePreviewPopup       ← scripts/ui/DamagePreview.gd
└── DisengageModal           ← scripts/ui/DisengageModal.gd
```

---

## 12. 구현 순서 (13단계)

| 단계 | 항목 | 규모 |
|------|------|------|
| UI-1 | TopBar (턴 + VP + 종료 버튼) | Small |
| UI-2 | UnitStatusPanel (게이지 5종 + 상태이상 태그) | Medium |
| UI-3 | SkillBar + SkillSlot (8슬롯 + 상태 표시) | Medium |
| UI-4 | SkillTooltip (호버 상세 정보) | Medium |
| UI-5 | 타겟팅 오버레이 (사거리/LOS/AoE) | Medium |
| UI-6 | DamagePreviewPopup (예상 결과) | Small |
| UI-7 | TargetStatusPanel | Small (재사용) |
| UI-8 | StatusTag (상태이상/원소) | Small |
| UI-9 | DisengageModal | Small |
| UI-10 | 3D 유닛 상태 아이콘 | Medium |
| UI-11 | JSON 데이터 3종 | Small |
| UI-12 | CombatScene 통합 | Large |
| UI-13 | 통합 테스트 | — |

---

## 13. 파일 목록

### 신규 (10개)

scripts/ui/UnitStatusPanel.gd, SkillBar.gd, SkillSlot.gd, SkillTooltip.gd, DamagePreview.gd, DisengageModal.gd
scenes/ui/skill_slot.tscn
data/ui/skill_bar_layout.json, gauge_colors.json, status_icons.json

### 수정 (3개)

combat_scene.tscn — CombatUI 교체
CombatScene.gd — UI 이벤트 연결
UnitRenderer3D.gd — 3D 상태 아이콘
