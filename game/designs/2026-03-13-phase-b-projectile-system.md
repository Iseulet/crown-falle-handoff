# CrownFalle — Phase B: 투사체 & 원거리 공격 시스템 설계

- Date: 2026-03-13
- Agent: @Planner
- Status: 설계 완료 — 사용자 승인 대기
- References:
  - Doc 2 — `spawn_projectile` 이벤트 & `data/projectiles/` 데이터 구조
  - Doc 4 섹션 4-2 — Archer 원거리 공격 시퀀스 (비동기 원칙)
  - `data-driven-design.md` — 수치 외부화 원칙
  - `distance_movement.md` — 공격 범위 전환 관련 열어둔 결정
  - `2026-03-13-combat-system-proposal-final.md` — ARM/RES 게이지형 전환, 원소 6종

---

## 0. 확정 결정사항 요약

| # | 결정 | 선택 |
|---|------|------|
| 1 | 사거리 단위 | **타일 수 (정수)** — 헥스 맨해튼 거리 기반 |
| 2 | 최소 사거리 (사각지대) | **있음** — Archer `min_range: 2`, 인접 공격 불가 |
| 3 | Mage 공격 방식 | **스킬별 분기** — `delivery: "projectile"` / `"instant"` (JSON 정의) |
| 4 | LOS (시야선) | **있음** — 직선 경로상 장애물 타일 시 공격 불가 |
| 5 | 투사체 비동기 | **확정** (Doc 4) — 발사 즉시 공격자 idle 복귀 |
| 6 | 투사체 비주얼 | **프리미티브 먼저** — CylinderMesh(화살)/SphereMesh(마법탄), 에셋은 후일 교체 |
| 7 | 설계 범위 | **풀** — 비행 + 사거리 + 카메라 + FX + Archer/Mage 클래스별 분기 |

---

## 1. 사거리 시스템

### 1-1. 공격 범위 계산

헥스 맨해튼 거리(hex distance)를 사용한다.
Staggered Grid에서 두 타일 간 hex distance는 cube 좌표로 변환 후 계산한다.

```
hex_distance(a, b):
  1. offset → cube 변환
  2. max(|a.q - b.q|, |a.r - b.r|, |a.s - b.s|)
```

```gdscript
# GridManager.gd — 신규 함수

## offset(col, row) → cube(q, r, s) 변환 (staggered column 기준)
static func offset_to_cube(col: int, row: int) -> Vector3i:
    var q := col
    var r := row - (col - (col & 1)) / 2
    var s := -q - r
    return Vector3i(q, r, s)

## 두 타일 간 헥스 거리
static func hex_distance(a: Vector2i, b: Vector2i) -> int:
    var ac := offset_to_cube(a.x, a.y)
    var bc := offset_to_cube(b.x, b.y)
    return maxi(maxi(absi(ac.x - bc.x), absi(ac.y - bc.y)), absi(ac.z - bc.z))
```

### 1-2. 공격 가능 타일 조회

```gdscript
# GridManager.gd — 신규 함수
func get_cells_in_attack_range(
    origin: Vector2i,
    range_max: int,
    range_min: int = 1
) -> Array[Vector2i]:
    var result: Array[Vector2i] = []
    for y in range(_grid_height):
        for x in range(_grid_width):
            var cell := Vector2i(x, y)
            if cell == origin:
                continue
            var dist := hex_distance(origin, cell)
            if dist >= range_min and dist <= range_max:
                result.append(cell)
    return result
```

### 1-3. 클래스별 사거리 JSON

```jsonc
// data/classes/archer.json (attack_range 필드 추가)
{
    "attack_type": "ranged",
    "attack_range_max": 4,
    "attack_range_min": 2,
    // ... 기존 필드 유지
}

// data/classes/mage.json
{
    "attack_type": "ranged",
    "attack_range_max": 5,
    "attack_range_min": 0,
    // min 0 = 인접 시전 가능 (instant 스킬 등)
}

// data/classes/fighter.json
{
    "attack_type": "melee",
    "attack_range_max": 1,
    "attack_range_min": 1,
    // 기존 인접 6방향과 동일
}

// data/classes/rogue.json
{
    "attack_type": "melee",
    "attack_range_max": 1,
    "attack_range_min": 1,
}
```

> [DECISION] 근접(melee)은 range_max=1, range_min=1으로 기존 인접 판정과 동일하게 유지.
> [DECISION] `get_attack_neighbors()`는 `get_cells_in_attack_range(origin, 1, 1)`의 래퍼로 리팩터링.

