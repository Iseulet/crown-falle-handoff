# Design: Phase 1-3 Turn System
> 🇰🇷 설계: Phase 1-3 턴 시스템

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Agent: @Planner
- Status: Pending User Approval
- Phase: 1-3 (Turn System)
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-movement_system.md` (Phase 1-2)
> 🇰🇷 Phase 1-2 설계 기반으로 확장

---

## Overview
> 🇰🇷 개요

Phase 1-3 adds a turn state machine (ally → enemy alternating), per-unit move limits,
an End Turn button, and a turn indicator label.
Enemy turn has no AI — it auto-skips to the next ally turn.
> 🇰🇷 아군↔적군 교대 턴 상태 머신, 유닛별 이동 제한, 턴 종료 버튼, 턴 표시 UI 추가.
> 🇰🇷 적군 턴은 AI 없이 자동 스킵.

---

## 1. Turn Rules (from concept doc)
> 🇰🇷 턴 규칙 (컨셉 문서 기준)

- **Order:** Ally → Enemy → Ally → … (alternating)
- **Limit:** 10 turns per combat (MAX_TURNS constant)
- **Per unit:** Each ally can move once per ally turn (`has_moved` flag)
- **End conditions:** Turn limit exceeded → defeat message (Phase 1-4 handles full win/lose)
> 🇰🇷 순서: 아군 → 적군 → 반복
> 🇰🇷 제한: 최대 10턴 (MAX_TURNS)
> 🇰🇷 유닛별: 아군 유닛당 1회 이동 (has_moved 플래그)
> 🇰🇷 종료 조건: 턴 초과 시 메시지 출력 (승/패 처리는 Phase 1-4)

---

## 2. Turn Flow
> 🇰🇷 턴 흐름

```
start_combat()
│
└── Ally Turn (current_turn = 1)
    │  Player selects and moves allies (has_moved blocks re-move)
    │  Player clicks "End Turn"
    └── Enemy Turn (auto-skip — no AI)
        └── Ally Turn (current_turn = 2)
            │  ...
            └── (repeat until current_turn > MAX_TURNS)
                └── combat_ended("turn_limit") → print message
```

> 🇰🇷 start_combat → 아군 턴 → 턴 종료 버튼 → 적군 턴 (자동 스킵) → 아군 턴 반복

---

## 3. New File: TurnManager.gd
> 🇰🇷 신규 파일: 턴 관리자

**Path:** `scripts/combat/TurnManager.gd`
**Attached to:** TurnManager (Node, child of CombatScene)

```gdscript
class_name TurnManager
extends Node

const MAX_TURNS: int = 10

enum Phase { ALLY_TURN, ENEMY_TURN }

var current_turn: int = 1
var current_phase: Phase = Phase.ALLY_TURN

signal phase_changed(phase: Phase)
signal turn_changed(turn: int)
signal combat_ended(reason: String)  # "turn_limit"

func start_combat() -> void
    # initialize turn 1, ally phase, emit signals

func end_ally_turn() -> void
    # guard: only if ALLY_TURN
    # switch to ENEMY_TURN, emit phase_changed
    # immediately call _end_enemy_turn()

func _end_enemy_turn() -> void
    # increment current_turn
    # if current_turn > MAX_TURNS → emit combat_ended("turn_limit")
    # else → switch to ALLY_TURN, emit phase_changed + turn_changed
```

> 🇰🇷 턴 상태 머신. 신호: phase_changed / turn_changed / combat_ended

---

## 4. Scene Changes: combat_scene.tscn
> 🇰🇷 씬 변경

Add to scene tree:
```
CombatScene (Node2D)
├── TurnManager (Node)          ← NEW: TurnManager.gd attached
├── GridManager (Node2D)
│   └── ...
├── UnitContainer (Node2D)
│   └── ...
└── CombatUI (CanvasLayer)
    └── HUD (VBoxContainer)     ← NEW
        ├── TurnLabel (Label)   ← NEW: "Turn 1 / 10 — Ally Phase"
        └── EndTurnButton (Button) ← NEW: "End Turn"
```

> 🇰🇷 TurnManager 노드 + HUD(VBoxContainer) + TurnLabel + EndTurnButton 추가

---

## 5. CombatScene.gd Changes
> 🇰🇷 CombatScene.gd 변경

**New @onready refs:**
```gdscript
@onready var turn_manager: TurnManager = $TurnManager
@onready var turn_label: Label = $CombatUI/HUD/TurnLabel
@onready var end_turn_button: Button = $CombatUI/HUD/EndTurnButton
```

**_ready() additions:**
```gdscript
# connect turn signals
turn_manager.phase_changed.connect(_on_phase_changed)
turn_manager.turn_changed.connect(_on_turn_changed)
turn_manager.combat_ended.connect(_on_combat_ended)
# connect button
end_turn_button.pressed.connect(_on_end_turn_button_pressed)
# start combat
turn_manager.start_combat()
```

**_unhandled_input() change:**
```gdscript
# Block input during enemy turn
if turn_manager.current_phase == TurnManager.Phase.ENEMY_TURN:
    return
```

**_select_unit() change:**
```gdscript
# Block re-move: only select if not has_moved
if unit.has_moved:
    return
```

**_move_selected_unit() change:**
```gdscript
selected_unit.has_moved = true  # add after move
```

**New functions:**
```gdscript
func _on_end_turn_button_pressed() -> void
    # deselect, call turn_manager.end_ally_turn()

func _on_phase_changed(phase: TurnManager.Phase) -> void
    # update turn_label text
    # toggle end_turn_button disabled state

func _on_turn_changed(turn: int) -> void
    # update turn_label turn number
    # reset has_moved = false for all ally units

func _on_combat_ended(reason: String) -> void
    # print/show "Turn limit reached" message
    # disable end_turn_button

func _reset_ally_moves() -> void
    # iterate ally_container children, set has_moved = false
```

---

## 6. Files Modified
> 🇰🇷 변경 파일 목록

| File | Change |
|------|--------|
| `scripts/combat/TurnManager.gd` | NEW — turn state machine |
| `scenes/combat/combat_scene.tscn` | ADD TurnManager node + HUD + TurnLabel + EndTurnButton |
| `scripts/combat/CombatScene.gd` | ADD turn integration, has_moved checks, UI refs |

> 🇰🇷 신규 1개 + 수정 2개

---

## 7. Out of Scope for Phase 1-3
> 🇰🇷 Phase 1-3 범위 외

- Enemy AI movement → Phase 1-4
- Win condition (all enemies dead) → Phase 1-4
- HP / attack → Phase 1-4
- Visual turn transition effects → future
> 🇰🇷 적 AI, 승리 조건, HP/공격, 시각 효과 제외

---

## 8. Acceptance Criteria
> 🇰🇷 완료 기준

- [ ] 화면 좌상단에 "Turn 1 / 10 — Ally Phase" 표시
- [ ] 아군 유닛은 턴당 1회만 이동 가능 (이동 후 선택 불가)
- [ ] "End Turn" 버튼 클릭 시 "Enemy Phase" 표시 후 즉시 "Ally Phase"로 전환
- [ ] 턴 카운터 1씩 증가 확인
- [ ] 10턴 초과 시 "Turn limit reached" 출력
- [ ] 적군 턴 중 아군 클릭 불가
- [ ] Godot 에러 없음

---

## 9. Next Step
> 🇰🇷 다음 단계

→ `@Implementor` implements based on this document.
→ Recommended tool: Claude Code
> 🇰🇷 승인 후 @Implementor 구현. 권장 도구: Claude Code
