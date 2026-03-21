# Design: Staggered Grid System
> 🇰🇷 설계: Staggered Grid 시스템

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-08
- Agent: @Planner
- Status: Pending User Approval
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-wartales-grid-analysis.md`

---

## Overview
> 🇰🇷 개요

Replace the current `TILE_SHAPE_ISOMETRIC` (pure diamond) grid with
`TILE_SHAPE_HALF_OFFSET_SQUARE` (staggered square) to match the Wartales visual style.
> 🇰🇷 현재 다이아몬드 격자(TILE_SHAPE_ISOMETRIC)를
> 🇰🇷 워테일즈 스타일 Staggered Square(TILE_SHAPE_HALF_OFFSET_SQUARE)로 교체.

### Why this change fixes existing bugs
> 🇰🇷 이 변경이 기존 버그를 해결하는 이유

Current ISOMETRIC uses `floor()` rounding in `local_to_map`.
This causes left-half-tile clicks to return the wrong cell (asymmetric click detection).
HALF_OFFSET_SQUARE uses rectangular tiles → `local_to_map` is deterministic and symmetric.
> 🇰🇷 현재 ISOMETRIC의 local_to_map은 floor() 기반 → 다이아몬드 타일 좌측 클릭 오감지 버그
> 🇰🇷 HALF_OFFSET_SQUARE는 직사각형 타일 → local_to_map 대칭 보장

---

## 1. Staggered Grid Coordinate System
> 🇰🇷 Staggered Grid 좌표계 정의

### Column Stagger Rule
> 🇰🇷 열 엇갈림 규칙

```
Odd columns (col % 2 == 1) shift DOWN by tile_h / 2.
Even columns (col % 2 == 0) are at standard height.
```

> 🇰🇷 홀수 열: 짝수 열보다 tile_h/2 아래로 이동
> 🇰🇷 짝수 열: 기준 높이

### Screen Position Formula
> 🇰🇷 화면 좌표 계산식

```
tile_w  = 128   (tile_size.x)
tile_h  = 64    (tile_size.y)
step_x  = tile_w / 2 = 64    (horizontal step between adjacent columns)
step_y  = tile_h / 2 = 32    (vertical step between adjacent rows)

screen_x = col * step_x
screen_y = row * tile_h + (col % 2) * step_y
```

> 🇰🇷 홀수 열이면 screen_y에 step_y(32px) 추가
> 🇰🇷 짝수 열은 그대로

### Visual Comparison
> 🇰🇷 비주얼 비교

```
Current (ISOMETRIC diamond):        Target (HALF_OFFSET staggered):

    (0,0)                           [0,0] [1,0] [2,0]
  (1,0) (0,1)                             [1,1]
    (1,1)                           [0,1] [1,2] [2,1]

Tiles look like pure diamonds.      Tiles look like tilted rectangles.
Grid feels abstract.                Grid reads as a battlefield board.
```

> 🇰🇷 현재: 순수 마름모 타일 (추상적)
> 🇰🇷 목표: 기울어진 직사각형 타일 (워테일즈 스타일)

---

## 2. Godot TileSet Configuration Change
> 🇰🇷 TileSet 설정 변경

### _create_tile_set() changes in GridManager.gd
> 🇰🇷 GridManager.gd의 _create_tile_set() 수정 내용

```gdscript
# BEFORE
ts.tile_shape = TileSet.TILE_SHAPE_ISOMETRIC
ts.tile_size  = Vector2i(128, 64)

