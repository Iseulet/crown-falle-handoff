# Design: Town System + Party Selection (Data-Driven)
> 🇰🇷 설계: 마을 시스템 + 출전 선택 (데이터 기반)

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-08
- Revised: 2026-03-08 (전면 데이터 기반 재설계)
- Agent: @Planner
- Status: Pending User Approval
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-crownfalle-roadmap.md`

---

## 핵심 원칙
> 🇰🇷 Core Principles

**모든 콘텐츠 설정은 데이터 파일로 관리. 코드에 수치/위치/이름 하드코딩 금지.**
- 마을 위치, 이름 → `data/towns/town_01.json`
- 상점 아이템 → `data/shop/shop_items.json`
- 모집 용병 → `data/recruits/recruit_pool.json`
- 퀘스트 보상 → `data/quests/*.json` (gold_reward 필드 추가)
- 휴식 비용/회복량 → `data/town_config.json`

---

## 1. 데이터 파일 구조
> 🇰🇷 Data File Definitions

### 1-1. data/town_config.json
공통 설정값. 전체 마을에 적용되는 기본값.

```json
{
  "rest_cost_gold": 5,
  "rest_fatigue_recovery": 30,
  "max_roster_size": 6,
  "max_party_size": 6
}
```

### 1-2. data/towns/town_01.json
마을 1개당 JSON 1개. 파일 추가만으로 신규 마을 자동 등록.

```json
{
  "id": "town_01",
  "name": "작은 마을",
  "world_position": { "x": 500, "y": 150 },
  "shop_id": "shop_01",
  "recruit_ids": ["recruit_01", "recruit_02", "recruit_03"],
  "quest_ids": ["quest_01"]
}
```

### 1-3. data/recruits/recruit_pool.json
전체 모집 가능 용병 풀. 마을 JSON의 `recruit_ids`로 참조.

```json
{
  "recruit_01": {
    "unit_name": "카를로",
    "unit_class": "FIGHTER",
    "max_hp": 30,
    "str": 7,
    "move_range": 3,
    "cost": 20
  },
  "recruit_02": {
    "unit_name": "에린",
    "unit_class": "ARCHER",
    "max_hp": 20,
    "str": 6,
    "move_range": 4,
    "cost": 20
  },
  "recruit_03": {
    "unit_name": "비올라",
    "unit_class": "ROGUE",
    "max_hp": 18,
    "str": 5,
    "move_range": 5,
    "cost": 20
  }
}
```

### 1-4. data/shop/shop_items.json
상점별 판매 아이템. 마을 JSON의 `shop_id`로 참조.

```json
{
  "shop_01": [
    {
      "id": "food",
      "name": "식량",
      "buy_price": 2,
      "sell_price": 1,
      "stat": "food",
      "amount": 1
    }
  ]
}
```

> `stat` 필드: WarbandManager의 어떤 static var에 반영할지 지정.
> 추후 `"stat": "gold"` 등 아이템 추가 시 코드 수정 없이 확장 가능.

### 1-5. data/quests/*.json (기존 + gold_reward 추가)

```json
{
  "id": "quest_01",
  "title": "마을의 위협",
  "description": "몬스터를 처치하고 마을을 지켜라.",
  "reward_food": 3,
  "gold_reward": 15
}
```

> `gold_reward` 신규 추가. 전투 승리 시 WorldMapScene._check_combat_rewards()에서 자동 지급.

---

## 2. WarbandManager 변경
> 🇰🇷 씬 간 데이터 공유 확장

```gdscript
# 추가
static var gold: int = 50
static var selected_units: Array[int] = []   # roster 인덱스 배열
static var current_town_id: String = ""       # 진입한 마을 ID

# 전환 (instance → static)
static var roster: Array[MercenaryData] = []
func _ready() -> void:
    if roster.is_empty():   # 최초 1회만 초기화
        roster = [
            load("res://data/mercenaries/fighter_01.tres"),
            load("res://data/mercenaries/archer_01.tres"),
            load("res://data/mercenaries/rogue_01.tres"),
        ]
```

> `roster`를 static으로 전환해야 씬 전환 후에도 모집 결과가 유지됨.
> `is_empty()` 체크로 중복 초기화 방지.

---

## 3. 월드맵 변경
> 🇰🇷 TownZone 동적 스폰

### 3-1. TownZone.gd (신규)

```gdscript
class_name TownZone
extends Node2D

const INTERACT_RADIUS: float = 80.0
var town_data: Dictionary = {}   # JSON에서 로드된 마을 데이터
```

### 3-2. WorldMapScene.gd — 마을 동적 스폰

`_ready()`에서 `data/towns/` 디렉토리를 스캔하여 TownZone을 동적으로 생성.
`.tscn`에 하드코딩하지 않음.

```gdscript
func _spawn_towns() -> void:
    var config := _load_json("res://data/town_config.json")
    var dir := DirAccess.open("res://data/towns/")
    if dir == null:
        return
    dir.list_dir_begin()
    var fname := dir.get_next()
    while fname != "":
        if fname.ends_with(".json"):
            var data: Dictionary = _load_json("res://data/towns/" + fname)
            var zone := TownZone.new()
            zone.town_data = data
            zone.position = Vector2(data["world_position"]["x"],
                                    data["world_position"]["y"])
            _add_town_marker(zone)
            town_layer.add_child(zone)
        fname = dir.get_next()

func _load_json(path: String) -> Dictionary:
    var file := FileAccess.open(path, FileAccess.READ)
    if file == null:
        return {}
    var result: Dictionary = JSON.parse_string(file.get_as_text())
    file.close()
    return result
```

> 마을 JSON 파일 추가만으로 월드맵에 자동 등록. 코드 수정 불필요.

### 3-3. 마을 진입 로직

```gdscript
func _enter_town(zone: TownZone) -> void:
    WarbandManager.player_world_pos = player_icon.position
    WarbandManager.current_town_id = zone.town_data["id"]
    get_tree().change_scene_to_file("res://scenes/town/town.tscn")
```

### 3-4. world_map.tscn 변경
- `TownLayer (Node2D)` 추가 (마커용 컨테이너)
- TownZone 노드는 코드에서 동적 생성 → tscn에 직접 배치 안 함
- `PartySelectPanel` 추가 (출전 선택 UI)

### 3-5. E키 우선순위
```
NPC → EncounterZone → TownZone
```

---

## 4. TownScene 씬 구조
> 🇰🇷 마을 씬 노드 트리

```
TownScene (Node2D)  [TownScene.gd]
├── Background (ColorRect)  — Color(0.18, 0.14, 0.10)
└── TownUI (CanvasLayer)
    ├── HUD (VBoxContainer)  offset(10, 10)
    │   ├── TownLabel       "[ {마을 이름} ]"   ← JSON에서 로드
    │   ├── GoldLabel       "골드: N"
    │   ├── FoodLabel       "식량: N"
    │   ├── FatigueLabel    "피로도: N / 100"
    │   └── LeaveButton     "마을 떠나기"
    │
    ├── MenuPanel (VBoxContainer)  offset(450, 200)
    │   ├── MenuTitle       "무엇을 하시겠습니까?"
    │   ├── RestButton      "휴식하기"           ← 비용은 town_config.json
    │   ├── QuestButton     "의뢰 게시판"
    │   ├── RecruitButton   "용병 모집"
    │   └── ShopButton      "상점"
    │
    ├── RestPanel (PanelContainer)  — hidden
    │   └── VBoxContainer
    │       ├── RestInfo    "{cost}G → 피로도 -{recovery}"  ← JSON 값
    │       ├── ConfirmButton  "휴식하기"
    │       └── BackButton
    │
    ├── QuestPanel (PanelContainer)  — hidden
    │   └── VBoxContainer
    │       ├── QuestTitle  "의뢰 게시판"
    │       ├── QuestList (VBoxContainer)  ← 동적 생성 (마을 JSON quest_ids 기반)
    │       └── BackButton
    │
    ├── RecruitPanel (PanelContainer)  — hidden
    │   └── VBoxContainer
    │       ├── RecruitTitle  "용병 모집"
    │       ├── RecruitList (VBoxContainer)  ← 동적 생성 (recruit_pool.json 기반)
    │       └── BackButton
    │
    └── ShopPanel (PanelContainer)  — hidden
        └── VBoxContainer
            ├── ShopTitle   "상점"
            ├── ItemList (VBoxContainer)  ← 동적 생성 (shop_items.json 기반)
            │     각 항목: HBoxContainer
            │       ├── BuyButton   "구매 ({buy_price}G)"
            │       ├── SellButton  "판매 ({sell_price}G)"
            │       └── ItemLabel   "{name}"
            ├── ShopStatus  "식량: N  골드: N"
            └── BackButton
```

---

## 5. 기능별 로직
> 🇰🇷 데이터 기반 처리 방식

### 5-1. TownScene 초기화
```gdscript
func _ready() -> void:
    var town_data := _load_json("res://data/towns/%s.json" % WarbandManager.current_town_id)
    var config := _load_json("res://data/town_config.json")
    town_label.text = "[ %s ]" % town_data["name"]
    _setup_rest(config)
    _setup_quests(town_data["quest_ids"])
    _setup_recruits(town_data["recruit_ids"])
    _setup_shop(town_data["shop_id"])
    _update_hud()
```

### 5-2. 휴식
- `town_config.json`에서 `rest_cost_gold`, `rest_fatigue_recovery` 로드
- 조건: `WarbandManager.gold >= cost AND WarbandManager.fatigue > 0`
- 버튼 텍스트: `"휴식하기 (%dG / 피로도 -%d)" % [cost, recovery]` — 런타임 조합

### 5-3. 의뢰 게시판
- 마을 JSON `quest_ids` 순회 → 각 quest JSON 로드
- 표시 조건: `active_quests`와 `completed_quests` 모두에 없는 퀘스트만
- 수락 → `WarbandManager.active_quests.append(id)`
- **재방문 시 중복 표시 없음**: 매번 active/completed 체크 후 동적 생성

### 5-4. 용병 모집
- 마을 JSON `recruit_ids` → `recruit_pool.json`에서 각 recruit 로드
- 이미 roster에 동일 `unit_name`이 있으면 "고용됨" disabled 표시
- 고용 조건: `gold >= cost AND roster.size() < config["max_roster_size"]`
- 고용 시: MercenaryData 인스턴스 동적 생성

```gdscript
const CLASS_MAP: Dictionary = {
    "FIGHTER": MercenaryData.UnitClass.FIGHTER,
    "ARCHER":  MercenaryData.UnitClass.ARCHER,
    "ROGUE":   MercenaryData.UnitClass.ROGUE,
    "MAGE":    MercenaryData.UnitClass.MAGE,
}

func _hire_recruit(recruit: Dictionary) -> void:
    var data := MercenaryData.new()
    data.unit_name  = recruit["unit_name"]
    data.unit_class = CLASS_MAP[recruit["unit_class"]]
    data.max_hp     = recruit["max_hp"]
    data.str        = recruit["str"]
    data.move_range = recruit["move_range"]
    WarbandManager.gold -= recruit["cost"]
    WarbandManager.roster.append(data)
```

### 5-5. 상점
- 마을 JSON `shop_id` → `shop_items.json`에서 아이템 목록 로드
- 각 아이템: `stat` 필드로 WarbandManager 어떤 값에 반영할지 결정
- 구매: `gold >= buy_price` 조건 / 판매: 해당 stat > 0 조건

```gdscript
func _buy_item(item: Dictionary) -> void:
    WarbandManager.gold -= item["buy_price"]
    WarbandManager[item["stat"]] += item["amount"]

func _sell_item(item: Dictionary) -> void:
    WarbandManager[item["stat"]] -= item["amount"]
    WarbandManager.gold += item["sell_price"]
```

---

## 6. 전투 승리 → 골드 보상
> 🇰🇷 quest gold_reward 자동 지급

기존 `WorldMapScene._check_combat_rewards()`에 골드 지급 추가:

```gdscript
func _check_combat_rewards() -> void:
    if WarbandManager.last_combat_result != "victory":
        return
    WarbandManager.last_combat_result = ""
    var reward_lines: Array[String] = []
    for qid in WarbandManager.active_quests.duplicate():
        var data := _load_json("res://data/quests/%s.json" % qid)
        WarbandManager.food += data.get("reward_food", 0)
        WarbandManager.gold += data.get("gold_reward", 0)    # ← 신규
        WarbandManager.active_quests.erase(qid)
        WarbandManager.completed_quests.append(qid)
        reward_lines.append("%s\n식량 +%d  골드 +%d" % [
            data.get("title", qid),
            data.get("reward_food", 0),
            data.get("gold_reward", 0)
        ])
    if reward_lines.size() > 0:
        reward_label.text = "퀘스트 완료!\n" + "\n".join(reward_lines)
        reward_popup.visible = true
        _refresh_quest_log()
```

---

## 7. 출전 멤버 선택 시스템
> 🇰🇷 Battle Party Selection System

### 7-1. 개요
전투 진입(EncounterZone E키) 시 즉시 전환하지 않고
**출전 멤버 선택 패널**을 먼저 표시.
인원 제한은 `town_config.json`의 `max_party_size` 값 사용.

### 7-2. WarbandManager 연동
```gdscript
static var selected_units: Array[int] = []
# roster의 인덱스 배열
# 예: [0, 2] → roster[0], roster[2]가 출전
```

### 7-3. 출전 선택 UI (world_map.tscn에 추가)
```
WorldMapUI/PartySelectPanel (PanelContainer)  — hidden
  offset: (300, 150) ~ (850, 500)
  └── VBoxContainer
      ├── TitleLabel    "출전 멤버 선택"
      ├── RosterList (VBoxContainer)  ← 동적 생성
      │     각 항목: HBoxContainer
      │       ├── CheckButton  (토글)
      │       └── InfoLabel    "이름  직업  HP:N  STR:N  MV:N"
      ├── StatusLabel   "선택: N / {max_party_size}명"
      ├── StartButton   "전투 시작"  (0명이면 disabled)
      └── CancelButton  "취소"
```

> RosterList는 `WarbandManager.roster` 전체 기반으로 동적 생성.
> 모집한 신규 용병도 roster에 추가되므로 자동 표시됨.

### 7-4. WorldMapScene.gd 흐름
```gdscript
func _enter_combat() -> void:
    WarbandManager.player_world_pos = player_icon.position
    WarbandManager.selected_units = []
    _open_party_select()

func _open_party_select() -> void:
    # RosterList 동적 생성 (roster 기반)
    # max_party_size는 town_config.json에서 로드
    party_select_panel.visible = true
    set_process_unhandled_input(false)

func _on_party_start_pressed() -> void:
    WarbandManager.last_combat_result = ""
    get_tree().change_scene_to_file("res://scenes/combat/combat_scene.tscn")

func _on_party_cancel_pressed() -> void:
    WarbandManager.selected_units = []
    party_select_panel.visible = false
    set_process_unhandled_input(true)
```

### 7-5. CombatScene.gd 변경
```gdscript
const ALLY_SPAWN_POSITIONS: Array[Vector2i] = [
    Vector2i(2, 2), Vector2i(3, 2), Vector2i(4, 2),
    Vector2i(5, 2), Vector2i(6, 2), Vector2i(7, 2),
]

func _spawn_units() -> void:
    var indices := WarbandManager.selected_units
    if indices.is_empty():   # fallback: roster 전체
        indices = range(WarbandManager.roster.size()) as Array[int]
    for i in range(mini(indices.size(), ALLY_SPAWN_POSITIONS.size())):
        var data := WarbandManager.roster[indices[i]]
        _spawn_unit(ALLY_SPAWN_POSITIONS[i], data, true, ally_container)
    for i in range(ENEMY_SPAWN_POSITIONS.size()):
        var d := load(ENEMY_DATA[i]) as MercenaryData
        _spawn_unit(ENEMY_SPAWN_POSITIONS[i], d, false, enemy_container)
```

---

## 8. 예상 파일 목록
> 🇰🇷 신규/수정 파일

| 파일 | 작업 |
|------|------|
| `data/town_config.json` | 신규 |
| `data/towns/town_01.json` | 신규 |
| `data/recruits/recruit_pool.json` | 신규 |
| `data/shop/shop_items.json` | 신규 |
| `data/quests/quest_01.json` | 수정 (gold_reward 추가) |
| `scripts/town/TownScene.gd` | 신규 |
| `scenes/town/town.tscn` | 신규 |
| `scripts/world/TownZone.gd` | 신규 |
| `scripts/world/WorldMapScene.gd` | TownZone 동적 스폰, 파티 선택, 골드 보상 |
| `scenes/world/world_map.tscn` | TownLayer + PartySelectPanel 추가 |
| `scripts/data/WarbandManager.gd` | gold, selected_units, current_town_id, roster static화 |
| `scripts/combat/CombatScene.gd` | selected_units 기반 스폰, ALLY_SPAWN 6개 확장 |

총 **12개 파일** (신규 8, 수정 4)

---

## 9. Acceptance Criteria
> 🇰🇷 완료 기준

**데이터 기반:**
- [ ] town_01.json 수정만으로 마을 위치/이름 변경 가능
- [ ] recruit_pool.json에 항목 추가만으로 신규 모집 용병 등장
- [ ] shop_items.json에 아이템 추가만으로 상점 확장 가능
- [ ] town_config.json으로 휴식 비용/회복량 조정 가능

**마을 시스템:**
- [ ] 월드맵에서 마을 마커 표시 (동적 스폰)
- [ ] E키로 마을 씬 진입 → 진입 위치로 복귀
- [ ] 휴식: town_config 기반 비용/회복
- [ ] 의뢰 게시판: 재방문 시 중복 표시 없음
- [ ] 용병 모집: 골드 소모, roster 추가 (최대 6명)
- [ ] 상점: shop_items.json 기반 동적 구성

**퀘스트 보상:**
- [ ] 전투 승리 시 gold_reward 자동 지급
- [ ] 보상 팝업에 골드 수치 표시

**출전 선택:**
- [ ] 전투 진입 시 파티 선택 패널 표시
- [ ] 신규 모집 용병 목록에 표시
- [ ] 0명 선택 시 전투 시작 비활성화
- [ ] selected_units 기반 스폰 확인

- [ ] Godot 에러 없음
