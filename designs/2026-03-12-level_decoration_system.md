# Doc 8 — 레벨 데코레이션 시스템 (배경 에셋 배치)
> 작성일: 2026-03-12 | 작성자: @Planner
> 대상 경로: `handoff/plans/design/2026-03-12-level_decoration_system.md`

---

## 개요

전투 맵에 3D 배경 에셋(건물, 소품)을 배치하는 데이터 기반 시스템.
두 영역으로 구분한다:

- **외곽 배경 (Exterior)** — 그리드 밖 1~2타일 간격에 건물/대형 소품 배치. JSON 고정 레이아웃.
- **내부 장식 (Interior)** — 그리드 내 타일 위 소형 소품 배치. 규칙 기반 랜덤 생성.

### 핵심 결정사항
- 외곽: 맵별 JSON 수동 배치 (맵 정체성 결정)
- 내부: 테마별 소품 풀 + density 기반 랜덤 (반복 플레이 변화)
- 4종 맵 전부 동시 구현
- 건물 배치 거리: 그리드 외곽 1~2타일 (TILE_SIZE=1.0 기준 1~2m)
- 에셋: Quaternius Medieval Village Pack (CC0)

---

## 1. 에셋 승격 계획

### 1.1 승격 대상

`assets/_library/environment/quaternius/medieval_village/` → `assets/environment/` 로 복사.

#### 건물 (Buildings) — 8종 승격

| 에셋 ID | 원본 파일 | 승격 경로 | 사용 맵 |
|---------|----------|----------|--------|
| bell_tower | Bell_Tower.fbx | assets/environment/buildings/ | large_01 |
| blacksmith | Blacksmith.fbx | assets/environment/buildings/ | medium_01 |
| house_1 | House_1.fbx | assets/environment/buildings/ | small_01, large_01 |
| house_2 | House_2.fbx | assets/environment/buildings/ | small_01, large_01 |
| house_3 | House_3.fbx | assets/environment/buildings/ | large_01 |
| inn | Inn.fbx | assets/environment/buildings/ | medium_02 |
| mill | Mill.fbx | assets/environment/buildings/ | large_01 |
| stable | Stable.fbx | assets/environment/buildings/ | medium_01 |

#### 소품 (Props) — 16종 승격

| 에셋 ID | 원본 파일 | 승격 경로 | 용도 |
|---------|----------|----------|------|
| barrel | Barrel.fbx | assets/environment/props/ | 내부 장식 |
| bench_1 | Bench_1.fbx | assets/environment/props/ | 내부/외곽 |
| bonfire | Bonfire.fbx | assets/environment/props/ | 내부 장식 |
| bonfire_lit | Bonfire_Lit.fbx | assets/environment/props/ | 내부 장식 |
| cart | Cart.fbx | assets/environment/props/ | 외곽 |
| cauldron | Cauldron.fbx | assets/environment/props/ | 내부 장식 |
| crate | Crate.fbx | assets/environment/props/ | 내부 장식 |
| fence | Fence.fbx | assets/environment/props/ | 외곽 |
| gazebo | Gazebo.fbx | assets/environment/props/ | 외곽 |
| hay | Hay.fbx | assets/environment/props/ | 내부 장식 |
| market_stand_1 | MarketStand_1.fbx | assets/environment/props/ | 외곽 |
| market_stand_2 | MarketStand_2.fbx | assets/environment/props/ | 외곽 |
| rock_1 | Rock_1.fbx | assets/environment/props/ | 내부 장식 |
| rock_2 | Rock_2.fbx | assets/environment/props/ | 내부 장식 |
| rock_3 | Rock_3.fbx | assets/environment/props/ | 내부 장식 |
| well | Well.fbx | assets/environment/props/ | 외곽 |

#### 승격 절차
1. `_library/` 원본 → `assets/environment/{category}/` 로 **복사** (원본 유지)
2. `ASSET_LOG.md`에 승격 이력 기록
3. Godot에서 Import 탭 확인 (FBX → 씬 자동 임포트)

### 1.2 폴더 구조

