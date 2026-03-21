# Doc 7 — 환경 시스템 (WorldEnvironment + 분위기 연출)
> 작성일: 2026-03-12 | 작성자: @Planner
> 대상 경로: `handoff/plans/design/2026-03-12-environment_system.md`

---

## 개요

전투 씬의 하늘/안개/조명/포스트프로세싱을 **데이터 기반**으로 제어하는 환경 시스템.
기존 AnimationConfig / CameraConfig 싱글톤 패턴을 그대로 따른다.

### 핵심 결정사항
- **Sky**: ProceduralSky (파라미터 제어, 다크 판타지 톤 고정 용이)
- **포스트프로세싱**: SSAO + Fog + Tonemap (Glow/Bloom 미사용)
- **프리셋 3종**: overcast_day / dusk / night
- **프리셋 선택**: map_presets 매핑 우선, 매핑 없으면 전체 프리셋 중 랜덤

---

## 1. 파일 구조

### 신규 생성

```
data/environment/
  └── environment_config.json          ← 환경 프리셋 데이터

scripts/singletons/
  └── EnvironmentConfig.gd             ← JSON 파싱 + 쿼리 (static class)

scripts/rendering/
  └── EnvironmentBuilder.gd            ← WorldEnvironment + Sun 생성 (static)
```

### 수정 대상

```
scripts/combat/CombatScene.gd          ← _setup_lighting() → _setup_environment() 교체
scripts/rendering/UnitRenderer3D.gd    ← 모델 layers = 1 | 2 설정 (_set_character_layers)
```

---

## 2. JSON 구조 — `data/environment/environment_config.json`

### 2.1 전체 스키마

```jsonc
{
  "presets": {
    "<preset_name>": {
      "sky": { ... },              // ProceduralSkyMaterial 파라미터
      "sun": { ... },              // DirectionalLight3D 파라미터
      "ambient": { ... },          // Environment ambient light
      "fog": { ... },              // 거리 안개 + 높이 안개
      "ssao": { ... },             // 앰비언트 오클루전
      "tonemap": { ... },          // 톤매핑
      "character_light": { ... }   // 캐릭터 전용 보조광 (레이어 2)
    }
  },
  "map_presets": {           // 맵 이름 → 프리셋 이름 (명시적 매핑)
    "<map_name>": "<preset_name>"
  },
  "default_preset": "<preset_name>"  // map_presets에 없고 랜덤 비활성 시 폴백
}
```

### 2.2 섹션별 스키마 상세

#### sky

| 키 | 타입 | 설명 | Godot 프로퍼티 |
|----|------|------|---------------|
| sky_top_color | [r, g, b] | 하늘 꼭대기 색 | ProceduralSkyMaterial.sky_top_color |
| sky_horizon_color | [r, g, b] | 하늘 수평선 색 | ProceduralSkyMaterial.sky_horizon_color |
| ground_bottom_color | [r, g, b] | 지면 바닥 색 | ProceduralSkyMaterial.ground_bottom_color |
| ground_horizon_color | [r, g, b] | 지면 수평선 색 | ProceduralSkyMaterial.ground_horizon_color |
| sun_angle_max | float | 태양 광원 크기 (도) | ProceduralSkyMaterial.sun_angle_max |
| sun_curve | float | 그래디언트 전환 커브 | ProceduralSkyMaterial.sun_curve |

#### sun

| 키 | 타입 | 설명 | Godot 프로퍼티 |
|----|------|------|---------------|
| color | [r, g, b] | 태양광 색상 | DirectionalLight3D.light_color |
| energy | float | 태양광 강도 | DirectionalLight3D.light_energy |
| rotation_deg | [x, y, z] | 회전 각도 (도) | DirectionalLight3D.rotation_degrees |
| shadow_enabled | bool | 그림자 활성화 | DirectionalLight3D.shadow_enabled |
| shadow_blur | float | 그림자 블러 | DirectionalLight3D.shadow_blur |

#### ambient

| 키 | 타입 | 설명 | Godot 프로퍼티 |
|----|------|------|---------------|
| color | [r, g, b] | 앰비언트 색상 | Environment.ambient_light_color |
| energy | float | 앰비언트 강도 | Environment.ambient_light_energy |
| sky_contribution | float | 하늘 기여도 (0~1) | Environment.ambient_light_sky_contribution |

#### fog

