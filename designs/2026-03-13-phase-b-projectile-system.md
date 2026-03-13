# Phase B — Projectile System Design Document

> 🇰🇷 Phase B — 투사체 시스템 설계 문서

**Date:** 2026-03-13
**Author:** @Planner
**Status:** Approved — Ready for Implementation
**References:** Doc 2 (spawn_projectile event), Doc 4 (async principle), `distance_movement.md`

> 🇰🇷 **날짜:** 2026-03-13 | **작성:** @Planner | **상태:** 승인 — 구현 준비 완료

---

## 0. Core Constraint

The non-negotiable principle from `CLAUDE.md`:

> **투사체 비동기** — Projectile flight must NOT block the turn flow.
> The attacker returns to idle immediately after its animation ends.
> Damage landing is deferred to the projectile's `hit_arrived` signal.

> 🇰🇷 **핵심 제약 (`CLAUDE.md` Non-Override)**
> 투사체 비동기: 투사체 비행이 턴 흐름을 블로킹하면 안 됨.
> 공격자는 자신의 애니메이션 종료 즉시 idle 복귀. 데미지는 `hit_arrived` 시그널로 지연 적용.

---

## 1. Architecture Overview

> 🇰🇷 **아키텍처 개요**

### 1.1 Visual-Only Projectile Pattern

```
_resolve_attack()           # pre-calculates ALL damage upfront, no randomness later
  ↓                         # 데미지 전체를 미리 계산 — 이후 랜덤 없음
store _dispatch_* vars      # CombatScene-level snapshot
  ↓
animation plays
  ↓  ratio 0.40 → spawn_projectile event fires
_handle_spawn_projectile()  # packs context Dict (VALUE COPY), calls ProjectileManager.spawn()
  ↓                         # 컨텍스트 딕셔너리(값 복사본) 생성 후 spawn() 호출
Projectile flies (async)    # attacker immediately returns to idle
  ↓  hit_arrived signal     # 공격자는 즉시 idle 복귀
_on_projectile_hit(ctx)     # restores _dispatch_* from snapshot, calls _handle_hit_frame()
                            # 스냅샷에서 _dispatch_* 복원 후 _handle_hit_frame() 호출
```

### 1.2 New Files (3)

> 🇰🇷 **신규 파일 3종**

| File | Role |
|------|------|
| `scripts/combat/Projectile.gd` | Flies A→B, emits `hit_arrived` |
| `scripts/combat/ProjectileManager.gd` | Scene-level spawner & active projectile tracker |
| `scripts/singletons/ProjectileConfig.gd` | Loads `data/projectiles/*.json` |

> 🇰🇷 `Projectile.gd`: A→B 비행 후 `hit_arrived` 발생. `ProjectileManager.gd`: 씬 수준 스포너, 활성 투사체 추적. `ProjectileConfig.gd`: `data/projectiles/*.json` 로딩 싱글톤.

### 1.3 Modified Files

> 🇰🇷 **수정 파일**

| File | Change |
|------|--------|
| `CombatScene.gd` | `_handle_spawn_projectile()` full impl; new `_on_projectile_hit()`; engagement guard |
| `CombatUnit.gd` | Add `get_attack_range()` |
| `GridManager.gd` | Add `grid_distance()`, `get_cells_in_range()` |
| `animation_config.json` | `attack_ranged`: `hit_frame@0.55` → `spawn_projectile@0.40` |
| `class_config.json` | Add `attack_range` + `projectile` per class |
| `data/projectiles/*.json` | Extend schema (`sfx_launch`, `arc_height`, `rotate_to_velocity`) |

---

## 2. ProjectileConfig.gd (Singleton)

> 🇰🇷 **ProjectileConfig.gd (싱글톤)** — `project.godot`에 오토로드 등록

**Path:** `scripts/singletons/ProjectileConfig.gd`
Registered in `project.godot` autoloads as `ProjectileConfig`.

