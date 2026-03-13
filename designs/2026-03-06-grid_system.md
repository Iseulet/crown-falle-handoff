# Design: Phase 1-1 Grid System
> 🇰🇷 설계: Phase 1-1 격자 시스템

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Agent: @Planner
- Status: Pending User Approval
- Phase: 1-1 (Grid System)
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-crownfalle-roadmap.md`
  - `handoff/DECISIONS.md`
> 🇰🇷 생성일: 2026-03-06 | 에이전트: @Planner | 상태: 사용자 승인 대기

---

## Overview
> 🇰🇷 개요

This document defines the grid system for CrownFalle's combat scene.
The grid is the foundation of all combat mechanics (movement, combat, turn order).
Phase 1-1 scope: grid generation + unit placement only.
AStar2D pathfinding and highlight are Phase 1-2 scope.
> 🇰🇷 전투 씬의 격자 시스템 설계. 이동/전투/턴 순서의 기반.
> 🇰🇷 Phase 1-1 범위: 격자 생성 + 유닛 배치만. AStar2D/하이라이트는 Phase 1-2.

---

## Godot Version Note
> 🇰🇷 Godot 버전 참고

Godot 4.3+ deprecates `TileMap` in favor of `TileMapLayer`.
All tile layers in this design use `TileMapLayer` nodes directly.
> 🇰🇷 Godot 4.3+에서 TileMap은 deprecated.
> 🇰🇷 이 설계는 TileMapLayer 노드를 직접 사용.

---

## 1. Tile Configuration
> 🇰🇷 타일 설정

### Tile Shape
> 🇰🇷 타일 형태

- **Tile shape:** Isometric (Godot TileSet → Tile Shape: Isometric)
- **Tile size:** 128 x 64 px (diamond bounding box, 2:1 ratio)
- **Grid type:** Square grid logic (x, y coordinates), isometric rendering
> 🇰🇷 타일 형태: 아이소메트릭 (다이아몬드)
> 🇰🇷 타일 크기: 128 x 64 px
> 🇰🇷 격자 로직: 사각형 좌표계 (x, y), 아이소메트릭으로 렌더링

### Grid Size
> 🇰🇷 격자 크기

- **Default:** 10 x 10 tiles
- **Configurable:** via `GridManager.GRID_WIDTH` / `GridManager.GRID_HEIGHT` constants
> 🇰🇷 기본 크기: 10 x 10
> 🇰🇷 GridManager 상수로 조정 가능

### Prototype Tiles
> 🇰🇷 프로토타입 타일

Two tile types for prototype (colored flat polygons, no art required):
- **floor** — walkable terrain (light gray)
- **wall** — impassable obstacle (dark gray)
> 🇰🇷 프로토타입용 타일 2종 (단색 다각형, 아트 불필요)
> 🇰🇷 floor: 이동 가능 (밝은 회색) | wall: 이동 불가 (어두운 회색)

---

## 2. Scene Structure (Node Tree)
> 🇰🇷 씬 구조 (노드 트리)

```
CombatScene (Node2D)          ← scenes/combat/combat_scene.tscn
│   script: CombatScene.gd
│
├── GridManager (Node2D)      ← grid root, holds all TileMapLayer nodes
│   │   script: GridManager.gd
│   │
│   ├── GroundLayer (TileMapLayer)   ← walkable floor tiles
│   ├── ObstacleLayer (TileMapLayer) ← impassable wall tiles
│   └── HighlightLayer (TileMapLayer)← movement/attack highlights (Phase 1-2)
│
├── UnitContainer (Node2D)    ← parent of all unit instances
│   ├── AllyUnits (Node2D)    ← player-controlled units
│   └── EnemyUnits (Node2D)   ← AI-controlled units
│
└── CombatUI (CanvasLayer)    ← HUD (HP bars, turn info — Phase 1-3/1-4)
```

> 🇰🇷 CombatScene: 최상위 씬 컨트롤러
> 🇰🇷 GridManager: 격자 로직 + TileMapLayer 3개 관리
> 🇰🇷 UnitContainer: 모든 유닛 인스턴스의 부모
> 🇰🇷 CombatUI: HUD (Phase 1-3/1-4에서 구현)

---

## 3. Unit Placement
> 🇰🇷 유닛 배치 방식

Units are **not** placed on TileMapLayer. They are independent Node2D instances
positioned at world coordinates calculated from grid positions.
> 🇰🇷 유닛은 TileMapLayer에 직접 배치하지 않음.
> 🇰🇷 격자 좌표에서 월드 좌표를 계산하여 독립 Node2D로 배치.

### Position Calculation
> 🇰🇷 위치 계산

```gdscript
# Convert grid coord → world position
var world_pos: Vector2 = ground_layer.map_to_local(grid_pos)
unit.position = world_pos
```

### Prototype Unit Visual
> 🇰🇷 프로토타입 유닛 비주얼

Colored polygon (Polygon2D) as placeholder:
- Ally unit: blue diamond
- Enemy unit: red diamond
> 🇰🇷 Polygon2D로 단색 다이아몬드 표현
> 🇰🇷 아군: 파랑 | 적군: 빨강

### Default Spawn Positions (10x10 grid)
> 🇰🇷 기본 스폰 위치 (10x10 격자)

```
Ally units:  (1,8), (2,8), (3,8)    ← bottom-left area
Enemy units: (6,1), (7,1), (8,1)    ← top-right area
```

---

## 4. Script List and Responsibilities
> 🇰🇷 스크립트 목록 및 역할

### 4-1. GridManager.gd
> 🇰🇷 격자 관리자

**Path:** `scripts/combat/GridManager.gd`
**Attached to:** GridManager (Node2D)

**Responsibilities:**
- Store grid dimensions (GRID_WIDTH, GRID_HEIGHT)
- Reference GroundLayer, ObstacleLayer, HighlightLayer
- Track tile occupancy (which unit is on which grid cell)
- Provide coordinate conversion utilities
- Provide tile walkability queries

**Key API:**
```gdscript
const GRID_WIDTH: int = 10
const GRID_HEIGHT: int = 10
const TILE_FLOOR: int = 0   # TileSet source ID
const TILE_WALL: int = 1    # TileSet source ID