| 키 | 타입 | 설명 | Godot 프로퍼티 |
|----|------|------|---------------|
| enabled | bool | 안개 활성화 | Environment.fog_enabled |
| color | [r, g, b] | 안개 색상 | Environment.fog_light_color |
| density | float | 거리 안개 밀도 | Environment.fog_density |
| sky_affect | float | 하늘에 대한 안개 영향 | Environment.fog_sky_affect |
| height_enabled | bool | 높이 안개 활성화 | — (조건부 적용) |
| height_max | float | 높이 안개 최대 높이 | Environment.fog_height |
| height_density | float | 높이 안개 밀도 | Environment.fog_height_density |

#### ssao

| 키 | 타입 | 설명 | Godot 프로퍼티 |
|----|------|------|---------------|
| enabled | bool | SSAO 활성화 | Environment.ssao_enabled |
| radius | float | 샘플링 반경 | Environment.ssao_radius |
| intensity | float | 강도 | Environment.ssao_intensity |
| power | float | 거듭제곱 | Environment.ssao_power |
| detail | float | 디테일 | Environment.ssao_detail |
| light_affect | float | 직사광 SSAO 감소량 | Environment.ssao_light_affect |

#### tonemap

| 키 | 타입 | 설명 | Godot 프로퍼티 |
|----|------|------|---------------|
| mode | string | "linear" / "reinhardt" / "filmic" / "aces" | Environment.tonemap_mode |
| exposure | float | 노출 | Environment.tonemap_exposure |
| white | float | 화이트 포인트 | Environment.tonemap_white |

#### character_light

| 키 | 타입 | 설명 | Godot 프로퍼티 |
|----|------|------|---------------|
| enabled | bool | 캐릭터 라이트 활성화 여부 | — (조건부 생성) |
| color | [r, g, b] | 라이트 색상 | DirectionalLight3D.light_color |
| energy | float | 라이트 강도 | DirectionalLight3D.light_energy |
| rotation_deg | [x, y, z] | 회전 각도 (도) | DirectionalLight3D.rotation_degrees |
| shadow_enabled | bool | 그림자 활성화 (기본 false) | DirectionalLight3D.shadow_enabled |

### 2.3 프리셋 데이터 (1차 튜닝값 적용 — 2026-03-12)

> 변경 이력: 초안 대비 sun.energy ×1.5~2×, fog.density ×1/4~1/5, character_light 섹션 추가