```gdscript
class_name ProjectileConfig
extends Node

# Loaded once at startup from data/projectiles/*.json.
# 시작 시 data/projectiles/*.json 전체 로드 — 이후 get_config()로 조회.
const DATA_DIR := "res://data/projectiles/"

var _configs: Dictionary = {}  # id → Dictionary

func _ready() -> void:
    _load_all()

func _load_all() -> void:
    var dir := DirAccess.open(DATA_DIR)
    if dir == null:
        push_error("[ProjectileConfig] Cannot open: %s" % DATA_DIR)
        return
    dir.list_dir_begin()
    var fname := dir.get_next()
    while fname != "":
        if fname.ends_with(".json"):
            var id := fname.get_basename()
            var text := FileAccess.get_file_as_string(DATA_DIR + fname)
            var parsed: Variant = JSON.parse_string(text)
            if parsed is Dictionary:
                _configs[id] = parsed
            else:
                push_warning("[ProjectileConfig] Parse failed: %s" % fname)
        fname = dir.get_next()

# Returns config dict for id; empty dict if not found.
# id에 해당하는 설정 반환. 없으면 빈 딕셔너리.
func get_config(id: String) -> Dictionary:
    return _configs.get(id, {})
```

---

## 3. Projectile.gd

> 🇰🇷 **Projectile.gd** — 개별 투사체 비행 로직

**Path:** `scripts/combat/Projectile.gd`

### 3.1 Public API

```gdscript
class_name Projectile
extends Node3D

# Emitted when the projectile reaches its destination.
# context = pre-calculated damage data (value copy from _handle_spawn_projectile).
# 투사체 목표 도달 시 발생. context = 미리 계산된 데미지 데이터 (값 복사본).
signal hit_arrived(context: Dictionary)

# Configure from JSON config, flight endpoints, and damage context.
# Call once before launch(). Do NOT call twice.
# JSON 설정, 비행 시작/끝 위치, 데미지 컨텍스트로 초기화. launch() 전 한 번만 호출.
func setup(config: Dictionary, start_pos: Vector3, end_pos: Vector3, context: Dictionary) -> void

# Begin flight. Called by ProjectileManager immediately after setup().
# 비행 시작. ProjectileManager가 setup() 직후 호출.
func launch() -> void
```

### 3.2 Straight-Line Flight (`arc: false`)

> 🇰🇷 **직선 비행** — `arc: false` 설정 시 Tween으로 직선 이동

```gdscript
func _fly_straight(duration: float) -> void:
    var tween := create_tween()
    tween.tween_property(self, "position", _end_pos, duration)
    await tween.finished
    _on_arrived()
```

### 3.3 Arc Flight (`arc: true`)

> 🇰🇷 **포물선 비행** — `arc: true` 설정 시 포물선 공식 적용

```gdscript
# Parabola: height_offset = arc_height × 4 × t × (1 - t),  t ∈ [0,1]
# 포물선: 높이 오프셋 = arc_height × 4 × t × (1 - t)
func _update_arc_position(t: float) -> void:
    var base_pos: Vector3 = _start_pos.lerp(_end_pos, t)
    base_pos.y += _arc_height * 4.0 * t * (1.0 - t)
    position = base_pos

func _fly_arc(duration: float) -> void:
    var tween := create_tween()
    tween.tween_method(_update_arc_position, 0.0, 1.0, duration)
    await tween.finished
    _on_arrived()
```

### 3.4 Arrival & Cleanup

> 🇰🇷 **도달 처리 및 정리** — VFX/SFX는 Phase B-4 이후 구현

```gdscript
func _on_arrived() -> void:
    # B-4+: play fx_hit, sfx_hit here
    # B-4+: 여기서 fx_hit, sfx_hit 재생
    print("[PROJ] hit → %s" % _config.get("fx_hit", ""))
    hit_arrived.emit(_context)
    queue_free()
```