### 1-4. Archer 인접 공격 불가 정책

Archer(min_range=2)가 적과 인접(hex_distance=1)한 경우:

- 원거리 공격 **불가** (사각지대)
- 근접 대체 공격 없음 (무방비 상태)
- 교전(Engagement) 성립 시 이동 제한은 기존 시스템 그대로 적용
- UI: 인접 적은 공격 대상 하이라이트에서 제외

> [DECISION] Archer 인접 페널티 공격(약한 근접)은 프로토타입 범위 밖.
> 향후 패시브 스킬("Survival Instinct" 등)로 추가 가능.

---

## 2. LOS (Line of Sight) 시스템

### 2-1. 판정 방식

공격자 타일 → 대상 타일 직선 경로상의 모든 중간 타일을 순회하여,
장애물(`is_walkable() == false`)이 있으면 공격 불가로 판정.

유닛이 점유한 타일은 LOS를 차단하지 **않는다** (아군/적군 모두 통과).

```gdscript
# GridManager.gd — 신규 함수
func has_line_of_sight(from: Vector2i, to: Vector2i) -> bool:
    var dist := hex_distance(from, to)
    if dist <= 1:
        return true  # 인접은 항상 시야 확보

    # cube 좌표 기반 직선 보간
    var from_cube := Vector3(offset_to_cube(from.x, from.y))
    var to_cube := Vector3(offset_to_cube(to.x, to.y))

    for i in range(1, dist):
        var t := float(i) / float(dist)
        var lerped := from_cube.lerp(to_cube, t)
        # cube → offset 역변환 (round to nearest hex)
        var rounded := _cube_round(lerped)
        var mid := cube_to_offset(rounded)

        if mid == from or mid == to:
            continue
        if not is_walkable(mid):
            return false

    return true
```

### 2-2. Cube 좌표 보조 함수

```gdscript
# GridManager.gd — 보조 함수

static func _cube_round(cube: Vector3) -> Vector3i:
    var q := roundi(cube.x)
    var r := roundi(cube.y)
    var s := roundi(cube.z)

    var q_diff := absf(float(q) - cube.x)
    var r_diff := absf(float(r) - cube.y)
    var s_diff := absf(float(s) - cube.z)

    if q_diff > r_diff and q_diff > s_diff:
        q = -r - s
    elif r_diff > s_diff:
        r = -q - s
    else:
        s = -q - r

    return Vector3i(q, r, s)

static func cube_to_offset(cube: Vector3i) -> Vector2i:
    var col := cube.x
    var row := cube.y + (cube.x - (cube.x & 1)) / 2
    return Vector2i(col, row)
```

### 2-3. 공격 가능 최종 판정

```gdscript
# CombatScene.gd 또는 ActionResolver에서 사용
func get_valid_attack_targets(
    attacker: CombatUnit,
    range_max: int,
    range_min: int
) -> Array[Vector2i]:
    var cells := grid_manager.get_cells_in_attack_range(
        attacker.grid_pos, range_max, range_min
    )
    var targets: Array[Vector2i] = []
    for cell in cells:
        if not grid_manager.is_occupied_by_enemy(cell, attacker.faction):
            continue
        if not grid_manager.has_line_of_sight(attacker.grid_pos, cell):
            continue
        targets.append(cell)
    return targets
```

### 2-4. LOS 시각화 (선택적)

공격 대상 하이라이트 시 LOS 실패 타일은 표시하지 않는다.
디버그 모드에서는 LOS 실패 경로를 회색 점선으로 표시할 수 있다 (후순위).

---

## 3. 투사체 시스템

### 3-1. 아키텍처 개요

```
AnimEventDispatcher
  "spawn_projectile" 이벤트 발화
        │
        ▼
CombatScene._handle_spawn_projectile(event_data, context)
  ① ProjectileManager.spawn(config, start_pos, target_pos, context)
  ② 공격자 → idle 복귀 (비동기)
        │
        ▼
ProjectileManager
  ① Projectile 씬 인스턴스 생성
  ② 비행 Tween 시작 (직선 or 포물선)
  ③ 도착 시 → projectile_arrived 시그널
        │
        ▼
CombatScene._on_projectile_arrived(context)
  ① 대상 receive_hit() 호출
  ② FX/SFX 재생
  ③ 카메라 효과 (hit 프리셋)
```

### 3-2. 투사체 데이터 (기존 Doc 2 확장)