```jsonc
{
  "presets": {
    "overcast_day": {
      "sky": {
        "sky_top_color":       [0.30, 0.33, 0.42],
        "sky_horizon_color":   [0.55, 0.55, 0.52],
        "ground_bottom_color": [0.18, 0.16, 0.14],
        "ground_horizon_color":[0.45, 0.42, 0.38],
        "sun_angle_max":       30.0,
        "sun_curve":           0.10
      },
      "sun": {
        "color":            [0.85, 0.82, 0.78],
        "energy":           1.2,
        "rotation_deg":     [-45.0, 45.0, 0.0],
        "shadow_enabled":   true,
        "shadow_blur":      1.5
      },
      "ambient": {
        "color":              [0.30, 0.30, 0.35],
        "energy":             0.6,
        "sky_contribution":   0.3
      },
      "fog": {
        "enabled":          true,
        "color":            [0.50, 0.50, 0.52],
        "density":          0.002,
        "sky_affect":       0.5,
        "height_enabled":   true,
        "height_max":       8.0,
        "height_density":   0.015
      },
      "ssao": {
        "enabled":       true,
        "radius":        1.0,
        "intensity":     1.5,
        "power":         1.5,
        "detail":        0.5,
        "light_affect":  0.2
      },
      "tonemap": {
        "mode":      "filmic",
        "exposure":  1.0,
        "white":     1.0
      },
      "character_light": {
        "enabled":        true,
        "color":          [1.0, 0.95, 0.90],
        "energy":         0.5,
        "rotation_deg":   [-40.0, 30.0, 0.0],
        "shadow_enabled": false
      }
    },

    "dusk": {
      "sky": {
        "sky_top_color":       [0.15, 0.12, 0.25],
        "sky_horizon_color":   [0.70, 0.40, 0.25],
        "ground_bottom_color": [0.10, 0.08, 0.08],
        "ground_horizon_color":[0.35, 0.25, 0.18],
        "sun_angle_max":       10.0,
        "sun_curve":           0.15
      },
      "sun": {
        "color":            [1.0, 0.65, 0.35],
        "energy":           0.9,
        "rotation_deg":     [-12.0, 60.0, 0.0],
        "shadow_enabled":   true,
        "shadow_blur":      2.5
      },
      "ambient": {
        "color":              [0.25, 0.18, 0.20],
        "energy":             0.4,
        "sky_contribution":   0.4
      },
      "fog": {
        "enabled":          true,
        "color":            [0.45, 0.30, 0.25],
        "density":          0.003,
        "sky_affect":       0.6,
        "height_enabled":   true,
        "height_max":       6.0,
        "height_density":   0.02
      },
      "ssao": {
        "enabled":       true,
        "radius":        1.2,
        "intensity":     2.0,
        "power":         1.5,
        "detail":        0.5,
        "light_affect":  0.3
      },
      "tonemap": {
        "mode":      "filmic",
        "exposure":  0.85,
        "white":     0.9
      },
      "character_light": {
        "enabled":        true,
        "color":          [1.0, 0.95, 0.90],
        "energy":         0.7,
        "rotation_deg":   [-40.0, 30.0, 0.0],
        "shadow_enabled": false
      }
    },

    "night": {
      "sky": {
        "sky_top_color":       [0.02, 0.02, 0.06],
        "sky_horizon_color":   [0.08, 0.08, 0.15],
        "ground_bottom_color": [0.02, 0.02, 0.03],
        "ground_horizon_color":[0.06, 0.06, 0.10],
        "sun_angle_max":       5.0,
        "sun_curve":           0.05
      },
      "sun": {
        "color":            [0.40, 0.45, 0.65],
        "energy":           0.35,
        "rotation_deg":     [-30.0, 45.0, 0.0],
        "shadow_enabled":   true,
        "shadow_blur":      3.0
      },
      "ambient": {
        "color":              [0.08, 0.08, 0.15],
        "energy":             0.3,
        "sky_contribution":   0.2
      },
      "fog": {
        "enabled":          true,
        "color":            [0.06, 0.06, 0.12],
        "density":          0.005,
        "sky_affect":       0.3,
        "height_enabled":   true,
        "height_max":       5.0,
        "height_density":   0.03
      },
      "ssao": {
        "enabled":       true,
        "radius":        1.5,
        "intensity":     2.5,
        "power":         2.0,
        "detail":        0.5,
        "light_affect":  0.4
      },
      "tonemap": {
        "mode":      "filmic",
        "exposure":  0.6,
        "white":     0.8
      },
      "character_light": {
        "enabled":        true,
        "color":          [1.0, 0.95, 0.90],
        "energy":         1.0,
        "rotation_deg":   [-40.0, 30.0, 0.0],
        "shadow_enabled": false
      }
    }
  },

  // 맵 → 프리셋 매핑 (명시적 지정)
  "map_presets": {
    "encounter_small_01":  "overcast_day",
    "encounter_medium_01": "overcast_day",
    "encounter_medium_02": "dusk",
    "encounter_large_01":  "night"
  },

  // map_presets에 없고 랜덤 미적용 시 폴백
  "default_preset": "overcast_day"
}
```

#### 프리셋별 분위기 의도

| 프리셋 | 의도 | 핵심 차이 |
|--------|------|----------|
| overcast_day | Wartales 기본 톤 — 흐린 하늘, 탈채도, 무겁고 칙칙한 전장 | 하늘/지면 색차 작음, fog 연하게 |
| dusk | 석양 전장 — 따뜻한 수평선, 긴 그림자, 불안한 분위기 | sun 저각(−12°), fog 따뜻한 색 |
| night | 야간 교전 — 달빛, 짙은 안개, 시야 제한 | energy 0.15, fog density 높음 |

---

## 3. 스크립트 구현

### 3.1 EnvironmentConfig.gd — `scripts/singletons/EnvironmentConfig.gd`

기존 AnimationConfig / CameraConfig 동일 패턴. static class, 오토로드 불필요.

