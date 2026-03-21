# Design: Phase 2-1 Character Data
> 🇰🇷 설계: Phase 2-1 캐릭터 데이터

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Agent: @Planner
- Status: Pending User Approval
- Phase: 2-1 (Character Data)
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-crownfalle-roadmap.md`

---

## Overview
> 🇰🇷 개요

Phase 2-1 introduces `MercenaryData` — a Godot Resource holding per-unit stats.
CombatUnit.gd is updated to read stats from MercenaryData instead of hardcoded values.
CombatScene.gd loads .tres files to spawn units with class-differentiated stats.
> 🇰🇷 MercenaryData Resource로 유닛 스탯을 외부 데이터로 분리.
> 🇰🇷 CombatUnit은 하드코딩 대신 MercenaryData를 참조.
> 🇰🇷 CombatScene은 .tres 파일을 로드해 유닛 스폰.

---

## 1. MercenaryData Resource
> 🇰🇷 MercenaryData 리소스 구조

**File:** `scripts/data/MercenaryData.gd`

```gdscript
class_name MercenaryData
extends Resource

enum UnitClass { FIGHTER, ARCHER, ROGUE, MAGE }

@export var unit_name: String = ""
@export var unit_class: UnitClass = UnitClass.FIGHTER
@export var max_hp: int = 20
@export var str: int = 5
@export var move_range: int = 4
```

| Property | Type | Description |
|----------|------|-------------|
| `unit_name` | String | Display name |
| `unit_class` | UnitClass (enum) | Fighter / Archer / Rogue / Mage |
| `max_hp` | int | Maximum hit points |
| `str` | int | Attack power |
| `move_range` | int | Movement tiles per turn |

> 🇰🇷 모든 속성은 @export로 Godot 에디터에서 편집 가능

---

## 2. Class Base Stats (Prototype)
> 🇰🇷 직업별 기본 스탯 (프로토타입)

| Class | max_hp | str | move_range |
|-------|--------|-----|------------|
| Fighter (전사) | 30 | 8 | 3 |
| Archer (궁수) | 20 | 5 | 4 |
| Rogue (도적) | 15 | 6 | 5 |
| Mage (마법사) | 15 | 10 | 3 |

Concept doc reference:
- Fighter: HP High / STR High / MV Low → 30 / 8 / 3
- Archer: HP Medium / STR Medium / MV Medium → 20 / 5 / 4
- Rogue: HP Low / STR Medium / MV High → 15 / 6 / 5
- Mage: HP Low / STR High / MV Low → 15 / 10 / 3

> 🇰🇷 컨셉 문서의 High/Medium/Low 기준으로 수치화

---

## 3. Resource File Storage
> 🇰🇷 Resource 파일 저장 경로

```
res://data/
└── mercenaries/
    ├── fighter_01.tres
    ├── archer_01.tres
    ├── rogue_01.tres
    └── mage_01.tres
```

Sample `.tres` content (fighter_01.tres):
```
[gd_resource type="Resource" script_class="MercenaryData" format=3]
[ext_resource type="Script" path="res://scripts/data/MercenaryData.gd" id="1"]
[resource]
script = ExtResource("1")
unit_name = "Fighter_01"
unit_class = 0
max_hp = 30
str = 8
move_range = 3
```

> 🇰🇷 data/mercenaries/ 신규 폴더 생성 필요
> 🇰🇷 각 직업당 샘플 .tres 파일 1개씩 생성

---

## 4. CombatUnit.gd Changes
> 🇰🇷 CombatUnit.gd 변경

**Add variable:**
```gdscript
var data: MercenaryData = null
```

**initialize() signature change:**
```gdscript
# Before:
func initialize(start_pos: Vector2i, p_unit_name: String, p_is_ally: bool) -> void

# After:
func initialize(start_pos: Vector2i, p_data: MercenaryData, p_is_ally: bool) -> void
    # data = p_data
    # unit_name = p_data.unit_name
    # max_hp = p_data.max_hp
    # current_hp = p_data.max_hp
    # str = p_data.str
    # move_range = p_data.move_range
    # is_ally = p_is_ally
    # _polygon.color = ...
    # _update_hp_label()
```

> 🇰🇷 하드코딩 스탯 제거 → MercenaryData에서 읽음

---

## 5. CombatScene.gd Changes
> 🇰🇷 CombatScene.gd 변경

**Spawn resources (preload):**
```gdscript
const ALLY_DATA: Array[String] = [
    "res://data/mercenaries/fighter_01.tres",
    "res://data/mercenaries/archer_01.tres",
    "res://data/mercenaries/rogue_01.tres",
]
const ENEMY_DATA: Array[String] = [
    "res://data/mercenaries/fighter_01.tres",
    "res://data/mercenaries/fighter_01.tres",
    "res://data/mercenaries/fighter_01.tres",
]
```

**_spawn_unit() signature change:**
```gdscript
# Before:
func _spawn_unit(pos: Vector2i, p_name: String, p_is_ally: bool, container: Node2D) -> void

# After:
func _spawn_unit(pos: Vector2i, p_data: MercenaryData, p_is_ally: bool, container: Node2D) -> void
    # unit.initialize(pos, p_data, p_is_ally)
```

**_spawn_units() change:**
```gdscript
func _spawn_units() -> void:
    for i in range(ALLY_SPAWN_POSITIONS.size()):
        var d := load(ALLY_DATA[i]) as MercenaryData
        _spawn_unit(ALLY_SPAWN_POSITIONS[i], d, true, ally_container)
    for i in range(ENEMY_SPAWN_POSITIONS.size()):
        var d := load(ENEMY_DATA[i]) as MercenaryData
        _spawn_unit(ENEMY_SPAWN_POSITIONS[i], d, false, enemy_container)
```

> 🇰🇷 스폰 시 .tres 로드 → MercenaryData 전달

---

## 6. Files Modified / Created
> 🇰🇷 변경/신규 파일 목록

| File | Change |
|------|--------|
| `scripts/data/MercenaryData.gd` | NEW — Resource 스크립트 |
| `data/mercenaries/fighter_01.tres` | NEW — 전사 샘플 데이터 |
| `data/mercenaries/archer_01.tres` | NEW — 궁수 샘플 데이터 |
| `data/mercenaries/rogue_01.tres` | NEW — 도적 샘플 데이터 |
| `data/mercenaries/mage_01.tres` | NEW — 마법사 샘플 데이터 |
| `scripts/combat/CombatUnit.gd` | MODIFY — initialize() 시그니처 변경 |
| `scripts/combat/CombatScene.gd` | MODIFY — 스폰 로직 변경 |

> 🇰🇷 신규 4개 + 수정 2개 = 총 7개

---

## 7. Acceptance Criteria
> 🇰🇷 완료 기준

- [ ] Fighter 유닛 HP=30, STR=8, MV=3으로 스폰
- [ ] Archer 유닛 HP=20, STR=5, MV=4로 스폰
- [ ] Rogue 유닛 HP=15, STR=6, MV=5로 스폰
- [ ] HP 레이블이 각 직업 수치로 표시 (예: "30/20")
- [ ] 이동 범위가 직업별로 다르게 적용
- [ ] Godot 에러 없음

---

## 8. Next Step
> 🇰🇷 다음 단계

→ `@Implementor` implements based on this document.
→ Recommended tool: Claude Code
> 🇰🇷 승인 후 @Implementor 구현. 권장 도구: Claude Code