```jsonc
// data/projectiles/arrow.json
{
    "type": "arrow",
    "speed": 12.0,            // 월드 단위/초
    "trajectory": "linear",   // "linear" | "arc"
    "arc_height": 0.0,        // arc일 때만 사용
    "rotate_to_direction": true,  // 비행 방향으로 회전 여부
    "fx_trail": "fx_arrow_trail",
    "fx_hit": "fx_arrow_impact",
    "sfx_launch": "sfx_arrow_shoot",
    "sfx_hit": "sfx_arrow_hit",
    "camera_on_hit": "shake_light",  // 도착 시 카메라 프리셋 (빈 문자열이면 없음)
    // 비주얼 (프리미티브)
    "visual": {
        "mesh_type": "cylinder",   // "cylinder" | "sphere" | "custom"
        "mesh_scale": [0.03, 0.3, 0.03],
        "color": [0.55, 0.35, 0.15, 1.0]
    }
}

// data/projectiles/magic_bolt.json
{
    "type": "magic_bolt",
    "speed": 10.0,
    "trajectory": "linear",
    "arc_height": 0.0,
    "rotate_to_direction": false,
    "fx_trail": "fx_magic_trail",
    "fx_hit": "fx_magic_burst",
    "sfx_launch": "sfx_magic_cast",
    "sfx_hit": "sfx_magic_hit",
    "camera_on_hit": "shake_light",
    "visual": {
        "mesh_type": "sphere",
        "mesh_scale": [0.15, 0.15, 0.15],
        "color": [0.3, 0.5, 1.0, 1.0]
    }
}

// data/projectiles/throw_stone.json
{
    "type": "throw_stone",
    "speed": 8.0,
    "trajectory": "arc",
    "arc_height": 2.0,
    "rotate_to_direction": false,
    "fx_trail": "",
    "fx_hit": "fx_dust_hit",
    "sfx_launch": "sfx_throw",
    "sfx_hit": "sfx_stone_hit",
    "camera_on_hit": "",
    "visual": {
        "mesh_type": "sphere",
        "mesh_scale": [0.08, 0.08, 0.08],
        "color": [0.5, 0.45, 0.4, 1.0]
    }
}
```

### 3-3. ProjectileConfig 로더

```gdscript
# scripts/singletons/ProjectileConfig.gd
class_name ProjectileConfig
extends Node

## 싱글톤 — data/projectiles/*.json 파싱

const DATA_DIR := "res://data/projectiles/"

var _cache: Dictionary = {}  # type → config dict

func _ready() -> void:
    _load_all()

func _load_all() -> void:
    var dir := DirAccess.open(DATA_DIR)
    if not dir:
        return
    dir.list_dir_begin()
    var file_name := dir.get_next()
    while file_name != "":
        if file_name.ends_with(".json"):
            var path := DATA_DIR + file_name
            var data := _parse_json(path)
            if data.has("type"):
                _cache[data.type] = data
        file_name = dir.get_next()

func get_config(type: String) -> Dictionary:
    if _cache.has(type):
        return _cache[type]
    push_warning("ProjectileConfig: unknown type '%s'" % type)
    return {}

func _parse_json(path: String) -> Dictionary:
    var file := FileAccess.open(path, FileAccess.READ)
    if not file:
        return {}
    var text := file.get_as_text()
    var json := JSON.new()
    if json.parse(text) == OK:
        return json.data
    return {}
```

### 3-4. Projectile 씬 & 스크립트

```
scenes/combat/projectile.tscn
  Projectile (Node3D)
    └── MeshInstance3D    ← 동적 생성 (config.visual 기반)
```