```gdscript
class_name EnvironmentConfig

const CONFIG_PATH := "res://data/environment/environment_config.json"

static var _data: Dictionary = {}
static var _loaded: bool = false


static func _ensure_loaded() -> void:
    if _loaded:
        return
    var file := FileAccess.open(CONFIG_PATH, FileAccess.READ)
    if file:
        var json := JSON.new()
        json.parse(file.get_as_text())
        _data = json.data
        _loaded = true
    else:
        push_error("EnvironmentConfig: failed to load %s" % CONFIG_PATH)


static func get_preset(preset_name: String) -> Dictionary:
    _ensure_loaded()
    var preset: Dictionary = _data.get("presets", {}).get(preset_name, {})
    if preset.is_empty():
        push_warning("EnvironmentConfig: preset '%s' not found" % preset_name)
    return preset


static func get_preset_for_map(map_name: String) -> String:
    _ensure_loaded()
    var map_presets: Dictionary = _data.get("map_presets", {})

    # 1) 명시적 매핑 우선
    if map_presets.has(map_name):
        return map_presets[map_name]

    # 2) 매핑 없으면 전체 프리셋 중 랜덤 선택
    var all_names: Array = get_all_preset_names()
    if all_names.size() > 0:
        return all_names[randi() % all_names.size()]

    # 3) 프리셋 자체가 없으면 default_preset 폴백
    return _data.get("default_preset", "overcast_day")


static func get_all_preset_names() -> Array:
    _ensure_loaded()
    return _data.get("presets", {}).keys()
```

### 3.2 EnvironmentBuilder.gd — `scripts/rendering/EnvironmentBuilder.gd`

WorldEnvironment + DirectionalLight3D를 프리셋 데이터로 생성하는 static 빌더.

```gdscript
class_name EnvironmentBuilder


## 프리셋 이름으로 WorldEnvironment + DirectionalLight3D 생성.
## 반환: {"world_env": WorldEnvironment, "sun": DirectionalLight3D, "character_light": DirectionalLight3D|null}
## character_light는 preset에 enabled: true일 때만 생성. 비활성화/실패 시 null.
## 실패 시 빈 Dictionary 반환.
static func build(parent: Node3D, preset_name: String) -> Dictionary:
    var preset := EnvironmentConfig.get_preset(preset_name)
    if preset.is_empty():
        push_warning("EnvironmentBuilder: preset '%s' not found, skipping" % preset_name)
        return {}

    var world_env := WorldEnvironment.new()
    var env := Environment.new()
    var sun := DirectionalLight3D.new()

    _apply_sky(env, preset.get("sky", {}))
    _apply_ambient(env, preset.get("ambient", {}))
    _apply_fog(env, preset.get("fog", {}))
    _apply_ssao(env, preset.get("ssao", {}))
    _apply_tonemap(env, preset.get("tonemap", {}))

    world_env.environment = env
    parent.add_child(world_env)

    _apply_sun(sun, preset.get("sun", {}))
    parent.add_child(sun)

    var char_light: DirectionalLight3D = null
    var char_data: Dictionary = preset.get("character_light", {})
    if char_data.get("enabled", false):
        char_light = DirectionalLight3D.new()
        char_light.light_color    = _to_color(char_data.get("color", [1.0, 0.95, 0.90]))
        char_light.light_energy   = char_data.get("energy", 0.6)
        var char_rot: Array       = char_data.get("rotation_deg", [-40.0, 30.0, 0.0])
        char_light.rotation_degrees = Vector3(char_rot[0], char_rot[1], char_rot[2])
        char_light.shadow_enabled = char_data.get("shadow_enabled", false)
        char_light.light_cull_mask = 2  # layer 2 only — characters
        parent.add_child(char_light)

    return {"world_env": world_env, "sun": sun, "character_light": char_light}


# --- Sky ---

static func _apply_sky(env: Environment, data: Dictionary) -> void:
    var proc_sky := ProceduralSkyMaterial.new()
    proc_sky.sky_top_color = _to_color(data.get("sky_top_color", [0.4, 0.45, 0.55]))
    proc_sky.sky_horizon_color = _to_color(data.get("sky_horizon_color", [0.6, 0.6, 0.6]))
    proc_sky.ground_bottom_color = _to_color(data.get("ground_bottom_color", [0.2, 0.17, 0.13]))
    proc_sky.ground_horizon_color = _to_color(data.get("ground_horizon_color", [0.5, 0.5, 0.45]))
    proc_sky.sun_angle_max = data.get("sun_angle_max", 30.0)
    proc_sky.sun_curve = data.get("sun_curve", 0.1)

    var sky := Sky.new()
    sky.sky_material = proc_sky
    env.sky = sky
    env.background_mode = Environment.BG_SKY


# --- Ambient ---

static func _apply_ambient(env: Environment, data: Dictionary) -> void:
    env.ambient_light_source = Environment.AMBIENT_SOURCE_COLOR
    env.ambient_light_color = _to_color(data.get("color", [0.3, 0.3, 0.35]))
    env.ambient_light_energy = data.get("energy", 0.5)
    env.ambient_light_sky_contribution = data.get("sky_contribution", 0.3)


# --- Fog ---

static func _apply_fog(env: Environment, data: Dictionary) -> void:
    env.fog_enabled = data.get("enabled", false)
    if not env.fog_enabled:
        return
    env.fog_light_color = _to_color(data.get("color", [0.5, 0.5, 0.5]))
    env.fog_density = data.get("density", 0.01)
    env.fog_sky_affect = data.get("sky_affect", 0.5)
    if data.get("height_enabled", false):
        env.fog_height = data.get("height_max", 8.0)
        env.fog_height_density = data.get("height_density", 0.05)


# --- SSAO ---

static func _apply_ssao(env: Environment, data: Dictionary) -> void:
    env.ssao_enabled = data.get("enabled", false)
    if not env.ssao_enabled:
        return
    env.ssao_radius = data.get("radius", 1.0)
    env.ssao_intensity = data.get("intensity", 1.5)
    env.ssao_power = data.get("power", 1.5)
    env.ssao_detail = data.get("detail", 0.5)
    env.ssao_light_affect = data.get("light_affect", 0.2)


# --- Tonemap ---

static func _apply_tonemap(env: Environment, data: Dictionary) -> void:
    var mode_str: String = data.get("mode", "filmic")
    match mode_str:
        "linear":    env.tonemap_mode = Environment.TONE_MAPPER_LINEAR
        "reinhardt": env.tonemap_mode = Environment.TONE_MAPPER_REINHARDT
        "filmic":    env.tonemap_mode = Environment.TONE_MAPPER_FILMIC
        "aces":      env.tonemap_mode = Environment.TONE_MAPPER_ACES
    env.tonemap_exposure = data.get("exposure", 1.0)
    env.tonemap_white = data.get("white", 1.0)


# --- Sun ---

static func _apply_sun(sun: DirectionalLight3D, data: Dictionary) -> void:
    sun.light_color = _to_color(data.get("color", [0.85, 0.82, 0.78]))
    sun.light_energy = data.get("energy", 0.7)
    var rot: Array = data.get("rotation_deg", [-45.0, 45.0, 0.0])
    sun.rotation_degrees = Vector3(rot[0], rot[1], rot[2])
    sun.shadow_enabled = data.get("shadow_enabled", true)
    sun.shadow_blur = data.get("shadow_blur", 1.5)


# --- Utility ---

static func _to_color(arr: Array) -> Color:
    if arr.size() >= 4:
        return Color(arr[0], arr[1], arr[2], arr[3])
    elif arr.size() == 3:
        return Color(arr[0], arr[1], arr[2])
    return Color.WHITE
```