# Coordinate conversion
func grid_to_world(grid_pos: Vector2i) -> Vector2
func world_to_grid(world_pos: Vector2) -> Vector2i

# Tile queries
func is_walkable(grid_pos: Vector2i) -> bool
func is_in_bounds(grid_pos: Vector2i) -> bool

# Occupancy tracking
func is_occupied(grid_pos: Vector2i) -> bool
func set_occupied(grid_pos: Vector2i, unit: Node2D) -> void
func clear_occupied(grid_pos: Vector2i) -> void
func get_unit_at(grid_pos: Vector2i) -> Node2D  # returns null if empty

# Grid generation
func generate_grid() -> void  # fills GroundLayer with floor tiles
```

> 🇰🇷 격자 크기 상수, TileMapLayer 참조, 점유 추적, 좌표 변환, 타일 쿼리 담당

---

### 4-2. CombatUnit.gd
> 🇰🇷 전투 유닛

**Path:** `scripts/combat/CombatUnit.gd`
**Attached to:** each unit instance (Node2D)

**Responsibilities:**
- Store current grid position
- Move unit to target grid position (instant for Phase 1-1, animated in Phase 1-2)
- Emit signals for state changes

**Key API:**
```gdscript
var grid_pos: Vector2i
var unit_name: String
var is_ally: bool

signal unit_moved(from: Vector2i, to: Vector2i)

func initialize(start_pos: Vector2i, name: String, ally: bool) -> void
func move_to(target_pos: Vector2i) -> void  # instant snap for Phase 1-1
```

> 🇰🇷 격자 좌표 보관, 목표 위치 이동, 상태 변경 시그널 발생

---

### 4-3. CombatScene.gd
> 🇰🇷 전투 씬 컨트롤러

**Path:** `scripts/combat/CombatScene.gd`
**Attached to:** CombatScene (Node2D)

**Responsibilities:**
- Initialize GridManager (call `generate_grid()`)
- Spawn ally and enemy units at default positions
- Handle unit click selection (highlight selected unit)
- Entry point for Phase 1-2 (movement) and Phase 1-3 (turn system)

**Key API:**
```gdscript
@onready var grid_manager: Node2D = $GridManager
@onready var ally_container: Node2D = $UnitContainer/AllyUnits
@onready var enemy_container: Node2D = $UnitContainer/EnemyUnits

var unit_scene: PackedScene = preload("res://scenes/combat/combat_unit.tscn")

func _ready() -> void  # init grid + spawn units
func _spawn_units() -> void
func _on_unit_clicked(unit: Node2D) -> void
```

> 🇰🇷 GridManager 초기화, 유닛 스폰, 클릭 선택 처리

---

## 5. File Output List
> 🇰🇷 산출 파일 목록

| File | Type | Description |
|------|------|-------------|
| `scenes/combat/combat_scene.tscn` | Scene | Main combat scene |
| `scenes/combat/combat_unit.tscn` | Scene | Unit instance scene |
| `scripts/combat/GridManager.gd` | Script | Grid logic and occupancy |
| `scripts/combat/CombatUnit.gd` | Script | Unit behavior |
| `scripts/combat/CombatScene.gd` | Script | Scene controller |

> 🇰🇷 산출 파일: 씬 2개 + 스크립트 3개

---

## 6. Out of Scope for Phase 1-1
> 🇰🇷 Phase 1-1 범위 외

The following are explicitly excluded from this phase:
- AStar2D pathfinding → Phase 1-2
- Tile highlight (movement/attack range) → Phase 1-2
- Click-to-move → Phase 1-2
- Turn queue system → Phase 1-3
- HP / damage → Phase 1-4
- Animations → Phase 1-2+
> 🇰🇷 이 Phase에서 제외: AStar2D, 하이라이트, 클릭 이동, 턴 큐, HP/데미지, 애니메이션

---

## 7. Acceptance Criteria
> 🇰🇷 완료 기준

Phase 1-1 is complete when:
- [ ] 10x10 isometric grid renders correctly in Godot editor
- [ ] 3 ally units (blue) and 3 enemy units (red) appear at spawn positions
- [ ] GridManager correctly reports occupancy and walkability for any tile
- [ ] No errors in Godot output panel on scene start
> 🇰🇷 격자 렌더링 확인, 유닛 스폰 확인, GridManager 쿼리 정상, 에러 없음

---

## 8. Next Step (after user approval)
> 🇰🇷 다음 단계 (승인 후)

→ `@Implementor` implements based on this document.
→ Recommended tool: Claude Code
> 🇰🇷 승인 후 @Implementor가 이 설계를 기반으로 구현.
> 🇰🇷 권장 도구: Claude Code
