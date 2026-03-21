# Design: Data Migration — 하드코딩 → 데이터 기반 전환
> 🇰🇷 기존 시스템 전면 데이터 기반 전환 설계

## Document Info
- Created: 2026-03-08
- Agent: @Planner
- Status: Pending User Approval
- Scope: 기존 구현 시스템 10개 전환 (신규 기능 제외)

---

## 핵심 원칙

- 코드에 수치/위치/이름 하드코딩 금지
- JSON 파일 추가만으로 콘텐츠 확장 가능
- GDScript 인터페이스 (함수명/시그널) 변경 최소화
- `.tres` 파일: 단기 유지, JSON 참조 방식으로 래핑

---

## 신규 데이터 파일 전체 목록

```
data/
├── combat_config.json          ← max_turns, ally spawn positions
├── world_config.json           ← fatigue 수치, 이동속도
├── camp_config.json            ← 캠프 회복 수치
├── encounters/
│   └── encounter_01.json       ← 적군 구성 + 위치
├── mercenaries/
│   └── roster_config.json      ← 초기 roster .tres 경로 목록
├── world/
│   └── world_nodes.json        ← NPC/조우 지점 위치+타입 통합
├── npcs/
│   └── npc_01.json             ← NPC 대화/퀘스트
└── classes/
    ├── class_config.json        ← 직업별 기본 스탯
    └── levelup_config.json      ← 직업별 레벨업 증가량
```

---

## 1순위 — 전투/핵심 시스템

---

### [1] 적군 스폰 데이터
**현재:** `CombatScene.gd` 상수로 하드코딩
**목표:** `data/encounters/encounter_01.json`

#### JSON 구조
```json
{
  "id": "encounter_01",
  "ally_spawn_positions": [
    [3, 2], [4, 2], [5, 2], [6, 2], [7, 2], [8, 2]
  ],
  "enemy_units": [
    { "data_path": "res://data/mercenaries/fighter_01.tres", "grid_pos": [3, 7] },
    { "data_path": "res://data/mercenaries/fighter_01.tres", "grid_pos": [4, 7] },
    { "data_path": "res://data/mercenaries/fighter_01.tres", "grid_pos": [5, 7] }
  ]
}
```

> `ally_spawn_positions`: 출전 선택 시스템과 연동. 최대 6개.
> `data_path`: 기존 .tres 유지. JSON이 경로를 참조하는 래핑 방식.

#### WarbandManager 추가
```gdscript
static var current_encounter_id: String = "encounter_01"
```

#### CombatScene.gd 변경 범위
- `const ALLY_SPAWN_POSITIONS`, `const ENEMY_SPAWN_POSITIONS`, `const ENEMY_DATA` 제거
- `_ready()`에서 encounter JSON 로드
- `_spawn_units()` — JSON 데이터 기반으로 교체

```gdscript
var _encounter: Dictionary = {}

func _ready() -> void:
    _encounter = _load_json(
        "res://data/encounters/%s.json" % WarbandManager.current_encounter_id)
    # ... 기존 초기화 ...

func _spawn_units() -> void:
    # 아군: selected_units + encounter ally_spawn_positions
    var ally_positions := _encounter["ally_spawn_positions"]
    var indices := WarbandManager.selected_units
    if indices.is_empty():
        indices = range(WarbandManager.roster.size())
    for i in range(mini(indices.size(), ally_positions.size())):
        var pos := Vector2i(ally_positions[i][0], ally_positions[i][1])
        _spawn_unit(pos, WarbandManager.roster[indices[i]], true, ally_container)
    # 적군
    for enemy in _encounter["enemy_units"]:
        var pos := Vector2i(enemy["grid_pos"][0], enemy["grid_pos"][1])
        var d := load(enemy["data_path"]) as MercenaryData
        _spawn_unit(pos, d, false, enemy_container)
```

#### .tres 처리 방향
유지. encounter JSON이 경로를 참조하는 구조로 래핑.

---

