# Design: Phase 2-3 Level Up System
> 🇰🇷 설계: Phase 2-3 레벨업 시스템

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Agent: @Planner
- Status: Pending User Approval
- Phase: 2-3 (Level Up System)
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-character_data.md` (Phase 2-1)

---

## Overview
> 🇰🇷 개요

Phase 2-3 adds experience gain on kill, level up detection, and a stat selection UI.
When a unit kills an enemy, it gains XP. On reaching the threshold, a popup
pauses combat and lets the player choose one stat to increase.
> 🇰🇷 적 처치 시 경험치 획득 → 임계값 도달 시 레벨업 팝업 표시.
> 🇰🇷 플레이어가 능력치 하나를 선택하면 팝업 닫힘, 전투 재개.

---

## 1. Prototype Rules
> 🇰🇷 프로토타입 규칙

| Item | Value |
|------|-------|
| XP per kill | 10 |
| XP threshold | `level × 10` (level 1→2: 10XP, level 2→3: 20XP) |
| Stat choices | +5 max_hp / +1 str / +1 move_range |

> 🇰🇷 킬당 XP: 10 / 임계값: 레벨 × 10 / 선택지: HP+5, STR+1, MV+1

---

## 2. MercenaryData.gd Changes
> 🇰🇷 MercenaryData.gd 변경

**Add:**
```gdscript
@export var level: int = 1
@export var xp: int = 0

func xp_to_next_level() -> int:
    return level * 10
```

---

## 3. CombatUnit.gd Changes
> 🇰🇷 CombatUnit.gd 변경

**New signal:**
```gdscript
signal unit_leveled_up(unit: CombatUnit)
```

**New function:**
```gdscript
func gain_xp(amount: int) -> void:
    data.xp += amount
    if data.xp >= data.xp_to_next_level():
        data.xp -= data.xp_to_next_level()
        data.level += 1
        unit_leveled_up.emit(self)
```

---

## 4. CombatScene.gd Changes
> 🇰🇷 CombatScene.gd 변경

**New @onready refs:**
```gdscript
@onready var level_up_panel: PanelContainer = $CombatUI/LevelUpPanel
@onready var level_up_label: Label = $CombatUI/LevelUpPanel/VBoxContainer/TitleLabel
@onready var hp_button: Button = $CombatUI/LevelUpPanel/VBoxContainer/HPButton
@onready var str_button: Button = $CombatUI/LevelUpPanel/VBoxContainer/STRButton
@onready var mv_button: Button = $CombatUI/LevelUpPanel/VBoxContainer/MVButton
```

**New variable:**
```gdscript
var _leveling_unit: CombatUnit = null
```

**_ready() additions:**
```gdscript
hp_button.pressed.connect(_on_levelup_hp)
str_button.pressed.connect(_on_levelup_str)
mv_button.pressed.connect(_on_levelup_mv)
```

**_spawn_unit() addition:**
```gdscript
unit.unit_leveled_up.connect(_on_unit_leveled_up)
```

**_attack_target() change:**
```gdscript
func _attack_target(target_pos: Vector2i) -> void:
    var target := ...
    var attacker := selected_unit   # save before deselect
    attacker.has_attacked = true
    target.take_damage(attacker.str)
    if target.current_hp == 0:      # killed → award XP
        attacker.gain_xp(10)
    _deselect_unit()
```

**New functions:**
```gdscript
func _on_unit_leveled_up(unit: CombatUnit) -> void:
    _leveling_unit = unit
    level_up_label.text = "%s  Lv.%d!" % [unit.unit_name, unit.data.level]
    level_up_panel.visible = true
    set_process_unhandled_input(false)

func _apply_levelup_choice() -> void:
    level_up_panel.visible = false
    _leveling_unit = null
    set_process_unhandled_input(true)

func _on_levelup_hp() -> void:
    _leveling_unit.data.max_hp += 5
    _leveling_unit.current_hp += 5
    _leveling_unit.max_hp += 5
    _leveling_unit._update_hp_label()
    _apply_levelup_choice()

func _on_levelup_str() -> void:
    _leveling_unit.data.str += 1
    _leveling_unit.str += 1
    _apply_levelup_choice()

func _on_levelup_mv() -> void:
    _leveling_unit.data.move_range += 1
    _leveling_unit.move_range += 1
    _apply_levelup_choice()
```

---

## 5. combat_scene.tscn Changes (LevelUpPanel)
> 🇰🇷 combat_scene.tscn 변경

```
CombatUI (CanvasLayer)
├── HUD
├── WarbandPanel
└── LevelUpPanel (PanelContainer)   ← NEW
    offset_left = 300, offset_top = 150
    custom_minimum_size = Vector2(200, 0)
    visible = false
    └── VBoxContainer
        ├── TitleLabel (Label)   text = "레벨 업!"
        ├── SubLabel (Label)     text = "능력치를 선택하세요"
        ├── HPButton (Button)    text = "+5 HP"
        ├── STRButton (Button)   text = "+1 STR"
        └── MVButton (Button)    text = "+1 MV"
```

---

## 6. Files Modified
> 🇰🇷 변경 파일 목록

| File | Change |
|------|--------|
| `scripts/data/MercenaryData.gd` | MODIFY — level, xp, xp_to_next_level() 추가 |
| `scripts/combat/CombatUnit.gd` | MODIFY — gain_xp(), unit_leveled_up 신호 추가 |
| `scripts/combat/CombatScene.gd` | MODIFY — XP 지급, 레벨업 패널 처리 |
| `scenes/combat/combat_scene.tscn` | MODIFY — LevelUpPanel UI 노드 추가 |

> 🇰🇷 신규 없음. 수정 4개.

---

## 7. Acceptance Criteria
> 🇰🇷 완료 기준

- [ ] 적 처치 시 공격 유닛 XP 10 획득
- [ ] XP 임계값 도달 시 레벨업 팝업 표시, 전투 입력 차단
- [ ] HP+5 선택 시 해당 유닛 HP 증가 + HP 레이블 갱신
- [ ] STR+1 선택 시 해당 유닛 STR 증가
- [ ] MV+1 선택 시 해당 유닛 이동 범위 증가
- [ ] 선택 후 팝업 닫힘, 전투 재개
- [ ] Godot 에러 없음

---

## 8. Next Step
> 🇰🇷 다음 단계

→ `@Implementor` implements based on this document.
→ Recommended tool: Claude Code