```
assets/environment/
├── buildings/
│   ├── Bell_Tower.fbx
│   ├── Blacksmith.fbx
│   ├── House_1.fbx
│   ├── House_2.fbx
│   ├── House_3.fbx
│   ├── Inn.fbx
│   ├── Mill.fbx
│   └── Stable.fbx
└── props/
    ├── Barrel.fbx
    ├── Bench_1.fbx
    ├── Bonfire.fbx
    ├── Bonfire_Lit.fbx
    ├── Cart.fbx
    ├── Cauldron.fbx
    ├── Crate.fbx
    ├── Fence.fbx
    ├── Gazebo.fbx
    ├── Hay.fbx
    ├── MarketStand_1.fbx
    ├── MarketStand_2.fbx
    ├── Rock_1.fbx
    ├── Rock_2.fbx
    ├── Rock_3.fbx
    └── Well.fbx
```

---

## 2. JSON 구조 — `data/levels/level_decoration.json`

### 2.1 전체 스키마

```jsonc
{
  // 에셋 레지스트리 — ID → 리소스 경로 + 기본 속성
  "assets": {
    "<asset_id>": {
      "path": "res://assets/environment/{category}/{File}.fbx",
      "category": "building" | "prop",
      "base_scale": 1.0
    }
  },

  // 맵별 데코레이션 정의
  "maps": {
    "<map_name>": {
      "theme": "<theme_name>",

      // 외곽 배경 — 고정 배치 (수동)
      "exterior": [
        {
          "asset": "<asset_id>",
          "position": [x, y, z],
          "rotation_y": 0.0,
          "scale": 1.0
        }
      ],

      // 내부 장식 — 랜덤 배치 (규칙 기반)
      "interior": {
        "density": 0.08,
        "eligible_terrain": ["grass", "mud"],
        "spawn_exclusion_radius": 3,
        "pool": [
          {"asset": "<asset_id>", "weight": 3},
          {"asset": "<asset_id>", "weight": 1}
        ]
      }
    }
  }
}
```

### 2.2 필드 상세

#### assets (에셋 레지스트리)

| 키 | 타입 | 설명 |
|----|------|------|
| path | string | `res://` 리소스 경로 |
| category | string | "building" 또는 "prop" |
| base_scale | float | 기본 스케일 (로드 시 적용) |

#### exterior (외곽 배치)

| 키 | 타입 | 설명 |
|----|------|------|
| asset | string | assets 레지스트리의 ID |
| position | [x, y, z] | 월드 좌표 (그리드 외부) |
| rotation_y | float | Y축 회전 (도) — 건물 정면 방향 제어 |
| scale | float | 개별 스케일 오버라이드 (생략 시 base_scale) |

#### interior (내부 랜덤 배치)

| 키 | 타입 | 설명 |
|----|------|------|
| density | float | 적격 타일 중 소품 배치 비율 (0.0~1.0) |
| eligible_terrain | [string] | 소품 배치 가능 지형 타입 |
| spawn_exclusion_radius | int | 유닛 스폰 위치 주변 배치 금지 반경 (타일 수) |
| pool[].asset | string | 소품 에셋 ID |
| pool[].weight | int | 가중치 (높을수록 자주 등장) |

### 2.3 맵별 데코레이션 데이터

#### 좌표 체계 참고

```
그리드 원점: (0, 0, 0)
X축: 열 방향 (0 ~ grid_width-1)
Z축: 행 방향 (0 ~ grid_height-1), 홀수 열 +0.5 오프셋
Y축: 높이 — 건물 바닥 = 0.0 (지면)
```

그리드 경계 (TILE_SIZE=1.0 기준):
- encounter_small_01: X=0~14, Z=0~14
- encounter_medium_01: X=0~17, Z=0~17
- encounter_medium_02: X=0~17, Z=0~17
- encounter_large_01: X=0~22, Z=0~22

외곽 1~2타일 = position.x 또는 position.z 가 -1~-2 또는 grid_max+1~+2.

---

#### encounter_small_01 (15×15) — "마을 외곽 조우"

컨셉: 마을 변두리에서 벌어진 기습 교전. 작은 가옥과 우물, 울타리가 주변에 보인다.