### [2] 전투 규칙 수치
**현재:** `TurnManager.gd` — `const MAX_TURNS: int = 10`
**목표:** `data/combat_config.json`

#### JSON 구조
```json
{
  "max_turns": 10,
  "attack_hit_radius": 35.0
}
```

> `attack_hit_radius`: CombatScene.gd `const ATTACK_HIT_RADIUS: float = 35.0` 도 동일 파일에서 관리.

#### TurnManager.gd 변경 범위
```gdscript
# 변경 전
const MAX_TURNS: int = 10

# 변경 후
var max_turns: int = 10   # CombatScene._ready()에서 주입
```

#### CombatScene.gd 변경 범위
```gdscript
var _config: Dictionary = {}

func _ready() -> void:
    _config = _load_json("res://data/combat_config.json")
    turn_manager.max_turns = _config.get("max_turns", 10)
    # ATTACK_HIT_RADIUS도 _config에서 읽음
```

> `MAX_TURNS` 참조 위치: `TurnManager._end_enemy_turn()`, `CombatScene._on_phase_changed()` — `turn_manager.max_turns`으로 교체.

---

### [3] 용병 초기 데이터
**현재:** `WarbandManager._ready()` — .tres 경로 3개 하드코딩
**목표:** `data/mercenaries/roster_config.json`

#### JSON 구조
```json
{
  "initial_roster": [
    "res://data/mercenaries/fighter_01.tres",
    "res://data/mercenaries/archer_01.tres",
    "res://data/mercenaries/rogue_01.tres"
  ]
}
```

#### WarbandManager.gd 변경 범위
```gdscript
static var roster: Array[MercenaryData] = []

func _ready() -> void:
    if roster.is_empty():
        var config := _load_json("res://data/mercenaries/roster_config.json")
        for path in config["initial_roster"]:
            roster.append(load(path) as MercenaryData)

func _load_json(path: String) -> Dictionary:
    var file := FileAccess.open(path, FileAccess.READ)
    if file == null: return {}
    var result: Dictionary = JSON.parse_string(file.get_as_text())
    file.close()
    return result
```

> `roster`를 동시에 static으로 전환 (town_system.md 요구사항과 통합).
> .tres 파일 유지. roster_config.json이 경로를 목록으로 관리.

---

## 2순위 — 월드/콘텐츠

---

### [4] 월드맵 노드 위치
**현재:** `NPC`, `EncounterZone` 위치가 `world_map.tscn`에 하드코딩 + `.tscn` 직접 배치
**목표:** `data/world/world_nodes.json` — 스캔 기반 동적 스폰

#### JSON 구조
```json
{
  "npcs": [
    {
      "id": "npc_01",
      "data_path": "res://data/npcs/npc_01.json",
      "world_position": { "x": 300, "y": 200 }
    }
  ],
  "encounters": [
    {
      "id": "encounter_01",
      "world_position": { "x": 750, "y": 400 }
    }
  ]
}
```

> 마을은 `data/towns/*.json` 디렉토리 스캔 방식 유지 (town_system.md 설계대로).
> NPC/조우 지점은 단일 파일 방식이 관리 편의상 유리 (월드맵 전체 배치를 한눈에 파악 가능).

#### WorldMapScene.gd 변경 범위
```gdscript
func _ready() -> void:
    _load_world_nodes()
    # ... 기존 초기화 ...

func _load_world_nodes() -> void:
    var data := _load_json("res://data/world/world_nodes.json")
    for npc_def in data.get("npcs", []):
        var npc := NPC.new()
        npc.load_data(npc_def["data_path"])
        npc.position = Vector2(npc_def["world_position"]["x"],
                               npc_def["world_position"]["y"])
        _add_npc_marker(npc)
        npc_layer.add_child(npc)
    for enc_def in data.get("encounters", []):
        var enc := EncounterZone.new()
        enc.encounter_id = enc_def["id"]
        enc.position = Vector2(enc_def["world_position"]["x"],
                               enc_def["world_position"]["y"])
        _add_encounter_marker(enc)
        encounter_layer.add_child(enc)
```