### 3.5 Placeholder Mesh

> 🇰🇷 **임시 메시** — Phase 4+에서 실제 GLB 에셋으로 교체

```gdscript
func _build_placeholder_mesh() -> void:
    var mesh_inst := MeshInstance3D.new()
    add_child(mesh_inst)
    match _config.get("type", ""):
        "arrow":
            var m := CylinderMesh.new()
            m.height = 0.4
            m.top_radius = 0.01
            m.bottom_radius = 0.04
            mesh_inst.mesh = m
        _:  # magic_bolt, throw_stone
            var m := SphereMesh.new()
            m.radius = 0.08
            mesh_inst.mesh = m
```

---

## 4. ProjectileManager.gd

> 🇰🇷 **ProjectileManager.gd** — CombatScene 자식 노드, 씬 수준 스포너

**Path:** `scripts/combat/ProjectileManager.gd`
Added as child node of `CombatScene`. Not a singleton — follows TurnManager/GridManager pattern.

> 🇰🇷 `CombatScene`의 자식 노드로 추가. 싱글톤 아님 — TurnManager/GridManager 패턴과 동일.

```gdscript
class_name ProjectileManager
extends Node

var _active: Array[Projectile] = []

# Spawn a projectile and begin flight immediately.
# Caller connects hit_arrived on the returned node.
# 투사체 스폰 후 즉시 비행 시작. 반환 노드에 hit_arrived 연결 권장.
func spawn(
    projectile_id: String,
    start_pos: Vector3,
    end_pos: Vector3,
    context: Dictionary
) -> Projectile:
    var cfg: Dictionary = ProjectileConfig.get_config(projectile_id)
    if cfg.is_empty():
        push_warning("[ProjectileManager] Unknown id: '%s'" % projectile_id)
        return null

    var proj := Projectile.new()
    add_child(proj)
    _active.append(proj)
    proj.tree_exiting.connect(func() -> void: _active.erase(proj))

    proj.setup(cfg, start_pos, end_pos, context)
    proj.launch()
    return proj

# Emergency cleanup — call on combat end to free any in-flight projectiles.
# 전투 종료 시 비행 중 투사체 강제 정리.
func clear_all() -> void:
    for p in _active.duplicate():
        if is_instance_valid(p):
            p.queue_free()
    _active.clear()
```

---

## 5. CombatScene.gd Integration

> 🇰🇷 **CombatScene.gd 연동**

### 5.1 animation_config.json Change

> 🇰🇷 **`animation_config.json` 수정** — `attack_ranged` 이벤트 변경

Replace `hit_frame@0.55` with `spawn_projectile@0.40` in `attack_ranged`.
`cast_spell` is **unchanged** — Mage `basic_attack` remains instant (`hit_frame`).

> 🇰🇷 `attack_ranged`의 `hit_frame@0.55`를 `spawn_projectile@0.40`으로 교체.
> `cast_spell`은 **수정 없음** — 마법사 기본 공격은 hit_frame 즉발 유지.

**Before (`attack_ranged`):**
```json
{ "ratio": 0.55, "type": "hit_frame",     "data": {} },
{ "ratio": 0.55, "type": "camera_effect", "data": { "preset": "shake_light" } }
```

**After (`attack_ranged`):**
```json
{ "ratio": 0.40, "type": "spawn_projectile", "data": { "projectile": "arrow" } }
```

> 🇰🇷 `shake_light`는 `_on_projectile_hit()` 내부에서 투사체 도달 시 대상 위치 기준으로 트리거.

### 5.2 _handle_spawn_projectile() — Full Implementation

> 🇰🇷 **`_handle_spawn_projectile()` 전체 구현** — CombatScene.gd 812번째 줄 교체