### 3.2.1 라이트 레이어 설계

Godot 4의 `light_cull_mask` (라이트) / `layers` (VisualInstance3D) 비트마스크를 활용해 캐릭터 전용 보조광을 구현한다.

| 레이어 | 비트 | 값 | 대상 | 설명 |
|--------|------|-----|------|------|
| Layer 1 | bit 0 | 1 | 지형, 모든 오브젝트 | 기본 레이어. sun이 이 레이어를 비춤 |
| Layer 2 | bit 1 | 2 | 캐릭터 모델 전용 | character_light가 이 레이어만 비춤 |

#### 적용 규칙

- `sun.light_cull_mask` = 기본값(0xFFFFFFFF, 전체) — 모든 오브젝트에 영향
- `character_light.light_cull_mask = 2` — layer 2 오브젝트만 비춤
- 캐릭터 VisualInstance3D 노드: `layers = 1 | 2 = 3` — 두 레이어 모두 소속

#### UnitRenderer3D 적용 방식

`_load_glb()` 및 `_create_placeholder()` 완료 후 `_set_character_layers(model)` 호출.
GLB 내부의 MeshInstance3D 등 모든 VisualInstance3D 자손에 재귀적으로 적용.

```gdscript
func _set_character_layers(node: Node) -> void:
    if node is VisualInstance3D:
        (node as VisualInstance3D).layers = 1 | 2
    for child in node.get_children():
        _set_character_layers(child)
```

#### 결과

캐릭터는 sun + character_light 두 광원을 받음 → 어두운 night 프리셋에서도 캐릭터 가시성 유지.
지형/배경은 sun만 받음 → 환경은 어둡고, 캐릭터는 판독 가능.

---

### 3.3 CombatScene.gd 변경

#### 제거

`_setup_lighting()` 함수 전체 삭제 (DirectionalLight3D + Environment 하드코딩 코드).

#### 추가

