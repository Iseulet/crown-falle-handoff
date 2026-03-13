# Design: Phase 1-2 Movement System
> 🇰🇷 설계: Phase 1-2 이동 시스템

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Agent: @Planner
- Status: Pending User Approval
- Phase: 1-2 (Movement System)
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-crownfalle-roadmap.md`
  - `handoff/plans/design/2026-03-06-grid_system.md` (Phase 1-1)
  - `handoff/DECISIONS.md`
> 🇰🇷 Phase 1-1 설계 문서 기반으로 확장

---

## Overview
> 🇰🇷 개요

Phase 1-2 adds unit selection, movement range calculation (BFS), tile highlight,
and click-to-move. Also includes the `move_to()` occupancy fix flagged in Phase 1-1 review.
No new files. All changes go in the existing 3 scripts.
> 🇰🇷 유닛 선택, BFS 이동 범위 계산, 타일 하이라이트, 클릭 이동 추가.
> 🇰🇷 Phase 1-1 리뷰 지적 move_to() 점유 수정 포함.
> 🇰🇷 신규 파일 없음. 기존 스크립트 3개 수정.

---

## 1. Movement Rules
> 🇰🇷 이동 규칙

- **Direction:** 4-directional (up / down / left / right, no diagonal)
- **Range:** Each unit has `move_range: int` (default 4 for all prototype units)
- **Blocking:** Cannot move to occupied tiles or wall tiles
- **Phase 1-2 scope:** No turn enforcement yet (Phase 1-3). Units can move freely for testing.
> 🇰🇷 4방향 이동 (대각선 없음)
> 🇰🇷 이동 범위: move_range (프로토타입 기본값 4)
> 🇰🇷 점유 타일 / wall 타일 이동 불가
> 🇰🇷 턴 제한 없음 (Phase 1-3에서 추가)

---

## 2. Input Flow
> 🇰🇷 입력 흐름

```
Left click on grid
│
├── Hit ally unit AND no unit selected
│   └── SELECT unit → calculate range → show highlight
│
├── Hit highlighted tile AND unit is selected
│   └── MOVE unit → clear highlight → deselect
│
└── Anything else
    └── DESELECT → clear highlight
```

> 🇰🇷 아군 유닛 클릭 → 선택 + 이동 범위 하이라이트
> 🇰🇷 하이라이트 타일 클릭 → 이동 + 하이라이트 제거
> 🇰🇷 그 외 클릭 → 선택 해제

---

## 3. Movement Range Algorithm (BFS)
> 🇰🇷 이동 범위 계산 (너비 우선 탐색)

BFS (Breadth-First Search) is used for movement range calculation.
AStar2D is reserved for Phase 1-3+ (pathfinding with obstacles for AI).
> 🇰🇷 이동 범위: BFS 사용 (AStar2D는 Phase 1-3+ AI 경로탐색용으로 예약)

```
start at unit.grid_pos, distance = 0
for each unvisited neighbor within is_walkable AND NOT is_occupied:
    if distance + 1 <= move_range → add to reachable, enqueue
return reachable cells (excludes starting position)
```

---

## 4. Script Changes
> 🇰🇷 스크립트 변경 내용

### 4-1. GridManager.gd — additions
> 🇰🇷 추가 내용

```gdscript
const TILE_HIGHLIGHT_MOVE: int = 2  # light blue diamond

# Add to _create_tile_set():
# highlight tile — semi-transparent light blue diamond

func get_reachable_cells(from: Vector2i, move_range: int) -> Array[Vector2i]
    # BFS from 'from', returns all cells reachable within move_range
    # excludes 'from' itself
    # respects is_walkable() and is_occupied()

func highlight_move_range(cells: Array[Vector2i]) -> void
    # sets TILE_HIGHLIGHT_MOVE on HighlightLayer for each cell

func clear_highlights() -> void
    # clears all cells on HighlightLayer

func _get_neighbors(pos: Vector2i) -> Array[Vector2i]
    # returns 4 cardinal neighbors within bounds
```

### 4-2. CombatUnit.gd — additions
> 🇰🇷 추가 내용

```gdscript
var move_range: int = 4       # movement range in tiles
var has_moved: bool = false   # Phase 1-3 turn prep

# move_to() fix — remove direct grid_pos update.
# occupancy update is handled by CombatScene._move_selected_unit()
func move_to(target_pos: Vector2i) -> void
    # updates grid_pos and emits unit_moved signal only
    # NOTE: caller must handle GridManager occupancy update
```

### 4-3. CombatScene.gd — additions
> 🇰🇷 추가 내용

```gdscript
var selected_unit: CombatUnit = null
var reachable_cells: Array[Vector2i] = []

func _unhandled_input(event: InputEvent) -> void
    # handles MouseButton.LEFT click
    # converts screen pos → grid pos → dispatch to select/move/deselect

func _select_unit(unit: CombatUnit) -> void
    # sets selected_unit, calculates reachable_cells, calls highlight_move_range

func _deselect_unit() -> void
    # clears selected_unit, reachable_cells, calls clear_highlights

func _move_selected_unit(target: Vector2i) -> void
    # 1. clear_occupied(selected_unit.grid_pos)
    # 2. selected_unit.move_to(target)
    # 3. selected_unit.position = grid_to_world(target)
    # 4. set_occupied(target, selected_unit)
    # 5. _deselect_unit()
```

---

## 5. Highlight Tile Color
> 🇰🇷 하이라이트 타일 색상

- **Movement range:** Light blue, semi-transparent (Color(0.3, 0.6, 1.0, 0.5))
> 🇰🇷 이동 범위: 연한 파랑 반투명

---

## 6. Files Modified
> 🇰🇷 수정 파일 목록

| File | Change |
|------|--------|
| `scripts/combat/GridManager.gd` | Add TILE_HIGHLIGHT_MOVE, get_reachable_cells (BFS), highlight_move_range, clear_highlights, _get_neighbors |
| `scripts/combat/CombatUnit.gd` | Add move_range, has_moved; fix move_to() (occupancy update removed) |
| `scripts/combat/CombatScene.gd` | Add selected_unit, reachable_cells, _unhandled_input, _select_unit, _deselect_unit, _move_selected_unit |

> 🇰🇷 신규 파일 없음. 기존 3개 수정.

---

## 7. Out of Scope for Phase 1-2
> 🇰🇷 Phase 1-2 범위 외

- Turn enforcement (one move per turn) → Phase 1-3
- Enemy unit movement → Phase 1-3
- Animation (smooth movement tween) → future
- Attack range highlight → Phase 1-4
- AStar2D pathfinding (path preview) → Phase 1-3+
> 🇰🇷 턴 제한, 적 이동, 이동 애니메이션, 공격 범위, AStar2D 경로 미리보기는 제외

---

## 8. Acceptance Criteria
> 🇰🇷 완료 기준

- [ ] 아군 유닛 클릭 → 이동 범위 하이라이트 표시
- [ ] 하이라이트 타일 클릭 → 유닛 이동, 하이라이트 제거
- [ ] 다른 곳 클릭 → 선택 해제
- [ ] 점유 타일 이동 불가 (유닛끼리 겹치지 않음)
- [ ] GridManager 점유 데이터 이동 후 정확히 갱신
- [ ] Godot 에러 없음

---

## 9. Next Step
> 🇰🇷 다음 단계

→ `@Implementor` implements based on this document.
→ Recommended tool: Claude Code
> 🇰🇷 승인 후 @Implementor 구현. 권장 도구: Claude Code