```jsonc
"encounter_small_01": {
  "theme": "village_outskirts",
  "exterior": [
    // === 북쪽 (Z = -2 ~ -1) — 카메라 뒤쪽, 가까이 배치 ===
    {"asset": "house_1",  "position": [3.0, 0.0, -2.0],  "rotation_y": 180.0, "scale": 1.0},
    {"asset": "house_2",  "position": [10.0, 0.0, -1.5], "rotation_y": 160.0, "scale": 1.0},
    {"asset": "well",     "position": [7.0, 0.0, -1.0],  "rotation_y": 0.0,   "scale": 1.0},

    // === 서쪽 (X = -2 ~ -1) ===
    {"asset": "fence",    "position": [-1.5, 0.0, 4.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "fence",    "position": [-1.5, 0.0, 7.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "fence",    "position": [-1.5, 0.0, 10.0], "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "bench_1",  "position": [-1.0, 0.0, 12.0], "rotation_y": 90.0,  "scale": 1.0},

    // === 동쪽 (X = 15 ~ 16) ===
    {"asset": "fence",    "position": [15.5, 0.0, 3.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "fence",    "position": [15.5, 0.0, 6.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "cart",     "position": [16.0, 0.0, 10.0], "rotation_y": -30.0, "scale": 1.0},

    // === 남쪽 (Z = 15 ~ 16) — 카메라 앞쪽 ===
    {"asset": "barrel",   "position": [5.0, 0.0, 15.5],  "rotation_y": 0.0,   "scale": 1.0},
    {"asset": "crate",    "position": [5.5, 0.0, 16.0],  "rotation_y": 15.0,  "scale": 1.0}
  ],
  "interior": {
    "density": 0.06,
    "eligible_terrain": ["grass"],
    "spawn_exclusion_radius": 3,
    "pool": [
      {"asset": "rock_1",  "weight": 3},
      {"asset": "rock_2",  "weight": 3},
      {"asset": "rock_3",  "weight": 2},
      {"asset": "hay",     "weight": 1}
    ]
  }
}
```

#### encounter_medium_01 (18×18) — "대장간 앞 교전"

컨셉: 대장간과 마구간 사이 공터에서 벌어진 전투. 수레와 나무통이 흩어져 있다.

```jsonc
"encounter_medium_01": {
  "theme": "blacksmith_yard",
  "exterior": [
    // === 북쪽 ===
    {"asset": "blacksmith","position": [5.0, 0.0, -2.0],  "rotation_y": 180.0, "scale": 1.0},
    {"asset": "barrel",    "position": [8.0, 0.0, -1.0],  "rotation_y": 0.0,   "scale": 1.0},
    {"asset": "barrel",    "position": [8.5, 0.0, -1.5],  "rotation_y": 25.0,  "scale": 1.0},

    // === 서쪽 ===
    {"asset": "stable",    "position": [-2.0, 0.0, 5.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "hay",       "position": [-1.0, 0.0, 9.0],  "rotation_y": 0.0,   "scale": 1.0},
    {"asset": "fence",     "position": [-1.5, 0.0, 12.0], "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "fence",     "position": [-1.5, 0.0, 15.0], "rotation_y": 90.0,  "scale": 1.0},

    // === 동쪽 ===
    {"asset": "cart",      "position": [19.0, 0.0, 6.0],  "rotation_y": -45.0, "scale": 1.0},
    {"asset": "crate",     "position": [18.5, 0.0, 8.0],  "rotation_y": 10.0,  "scale": 1.0},
    {"asset": "bench_1",   "position": [18.5, 0.0, 13.0], "rotation_y": -90.0, "scale": 1.0},

    // === 남쪽 ===
    {"asset": "cauldron",  "position": [4.0, 0.0, 19.0],  "rotation_y": 0.0,   "scale": 1.0},
    {"asset": "bonfire",   "position": [12.0, 0.0, 18.5], "rotation_y": 0.0,   "scale": 1.0}
  ],
  "interior": {
    "density": 0.07,
    "eligible_terrain": ["grass", "mud"],
    "spawn_exclusion_radius": 3,
    "pool": [
      {"asset": "barrel",  "weight": 3},
      {"asset": "crate",   "weight": 3},
      {"asset": "rock_1",  "weight": 2},
      {"asset": "rock_2",  "weight": 1}
    ]
  }
}
```