```gdscript
func _handle_spawn_projectile(event: Dictionary) -> void:
    var data: Dictionary = event.get("data", {})
    var proj_id: String = data.get("projectile", "")
    if proj_id.is_empty():
        push_warning("[PROJ] spawn_projectile event missing 'projectile' key")
        return

    # Attacker chest height (+1.2m), target body center (+0.9m).
    # 공격자 가슴 높이(+1.2m) → 대상 몸통 중심(+0.9m) 경로.
    var start_pos: Vector3 = _dispatch_attacker.global_position + Vector3(0.0, 1.2, 0.0)
    var end_pos: Vector3   = _dispatch_target.global_position   + Vector3(0.0, 0.9, 0.0)

    # VALUE COPY — critical for concurrent projectile isolation.
    # 값 복사 필수 — 동시 비행 투사체 간 컨텍스트 오염 방지.
    var ctx: Dictionary = {
        "attacker":        _dispatch_attacker,
        "target":          _dispatch_target,
        "atk_ctx":         _dispatch_atk_ctx.duplicate(),
        "did_hit":         _dispatch_did_hit,
        "final_dmg":       _dispatch_final_dmg,
        "did_crit":        _dispatch_did_crit,
        "dmg_before_crit": _dispatch_dmg_before_crit,
        "back_bonus":      _dispatch_back_bonus,
        "hit_chance":      _dispatch_hit_chance,
        "is_magic":        _dispatch_is_magic,
    }

    var proj: Projectile = projectile_manager.spawn(proj_id, start_pos, end_pos, ctx)
    if proj != null:
        proj.hit_arrived.connect(_on_projectile_hit)
```

### 5.3 _on_projectile_hit() — New Function

> 🇰🇷 **`_on_projectile_hit()` 신규 함수** — 투사체 도달 콜백

```gdscript
func _on_projectile_hit(ctx: Dictionary) -> void:
    # Guard: target may have died while the projectile was in flight.
    # 가드: 투사체 비행 중 대상이 사망했을 수 있음.
    var target: CombatUnit = ctx.get("target")
    if not is_instance_valid(target) or not target.is_alive():
        push_warning("[PROJ] Target invalid or dead on arrival — skipping")
        return

    # Restore _dispatch_* vars from context snapshot.
    # 컨텍스트 스냅샷에서 _dispatch_* 변수 복원.
    _dispatch_attacker        = ctx["attacker"]
    _dispatch_target          = ctx["target"]
    _dispatch_atk_ctx         = ctx["atk_ctx"]
    _dispatch_did_hit         = ctx["did_hit"]
    _dispatch_final_dmg       = ctx["final_dmg"]
    _dispatch_did_crit        = ctx["did_crit"]
    _dispatch_dmg_before_crit = ctx["dmg_before_crit"]
    _dispatch_back_bonus      = ctx["back_bonus"]
    _dispatch_hit_chance      = ctx["hit_chance"]
    _dispatch_is_magic        = ctx["is_magic"]

    _handle_hit_frame()
```

### 5.4 Engagement Guard — Modified

> 🇰🇷 **교전 생성 가드 수정** — 원거리 공격(거리 > 1)은 교전 생성 안 함

**Location:** `CombatScene.gd` line ~700, inside `_resolve_attack()`.

**Before:**
```gdscript
if target.engaged_with == null and attacker.engaged_with == null:
    attacker.engaged_with = target
    ...
```

**After:**
```gdscript
# Only melee attacks (grid distance == 1) create engagement.
# 근접 공격(거리 1)만 교전 생성. 원거리는 교전 없음.
if grid_manager.grid_distance(attacker.grid_pos, target.grid_pos) <= 1:
    if target.engaged_with == null and attacker.engaged_with == null:
        attacker.engaged_with = target
        ...
```

---

## 6. Range System

> 🇰🇷 **사거리 시스템**

### 6.1 class_config.json — New Fields

> 🇰🇷 **`class_config.json` 수정** — `attack_range`와 `projectile` 필드 추가