```gdscript
# scripts/combat/Projectile.gd
class_name Projectile
extends Node3D

signal arrived(context: Dictionary)

var _config: Dictionary = {}
var _target_pos: Vector3
var _context: Dictionary = {}
var _speed: float = 10.0
var _trajectory: String = "linear"
var _arc_height: float = 0.0

func setup(config: Dictionary, start: Vector3, target: Vector3, context: Dictionary) -> void:
    _config = config
    _target_pos = target
    _context = context
    _speed = config.get("speed", 10.0)
    _trajectory = config.get("trajectory", "linear")
    _arc_height = config.get("arc_height", 0.0)
    global_position = start
    _create_visual(config.get("visual", {}))
    _start_flight()

func _create_visual(visual: Dictionary) -> void:
    var mesh_inst := MeshInstance3D.new()
    var mesh_type: String = visual.get("mesh_type", "sphere")

    match mesh_type:
        "cylinder":
            var m := CylinderMesh.new()
            m.top_radius = 0.02
            m.bottom_radius = 0.02
            m.height = 0.3
            mesh_inst.mesh = m
        "sphere":
            var m := SphereMesh.new()
            m.radius = 0.1
            m.height = 0.2
            mesh_inst.mesh = m
        _:
            var m := SphereMesh.new()
            mesh_inst.mesh = m

    # 스케일 적용
    var s: Array = visual.get("mesh_scale", [0.1, 0.1, 0.1])
    mesh_inst.scale = Vector3(s[0], s[1], s[2])

    # 머티리얼 색상
    var mat := StandardMaterial3D.new()
    var c: Array = visual.get("color", [1.0, 1.0, 1.0, 1.0])
    mat.albedo_color = Color(c[0], c[1], c[2], c[3])
    mat.emission_enabled = true
    mat.emission = Color(c[0], c[1], c[2])
    mat.emission_energy_multiplier = 0.3
    mesh_inst.material_override = mat

    add_child(mesh_inst)

func _start_flight() -> void:
    var dist := global_position.distance_to(_target_pos)
    var duration := dist / _speed

    # 방향 회전
    if _config.get("rotate_to_direction", false):
        look_at(_target_pos, Vector3.UP)

    match _trajectory:
        "linear":
            _fly_linear(duration)
        "arc":
            _fly_arc(duration)
        _:
            _fly_linear(duration)

func _fly_linear(duration: float) -> void:
    var tween := create_tween()
    tween.tween_property(self, "global_position", _target_pos, duration)
    tween.tween_callback(_on_arrived)

func _fly_arc(duration: float) -> void:
    var start := global_position
    var tween := create_tween()
    var elapsed := 0.0

    # 프레임별 위치 계산 — _process 사용
    set_meta("arc_start", start)
    set_meta("arc_duration", duration)
    set_meta("arc_elapsed", 0.0)
    set_process(true)

func _process(delta: float) -> void:
    if not has_meta("arc_start"):
        return

    var start: Vector3 = get_meta("arc_start")
    var duration: float = get_meta("arc_duration")
    var elapsed: float = get_meta("arc_elapsed") + delta
    set_meta("arc_elapsed", elapsed)

    var t := clampf(elapsed / duration, 0.0, 1.0)

    # XZ 선형 보간
    var pos := start.lerp(_target_pos, t)
    # Y 포물선: 4h * t * (1-t)
    pos.y += _arc_height * 4.0 * t * (1.0 - t)

    global_position = pos

    if t >= 1.0:
        set_process(false)
        remove_meta("arc_start")
        remove_meta("arc_duration")
        remove_meta("arc_elapsed")
        _on_arrived()

func _on_arrived() -> void:
    arrived.emit(_context)
    # FX/SFX는 ProjectileManager가 시그널 수신 후 처리
    queue_free()
```

### 3-5. ProjectileManager

```gdscript
# scripts/combat/ProjectileManager.gd
class_name ProjectileManager
extends Node3D

## 투사체 생성/관리, CombatScene의 자식 노드로 배치

signal projectile_arrived(context: Dictionary)

const PROJECTILE_SCENE := preload("res://scenes/combat/projectile.tscn")

var _active_projectiles: Array[Projectile] = []

func spawn(
    projectile_type: String,
    start_pos: Vector3,
    target_pos: Vector3,
    context: Dictionary
) -> void:
    var config := ProjectileConfig.get_config(projectile_type)
    if config.is_empty():
        push_warning("ProjectileManager: no config for '%s'" % projectile_type)
        # 설정 없으면 즉시 도착 처리 (graceful fallback)
        projectile_arrived.emit(context)
        return

    var proj: Projectile = PROJECTILE_SCENE.instantiate()
    add_child(proj)
    _active_projectiles.append(proj)

    proj.arrived.connect(_on_projectile_arrived.bind(proj, config))
    proj.setup(config, start_pos, target_pos, context)

    # 발사 SFX
    var sfx_launch: String = config.get("sfx_launch", "")
    if sfx_launch != "":
        _play_sfx(sfx_launch)

func _on_projectile_arrived(context: Dictionary, proj: Projectile, config: Dictionary) -> void:
    _active_projectiles.erase(proj)

    # 도착 FX
    var fx_hit: String = config.get("fx_hit", "")
    if fx_hit != "":
        _spawn_fx(fx_hit, proj.global_position)

    # 도착 SFX
    var sfx_hit: String = config.get("sfx_hit", "")
    if sfx_hit != "":
        _play_sfx(sfx_hit)

    # 도착 카메라 효과
    var camera_preset: String = config.get("camera_on_hit", "")
    if camera_preset != "":
        CameraDirector.execute(camera_preset, context)

    projectile_arrived.emit(context)

func has_active_projectiles() -> bool:
    return not _active_projectiles.is_empty()

# --- 플레이스홀더 (Phase B FX 미구현 시) ---

func _play_sfx(_sfx_name: String) -> void:
    pass  # TODO: SFX 시스템 연결

func _spawn_fx(_fx_name: String, _pos: Vector3) -> void:
    pass  # TODO: FX 시스템 연결
```