#### encounter_medium_02 (18×18) — "시장터 약탈"

컨셉: 여관과 시장 노점 주변. 전투로 물건이 흩어진 약탈 현장.

```jsonc
"encounter_medium_02": {
  "theme": "market_raid",
  "exterior": [
    // === 북쪽 ===
    {"asset": "inn",            "position": [9.0, 0.0, -2.0],  "rotation_y": 180.0, "scale": 1.0},
    {"asset": "bench_1",        "position": [4.0, 0.0, -1.0],  "rotation_y": 0.0,   "scale": 1.0},

    // === 서쪽 ===
    {"asset": "market_stand_1", "position": [-2.0, 0.0, 4.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "market_stand_2", "position": [-2.0, 0.0, 9.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "barrel",         "position": [-1.0, 0.0, 13.0], "rotation_y": 0.0,   "scale": 1.0},

    // === 동쪽 ===
    {"asset": "gazebo",         "position": [19.5, 0.0, 7.0],  "rotation_y": 0.0,   "scale": 1.0},
    {"asset": "fence",          "position": [18.5, 0.0, 13.0], "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "fence",          "position": [18.5, 0.0, 16.0], "rotation_y": 90.0,  "scale": 1.0},

    // === 남쪽 ===
    {"asset": "cart",           "position": [7.0, 0.0, 19.0],  "rotation_y": 20.0,  "scale": 1.0},
    {"asset": "crate",          "position": [13.0, 0.0, 18.5], "rotation_y": -10.0, "scale": 1.0},
    {"asset": "crate",          "position": [13.5, 0.0, 19.0], "rotation_y": 30.0,  "scale": 1.0}
  ],
  "interior": {
    "density": 0.08,
    "eligible_terrain": ["grass", "mud"],
    "spawn_exclusion_radius": 3,
    "pool": [
      {"asset": "barrel",      "weight": 2},
      {"asset": "crate",       "weight": 3},
      {"asset": "hay",         "weight": 2},
      {"asset": "bonfire_lit", "weight": 1},
      {"asset": "cauldron",    "weight": 1}
    ]
  }
}
```

#### encounter_large_01 (23×23) — "마을 방어전"

컨셉: 마을 중심부 방어전. 종탑이 뒤에 보이고, 방앗간과 가옥들이 전장을 둘러싼다. 규모가 커서 에셋도 많이 배치.

```jsonc
"encounter_large_01": {
  "theme": "village_defense",
  "exterior": [
    // === 북쪽 ===
    {"asset": "bell_tower","position": [11.0, 0.0, -2.0],  "rotation_y": 180.0, "scale": 1.0},
    {"asset": "house_1",   "position": [3.0, 0.0, -2.0],   "rotation_y": 180.0, "scale": 1.0},
    {"asset": "house_3",   "position": [19.0, 0.0, -1.5],  "rotation_y": 200.0, "scale": 1.0},

    // === 서쪽 ===
    {"asset": "mill",      "position": [-2.5, 0.0, 5.0],   "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "fence",     "position": [-1.5, 0.0, 10.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "fence",     "position": [-1.5, 0.0, 13.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "house_2",   "position": [-2.0, 0.0, 17.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "barrel",    "position": [-1.0, 0.0, 20.0],  "rotation_y": 0.0,   "scale": 1.0},

    // === 동쪽 ===
    {"asset": "stable",    "position": [24.0, 0.0, 4.0],   "rotation_y": -90.0, "scale": 1.0},
    {"asset": "cart",      "position": [24.0, 0.0, 9.0],   "rotation_y": 45.0,  "scale": 1.0},
    {"asset": "well",      "position": [24.0, 0.0, 14.0],  "rotation_y": 0.0,   "scale": 1.0},
    {"asset": "fence",     "position": [23.5, 0.0, 18.0],  "rotation_y": 90.0,  "scale": 1.0},
    {"asset": "fence",     "position": [23.5, 0.0, 21.0],  "rotation_y": 90.0,  "scale": 1.0},

    // === 남쪽 ===
    {"asset": "bonfire_lit","position": [6.0, 0.0, 24.0],  "rotation_y": 0.0,   "scale": 1.0},
    {"asset": "bench_1",    "position": [11.0, 0.0, 23.5], "rotation_y": 0.0,   "scale": 1.0},
    {"asset": "crate",      "position": [17.0, 0.0, 24.0], "rotation_y": -20.0, "scale": 1.0},
    {"asset": "crate",      "position": [17.5, 0.0, 24.5], "rotation_y": 15.0,  "scale": 1.0}
  ],
  "interior": {
    "density": 0.05,
    "eligible_terrain": ["grass"],
    "spawn_exclusion_radius": 4,
    "pool": [
      {"asset": "rock_1",      "weight": 3},
      {"asset": "rock_2",      "weight": 3},
      {"asset": "rock_3",      "weight": 2},
      {"asset": "barrel",      "weight": 1},
      {"asset": "hay",         "weight": 1}
    ]
  }
}
```

