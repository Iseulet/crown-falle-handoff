# Design: Phase 1-4 Combat Action
> 🇰🇷 설계: Phase 1-4 전투 액션

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Agent: @Planner
- Status: Pending User Approval
- Phase: 1-4 (Combat Action)
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-turn_system.md` (Phase 1-3)
> 🇰🇷 Phase 1-3 설계 기반으로 확장

---

## Overview
> 🇰🇷 개요

Phase 1-4 adds attack actions, HP stats, damage calculation, HP label display,
unit death handling, and win/lose conditions.
No new scripts — 4 existing files modified.
> 🇰🇷 공격 액션, HP 스탯, 데미지 계산, HP 레이블, 유닛 사망, 승/패 조건 추가.
> 🇰🇷 신규 스크립트 없음. 기존 파일 4개 수정.

---

## 1. Prototype Stats
> 🇰🇷 프로토타입 스탯

All placeholder units share the same flat stats for Phase 1-4:
> 🇰🇷 Phase 1-4는 모든 유닛에 동일한 기본값 사용

| Stat | Value | Description |
|------|-------|-------------|
| `max_hp` | 20 | Maximum hit points |
| `str` | 5 | Attack power (damage dealt) |
| `move_range` | 4 | (unchanged from Phase 1-2) |

Damage formula: `damage = attacker.str` (no defense stat in Phase 1-4)
> 🇰🇷 데미지 공식: 공격자의 STR (방어 스탯은 Phase 1-4 범위 외)

---

## 2. Action Economy
> 🇰🇷 행동 경제

Per ally unit per turn:
- Move: once (`has_moved`)
- Attack: once (`has_attacked`)
- Can do both, either, or neither in any order
> 🇰🇷 유닛당 턴 1회 이동 + 1회 공격 (순서 자유)

---

## 3. Input Flow (extended from Phase 1-2/1-3)
> 🇰🇷 입력 흐름 (Phase 1-2/1-3 확장)

```
Click ally unit (not has_moved OR not has_attacked):
  → Select
  → Show BLUE highlights (move range, if not has_moved)
  → Show RED highlights (adjacent enemies, if not has_attacked)

Click BLUE tile:
  → Move unit
  → Refresh RED highlights only (move consumed)

Click RED tile (adjacent enemy):
  → Attack enemy
  → Apply damage → check death
  → Deselect

Click elsewhere:
  → Deselect, clear all highlights
```

> 🇰🇷 선택 → 파랑(이동)/빨강(공격 대상) 동시 표시
> 🇰🇷 파랑 클릭 → 이동 후 빨강만 갱신
> 🇰🇷 빨강 클릭 → 공격 + 사망 처리 + 선택 해제

---

## 4. CombatUnit.gd Changes
> 🇰🇷 CombatUnit.gd 변경

**New variables:**
```gdscript
var max_hp: int = 20
var current_hp: int = 20
var str: int = 5
var has_attacked: bool = false

@onready var _hp_label: Label = $HPLabel
```

**New signals:**
```gdscript
signal unit_died
```

**New functions:**
```gdscript
func take_damage(amount: int) -> void
    # current_hp -= amount
    # clamp to 0
    # update _hp_label
    # if current_hp <= 0 → emit unit_died

func _update_hp_label() -> void
    # _hp_label.text = "%d/%d" % [current_hp, max_hp]
```

**initialize() change:**
```gdscript
# Add: _update_hp_label()
```

**_reset_for_new_turn() — called by CombatScene._reset_ally_moves():**
```gdscript
# Add has_attacked = false reset alongside has_moved
```
> 🇰🇷 has_attacked도 턴 시작 시 리셋

---

## 5. combat_unit.tscn Changes
> 🇰🇷 combat_unit.tscn 변경

Add `HPLabel` (Label) as child of CombatUnit:
```
CombatUnit (Node2D)
├── Polygon2D       ← existing
└── HPLabel (Label) ← NEW, position (−20, −35)
```

> 🇰🇷 유닛 위에 HP 표시 레이블 추가

---

## 6. GridManager.gd Changes
> 🇰🇷 GridManager.gd 변경

```gdscript
const TILE_HIGHLIGHT_ATTACK: int = 3  # red semi-transparent

