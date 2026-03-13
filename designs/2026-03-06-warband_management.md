# Design: Phase 2-2 Warband Management
> 🇰🇷 설계: Phase 2-2 용병단 관리

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Agent: @Planner
- Status: Pending User Approval
- Phase: 2-2 (Warband Management)
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-character_data.md` (Phase 2-1)

---

## Overview
> 🇰🇷 개요

Phase 2-2 adds a `WarbandManager` node that owns the player's mercenary roster.
CombatScene reads from WarbandManager (instead of hardcoded ALLY_DATA) to spawn allies.
A toggle panel in the HUD shows the roster list.
> 🇰🇷 WarbandManager 노드가 용병 로스터를 소유.
> 🇰🇷 CombatScene은 하드코딩 대신 WarbandManager에서 아군 데이터 읽어 스폰.
> 🇰🇷 HUD에 토글 패널로 로스터 목록 표시.

---

## 1. WarbandManager.gd
> 🇰🇷 WarbandManager 스크립트

**File:** `scripts/data/WarbandManager.gd`

```gdscript
class_name WarbandManager
extends Node

var roster: Array[MercenaryData] = []

func _ready() -> void:
    roster = [
        load("res://data/mercenaries/fighter_01.tres"),
        load("res://data/mercenaries/archer_01.tres"),
        load("res://data/mercenaries/rogue_01.tres"),
    ]
```

- `roster`: player's active mercenary list
- Populated on `_ready()` from .tres files
- Phase 3+ will replace with persistent save data
> 🇰🇷 roster: 플레이어 용병 목록
> 🇰🇷 Phase 3+에서 영속 저장 데이터로 교체 예정

---

## 2. UI Layout (combat_scene.tscn)
> 🇰🇷 UI 레이아웃

```
CombatScene (Node2D)
├── WarbandManager (Node)      ← NEW
├── ...
└── CombatUI (CanvasLayer)
    ├── HUD (VBoxContainer)
    │   ├── TurnLabel
    │   ├── EndTurnButton
    │   └── WarbandButton      ← NEW ("용병단")
    └── WarbandPanel (PanelContainer)  ← NEW
        └── VBoxContainer
            └── (labels populated by code)
```

WarbandPanel properties:
- `offset_left = 200`, `offset_top = 10` (right of HUD)
- `visible = false` by default
- `custom_minimum_size = Vector2(220, 0)`
> 🇰🇷 WarbandPanel: 기본 숨김, 버튼 클릭 시 토글
> 🇰🇷 HUD 우측에 배치

---

## 3. WarbandPanel Content
> 🇰🇷 패널 표시 내용

Each roster entry displays one line:
```
[Name]  [Class]  HP:[max_hp]  STR:[str]  MV:[move_range]
예: Fighter_01  Fighter  HP:30  STR:8  MV:3
```

Populated programmatically by CombatScene on `_ready()`.
> 🇰🇷 CombatScene._ready()에서 코드로 레이블 생성

---

## 4. CombatScene.gd Changes
> 🇰🇷 CombatScene.gd 변경

**New @onready refs:**
```gdscript
@onready var warband_manager: WarbandManager = $WarbandManager
@onready var warband_button: Button = $CombatUI/HUD/WarbandButton
@onready var warband_panel: PanelContainer = $CombatUI/WarbandPanel
@onready var warband_list: VBoxContainer = $CombatUI/WarbandPanel/VBoxContainer
```

**Remove ALLY_DATA / ENEMY_DATA constants** — replaced by warband_manager.roster

**_spawn_units() change:**
```gdscript
func _spawn_units() -> void:
    for i in range(warband_manager.roster.size()):
        _spawn_unit(ALLY_SPAWN_POSITIONS[i], warband_manager.roster[i], true, ally_container)
    for i in range(ENEMY_SPAWN_POSITIONS.size()):
        var d := load(ENEMY_DATA[i]) as MercenaryData
        _spawn_unit(ENEMY_SPAWN_POSITIONS[i], d, false, enemy_container)
```

Note: ENEMY_DATA remains hardcoded (enemies not player-managed)
> 🇰🇷 아군은 WarbandManager에서, 적군은 하드코딩 유지

**New in _ready():**
```gdscript
warband_button.pressed.connect(_on_warband_button_pressed)
_populate_warband_panel()
```

**New functions:**
```gdscript
func _on_warband_button_pressed() -> void:
    warband_panel.visible = not warband_panel.visible

func _populate_warband_panel() -> void:
    for entry in warband_manager.roster:
        var label := Label.new()
        var class_name_str := MercenaryData.UnitClass.keys()[entry.unit_class]
        label.text = "%s  %s  HP:%d  STR:%d  MV:%d" % [
            entry.unit_name, class_name_str,
            entry.max_hp, entry.str, entry.move_range
        ]
        warband_list.add_child(label)
```

---

## 5. Files Modified / Created
> 🇰🇷 변경/신규 파일 목록

| File | Change |
|------|--------|
| `scripts/data/WarbandManager.gd` | NEW — 용병단 로스터 관리 |
| `scenes/combat/combat_scene.tscn` | MODIFY — WarbandManager 노드 + UI 추가 |
| `scripts/combat/CombatScene.gd` | MODIFY — WarbandManager 연동, 패널 토글 |

> 🇰🇷 신규 1개 + 수정 2개 = 총 3개

---

## 6. Acceptance Criteria
> 🇰🇷 완료 기준

- [ ] "용병단" 버튼 클릭 시 패널 표시/숨김 토글
- [ ] 패널에 Fighter_01 / Archer_01 / Rogue_01 목록 표시
- [ ] 각 항목에 직업명 / HP / STR / MV 표시
- [ ] 아군 스폰이 WarbandManager.roster 기준으로 동작
- [ ] Godot 에러 없음

---

## 7. Next Step
> 🇰🇷 다음 단계

→ `@Implementor` implements based on this document.
→ Recommended tool: Claude Code
> 🇰🇷 승인 후 @Implementor 구현. 권장 도구: Claude Code