---

## 3. 스크립트 구현

### 3.1 파일 구조

```
data/levels/
  └── level_decoration.json              ← 에셋 레지스트리 + 맵별 데코레이션

scripts/singletons/
  └── LevelDecorationConfig.gd           ← JSON 파싱 + 쿼리 (static class)

scripts/rendering/
  └── LevelDecorationBuilder.gd          ← 에셋 로드 + 배치 (static)
```

### 3.2 LevelDecorationConfig.gd — `scripts/singletons/LevelDecorationConfig.gd`

기존 EnvironmentConfig / AnimationConfig 동일 패턴.

```gdscript
class_name LevelDecorationConfig

const CONFIG_PATH := "res://data/levels/level_decoration.json"

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
        push_error("LevelDecorationConfig: failed to load %s" % CONFIG_PATH)


## 에셋 레지스트리에서 에셋 정보 반환
static func get_asset(asset_id: String) -> Dictionary:
    _ensure_loaded()
    return _data.get("assets", {}).get(asset_id, {})


## 맵별 외곽 배치 데이터 반환
static func get_exterior(map_name: String) -> Array:
    _ensure_loaded()
    var map_data: Dictionary = _data.get("maps", {}).get(map_name, {})
    return map_data.get("exterior", [])


## 맵별 내부 장식 규칙 반환
static func get_interior(map_name: String) -> Dictionary:
    _ensure_loaded()
    var map_data: Dictionary = _data.get("maps", {}).get(map_name, {})
    return map_data.get("interior", {})


## 맵 테마 이름 반환
static func get_theme(map_name: String) -> String:
    _ensure_loaded()
    var map_data: Dictionary = _data.get("maps", {}).get(map_name, {})
    return map_data.get("theme", "")
```

### 3.3 LevelDecorationBuilder.gd — `scripts/rendering/LevelDecorationBuilder.gd`

에셋 로드 + 외곽/내부 배치를 수행하는 static 빌더.