# AFTER
ts.tile_shape       = TileSet.TILE_SHAPE_HALF_OFFSET_SQUARE
ts.tile_layout      = TileSet.TILE_LAYOUT_STAGGER_ODD    # odd columns stagger
ts.tile_offset_axis = TileSet.TILE_OFFSET_AXIS_VERTICAL  # vertical stagger per column
ts.tile_size        = Vector2i(64, 64)  # square tiles (fit the staggered step geometry)
```

> 🇰🇷 tile_shape: 다이아몬드 → Half-Offset Square
> 🇰🇷 tile_layout: 홀수 열 엇갈림
> 🇰🇷 tile_size: (128,64)→(64,64) — Half-Offset에서 시각적으로 올바른 정사각형 타일

### Tile Texture Change
> 🇰🇷 타일 텍스처 변경

Replace `_create_diamond_texture` with `_create_parallelogram_texture`.
Tile graphic changes from rhombus to a filled rectangle (viewed as tilted square).
> 🇰🇷 _create_diamond_texture → _create_parallelogram_texture 교체
> 🇰🇷 텍스처 형태: 마름모 → 직사각형 (기울어진 정사각형처럼 보임)

```gdscript
func _create_parallelogram_texture(color: Color) -> ImageTexture:
    # Draw a solid rectangle that visually represents a top-down tilted square.
    # Exact shape will be tuned during implementation.
    var img := Image.create(64, 64, false, Image.FORMAT_RGBA8)
    img.fill(Color(0.0, 0.0, 0.0, 0.0))
    for y in range(64):
        for x in range(64):
            img.set_pixel(x, y, color)  # solid fill — border added separately
    return ImageTexture.create_from_image(img)
```

> 🇰🇷 프로토타입: 단색 직사각형. 추후 아이소메트릭 경사 적용 가능.

---

## 3. Neighbor Calculation (Stagger-Aware)
> 🇰🇷 인접 타일 계산 (Stagger 적용)

Staggered grid neighbors differ depending on column parity.
> 🇰🇷 열의 짝수/홀수에 따라 인접 타일 좌표가 달라짐.

### 6-Direction Neighbors (Movement BFS)
> 🇰🇷 6방향 인접 (이동 BFS용)

```
For even col (col % 2 == 0):
  Top        = (col,   row-1)
  Bottom     = (col,   row+1)
  Top-Right  = (col+1, row-1)
  Bot-Right  = (col+1, row)
  Top-Left   = (col-1, row-1)
  Bot-Left   = (col-1, row)

For odd col (col % 2 == 1):
  Top        = (col,   row-1)
  Bottom     = (col,   row+1)
  Top-Right  = (col+1, row)
  Bot-Right  = (col+1, row+1)
  Top-Left   = (col-1, row)
  Bot-Left   = (col-1, row+1)
```

> 🇰🇷 이동 BFS는 6방향 전체 사용 (워테일즈 자유 이동 방향과 동일)

### 6-Direction Neighbors (Attack / Engagement)
> 🇰🇷 6방향 인접 (공격 / 교전 판정용)

In a staggered grid, all 6 directions share an edge with the tile.
Attack and engagement use the same 6-direction set as movement BFS.
> 🇰🇷 Staggered Grid에서 변(Edge)을 공유하는 방향은 직상하 포함 6방향 전부
> 🇰🇷 이동(BFS)과 공격/교전 모두 6방향으로 통일

```
For even col:
  attack_neighbors = [
    (col,   row-1),  # Top
    (col,   row+1),  # Bottom
    (col+1, row-1),  # Top-Right
    (col+1, row),    # Bot-Right
    (col-1, row-1),  # Top-Left
    (col-1, row),    # Bot-Left
  ]

For odd col:
  attack_neighbors = [
    (col,   row-1),  # Top
    (col,   row+1),  # Bottom
    (col+1, row),    # Top-Right
    (col+1, row+1),  # Bot-Right
    (col-1, row),    # Top-Left
    (col-1, row+1),  # Bot-Left
  ]
