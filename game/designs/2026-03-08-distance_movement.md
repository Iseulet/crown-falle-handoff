# Design: Distance-Based Movement System
> 🇰🇷 설계: 거리 기반 이동 시스템

- **Agent:** @Planner
- **Date:** 2026-03-08
- **Status:** Design Complete — Pending @Implementor
- **References:**
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-wartales-grid-analysis.md`

---

## 1. 설계 목표
> Goals

| 항목 | 현재 (BFS 타일) | 목표 (거리 기반) |
|------|----------------|----------------|
| 이동 단위 | 칸 수 (int) | 거리 (float, 픽셀) |
| 이동 범위 계산 | BFS 6방향 탐색 | 원형 반경 + 장애물 클리핑 |
| 시각화 | TileMapLayer 타일 하이라이트 | 청록색 폴리곤 외곽선 |
| 이동 목적지 | 타일 중심으로 스냅 | 타일 중심으로 스냅 (유지) |
| 그리드 구조 | Staggered Hex (6방향) | Staggered Hex 유지 |

타일은 점유/인접 판정 전용으로 유지.
이동 가능 여부는 **픽셀 거리**로 판정.

---

## 2. 거리 계산 방식
> Distance Calculation

### 단위: 픽셀 (world-space)

타일 중심 간 픽셀 거리를 사용.
`move_range`를 픽셀 단위 float로 변환.

```gdscript
# MercenaryData.gd — 변경
@export var move_range: float = 192.0  # pixels (기존 int 3 × 64px/tile)
```

### 타일 크기 기준

현재 타일 크기: **64×64px** (Staggered Hex)
인접 타일 중심 간 거리:
- 직선 방향(상/하): **64px**
- 사선 방향(hex 인접): **≈64px** (HALF_OFFSET_SQUARE 구조상 동일 단위)

### move_range 환산표

| 기존 칸 수 | 픽셀 거리 |
|-----------|---------|
| 3 | 192.0px |
| 4 | 256.0px |
| 5 | 320.0px |

### 도달 가능 판정

```
유닛 월드 좌표 → 후보 타일 중심 좌표
거리 = 유닛 위치.distance_to(타일 중심)
도달 가능 조건: 거리 <= move_range AND 경로상 장애물 없음
```

---

## 3. 이동 가능 구역 계산
> Reachable Area Calculation

### 방식: 타일 중심 기반 범위 필터링

NavigationRegion2D나 연속 공간 pathfinding을 쓰지 않고,
**기존 타일 격자를 순회**하되 판정 기준을 BFS 깊이 → 픽셀 거리로 교체.

```gdscript
# GridManager.gd — get_reachable_cells() 교체
func get_reachable_cells(from: Vector2i, move_range: float) -> Array[Vector2i]:
    var origin_world: Vector2 = grid_to_world(from)
    var reachable: Array[Vector2i] = []

    for y in range(_grid_height):
        for x in range(_grid_width):
            var candidate := Vector2i(x, y)
            if candidate == from:
                continue
            if not is_walkable(candidate) or is_occupied(candidate):
                continue
            var candidate_world: Vector2 = grid_to_world(candidate)
            if origin_world.distance_to(candidate_world) <= move_range:
                if _has_clear_path(from, candidate):
                    reachable.append(candidate)

    return reachable
```

### 장애물 경로 체크: `_has_clear_path()`

직선 경로상 장애물 타일을 DDA(Digital Differential Analyzer) 또는
타일 순회로 체크.

```gdscript
func _has_clear_path(from: Vector2i, to: Vector2i) -> bool:
    var steps: int = maxi(abs(to.x - from.x), abs(to.y - from.y))
    if steps == 0:
        return true
    for i in range(1, steps):
        var t := float(i) / float(steps)
        var ix := roundi(lerp(float(from.x), float(to.x), t))
        var iy := roundi(lerp(float(from.y), float(to.y), t))
        var mid := Vector2i(ix, iy)
        if not is_walkable(mid):
            return false
    return true