```json
"Fighter": {
  "attack_range": 1,
  "actions": { "basic_attack": { "motion": "attack_melee" } }
},
"Archer": {
  "attack_range": 4,
  "actions": { "basic_attack": { "motion": "attack_ranged", "projectile": "arrow" } }
},
"Rogue": {
  "attack_range": 1,
  "actions": { "basic_attack": { "motion": "attack_melee" } }
},
"Mage": {
  "attack_range": 3,
  "actions": { "basic_attack": { "motion": "cast_spell" } }
}
```

> 🇰🇷 `attack_range`: 공격 가능 최대 거리(셀 단위). `projectile` 필드 있으면 해당 JSON 사용.
> Mage `basic_attack`에 `projectile` 없음 — `hit_frame` 즉발 유지.

### 6.2 GridManager.gd — New Functions

> 🇰🇷 **GridManager.gd 신규 함수 2개** — 거리 계산 및 범위 셀 조회

The grid uses a **staggered column** layout (odd columns: +0.5 Z offset).
Distance uses offset→cube conversion, then standard hex distance formula.

> 🇰🇷 그리드는 **스태거드 열** 레이아웃 (홀수 열: Z방향 +0.5 오프셋).
> 거리 계산: 오프셋 좌표 → 큐브 좌표 변환 후 hex 거리 공식 적용.

```gdscript
# Offset → cube coordinate conversion (staggered column grid).
# 오프셋 → 큐브 좌표 변환 (스태거드 열 그리드 기준).
func _offset_to_cube(pos: Vector2i) -> Vector3i:
    var q: int = pos.x
    var r: int = pos.y - (pos.x - (pos.x & 1)) / 2
    return Vector3i(q, r, -q - r)


# Returns the grid-cell distance between two positions.
# 두 위치 사이의 그리드 셀 거리 반환.
func grid_distance(a: Vector2i, b: Vector2i) -> int:
    var ca: Vector3i = _offset_to_cube(a)
    var cb: Vector3i = _offset_to_cube(b)
    return maxi(
        maxi(absi(ca.x - cb.x), absi(ca.y - cb.y)),
        absi(ca.z - cb.z)
    )


# Returns all cells within range_cells distance of center (inclusive).
# center에서 range_cells 이내의 모든 셀 반환 (경계 포함).
func get_cells_in_range(center: Vector2i, range_cells: int) -> Array[Vector2i]:
    var result: Array[Vector2i] = []
    for x in range(center.x - range_cells, center.x + range_cells + 1):
        for y in range(center.y - range_cells, center.y + range_cells + 1):
            var c := Vector2i(x, y)
            if is_in_bounds(c) and grid_distance(center, c) <= range_cells:
                result.append(c)
    return result
```

### 6.3 CombatUnit.gd — New Function

> 🇰🇷 **CombatUnit.gd 신규 함수** — 공격 사거리 조회

```gdscript
# Returns attack range in grid cells from class_config.
# class_config에서 공격 사거리(셀 단위) 반환. 기본값 1 (근접).
func get_attack_range() -> int:
    var class_cfg: Dictionary = ClassConfig.get_config(data.class_id)
    return int(class_cfg.get("attack_range", 1))
```

### 6.4 _get_attack_targets() — Modified

> 🇰🇷 **`_get_attack_targets()` 수정** — 사거리 기반 타겟 탐색 (CombatScene.gd 572번째 줄)

**Before:**
```gdscript
for cell in grid_manager.get_attack_neighbors(unit.grid_pos):
```

**After:**
```gdscript
var attack_range: int = unit.get_attack_range()
var cells: Array[Vector2i]
if attack_range <= 1:
    # Melee: neighbors only — keeps existing behavior.
    # 근접: 이웃 셀만 — 기존 동작 유지.
    cells = grid_manager.get_attack_neighbors(unit.grid_pos)
else:
    # Ranged: all cells within range.
    # 원거리: 사거리 내 모든 셀.
    cells = grid_manager.get_cells_in_range(unit.grid_pos, attack_range)
for cell in cells:
```