#### world_map.tscn 변경
- `NPCLayer`, `EncounterLayer` 컨테이너만 유지
- 하위 NPC/EncounterZone 노드 직접 배치 제거 (코드에서 동적 생성)

#### EncounterZone.gd 변경
```gdscript
var encounter_id: String = "encounter_01"   # 추가
```

---

### [5] NPC 대화 데이터
**현재:** `NPC.gd` — `npc_name`, `quest_id`, `dialogue` 배열 하드코딩
**목표:** `data/npcs/npc_01.json`

#### JSON 구조
```json
{
  "id": "npc_01",
  "name": "마을 사람",
  "quest_id": "quest_01",
  "dialogue": [
    { "speaker": "마을 사람", "text": "안녕하시오, 용병이여." },
    { "speaker": "마을 사람", "text": "이 근방은 요즘 몬스터가 자주 출몰한다오." },
    { "speaker": "마을 사람", "text": "부디 조심하시게나." }
  ]
}
```

#### NPC.gd 변경 범위
```gdscript
# 변경 전
var npc_name: String = "마을 사람"
var quest_id: String = "quest_01"
var dialogue: Array[Dictionary] = [...]

# 변경 후
var npc_name: String = ""
var quest_id: String = ""
var dialogue: Array[Dictionary] = []

func load_data(path: String) -> void:
    var file := FileAccess.open(path, FileAccess.READ)
    if file == null: return
    var data: Dictionary = JSON.parse_string(file.get_as_text())
    file.close()
    npc_name  = data.get("name", "")
    quest_id  = data.get("quest_id", "")
    dialogue  = data.get("dialogue", [])
```

---

### [6] 퀘스트 보상 확장
**현재:** `data/quests/quest_01.json` — `reward_food`만 존재
**목표:** `gold_reward`, `item_rewards` 필드 추가

#### JSON 구조 (quest_01.json 수정)
```json
{
  "id": "quest_01",
  "title": "마을의 위협",
  "description": "몬스터를 처치하고 마을을 지켜라.",
  "reward_food": 3,
  "gold_reward": 15,
  "item_rewards": []
}
```

> `item_rewards`: 향후 아이템 시스템 연동용 예약 필드. 현재는 빈 배열.

#### WorldMapScene.gd 변경 범위
`_check_combat_rewards()`에서 `gold_reward` 지급 추가:
```gdscript
WarbandManager.gold += data.get("gold_reward", 0)
```
보상 팝업 텍스트에 골드 수치 포함.

---

### [7] 레벨업 수치
**현재:** `CombatScene.gd` — `+5 HP`, `+1 STR`, `+1 MV` 하드코딩
**목표:** `data/classes/levelup_config.json`

#### JSON 구조
```json
{
  "FIGHTER": { "hp_choice": 6, "str_choice": 2, "mv_choice": 1 },
  "ARCHER":  { "hp_choice": 4, "str_choice": 1, "mv_choice": 1 },
  "ROGUE":   { "hp_choice": 3, "str_choice": 1, "mv_choice": 2 },
  "MAGE":    { "hp_choice": 3, "str_choice": 2, "mv_choice": 0 }
}
```

> 현재 레벨업은 플레이어가 HP/STR/MV 중 선택하는 방식.
> 클래스별로 각 선택지의 증가량을 다르게 지정하여 직업 차별화.

#### CombatScene.gd 변경 범위
```gdscript
var _levelup_config: Dictionary = {}

func _ready() -> void:
    _levelup_config = _load_json("res://data/classes/levelup_config.json")

func _on_levelup_hp() -> void:
    var class_key := MercenaryData.UnitClass.keys()[_leveling_unit.data.unit_class]
    var gain: int = _levelup_config.get(class_key, {}).get("hp_choice", 5)
    _leveling_unit.data.max_hp  += gain
    _leveling_unit.max_hp       += gain
    _leveling_unit.current_hp   += gain
    _leveling_unit._update_hp_label()
    _apply_levelup_choice()

# _on_levelup_str, _on_levelup_mv도 동일 패턴
```