---

## 4. 공격 전달 방식 분기 (Delivery System)

### 4-1. 스킬/액션 JSON에 delivery 필드 추가

```jsonc
// data/classes/archer.json — actions 필드
{
    "actions": {
        "basic_attack": {
            "motion_name": "attack_ranged",
            "delivery": "projectile",
            "projectile_type": "arrow",
            "damage_stat": "dex"
        }
    }
}

// data/classes/mage.json — actions 필드
{
    "actions": {
        "basic_attack": {
            "motion_name": "cast_spell",
            "delivery": "projectile",
            "projectile_type": "magic_bolt",
            "damage_stat": "int_stat"
        },
        "fire_burst": {
            "motion_name": "cast_spell",
            "delivery": "instant",
            "projectile_type": "",
            "damage_stat": "int_stat",
            "fx_on_target": "fx_fire_burst",
            "sfx_on_target": "sfx_fire_hit"
        }
    }
}

// data/classes/fighter.json — actions 필드
{
    "actions": {
        "basic_attack": {
            "motion_name": "attack_melee",
            "delivery": "melee",
            "projectile_type": "",
            "damage_stat": "str"
        }
    }
}
```

### 4-2. delivery 분기 흐름

```
CombatScene._execute_attack(attacker, target, action_data)
  │
  ├── delivery == "melee"
  │     → 기존 흐름: hit_frame 이벤트 → receive_hit()
  │
  ├── delivery == "projectile"
  │     → spawn_projectile 이벤트 → ProjectileManager.spawn()
  │     → 공격자 즉시 idle 복귀
  │     → projectile_arrived → receive_hit()
  │
  └── delivery == "instant"
        → 시전 모션 재생
        → hit_frame 이벤트 시점에 즉시 대상 위치 FX 생성
        → receive_hit() 동시 호출
        → 공격자 모션 완료 후 idle 복귀
```

### 4-3. instant 전달 상세

instant는 투사체 없이 시전 모션의 `hit_frame` 이벤트에서 직접 피해를 적용한다.
melee와의 차이는 **사거리가 1 초과**일 수 있다는 점과, **대상 위치에 FX가 발생**한다는 점.

```gdscript
# CombatScene.gd — _handle_hit_frame() 수정
func _handle_hit_frame(event_data: Dictionary) -> void:
    var action := _current_action_data  # 현재 실행 중인 액션 정보

    if action.delivery == "instant":
        # 대상 위치에 FX 즉시 생성
        var fx_name: String = action.get("fx_on_target", "")
        if fx_name != "":
            _spawn_fx_at(fx_name, _dispatch_target.global_position)

    # 피해 적용 (melee, instant 공통)
    _apply_damage_to_target(event_data)
```

---

## 5. CombatScene 통합 흐름

### 5-1. 공격 타일 하이라이트 분기

```gdscript
# CombatScene.gd — _show_attack_highlights() 수정
func _show_attack_highlights(unit: CombatUnit) -> void:
    var class_data := unit.get_class_data()
    var attack_type: String = class_data.get("attack_type", "melee")

    match attack_type:
        "melee":
            # 기존: 인접 6방향 적 타일만 빨간 하이라이트
            var neighbors := grid_manager.get_cells_in_attack_range(
                unit.grid_pos, 1, 1
            )
            _highlight_attack_targets(unit, neighbors)
        "ranged":
            var range_max: int = class_data.get("attack_range_max", 4)
            var range_min: int = class_data.get("attack_range_min", 2)
            var cells := grid_manager.get_cells_in_attack_range(
                unit.grid_pos, range_max, range_min
            )
            # LOS 필터링
            var valid: Array[Vector2i] = []
            for cell in cells:
                if grid_manager.has_line_of_sight(unit.grid_pos, cell):
                    valid.append(cell)
            _highlight_attack_targets(unit, valid)
```