---

## 7. Extended Projectile JSON Schema

> 🇰🇷 **확장 투사체 JSON 스키마**

Full schema — all supported fields:

> 🇰🇷 지원하는 모든 필드를 포함한 전체 스키마.

```json
{
  "type": "arrow",
  "speed": 12.0,
  "arc": false,
  "arc_height": 0.0,
  "fx_trail": "fx_arrow_trail",
  "fx_hit": "fx_arrow_impact",
  "sfx_launch": "sfx_arrow_shoot",
  "sfx_hit": "sfx_arrow_hit",
  "rotate_to_velocity": true
}
```

Fields to add to each existing file:

> 🇰🇷 기존 JSON에 추가할 필드 목록:

`arrow.json` — add: `"arc_height": 0.0`, `"sfx_launch": "sfx_arrow_shoot"`, `"rotate_to_velocity": true`

`magic_bolt.json` — add: `"arc_height": 0.0`, `"sfx_launch": "sfx_magic_cast"`, `"rotate_to_velocity": false`

`throw_stone.json` — add: `"sfx_launch": "sfx_stone_throw"`, `"rotate_to_velocity": false`

---

## 8. Context Isolation (Critical)

> 🇰🇷 **컨텍스트 격리 — 설계 핵심 주의사항**

When multiple projectiles are in flight simultaneously, the CombatScene `_dispatch_*` vars will be overwritten by subsequent attacks before earlier projectiles arrive.

> 🇰🇷 동시에 여러 투사체가 날아가는 경우 `_dispatch_*` 변수가 덮어씌워짐.
> 예: 궁수 공격 → 적 궁수 반격 → 두 번째 공격이 변수 덮어쓰기.

**Solution:** Each projectile owns `_context: Dictionary` — a **value copy** (`.duplicate()` for nested dicts) made at spawn time. `_on_projectile_hit()` restores from the snapshot, not from live `_dispatch_*` vars.

> 🇰🇷 **해법:** 각 투사체는 스폰 시점에 `.duplicate()`로 만든 값 복사본을 보유.
> `_on_projectile_hit()`은 현재 `_dispatch_*`가 아닌 복사본에서 복원.

Required guard every time:

> 🇰🇷 **필수 가드** — 항상 확인:

```gdscript
# Target may have died to another projectile's earlier hit.
# 다른 투사체에 먼저 맞아 사망했을 수 있음.
if not is_instance_valid(target) or not target.is_alive():
    return
```

---

## 9. Implementation Priority

> 🇰🇷 **구현 우선순위**

### B-1 Core — Archer functional

> 🇰🇷 **B-1 코어** — 궁수가 동작하는 최소 구현

1. `ProjectileConfig.gd` — singleton, JSON loading
2. `Projectile.gd` — straight-line flight, `hit_arrived`, placeholder mesh
3. `ProjectileManager.gd` — `spawn()`, `clear_all()`, added to `combat_scene.tscn`
4. `animation_config.json` — `attack_ranged` events updated
5. `class_config.json` — `attack_range` + `projectile` fields
6. `CombatScene.gd` — `_handle_spawn_projectile()` full impl + `_on_projectile_hit()`
7. `CombatUnit.gd` — `get_attack_range()`
8. `GridManager.gd` — `grid_distance()` + `get_cells_in_range()`
9. `CombatScene.gd` — `_get_attack_targets()` range support + engagement guard

### B-2 Range Polish

> 🇰🇷 **B-2 사거리 다듬기**

- Enemy AI: add ranged target selection (currently uses `get_attack_neighbors`)
- Out-of-range indicator (dim tiles beyond `attack_range`)

### B-3 Flight Polish

> 🇰🇷 **B-3 비행 연출 다듬기**

- Arc flight for `throw_stone` (`arc: true` already in JSON)
- `rotate_to_velocity` for arrow (mesh faces direction of travel)