```gdscript
class_name LevelDecorationBuilder


## 맵 이름으로 외곽 + 내부 데코레이션 전체 생성.
## parent: 데코레이션 노드의 부모 (CombatScene 등)
## map_name: 전투 맵 이름
## tile_data: 타일 그리드 정보 (지형 타입 + 유닛 스폰 위치 판별용)
## 반환: {"exterior_root": Node3D, "interior_root": Node3D}
static func build(parent: Node3D, map_name: String, tile_data: Dictionary) -> Dictionary:
    var exterior_root := Node3D.new()
    exterior_root.name = "ExteriorDecoration"
    parent.add_child(exterior_root)

    var interior_root := Node3D.new()
    interior_root.name = "InteriorDecoration"
    parent.add_child(interior_root)

    _build_exterior(exterior_root, map_name)
    _build_interior(interior_root, map_name, tile_data)

    return {"exterior_root": exterior_root, "interior_root": interior_root}


# --- 외곽 배치 (JSON 고정) ---

static func _build_exterior(root: Node3D, map_name: String) -> void:
    var placements: Array = LevelDecorationConfig.get_exterior(map_name)
    for placement in placements:
        var asset_id: String = placement.get("asset", "")
        var asset_info: Dictionary = LevelDecorationConfig.get_asset(asset_id)
        if asset_info.is_empty():
            push_warning("LevelDecorationBuilder: asset '%s' not found in registry" % asset_id)
            continue

        var scene: PackedScene = load(asset_info.get("path", ""))
        if scene == null:
            push_warning("LevelDecorationBuilder: failed to load '%s'" % asset_info.get("path", ""))
            continue

        var instance: Node3D = scene.instantiate()

        # 위치
        var pos: Array = placement.get("position", [0.0, 0.0, 0.0])
        instance.position = Vector3(pos[0], pos[1], pos[2])

        # Y축 회전
        instance.rotation_degrees.y = placement.get("rotation_y", 0.0)

        # 스케일 (개별 오버라이드 > base_scale)
        var s: float = placement.get("scale", asset_info.get("base_scale", 1.0))
        instance.scale = Vector3(s, s, s)

        root.add_child(instance)


# --- 내부 장식 (랜덤 배치) ---

static func _build_interior(root: Node3D, map_name: String, tile_data: Dictionary) -> void:
    var rules: Dictionary = LevelDecorationConfig.get_interior(map_name)
    if rules.is_empty():
        return

    var density: float = rules.get("density", 0.05)
    var eligible: Array = rules.get("eligible_terrain", ["grass"])
    var exclusion_r: int = rules.get("spawn_exclusion_radius", 3)
    var pool: Array = rules.get("pool", [])
    if pool.is_empty():
        return

    # 가중치 기반 소품 선택 준비
    var weighted_assets: Array = []  # [asset_id, ...]  가중치만큼 반복
    for entry in pool:
        var weight: int = entry.get("weight", 1)
        for i in range(weight):
            weighted_assets.append(entry.get("asset", ""))

    # 적격 타일 수집
    # tile_data 구조 기대:
    #   tile_data.tiles = {Vector2i: {"terrain": "grass", ...}, ...}
    #   tile_data.spawn_positions = [Vector2i, ...]  (유닛 스폰 좌표)
    var tiles: Dictionary = tile_data.get("tiles", {})
    var spawn_positions: Array = tile_data.get("spawn_positions", [])

    var eligible_tiles: Array = []
    for coord in tiles.keys():
        var terrain: String = tiles[coord].get("terrain", "")
        if terrain not in eligible:
            continue
        # 스폰 배제 영역 체크
        var excluded := false
        for sp in spawn_positions:
            if _grid_distance(coord, sp) <= exclusion_r:
                excluded = true
                break
        if excluded:
            continue
        eligible_tiles.append(coord)

    # density 비율만큼 랜덤 선택
    eligible_tiles.shuffle()
    var count: int = int(eligible_tiles.size() * density)
    count = max(count, 0)

    for i in range(count):
        var coord: Vector2i = eligible_tiles[i]
        var asset_id: String = weighted_assets[randi() % weighted_assets.size()]
        var asset_info: Dictionary = LevelDecorationConfig.get_asset(asset_id)
        if asset_info.is_empty():
            continue

        var scene: PackedScene = load(asset_info.get("path", ""))
        if scene == null:
            continue

        var instance: Node3D = scene.instantiate()

        # 타일 중앙 + 약간의 랜덤 오프셋 (타일 내에서 자연스럽게)
        var world_pos := _grid_to_world(coord)
        world_pos.x += randf_range(-0.3, 0.3)
        world_pos.z += randf_range(-0.3, 0.3)
        instance.position = world_pos

        # 랜덤 Y 회전 (자연스러운 배치)
        instance.rotation_degrees.y = randf_range(0.0, 360.0)

        # 스케일
        var base_s: float = asset_info.get("base_scale", 1.0)
        instance.scale = Vector3(base_s, base_s, base_s)

        root.add_child(instance)


# --- 유틸리티 ---

## 스태거드 그리드 좌표 → 월드 좌표 변환
## 홀수 열은 Z축 +0.5 오프셋
static func _grid_to_world(coord: Vector2i) -> Vector3:
    var x: float = coord.x * 1.0  # TILE_SIZE = 1.0
    var z: float = coord.y * 1.0
    if coord.x % 2 == 1:
        z += 0.5
    return Vector3(x, 0.0, z)


## 그리드 상 두 좌표 간 거리 (체비셰프/맨해튼 대신 단순 유클리드)
static func _grid_distance(a: Vector2i, b: Vector2i) -> float:
    var wa := _grid_to_world(a)
    var wb := _grid_to_world(b)
    return wa.distance_to(wb)
```