```

> **설계 근거:**
> - BFS 대비 구현 단순, 타일 수가 적어 O(W×H) 순회 충분 (10×10 = 100타일)
> - 장애물 감지: 직선 경로 보간으로 근사 — 프로토타입 수준에서 충분한 정확도
> - 향후 AStar2D로 교체 가능 (인터페이스 동일 유지)

---

## 4. 시각화 방식
> Visualization

### 결정: 커스텀 폴리곤 외곽선 (Line2D)

| 방식 | 장점 | 단점 | 결정 |
|------|------|------|------|
| TileMapLayer 하이라이트 | 기존 코드 재사용 | 각진 타일 경계, 워테일즈 느낌 없음 | ❌ |
| NavigationRegion2D | 연속 공간 지원 | 타일 격자와 혼용 복잡, 과함 | ❌ |
| **커스텀 Line2D 폴리곤** | 워테일즈 스타일, 구현 명확 | 외곽선 순서 정렬 필요 | ✅ |

### 구현 방법

도달 가능 타일 외곽 꼭짓점을 수집 → 볼록 껍질(Convex Hull) 또는 외곽선 정렬 → Line2D로 그리기.

```gdscript
# CombatScene.gd — highlight_move_range() 교체 대상
# GridManager.gd — get_reachable_cells() 결과 사용

# 새 노드 추가: MoveRangeOverlay (Line2D)
@onready var move_range_overlay: Line2D = $MoveRangeOverlay

func _draw_move_range(cells: Array[Vector2i]) -> void:
    if cells.is_empty():
        move_range_overlay.points = PackedVector2Array()
        return
    # 각 타일 중심 월드 좌표 수집
    var points: PackedVector2Array
    for cell in cells:
        points.append(grid_to_world(cell))
    # 볼록 껍질 정렬 후 Line2D에 적용
    move_range_overlay.points = _convex_hull(points)
    move_range_overlay.closed = true
```

### Line2D 속성

```gdscript
move_range_overlay.default_color = Color(0.0, 0.9, 0.8, 0.9)  # 청록색
move_range_overlay.width = 3.0
move_range_overlay.closed = true
```

- TileMapLayer `highlight_layer`의 파란 타일 하이라이트는 **제거**
- 외곽선만 표시 — 내부는 채우지 않음 (워테일즈 스타일)

### Convex Hull 구현 (GDScript)

```gdscript
func _convex_hull(points: PackedVector2Array) -> PackedVector2Array:
    if points.size() < 3:
        return points
    # Graham scan (bottom-left pivot)
    var pts := Array(points)
    pts.sort_custom(func(a, b): return a.y < b.y or (a.y == b.y and a.x < b.x))
    var pivot: Vector2 = pts[0]
    pts.sort_custom(func(a, b):
        var angle_a := pivot.angle_to_point(a)
        var angle_b := pivot.angle_to_point(b)
        return angle_a < angle_b
    )
    var hull: Array[Vector2] = [pts[0], pts[1]]
    for i in range(2, pts.size()):
        while hull.size() >= 2:
            var cross := (hull[-1] - hull[-2]).cross(pts[i] - hull[-2])
            if cross <= 0:
                hull.pop_back()
            else:
                break
        hull.append(pts[i])
    var result: PackedVector2Array
    for p in hull:
        result.append(p)
    return result