### B-4 VFX (Phase 4+)

> 🇰🇷 **B-4 시각 효과** (Phase 4 이후)

- Trail particles (`fx_trail`), impact VFX (`fx_hit`), audio (`sfx_launch`, `sfx_hit`)
- Camera shake on projectile impact

---

## 10. Deferred / Extended Scope

> 🇰🇷 **연기된/확장 범위 항목** — 이전 설계 문서에서 고려한 기능, 현재 범위 외

The following features were considered in prior design discussions but are intentionally deferred:

> 🇰🇷 아래 기능들은 이전 설계 논의에서 검토됐으나 현재 Phase B 범위에서 의도적으로 제외.

| Feature | Reason Deferred |
|---------|----------------|
| Archer `min_range` blind spot (range_min=2) | Adds UI complexity; revisit with passive skill system |
| LOS (Line of Sight) blocking by terrain | Requires full 3D terrain height data; Phase C+ |
| Mage `basic_attack` projectile (magic_bolt) | `cast_spell` stays instant; projectile Mage is skill-tier |
| Delivery system (`melee`/`projectile`/`instant` split) | Over-engineered for B-1; delivery is implied by motion |
| `visual` mesh config in projectile JSON | Placeholder mesh code handles it; data-driven later |

> 🇰🇷 위 항목들은 B-1~B-3 구현 완료 후 별도 설계 문서로 다룰 예정.

---

## 11. Verification Scenarios

> 🇰🇷 **검증 시나리오**

| # | Scenario | Expected Result |
|---|----------|-----------------|
| V-1 | Archer attacks enemy at distance ≥ 2 | Arrow flies; damage on arrival; no engagement |
| V-2 | Archer attacks adjacent enemy (distance 1) | Arrow flies; damage on arrival; engagement created |
| V-3 | Engaged Archer attacks | Forced to attack engagement partner (line 573); arrow still fires |
| V-4 | Throw stone (arc: true) | Parabolic arc to target; impact on arrival |
| V-5 | Target dies before arrow arrives | `is_instance_valid` guard triggers; hit skipped gracefully |
| V-6 | Two arrows in flight simultaneously | Independent contexts; no corruption |
| V-7 | Mage `basic_attack` | No projectile; `hit_frame` instant damage; unchanged |
| V-8 | Combat ends mid-flight | `ProjectileManager.clear_all()` frees all nodes safely |

---

## 12. File Change Summary

> 🇰🇷 **파일 변경 요약**

| File | Action | Key Change |
|------|--------|------------|
| `scripts/combat/Projectile.gd` | **NEW** | Full class |
| `scripts/combat/ProjectileManager.gd` | **NEW** | Full class |
| `scripts/singletons/ProjectileConfig.gd` | **NEW** | Full class; register in `project.godot` |
| `scripts/combat/CombatScene.gd` | MODIFY | Lines 572, ~700, 812 + new `_on_projectile_hit()` |
| `scripts/combat/CombatUnit.gd` | MODIFY | Add `get_attack_range()` |
| `scripts/combat/GridManager.gd` | MODIFY | Add `_offset_to_cube()`, `grid_distance()`, `get_cells_in_range()` |
| `data/animations/animation_config.json` | MODIFY | `attack_ranged`: remove `hit_frame@0.55`, add `spawn_projectile@0.40` |
| `data/classes/class_config.json` | MODIFY | Add `attack_range` + `projectile` per class |
| `data/projectiles/arrow.json` | MODIFY | Add `arc_height`, `sfx_launch`, `rotate_to_velocity` |
| `data/projectiles/magic_bolt.json` | MODIFY | Add `arc_height`, `sfx_launch`, `rotate_to_velocity` |
| `data/projectiles/throw_stone.json` | MODIFY | Add `sfx_launch`, `rotate_to_velocity` |

---

*@Planner — 2026-03-13*