> 레벨업 UI 버튼 텍스트도 로드된 수치로 동적 갱신 필요:
> `hp_button.text = "+%d HP" % gain`

---

## 3순위 — 밸런스 수치

---

### [8] 피로도 수치
**현재:** `WorldMapScene.gd` — `const MAX_FATIGUE = 100`, `const FATIGUE_PER_100PX = 5`
**목표:** `data/world_config.json`

#### JSON 구조
```json
{
  "max_fatigue": 100,
  "fatigue_per_100px": 5,
  "player_move_speed": 200.0
}
```

> `player_move_speed`: `PlayerIcon.gd`의 `const MOVE_SPEED = 200.0`도 동일 파일에서 관리.

#### WorldMapScene.gd 변경 범위
```gdscript
var _max_fatigue: int = 100
var _fatigue_per_100px: int = 5

func _ready() -> void:
    var cfg := _load_json("res://data/world_config.json")
    _max_fatigue      = cfg.get("max_fatigue", 100)
    _fatigue_per_100px = cfg.get("fatigue_per_100px", 5)
```

#### PlayerIcon.gd 변경 범위
```gdscript
# const MOVE_SPEED 제거 → WorldMapScene에서 주입
var move_speed: float = 200.0

func move_to(target: Vector2) -> void:
    var duration := position.distance_to(target) / move_speed
    # ...
```

`WorldMapScene._ready()`에서:
```gdscript
player_icon.move_speed = cfg.get("player_move_speed", 200.0)
```

---

### [9] 캠프 회복 수치
**현재:** `CampScene.gd` — `food -= 1`, `fatigue -= 30`, 레이블에 `100` 하드코딩
**목표:** `data/camp_config.json`

#### JSON 구조
```json
{
  "food_cost_per_rest": 1,
  "fatigue_recovery_per_rest": 30
}
```

#### CampScene.gd 변경 범위
```gdscript
var _cfg: Dictionary = {}

func _ready() -> void:
    _cfg = _load_json("res://data/camp_config.json")
    rest_button.text = "휴식 (식량 -%d / 피로도 -%d)" % [
        _cfg.get("food_cost_per_rest", 1),
        _cfg.get("fatigue_recovery_per_rest", 30)
    ]
    _update_labels()

func _on_rest_button_pressed() -> void:
    WarbandManager.food    -= _cfg.get("food_cost_per_rest", 1)
    WarbandManager.fatigue  = maxi(
        WarbandManager.fatigue - _cfg.get("fatigue_recovery_per_rest", 30), 0)
    _update_labels()

func _update_labels() -> void:
    var max_fatigue: int = _load_json("res://data/world_config.json").get("max_fatigue", 100)
    fatigue_label.text = "피로도: %d / %d" % [WarbandManager.fatigue, max_fatigue]
    food_label.text    = "식량: %d" % WarbandManager.food
    rest_button.disabled = WarbandManager.food < _cfg.get("food_cost_per_rest", 1)
```

> world_config.json과 중복 로드 회피를 위해 max_fatigue를 WarbandManager static으로 캐싱하는 방안 검토 가능. 우선 단순 구현 후 최적화.

---

### [10] 직업별 기본 스탯
**현재:** `data/mercenaries/*.tres` — 스탯 개별 파일에 산재
**목표:** `data/classes/class_config.json` — 클래스 기본값 중앙 관리

#### JSON 구조
```json
{
  "FIGHTER": { "max_hp": 30, "str": 8, "move_range": 3 },
  "ARCHER":  { "max_hp": 22, "str": 6, "move_range": 4 },
  "ROGUE":   { "max_hp": 18, "str": 5, "move_range": 5 },
  "MAGE":    { "max_hp": 16, "str": 9, "move_range": 3 }
}
```