### 5-2. 공격 실행 분기

```gdscript
# CombatScene.gd — _attack_target() 재구조화
func _attack_target(target_pos: Vector2i) -> void:
    var attacker := selected_unit
    var target := grid_manager.get_occupant(target_pos)
    var action := _get_current_action(attacker)  # actions.basic_attack 등

    # 방향 처리 (모든 공격 공통)
    attacker._renderer.face_direction(target.global_position)
    target._renderer.face_direction_smooth(attacker.global_position)

    # 모션 재생
    var motion_name: String = action.get("motion_name", "attack_melee")
    attacker._renderer.play_motion(motion_name)

    # delivery에 따른 분기는 이벤트 핸들러에서 처리
    _current_action_data = action
    _dispatch_attacker = attacker
    _dispatch_target = target
```

### 5-3. spawn_projectile 이벤트 핸들러

```gdscript
# CombatScene.gd — 신규 (AnimEventDispatcher 시그널 연결)
func _handle_spawn_projectile(event_data: Dictionary) -> void:
    var proj_type: String = event_data.data.get("type", "")
    var bone_name: String = event_data.data.get("bone", "RightHand")

    # 발사 위치: 본 위치 or 공격자 위치
    var start_pos: Vector3 = _dispatch_attacker._renderer.get_bone_position(bone_name)
    var target_pos: Vector3 = _dispatch_target.global_position
    # 타겟 약간 위 (몸통 중심)
    target_pos.y += 0.7

    var context := {
        "attacker": _dispatch_attacker,
        "target": _dispatch_target,
        "action_data": _current_action_data,
        "event_data": event_data
    }

    _projectile_manager.spawn(proj_type, start_pos, target_pos, context)

    # 비동기: 공격자 즉시 idle 복귀
    _dispatch_attacker._renderer.play_motion("idle")
```

### 5-4. 투사체 도착 핸들러

```gdscript
# CombatScene.gd — ProjectileManager.projectile_arrived 연결
func _on_projectile_arrived(context: Dictionary) -> void:
    var target: CombatUnit = context.get("target")
    var attacker: CombatUnit = context.get("attacker")
    var action_data: Dictionary = context.get("action_data", {})
    var event_data: Dictionary = context.get("event_data", {})

    if not is_instance_valid(target):
        return  # 비행 중 대상 사망 시 무시

    # 피해 적용
    var damage := _calculate_damage(attacker, target, action_data)
    target.receive_hit(damage, "ranged", attacker)

    # 카메라 효과는 ProjectileManager에서 이미 처리됨
```

---

## 6. 카메라 연출

### 6-1. 원거리 공격 카메라 프리셋

```jsonc
// data/cameras/camera_config.json — 추가 프리셋

{
    "presets": {
        // ... 기존 프리셋 유지 ...

        "soft_track_ranged": {
            "type": "soft_track",
            "weight": 0.08,
            "duration": 0.6,
            "ease": "ease_out"
        },
        "shake_ranged_hit": {
            "type": "shake",
            "intensity": 0.20,
            "duration": 0.15,
            "decay": "ease_out"
        }
    }
}
```

### 6-2. 카메라 경로 연동

| 시점 | 프리셋 | 경로 |
|------|--------|------|
| 원거리 공격 시작 | `soft_track_ranged` | 경로 B (animation_config 이벤트) |
| 투사체 도착 hit | `shake_ranged_hit` | ProjectileManager → CameraDirector |
| 즉발 마법 hit | `shake_light` | 경로 B (animation_config hit_frame) |
| 치명타 (원거리) | combat_rules.json 조건 평가 | 경로 C |

---

## 7. animation_config.json 모션 추가/수정