### 3.4 CombatScene.gd 연동

**tile_data 실제 구조 (구현 확인 결과):**
- terrain_map 키: `"x,y"` 문자열 (encounter JSON과 동일)
- 지형 기본값: `"grass"` (terrain_map에 없는 타일)
- 지형 타입: `"grass"`, `"mud"`, `"elevated"`, `"water"`, `"wall"`
- spawn_positions: ally_spawn_positions + enemies[].grid_pos

```gdscript
var _decoration_nodes: Dictionary = {}


func _setup_decorations() -> void:
    var tile_data := _build_tile_data_for_decoration()
    _decoration_nodes = LevelDecorationBuilder.build(self, WarbandManager.current_encounter_id, tile_data)
    if _decoration_nodes.is_empty():
        push_warning("CombatScene: decoration build failed for map '%s'" % WarbandManager.current_encounter_id)


func _build_tile_data_for_decoration() -> Dictionary:
    # terrain_map의 키 형식: "x,y" 문자열 (encounter JSON과 동일)
    var terrain_map: Dictionary = _encounter.get("terrain_map", {})
    var grid_w: int = _encounter.get("grid_width", int(_config.get("grid_width", 10)))
    var grid_h: int = _encounter.get("grid_height", int(_config.get("grid_height", 10)))

    # 전체 타일 목록 구성 {Vector2i → {"terrain": String}}
    var tiles: Dictionary = {}
    for x in range(grid_w):
        for y in range(grid_h):
            var key: String = "%d,%d" % [x, y]
            var terrain: String = terrain_map.get(key, "grass")
            tiles[Vector2i(x, y)] = {"terrain": terrain}

    # 스폰 배제 영역: 아군 + 적군 스폰 위치
    var spawn_positions: Array = []
    for raw in _encounter.get("ally_spawn_positions", []):
        spawn_positions.append(Vector2i(int(raw[0]), int(raw[1])))
    for enemy in _encounter.get("enemies", []):
        var raw: Array = enemy.get("grid_pos", [0, 0])
        spawn_positions.append(Vector2i(int(raw[0]), int(raw[1])))

    return {"tiles": tiles, "spawn_positions": spawn_positions}
```

호출 위치: `_setup_environment()` 이후, `_spawn_units()` 이전 (`_ready()` 내).

---

## 4. 맵별 테마 컨셉 요약

| 맵 | 테마 | 분위기 | 핵심 외곽 에셋 | 내부 소품 |
|----|------|--------|-------------|----------|
| small_01 | 마을 외곽 조우 | 울타리와 작은 집 — 조용한 변두리 급습 | house_1/2, well, fence | rock, hay |
| medium_01 | 대장간 앞 교전 | 대장간+마구간 공터 — 물자가 흩어진 작업장 | blacksmith, stable, cart | barrel, crate, rock |
| medium_02 | 시장터 약탈 | 여관+노점 — 전투로 무너진 시장 | inn, market_stand, gazebo | barrel, crate, hay, bonfire |
| large_01 | 마을 방어전 | 종탑 중심 마을 — 대규모 방어전 | bell_tower, mill, house_1/2/3 | rock, barrel, hay |

---

## 5. 구현 순서 (Claude Code 태스크)

### Task 1 — 에셋 승격
- 섹션 1.1의 24종(건물 8 + 소품 16) FBX를 `_library/` → `assets/environment/` 복사
- `ASSET_LOG.md` 승격 이력 기록
- Godot Import 확인은 에디터에서 수동

### Task 2 — JSON 생성
- `data/levels/level_decoration.json` 생성
- 섹션 2.3의 전체 데이터 기입 (순수 JSON, 주석 제거)
- 에셋 레지스트리 + 4개 맵 데코레이션 데이터