```

> 🇰🇷 이동 6방향 = 공격 6방향 — 방향 판정 통일

---

## 4. GridManager.gd Change Scope
> 🇰🇷 GridManager.gd 변경 범위

### Functions to Modify
> 🇰🇷 수정 대상 함수

| Function | Change |
|----------|--------|
| `_create_tile_set()` | tile_shape / tile_layout / tile_offset_axis / tile_size 변경 |
| `_create_diamond_texture()` | 제거 → `_create_parallelogram_texture()` 로 교체 |
| `_get_neighbors(pos)` | 4방향 고정 → stagger-aware 6방향 |
| `generate_grid()` | 변경 없음 (set_cell 루프 그대로) |
| `grid_to_world()` | 변경 없음 (map_to_local 그대로) |
| `global_to_grid()` | 변경 없음 (local_to_map이 이제 HALF_OFFSET 기준으로 올바르게 작동) |
| `is_in_bounds()` | 변경 없음 |

### New Function to Add
> 🇰🇷 신규 추가 함수

```gdscript
# Returns 6 edge-sharing attack neighbors (stagger-aware, same as movement)
func get_attack_neighbors(pos: Vector2i) -> Array[Vector2i]:
    var result: Array[Vector2i] = []
    var is_even_col: bool = (pos.x % 2 == 0)
    var candidates: Array[Vector2i]
    if is_even_col:
        candidates = [
            Vector2i(pos.x,   pos.y-1),
            Vector2i(pos.x,   pos.y+1),
            Vector2i(pos.x+1, pos.y-1), Vector2i(pos.x+1, pos.y),
            Vector2i(pos.x-1, pos.y-1), Vector2i(pos.x-1, pos.y),
        ]
    else:
        candidates = [
            Vector2i(pos.x,   pos.y-1),
            Vector2i(pos.x,   pos.y+1),
            Vector2i(pos.x+1, pos.y),   Vector2i(pos.x+1, pos.y+1),
            Vector2i(pos.x-1, pos.y),   Vector2i(pos.x-1, pos.y+1),
        ]
    for c in candidates:
        if is_in_bounds(c):
            result.append(c)
    return result
```

> 🇰🇷 이동 BFS와 동일한 6방향 반환 — CombatScene.gd의 _get_attack_targets 호출용

---

## 5. BFS Movement Update
> 🇰🇷 BFS 이동 범위 수정 방향

### _get_neighbors() replacement
> 🇰🇷 _get_neighbors() 교체

```gdscript
# BEFORE (4 fixed directions)
var directions: Array[Vector2i] = [
    Vector2i(1,0), Vector2i(-1,0), Vector2i(0,1), Vector2i(0,-1),
]

# AFTER (stagger-aware 6 directions)
func _get_neighbors(pos: Vector2i) -> Array[Vector2i]:
    var result: Array[Vector2i] = []
    var is_even: bool = (pos.x % 2 == 0)
    var candidates: Array[Vector2i]
    if is_even:
        candidates = [
            Vector2i(pos.x,   pos.y-1),
            Vector2i(pos.x,   pos.y+1),
            Vector2i(pos.x+1, pos.y-1),
            Vector2i(pos.x+1, pos.y),
            Vector2i(pos.x-1, pos.y-1),
            Vector2i(pos.x-1, pos.y),
        ]
    else:
        candidates = [
            Vector2i(pos.x,   pos.y-1),
            Vector2i(pos.x,   pos.y+1),
            Vector2i(pos.x+1, pos.y),
            Vector2i(pos.x+1, pos.y+1),
            Vector2i(pos.x-1, pos.y),
            Vector2i(pos.x-1, pos.y+1),
        ]
    for c in candidates:
        if is_in_bounds(c):
            result.append(c)
    return result
```

> 🇰🇷 get_reachable_cells() 자체 로직(BFS)은 변경 없음
> 🇰🇷 _get_neighbors만 교체하면 BFS는 자동으로 6방향 동작

---

## 6. Attack Adjacency Update (CombatScene.gd)
> 🇰🇷 공격 인접 계산 수정 (CombatScene.gd)

### _get_attack_targets() replacement
> 🇰🇷 _get_attack_targets() 교체

```gdscript
# BEFORE (Manhattan distance == 1)
var diff := enemy.grid_pos - unit.grid_pos
if absi(diff.x) + absi(diff.y) == 1:

# AFTER (stagger-aware neighbor list)
func _get_attack_targets(unit: CombatUnit) -> Array[Vector2i]:
    var targets: Array[Vector2i] = []
    var adjacent := grid_manager.get_attack_neighbors(unit.grid_pos)
    for cell in adjacent:
        var enemy := grid_manager.get_unit_at(cell) as CombatUnit
        if enemy != null and not enemy.is_ally and not enemy.is_queued_for_deletion():
            targets.append(cell)
    return targets
```

> 🇰🇷 occupancy 조회 방식으로 복귀 (get_attack_neighbors가 정확한 인접 좌표만 반환하므로 안전)
> 🇰🇷 Manhattan distance 방식 제거

---

## 7. Boundary Masking
> 🇰🇷 전장 외곽 마스킹 방식

### Approach: Non-Walkable Mask Dictionary
> 🇰🇷 방식: 비통행 마스크 딕셔너리

Add a `_mask: Dictionary` (Vector2i → bool) to GridManager.
Cells in the mask are treated as out-of-bounds for BFS and attacks.
This allows arbitrary battlefield shapes (not just rectangles).
> 🇰🇷 GridManager에 _mask: Dictionary 추가
> 🇰🇷 마스크된 셀은 BFS/공격에서 제외 → 비직사각형 전장 지원

```gdscript
# GridManager.gd addition
var _mask: Dictionary = {}  # Vector2i -> true (masked = off-battlefield)

func mask_cell(pos: Vector2i) -> void:
    _mask[pos] = true

func is_in_bounds(grid_pos: Vector2i) -> bool:
    if _mask.has(grid_pos):
        return false
    return grid_pos.x >= 0 and grid_pos.x < GRID_WIDTH \
        and grid_pos.y >= 0 and grid_pos.y < GRID_HEIGHT
```

> 🇰🇷 프로토타입: 기본 직사각형 사용 (마스킹 불필요)
> 🇰🇷 Phase 5+: 비직사각형 전장 시 mask_cell()로 외곽 제거

---

## 8. Expected File Changes
> 🇰🇷 예상 변경 파일 목록

| File | Type | Changes |
|------|------|---------|
| `scripts/combat/GridManager.gd` | Modify | tile_shape, tile_layout, tile_size, _create_parallelogram_texture, _get_neighbors (6방향), get_attack_neighbors (신규), _mask (신규, 선택) |
| `scripts/combat/CombatScene.gd` | Modify | _get_attack_targets (stagger 인접으로 교체) |
| `scenes/combat/combat_scene.tscn` | No change | TileMapLayer 노드 구조 동일 |
| `scenes/combat/combat_unit.tscn` | No change | 유닛 비주얼 변경 없음 |

> 🇰🇷 총 2개 파일 수정. 씬 파일 변경 없음.

---

## 9. Spawn Position Note
> 🇰🇷 스폰 위치 주의사항

After converting to staggered grid, existing spawn positions
`(3,4)~(5,5)` remain valid — they're just grid coordinates.
Visual positions will shift due to the new tile geometry, but no code change needed.
> 🇰🇷 스폰 좌표는 그대로 유효. 화면상 위치만 달라짐. 코드 수정 불필요.

---

## 10. Acceptance Criteria
> 🇰🇷 완료 기준

- [ ] Godot 실행 시 Staggered 패턴 타일 렌더링 (짝수/홀수 열 엇갈림 확인)
- [ ] 아군 선택 → 이동 하이라이트 6방향 대칭 표시
- [ ] 클릭 이동: 좌우 모든 방향 타일 클릭 정상 동작
- [ ] 공격 하이라이트: 6방향 인접 적군 모두 빨간 표시
- [ ] 공격 클릭: 6방향 모두 정상 공격
- [ ] Godot 에러 없음

---

## Reference
> 🇰🇷 참조

- `handoff/plans/design/2026-03-06-wartales-grid-analysis.md`
- `handoff/plans/design/2026-03-06-crownfalle-concept.md`
- Godot 4 TileSet docs: TILE_SHAPE_HALF_OFFSET_SQUARE