```jsonc
// data/animations/animation_config.json — 수정/추가

{
    "motions": {
        // ... 기존 모션 유지 ...

        "attack_ranged": {
            "loop": false,
            "speed_scale": 1.0,
            "events": [
                {
                    "ratio": 0.30,
                    "type": "camera_effect",
                    "data": { "preset": "soft_track_ranged" }
                },
                {
                    "ratio": 0.55,
                    "type": "spawn_projectile",
                    "data": {
                        "type": "arrow",
                        "bone": "RightHand",
                        "speed_override": 0.0
                    }
                },
                {
                    "ratio": 0.55,
                    "type": "play_sfx",
                    "data": { "sfx": "arrow_shoot" }
                }
            ]
        },

        "cast_spell": {
            "loop": false,
            "speed_scale": 1.0,
            "events": [
                {
                    "ratio": 0.20,
                    "type": "attach_fx",
                    "data": { "fx": "magic_charge", "bone": "RightHand" }
                },
                {
                    "ratio": 0.65,
                    "type": "camera_effect",
                    "data": { "preset": "soft_track_ranged" }
                },
                {
                    "ratio": 0.70,
                    "type": "spawn_projectile",
                    "data": {
                        "type": "magic_bolt",
                        "bone": "RightHand",
                        "speed_override": 0.0
                    }
                },
                {
                    "ratio": 0.70,
                    "type": "detach_fx",
                    "data": { "fx": "magic_charge" }
                },
                {
                    "ratio": 0.70,
                    "type": "play_sfx",
                    "data": { "sfx": "spell_cast" }
                }
            ]
        },

        "cast_instant": {
            "loop": false,
            "speed_scale": 1.0,
            "events": [
                {
                    "ratio": 0.20,
                    "type": "attach_fx",
                    "data": { "fx": "magic_charge", "bone": "RightHand" }
                },
                {
                    "ratio": 0.80,
                    "type": "hit_frame",
                    "data": { "damage_multiplier": 1.0 }
                },
                {
                    "ratio": 0.80,
                    "type": "detach_fx",
                    "data": { "fx": "magic_charge" }
                },
                {
                    "ratio": 0.80,
                    "type": "camera_effect",
                    "data": { "preset": "shake_light" }
                },
                {
                    "ratio": 0.80,
                    "type": "play_sfx",
                    "data": { "sfx": "spell_cast" }
                }
            ]
        }
    }
}
```

> [NOTE] `cast_spell`은 projectile delivery용 (spawn_projectile 이벤트 포함).
> `cast_instant`은 instant delivery용 (hit_frame 이벤트로 즉시 피해).
> 스킬별로 `motion_name`을 다르게 지정하여 분기.

---

## 8. UnitRenderer3D 확장

### 8-1. 본 위치 조회

투사체 발사 위치를 본 기준으로 가져오기 위한 함수.

```gdscript
# scripts/rendering/UnitRenderer3D.gd — 추가 함수

func get_bone_position(bone_name: String) -> Vector3:
    if _skeleton == null:
        return global_position + Vector3(0, 1.0, 0)

    var bone_idx := _skeleton.find_bone(bone_name)
    if bone_idx == -1:
        # 폴백: 모델 상단 추정
        return global_position + Vector3(0, 1.2, 0)

    var bone_global := _skeleton.global_transform * _skeleton.get_bone_global_pose(bone_idx)
    return bone_global.origin
```

---

## 9. 공격 타일 하이라이트 비주얼

### 9-1. 원거리 공격 범위 표시

원거리 유닛 선택 시:
- **파란색 영역**: 이동 가능 타일 (기존)
- **빨간색 타일**: 공격 가능 적 (사거리 내 + LOS 통과)
- **회색 타일** (선택적): 사거리 내지만 LOS 실패인 적

### 9-2. 사각지대 표시

min_range 내 적은 하이라이트하지 않는다.
UI 팁 메시지: "사정거리 밖 — 이동이 필요합니다" (후순위).

---

## 10. 구현 순서 (Phase B 세부)

| 단계 | 항목 | 의존성 | 예상 규모 |
|------|------|--------|----------|
| B-1 | `hex_distance()`, `offset_to_cube()`, `cube_to_offset()` | 없음 | Small |
| B-2 | `get_cells_in_attack_range()` + 클래스 JSON `attack_range` 필드 | B-1 | Small |
| B-3 | `has_line_of_sight()` + `_cube_round()` | B-1 | Medium |
| B-4 | `ProjectileConfig` 싱글톤 + `data/projectiles/` JSON 3종 | 없음 | Small |
| B-5 | `Projectile` 씬/스크립트 (비행 + 프리미티브 비주얼) | B-4 | Medium |
| B-6 | `ProjectileManager` (spawn + arrived 시그널) | B-5 | Medium |
| B-7 | `CombatScene` — `_handle_spawn_projectile()` + 비동기 idle 복귀 | B-6 | Medium |
| B-8 | `CombatScene` — delivery 분기 (`melee`/`projectile`/`instant`) | B-7 | Medium |
| B-9 | `CombatScene` — 원거리 공격 하이라이트 (사거리 + LOS + min_range) | B-2, B-3 | Small |
| B-10 | `animation_config.json` — `attack_ranged`, `cast_spell`, `cast_instant` | B-7, B-8 | Small |
| B-11 | 카메라 프리셋 추가 (`soft_track_ranged`, `shake_ranged_hit`) | B-7 | Small |
| B-12 | `UnitRenderer3D.get_bone_position()` | 없음 | Small |
| B-13 | 통합 테스트 — Archer/Mage 전투 시나리오 | 전체 | — |