### Task 3 — LevelDecorationConfig.gd
- `scripts/singletons/LevelDecorationConfig.gd` 생성
- 섹션 3.2 코드 구현

### Task 4 — LevelDecorationBuilder.gd
- `scripts/rendering/LevelDecorationBuilder.gd` 생성
- 섹션 3.3 코드 구현
- 주의: `_build_tile_data_for_decoration()`의 tile_data 구조는 실제 TileRenderer3D / CombatScene의 기존 타일 데이터 구조를 확인하고 맞출 것

### Task 5 — CombatScene 연동
- `_setup_decorations()` 함수 추가
- `_build_tile_data_for_decoration()` — 실제 타일 그리드 데이터에서 terrain + spawn_positions 추출
- `_setup_environment()` 이후, 유닛 스폰 이전 시점에 호출

### Task 6 — 설계 문서 + CURRENT.md 업데이트
- `handoff/plans/design/2026-03-12-level_decoration_system.md` 에 실제 구현 결과 반영
- `CURRENT.md` 백업(handoff/LOG/) 후 갱신

---

## 6. tile_data 인터페이스 주의사항

`LevelDecorationBuilder._build_interior()` 는 다음 tile_data 구조를 기대한다:

```gdscript
{
    "tiles": {
        Vector2i(0, 0): {"terrain": "grass"},
        Vector2i(1, 0): {"terrain": "mud"},
        Vector2i(2, 0): {"terrain": "elevated"},
        # ...
    },
    "spawn_positions": [
        Vector2i(3, 2),   # 아군 스폰
        Vector2i(10, 12), # 적군 스폰
        # ...
    ]
}
```

실제 CombatScene / TileRenderer3D의 기존 데이터 구조가 이와 다를 수 있으므로, Task 5에서 `_build_tile_data_for_decoration()` 구현 시 **기존 코드를 먼저 확인**하고 변환 로직을 작성할 것.

특히:
- 타일 좌표가 Vector2i인지 String 키인지
- terrain 타입 이름이 "grass"/"mud"/"elevated"/"wall"/"water" 그대로인지
- 스폰 위치를 어디서 가져올 수 있는지 (맵 JSON? CombatScene 변수?)

---

## 7. 카메라 시점 고려사항

카메라가 BG3 스타일 ~45° 고각이므로:
- **남쪽 (카메라 앞쪽)** 에는 높은 건물 배치를 피한다 → 전장 가림
- **북쪽 (카메라 뒤쪽)** 에 주요 건물 배치 → 배경으로 잘 보임
- 남쪽에는 낮은 소품(barrel, crate, bench) 위주로 배치
- 이 원칙이 위 데이터에 이미 반영되어 있음

---

## 8. 미래 확장 고려

현재 스코프에 포함하지 않지만 이 구조로 지원 가능:

- **엄폐/장애물 시스템**: interior 소품에 `gameplay_type: "half_cover"` 등 추가 → 전투 시스템 연동
- **파괴 가능 소품**: 에셋 레지스트리에 `destructible: true` + HP 추가
- **맵 에디터**: JSON 기반이므로 나중에 비주얼 맵 에디터에서 직접 배치 가능
- **계절/시간 변형**: 에셋 레지스트리에 variant 추가 (예: `bonfire` vs `bonfire_lit`)
- **LOD**: 대형 맵에서 먼 건물은 LOD 적용 (에셋 레지스트리에 lod_path 추가)

---

## 9. 설계 원칙 준수 체크

| 원칙 | 준수 |
|------|------|
| 모든 수치 JSON 외부화 | ✅ level_decoration.json |
| 코드에 매직 넘버 금지 | ✅ TILE_SIZE=1.0만 _grid_to_world에서 사용 (기존 타일 시스템과 동일) |
| 에셋 승격 절차 준수 | ✅ _library/ → assets/ 복사, ASSET_LOG 기록 |
| 기존 패턴 따름 | ✅ static class 싱글톤 + static 빌더 |
| project.godot 미수정 | ✅ 오토로드 불필요 |
| _library 원본 수정 금지 | ✅ 복사만 수행 |