#### .tres 파일 처리 방향
**단기:** `.tres` 파일 유지. `class_config.json`은 신규 MercenaryData 생성 시(용병 모집 등)에만 사용.
**장기:** `.tres`를 JSON으로 완전 대체 가능하나 Godot 에디터 호환성 고려 필요 → 프로토타입 단계에서는 스킵.

#### 적용 범위
- 마을 용병 모집(`TownScene.gd`): `_hire_recruit()` 시 class_config에서 기본 스탯 로드
- 기존 .tres 파일의 스탯은 그대로 유지 (별도 마이그레이션 없음)

---

## 수정 파일 전체 목록
> 🇰🇷 신규/수정 파일 요약

### 신규 데이터 파일 (9개)
| 파일 | 항목 |
|------|------|
| `data/combat_config.json` | max_turns, attack_hit_radius |
| `data/world_config.json` | max_fatigue, fatigue_per_100px, move_speed |
| `data/camp_config.json` | food_cost, fatigue_recovery |
| `data/encounters/encounter_01.json` | 적군 구성/위치, 아군 spawn positions |
| `data/mercenaries/roster_config.json` | 초기 roster .tres 경로 목록 |
| `data/world/world_nodes.json` | NPC/조우 지점 위치+참조 |
| `data/npcs/npc_01.json` | NPC 대화/퀘스트 |
| `data/classes/class_config.json` | 직업별 기본 스탯 |
| `data/classes/levelup_config.json` | 직업별 레벨업 증가량 |

### 수정 GDScript (7개)
| 파일 | 변경 내용 |
|------|-----------|
| `scripts/combat/CombatScene.gd` | 상수 제거, JSON 로드, 스폰 로직 교체 |
| `scripts/combat/TurnManager.gd` | `const MAX_TURNS` → `var max_turns` |
| `scripts/data/WarbandManager.gd` | roster static화, roster_config 로드, gold/encounter_id 추가 |
| `scripts/world/WorldMapScene.gd` | world_nodes.json 기반 동적 스폰, world_config 로드 |
| `scripts/world/NPC.gd` | 하드코딩 제거, load_data() 추가 |
| `scripts/world/PlayerIcon.gd` | MOVE_SPEED → var, 외부 주입 |
| `scripts/camp/CampScene.gd` | camp_config 로드 |

### 수정 데이터 파일 (1개)
| 파일 | 변경 내용 |
|------|-----------|
| `data/quests/quest_01.json` | `gold_reward`, `item_rewards` 추가 |

### 수정 씬 파일 (1개)
| 파일 | 변경 내용 |
|------|-----------|
| `scenes/world/world_map.tscn` | NPC/EncounterZone 노드 직접 배치 제거 (코드 동적 생성으로 교체) |

---

## 구현 권장 순서

```
Step 1: 데이터 파일 전체 생성 (JSON 9개 + quest_01.json 수정)
Step 2: WarbandManager (roster static화 + 헬퍼 함수)
Step 3: CombatScene + TurnManager (1순위)
Step 4: WorldMapScene + NPC + PlayerIcon (2순위 동적 스폰)
Step 5: CampScene (3순위)
Step 6: levelup_config 연동 (CombatScene 레벨업 버튼)
Step 7: class_config 연동 (TownScene 모집 시 사용)
```

---

## Acceptance Criteria

- [ ] JSON 수치 변경만으로 max_turns, 피로도, 캠프 회복량 조정 가능
- [ ] encounter_01.json 수정으로 적군 구성/위치 변경 가능
- [ ] world_nodes.json 수정으로 NPC/조우 지점 위치 변경 가능
- [ ] npc_01.json 수정으로 대화 텍스트 변경 가능
- [ ] roster_config.json으로 초기 용병 구성 변경 가능
- [ ] 직업별 레벨업 증가량 차별화 동작
- [ ] 기존 전투/이동/캠프 동작 이상 없음 (회귀 없음)
- [ ] Godot 에러 없음