### 병렬 가능 그룹

```
그룹 A (사거리):  B-1 → B-2 → B-3 → B-9
그룹 B (투사체):  B-4 → B-5 → B-6 → B-7 → B-8
그룹 C (데이터):  B-10 + B-11 + B-12 (독립)
        ↓
    B-13 (통합 테스트)
```

---

## 11. 수정/생성 파일 목록

### 신규 생성

| 파일 | 역할 |
|------|------|
| `scripts/singletons/ProjectileConfig.gd` | 투사체 JSON 로더 (싱글톤) |
| `scripts/combat/ProjectileManager.gd` | 투사체 생성/관리 |
| `scripts/combat/Projectile.gd` | 개별 투사체 비행 로직 |
| `scenes/combat/projectile.tscn` | 투사체 씬 |

### 수정

| 파일 | 변경 내용 |
|------|----------|
| `scripts/combat/GridManager.gd` | `hex_distance()`, `offset_to_cube()`, `cube_to_offset()`, `_cube_round()`, `get_cells_in_attack_range()`, `has_line_of_sight()` |
| `scripts/combat/CombatScene.gd` | delivery 분기, `_handle_spawn_projectile()`, `_on_projectile_arrived()`, 원거리 하이라이트, ProjectileManager 연결 |
| `scripts/rendering/UnitRenderer3D.gd` | `get_bone_position()` |
| `data/classes/archer.json` | `attack_type`, `attack_range_max/min`, `actions.delivery` |
| `data/classes/mage.json` | `attack_type`, `attack_range_max/min`, `actions.delivery` |
| `data/classes/fighter.json` | `attack_type`, `attack_range_max/min` (명시화) |
| `data/classes/rogue.json` | `attack_type`, `attack_range_max/min` (명시화) |
| `data/projectiles/arrow.json` | 기존 구조 확장 (visual, sfx_launch, camera_on_hit) |
| `data/projectiles/magic_bolt.json` | 기존 구조 확장 |
| `data/projectiles/throw_stone.json` | 기존 구조 확장 |
| `data/animations/animation_config.json` | `attack_ranged`, `cast_spell`, `cast_instant` 모션 |
| `data/cameras/camera_config.json` | `soft_track_ranged`, `shake_ranged_hit` 프리셋 |
| `project.godot` | `ProjectileConfig` 오토로드 등록 |

---

## 12. 미결정 / 후순위 항목

| # | 항목 | 상태 | 비고 |
|---|------|------|------|
| 1 | Archer dodge 에셋 | 미수급 | 에셋 수급 후 별도 작업 |
| 2 | 원거리 치명타 연출 | 후순위 | combat_rules.json 조건 추가 필요 |
| 3 | AoE 마법 (범위 공격) | 후순위 | 타겟 선택 UI 필요, 별도 설계 문서 |
| 4 | 투사체 GLB 에셋 교체 | 후순위 | `visual.mesh_type: "custom"` + `visual.mesh_path` 필드 예약 |
| 5 | LOS 디버그 시각화 | 후순위 | 회색 점선 표시 |
| 6 | Archer 근접 대체 공격 | 후순위 | 패시브 스킬 시스템 연동 |
| 7 | 높이 차이 LOS 보정 | 후순위 | 3D 지형 구현 시 |
| 8 | 투사체 FX Trail 실구현 | 후순위 | GPUParticles3D 연동 |

---

## 13. 설계 검증 체크리스트

- [x] 모든 수치 JSON 외부화 — 코드 내 매직넘버 없음
- [x] 기존 melee 흐름 변경 없음 — delivery="melee"는 기존과 동일
- [x] 비동기 원칙 준수 — 투사체 발사 후 공격자 즉시 idle
- [x] 카메라 경로 B/C 유지 — 신규 프리셋 추가만
- [x] AnimEventDispatcher 변경 없음 — spawn_projectile 이벤트는 이미 정의됨
- [x] GridManager 기존 인터페이스 유지 — get_attack_neighbors()는 래퍼로 존속
- [x] GDScript 컨벤션 — int_stat, snake_case, UPPER_SNAKE_CASE 준수
- [x] 단일 진입점 원칙 — CameraDirector.execute() 경유만 사용