```

---

## 5. 장애물 처리
> Obstacle Handling

| 장애물 종류 | 처리 |
|------------|------|
| Wall (obstacle_layer) | `is_walkable()` false → 목록 제외, 경계선 안쪽으로 꺾임 |
| 적 점유 타일 | `is_occupied()` true → 목적지 제외 (근접 공격 대상으로 표시) |
| 아군 점유 타일 | 목적지 제외 (경로 통과는 `_has_clear_path` 에서 무시 가능) |
| 범위 외 타일 | `distance > move_range` → 자동 제외 |

기존 `is_walkable()` / `is_occupied()` 인터페이스 그대로 유지.

---

## 6. 공격 범위 거리 기반 변경 여부
> Attack Range — Distance or Tile?

### 결정: **현재는 타일 인접(6방향) 유지**

| 이유 | 설명 |
|------|------|
| 범위 단순성 | 근접 공격은 인접 1타일이면 충분 |
| 교전 시스템 의존 | Engagement 판정이 타일 인접에 기반 |
| 원거리 미구현 | Archer 원거리 공격은 별도 설계 예정 |

원거리 공격(Archer, Mage) 구현 시점에 attack_range도 float 거리로 전환.
현재 `get_attack_neighbors()` 함수 유지.

---

## 7. 교전 시스템 인접 판정 재정의
> Engagement Adjacency Redefinition

### 결정: **타일 인접(6방향) 유지**

이동이 거리 기반으로 바뀌어도, 이동 후 **도착 타일**은 격자에 스냅됨.
교전 판정은 점유 타일 기준이므로 `_get_neighbors()` 로직 변경 없음.

```
이동: 거리 기반 (float)
도착: 타일 중심 스냅 (Vector2i)
교전: 도착 타일 기준 6방향 인접 (기존 동일)
```

`get_attack_neighbors()` / `is_in_proximity()` 변경 없음.

---

## 8. 수정 파일 목록 및 작업량
> Changed Files & Estimated Work

### 수정 파일

| 파일 | 변경 내용 | 규모 |
|------|-----------|------|
| `scripts/combat/GridManager.gd` | `get_reachable_cells()` BFS→거리 필터, `_has_clear_path()` 추가, `highlight_move_range()` 제거 또는 비워두기 | Medium |
| `scripts/combat/CombatScene.gd` | `_draw_move_range()` 추가, `highlight_move_range()` 호출 교체, `move_range_overlay` 노드 참조 | Small |
| `scenes/combat/combat_scene.tscn` | `MoveRangeOverlay` (Line2D) 노드 추가 | Small |
| `scripts/data/MercenaryData.gd` | `move_range: int` → `move_range: float` | Minimal |
| `data/mercenaries/*.tres` | `move_range` 값 픽셀 단위로 변환 (3→192.0, 4→256.0, 5→320.0) | Minimal |
| `data/classes/levelup_config.json` | `mv_choice` 픽셀 단위로 변환 (1→64.0) | Minimal |

### 변경 없는 파일

| 파일 | 이유 |
|------|------|
| `GridManager.gd` — `_get_neighbors()`, `get_attack_neighbors()`, `is_in_proximity()` | 타일 인접 유지 |
| `GridManager.gd` — `is_walkable()`, `is_occupied()`, occupancy | 인터페이스 동일 |
| `CombatScene.gd` — `_attack_target()`, `_select_unit()` engagement 분기 | 변경 없음 |
| `CombatUnit.gd` | 변경 없음 |

### 예상 작업량

| 단계 | 내용 | 난이도 |
|------|------|--------|
| 1 | `get_reachable_cells()` 교체 | Low |
| 2 | `_has_clear_path()` 구현 | Low |
| 3 | Line2D 노드 추가 + `_draw_move_range()` | Low |
| 4 | `_convex_hull()` 구현 | Medium |
| 5 | MercenaryData float 전환 + .tres 값 수정 | Low |
| 6 | 정적 분석 + 동작 테스트 | — |

**총 변경량:** 6개 파일, 약 80줄 추가/수정

---

## 9. 주요 설계 판단 근거
> Design Rationale

**왜 NavigationRegion2D를 쓰지 않는가?**
- NavigationRegion2D는 연속 공간(continuous space) 전용
- 타일 격자 + 점유 시스템과 혼용 시 동기화 복잡
- 프로토타입 규모(10×10)에서 과설계
- 타일 순회 O(100) 충분

**왜 BFS를 완전히 제거하는가?**
- BFS는 칸 수 기반 → 거리 단위와 근본적으로 맞지 않음
- 동일 거리 내 타일도 BFS 깊이에 따라 결과 다를 수 있음 (hex offset 특성)
- 거리 필터로 교체하면 단순하고 정확

**왜 타일 스냅을 유지하는가?**
- 교전 시스템, 점유 추적, 공격 판정 모두 타일 기반
- 연속 위치를 도입하면 점유 충돌 등 edge case 급증
- 워테일즈도 내부적으로 타일 스냅 사용 (이동 범위만 거리 단위)

---

## 구현 체크리스트 (for @Implementor)

- [ ] `MercenaryData.gd` — `move_range: int` → `float`
- [ ] `*.tres` — move_range 값 픽셀 단위 변환
- [ ] `levelup_config.json` — `mv_choice` 픽셀 단위 변환
- [ ] `GridManager.gd` — `get_reachable_cells()` 거리 필터로 교체
- [ ] `GridManager.gd` — `_has_clear_path()` 추가
- [ ] `combat_scene.tscn` — `MoveRangeOverlay` (Line2D) 노드 추가
- [ ] `CombatScene.gd` — `_draw_move_range()` + `_convex_hull()` 추가
- [ ] `CombatScene.gd` — `highlight_move_range()` 호출 → `_draw_move_range()` 교체
- [ ] 동작 확인: 원형 반경 표시, 장애물 시 경계선 변형, 타일 스냅 이동