# Add to _create_tile_set():
# attack highlight — red semi-transparent diamond (Color(1.0, 0.2, 0.2, 0.5))

func highlight_attack_range(cells: Array[Vector2i]) -> void
    # sets TILE_HIGHLIGHT_ATTACK on HighlightLayer for each cell
    # (alongside existing move highlights — both layers share HighlightLayer)

# clear_highlights() already clears all — no change needed
```

> 🇰🇷 공격 범위 빨간 하이라이트 추가

---

## 7. CombatScene.gd Changes
> 🇰🇷 CombatScene.gd 변경

**New variables:** none

**_unhandled_input() change:**
```gdscript
# Extend hit detection:
# if click is on RED highlighted tile → call _attack_target(grid_pos)
# Guard: only select unit if not (has_moved AND has_attacked)
```

**_select_unit() change:**
```gdscript
# After highlighting move range, also call:
# _refresh_attack_highlights()
```

**_move_selected_unit() change:**
```gdscript
# After move, call _refresh_attack_highlights()
# so red highlights update to new position
```

**New functions:**
```gdscript
func _get_attack_targets(unit: CombatUnit) -> Array[Vector2i]
    # returns adjacent grid positions occupied by enemy units

func _refresh_attack_highlights() -> void
    # if selected_unit != null and not selected_unit.has_attacked:
    #   highlight_attack_range(_get_attack_targets(selected_unit))

func _attack_target(target_pos: Vector2i) -> void
    # get target unit at pos
    # selected_unit.has_attacked = true
    # target.take_damage(selected_unit.str)
    # _deselect_unit()
    # (death handled via unit_died signal)

func _on_unit_died(unit: CombatUnit) -> void
    # grid_manager.clear_occupied(unit.grid_pos)
    # unit.queue_free()
    # _check_combat_end()

func _check_combat_end() -> void
    # if enemy_container.get_child_count() == 0 → victory
    # if ally_container.get_child_count() == 0 → defeat
    # call turn_manager.combat_ended or handle directly

func _on_combat_ended(reason: String) -> void  # extend existing
    # add "victory" and "defeat" cases to label update
```

**_spawn_unit() change:**
```gdscript
# Connect unit.unit_died signal → _on_unit_died
```

**_reset_ally_moves() change:**
```gdscript
# Also reset has_attacked = false for all allies
```

---

## 8. Files Modified
> 🇰🇷 변경 파일 목록

| File | Change |
|------|--------|
| `scripts/combat/CombatUnit.gd` | Add max_hp, current_hp, str, has_attacked, take_damage, unit_died, HP label |
| `scenes/combat/combat_unit.tscn` | Add HPLabel node |
| `scripts/combat/GridManager.gd` | Add TILE_HIGHLIGHT_ATTACK, red highlight tile, highlight_attack_range |
| `scripts/combat/CombatScene.gd` | Add attack flow, death handling, win/lose, attack highlights |

> 🇰🇷 신규 파일 없음. 기존 4개 수정.

---

## 9. Acceptance Criteria
> 🇰🇷 완료 기준

- [ ] 유닛 선택 시 파란 하이라이트(이동) + 빨간 하이라이트(인접 적) 동시 표시
- [ ] 이동 후 빨간 하이라이트 새 위치 기준으로 갱신
- [ ] 빨간 하이라이트 클릭 시 데미지 적용, HP 레이블 감소
- [ ] HP 0 도달 시 유닛 제거
- [ ] 적 전멸 시 "Victory!" 표시
- [ ] 아군 전멸 시 "Defeat!" 표시
- [ ] Godot 에러 없음

---

## 10. Next Step
> 🇰🇷 다음 단계

→ `@Implementor` implements based on this document.
→ Recommended tool: Claude Code
> 🇰🇷 승인 후 @Implementor 구현. 권장 도구: Claude Code