```gdscript
var _env_nodes: Dictionary = {}


func _setup_environment() -> void:
    var preset_name := EnvironmentConfig.get_preset_for_map(_current_map_name)
    _env_nodes = EnvironmentBuilder.build(self, preset_name)
    if _env_nodes.is_empty():
        push_warning("CombatScene: environment build failed for map '%s'" % _current_map_name)
```

#### 호출 위치

기존 `_setup_lighting()` 호출 위치를 `_setup_environment()`로 교체.

---

## 4. 프리셋 선택 로직 상세

```
CombatScene._setup_environment()
  └─ EnvironmentConfig.get_preset_for_map(map_name)
       ├─ map_presets에 map_name 있음 → 해당 프리셋 이름 반환
       ├─ map_presets에 없음 → 전체 프리셋 중 randi() 랜덤 선택
       └─ 프리셋 0개 (비정상) → default_preset 폴백
```

이 구조로 나중에 필요하면:
- 특정 맵에 고정 분위기 지정 가능 (map_presets에 추가)
- 매핑 안 된 새 맵은 자동으로 분위기 랜덤 적용
- 스토리 연동이 필요하면 CombatScene에서 preset_name을 직접 오버라이드 가능

---

## 5. 구현 순서 (Claude Code 태스크)

### Task 1 — JSON 생성
- `data/environment/environment_config.json` 생성
- 위 2.3의 데이터 기입

### Task 2 — EnvironmentConfig.gd
- `scripts/singletons/EnvironmentConfig.gd` 생성
- 위 3.1 코드 구현

### Task 3 — EnvironmentBuilder.gd
- `scripts/rendering/EnvironmentBuilder.gd` 생성
- 위 3.2 코드 구현

### Task 4 — CombatScene 연동
- `_setup_lighting()` 함수 삭제
- `_setup_environment()` 함수 추가
- `_setup_lighting()` 호출부 → `_setup_environment()` 교체
- `_env_nodes` 변수 추가

### Task 5 — 검증
- Godot 에디터에서 전투 시작 → 하늘/안개/조명 렌더링 확인
- 각 프리셋별 시각적 차이 확인 (map_presets 매핑 임시 변경으로 테스트)
- map_presets에 없는 맵 이름으로 랜덤 선택 동작 확인

---

## 6. 튜닝 가이드

JSON 수치는 초안이다. 에디터에서 튜닝 후 JSON만 업데이트하면 된다.

### 우선 튜닝 대상

| 파라미터 | 이유 |
|---------|------|
| fog.density | 맵 크기(15×15~23×23)에 따라 안개 농도 체감이 크게 다름 |
| fog.height_density | 높이 안개가 타일 위에 어떻게 깔리는지 실제로 봐야 판단 가능 |
| ssao.intensity | 로우폴리 모델에서 SSAO가 과하면 노이즈처럼 보일 수 있음 |
| sun.rotation_deg | dusk의 −12° 태양 각도가 맵 크기 대비 그림자 길이에 적절한지 |
| tonemap.exposure | night 0.6이 너무 어두우면 0.7~0.8로 올릴 것 |

### 새 프리셋 추가 시

1. `presets`에 새 키 추가 (6개 섹션 모두 포함)
2. 필요하면 `map_presets`에 매핑 추가
3. 코드 변경 불필요 — JSON만으로 확장

---

## 7. 미래 확장 고려

현재 스코프에 포함하지 않지만 이 구조로 지원 가능한 확장:

- **프리셋 전환 애니메이션**: EnvironmentBuilder에 `transition(from, to, duration)` 추가 → Tween으로 각 파라미터 보간
- **맵별 오버라이드**: 특정 맵이 프리셋의 일부 값만 덮어쓰는 `map_overrides` 섹션
- **날씨 시스템**: rain/snow 파티클 + fog density 변경을 프리셋 위에 레이어링
- **시간 경과**: 턴 진행에 따라 overcast_day → dusk → night 점진적 전환

---

## 8. 설계 원칙 준수 체크

| 원칙 | 준수 |
|------|------|
| 모든 수치 JSON 외부화 | ✅ environment_config.json |
| 코드에 매직 넘버 금지 | ✅ 모든 기본값은 .get() 폴백용, JSON이 권위 소스 |
| 단일 진입점 | ✅ EnvironmentBuilder.build(parent, preset_name) |
| 기존 패턴 따름 | ✅ static class 싱글톤 (AnimationConfig/CameraConfig 동일) |
| project.godot 미수정 | ✅ 오토로드 불필요 (static class) |
