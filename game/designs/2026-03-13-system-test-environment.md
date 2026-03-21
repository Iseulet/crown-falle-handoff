# CrownFalle — 시스템 테스트 환경 설계

- Date: 2026-03-13
- Agent: @Planner
- Status: 설계 완료 — 사용자 승인 대기
- References:
  - `2026-03-08-stat_system.md` — 스탯 파생 공식, combat_config.jsonc 계수
  - `2026-03-08-stamina_system.md` — STA 공식, 소모/회복 규칙
  - `2026-03-08-passive_skill_system.md` — trigger/condition/effects 구조
  - `2026-03-08-unit_data_architecture.md` — UnitData/MercenaryData/EnemyData, 레벨 성장
  - `2026-03-08-engagement_system.md` — 교전 판정 로직

---

## 0. 설계 목표

| 목표 | 설명 |
|------|------|
| 공식 검증 | JSON 계수 변경 → 결과 즉시 확인 (코드 재시작 불필요) |
| 시각적 확인 | 숫자만이 아닌, 바/색상/로그로 시스템 동작을 눈으로 확인 |
| 격리 테스트 | 전투 흐름 없이 개별 시스템만 독립 검증 가능 |
| 실전 모니터링 | 실제 전투 중 내부 수치를 실시간 관찰 |
| 데이터 기반 | 테스트 UI 자체도 수치 하드코딩 없음 |

---

## 1. 전체 구조

```
┌─────────────────────────────────────────────┐
│            Part A: 독립 샌드박스              │
│         scenes/test/system_test.tscn         │
│                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │SpawnPanel│ │ StatPanel │ │  LogPanel    │ │
│  │ 유닛생성  │ │ 스탯뷰어  │ │  이벤트로그  │ │
│  └──────────┘ └──────────┘ └──────────────┘ │
│  ┌─────────────────────────────────────────┐ │
│  │          ControlPanel                   │ │
│  │  전투 시뮬레이션 버튼 + 시나리오 선택    │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│        Part B: 전투 씬 디버그 오버레이       │
│     기존 combat_scene.tscn에 토글 패널      │
│                                              │
│  F3 키 → 디버그 HUD 토글                    │
│  ┌────────────────┐ ┌────────────────────┐  │
│  │ UnitInspector  │ │  EventStream       │  │
│  │ 선택 유닛 상세  │ │  실시간 이벤트 흐름 │  │
│  └────────────────┘ └────────────────────┘  │
└─────────────────────────────────────────────┘
```

---

# Part A: 독립 샌드박스 씬

## A-1. 씬 구조

```
SystemTestScene (Control)                    ← 전체화면 UI
├── HSplitContainer
│   ├── LeftPanel (VBoxContainer)            ← 좌측 40%
│   │   ├── SpawnPanel (PanelContainer)
│   │   │   └── SpawnContent (VBoxContainer)
│   │   │       ├── TitleLabel "유닛 생성"
│   │   │       ├── ClassSelector (OptionButton)     ← Fighter/Archer/Rogue/Mage
│   │   │       ├── LevelSpinBox (SpinBox)           ← 1~20
│   │   │       ├── FactionSelector (OptionButton)   ← 아군/적군
│   │   │       ├── SpawnButton (Button) "생성"
│   │   │       └── UnitList (VBoxContainer)         ← 생성된 유닛 목록
│   │   │
│   │   └── ControlPanel (PanelContainer)
│   │       └── ControlContent (VBoxContainer)
│   │           ├── TitleLabel "시뮬레이션"
│   │           ├── ScenarioSelector (OptionButton)
│   │           │     ← "단일 공격" / "교전 턴 진행" / "패시브 발동 체크"
│   │           ├── AttackerPicker (OptionButton)     ← 생성된 유닛 중 선택
│   │           ├── TargetPicker (OptionButton)
│   │           ├── ExecuteButton (Button) "실행"
│   │           ├── StepTurnButton (Button) "턴 진행"
│   │           └── ResetButton (Button) "전체 초기화"
│   │
│   └── RightPanel (VBoxContainer)           ← 우측 60%
│       ├── StatPanel (PanelContainer)       ← 상단 50%
│       │   └── StatContent (HSplitContainer)
│       │       ├── UnitAPanel (VBoxContainer)  ← 유닛 A 스탯 전체 표시
│       │       └── UnitBPanel (VBoxContainer)  ← 유닛 B 스탯 전체 표시 (비교용)
│       │
│       └── LogPanel (PanelContainer)        ← 하단 50%
│           └── LogContent (VBoxContainer)
│               ├── FilterBar (HBoxContainer)
│               │   ├── FilterAll (CheckBox) "전체"
│               │   ├── FilterStat (CheckBox) "스탯"
│               │   ├── FilterCombat (CheckBox) "전투"
│               │   ├── FilterSTA (CheckBox) "STA"
│               │   ├── FilterPassive (CheckBox) "패시브"
│               │   ├── FilterHIT (CheckBox) "적중"
│               │   ├── FilterCRIT (CheckBox) "치명타"
│               │   └── ClearButton (Button) "로그 클리어"
│               └── LogScroll (ScrollContainer)
│                   └── LogList (RichTextLabel)
```

---

## A-2. SpawnPanel — 유닛 생성기

### 기능

- 클래스 선택(4종) + 레벨(1~20) + 팩션(아군/적군) 조합으로 유닛 생성
- 생성된 유닛은 UnitList에 요약 라벨로 표시
- 최대 8개까지 생성 (아군 4 + 적군 4)
- 개별 유닛 삭제 버튼

### 유닛 생성 로직

```gdscript
# scripts/test/SystemTestScene.gd

func _on_spawn_pressed() -> void:
    var class_id: String = _class_selector.get_item_text(_class_selector.selected)
    var level: int = int(_level_spin.value)
    var is_ally: bool = _faction_selector.selected == 0

    # class_config.json에서 기본 스탯 로드
    var class_data: Dictionary = DataLoader.get_class_config(class_id)
    var base_stats: Dictionary = class_data.base_stats.duplicate()

    # 적 자동 성장 적용 (레벨 > 1일 때)
    if level > 1:
        base_stats = _apply_level_growth(class_id, base_stats, level)

    # TestUnit 데이터 구조 생성
    var unit := TestUnitData.new()
    unit.unit_id = "TEST_%03d" % _unit_counter
    unit.unit_name = "%s Lv.%d" % [class_id, level]
    unit.class_id = class_id
    unit.level = level
    unit.is_ally = is_ally
    unit.primary_stats = base_stats
    unit.class_data = class_data

    # 2차 스탯 계산
    _recalculate_derived_stats(unit)

    # 패시브 로드
    unit.passives = _load_passives(class_id)

    _spawned_units.append(unit)
    _refresh_unit_list()
    _refresh_pickers()
    _log("SPAWN", "[%s] %s 생성 (Lv.%d, %s)" % [
        unit.unit_id, class_id, level, "아군" if is_ally else "적군"
    ], Color.WHITE)
```

### 레벨 성장 함수

```gdscript
func _apply_level_growth(class_id: String, stats: Dictionary, level: int) -> Dictionary:
    # unit_data_architecture.md 확정 규칙 적용
    var growth_map := {
        "Fighter": [["str", "con"]],      # 홀수→STR, 짝수→CON
        "Archer":  [["dex"]],              # 매 레벨→DEX
        "Rogue":   [["dex"]],              # 매 레벨→DEX
        "Mage":    [["int_stat", "wil"]],  # 홀수→INT, 짝수→WIL
    }
    var growth_stats: Array = growth_map.get(class_id, [["str"]])[0]

    for lv in range(2, level + 1):
        if growth_stats.size() == 1:
            stats[growth_stats[0]] += 1
        else:
            var idx := (lv % 2)  # 홀수=0(첫번째), 짝수=1(두번째)
            # 홀수 레벨 → 첫 번째 스탯, 짝수 레벨 → 두 번째 스탯
            var stat_key: String = growth_stats[0] if lv % 2 == 1 else growth_stats[1]
            stats[stat_key] += 1

    return stats
```

---

## A-3. StatPanel — 스탯 뷰어

### 표시 구조 (유닛 1개당)

```
┌─ Fighter Lv.3 (아군) ─────────────────────┐
│                                            │
│  ── 1차 스탯 ──                            │
│  STR  9   DEX  4   CON  9   INT  2  WIL 2 │
│                                            │
│  ── 2차 스탯 (파생) ──                      │
│  HP   18  (CON 9 × 2)                     │
│  MP    6  (WIL 2 × 3)                     │
│  STA  28  (10 + CON 9 × 2)               │
│  HIT  72% (70 + DEX 4 × 0.5)             │
│  MV    4  (base 4 + DEX 4 × 0.2 → 4)     │
│  CRIT  2% (DEX 4 × 0.5 + bonus 0)        │
│  ARM  2.7% (CON 9 × 0.3)                 │
│  RES  0.6% (WIL 2 × 0.3)                 │
│                                            │
│  ── 전투 자원 (현재/최대) ──                │
│  HP  ████████████████████ 18/18            │
│  STA ████████████████████ 28/28            │
│  MP  ████████████████████  6/6             │
│                                            │
│  ── 패시브 스킬 ──                          │
│  ◆ 강인한 체력 (on_combat_start)           │
│  ◆ 방어 본능 (on_hit, HP≤50%)             │
│  ◆ 전열 유지 (on_engage)                   │
│                                            │
│  ── 활성 버프/디버프 ──                     │
│  (없음)                                    │
└────────────────────────────────────────────┘
```

### 핵심: 파생 공식 실시간 표시

2차 스탯 옆에 계산 과정을 괄호로 함께 표시한다.
이를 통해 `combat_config.jsonc`의 계수가 올바르게 적용되는지 즉시 확인 가능.

```gdscript
func _format_derived_stat(label: String, value, formula_str: String) -> String:
    return "%s  %s  (%s)" % [label, str(value), formula_str]

# 예시 출력:
# "HIT  72%  (70 + DEX 4 × 0.5)"
# "MV   4    (base 4 + DEX 4 × 0.2 → floor 4)"
```

### 2차 스탯 계산 (combat_config.jsonc 계수 사용)

```gdscript
func _recalculate_derived_stats(unit: TestUnitData) -> void:
    var cfg := DataLoader.get_combat_config()
    var sd: Dictionary = cfg.stat_derivation
    var s: Dictionary = unit.primary_stats
    var cd: Dictionary = unit.class_data

    unit.derived = {
        "hp": s.con * sd.hp_per_con,
        "mp": s.wil * sd.mp_per_wil,
        "sta": cfg.stamina.sta_base + s.con * cfg.stamina.sta_per_con,
        "hit": cfg.hit.base_hit_rate + s.dex * sd.dex_hit_coefficient,
        "mv": int(cd.mv_base + s.dex * sd.dex_mv_coefficient),
        "crit": s.dex * sd.dex_crit_coefficient + cd.crit_bonus,
        "arm": s.con * sd.con_arm_coefficient,
        "res": s.wil * sd.wil_res_coefficient,
    }

    # 상한/하한 적용
    unit.derived.hit = clampf(unit.derived.hit, cfg.hit.hit_min, cfg.hit.hit_max)
    unit.derived.arm = minf(unit.derived.arm, cfg.damage.arm_max)
    unit.derived.res = minf(unit.derived.res, cfg.damage.res_max)

    # 현재 자원 초기화
    unit.current_hp = unit.derived.hp
    unit.current_sta = unit.derived.sta
    unit.current_mp = unit.derived.mp

    # 공식 문자열 생성 (표시용)
    unit.formulas = {
        "hp": "CON %d × %s" % [s.con, sd.hp_per_con],
        "mp": "WIL %d × %s" % [s.wil, sd.mp_per_wil],
        "sta": "%s + CON %d × %s" % [cfg.stamina.sta_base, s.con, cfg.stamina.sta_per_con],
        "hit": "%s + DEX %d × %s" % [cfg.hit.base_hit_rate, s.dex, sd.dex_hit_coefficient],
        "mv": "base %d + DEX %d × %s → floor %d" % [cd.mv_base, s.dex, sd.dex_mv_coefficient, unit.derived.mv],
        "crit": "DEX %d × %s + bonus %s" % [s.dex, sd.dex_crit_coefficient, cd.crit_bonus],
        "arm": "CON %d × %s" % [s.con, sd.con_arm_coefficient],
        "res": "WIL %d × %s" % [s.wil, sd.wil_res_coefficient],
    }
```

### 게이지 바 표시

HP/STA/MP를 색상 바로 표시. 비율에 따라 색상 전환:

```gdscript
func _get_gauge_color(current: float, max_val: float, type: String) -> Color:
    var ratio := current / max_val if max_val > 0 else 0.0
    match type:
        "hp":
            if ratio > 0.5: return Color(0.2, 0.8, 0.2)    # 초록
            if ratio > 0.25: return Color(0.9, 0.7, 0.1)   # 노랑
            return Color(0.9, 0.2, 0.2)                      # 빨강
        "sta":
            if ratio > 0.3: return Color(0.9, 0.7, 0.1)    # 노랑
            return Color(0.9, 0.4, 0.1)                      # 주황
        "mp":
            return Color(0.3, 0.4, 0.9)                      # 파랑
    return Color.WHITE
```

---

## A-4. ControlPanel — 시뮬레이션 실행

### 시나리오 목록

| # | 시나리오 | 설명 | 필요 유닛 |
|---|---------|------|----------|
| 1 | 단일 공격 (물리) | 공격자→타겟 물리 공격 1회 (HIT/데미지/CRIT/ARM 전체 흐름) | 2 |
| 2 | 단일 공격 (마법) | 공격자→타겟 마법 공격 1회 (항상 명중, RES 감소) | 2 |
| 3 | 교전 성립 → 턴 진행 | 공격 → Engagement 형성 → 턴 시작 시 STA 변화 | 2 |
| 4 | 패시브 발동 체크 | 선택된 유닛의 패시브를 지정 트리거로 발동 테스트 | 1~2 |
| 5 | 레벨 비교 | 같은 클래스, 다른 레벨 유닛 2개의 스탯 나란히 비교 | 2 |
| 6 | STA 소진 시뮬레이션 | 이동N칸 + 공격M회 반복 → STA 0 도달 확인 | 1 |
| 7 | 연속 전투 | 공격 10회 자동 반복 → 평균 명중률/데미지 통계 출력 | 2 |

### 시나리오 1: 단일 공격 (물리) 상세 흐름

```gdscript
func _execute_physical_attack(attacker: TestUnitData, target: TestUnitData) -> void:
    var cfg := DataLoader.get_combat_config()

    # 1. STA 체크
    var sta_cost: int = cfg.stamina.sta_cost_attack
    if attacker.current_sta < sta_cost:
        _log("STA", "STA 부족 (%d < %d) — 공격 불가" % [attacker.current_sta, sta_cost], Color.ORANGE)
        return
    attacker.current_sta -= sta_cost
    _log("STA", "%s STA -%d → %d/%d" % [attacker.unit_name, sta_cost, attacker.current_sta, attacker.derived.sta], Color.YELLOW)

    # 2. 적중 판정
    var level_diff: int = attacker.level - target.level
    var final_hit: float = attacker.derived.hit + level_diff * cfg.hit.level_diff_coefficient
    final_hit = clampf(final_hit, cfg.hit.hit_min, cfg.hit.hit_max)
    var roll: float = randf() * 100.0
    var hit_success: bool = roll < final_hit

    _log("HIT", "적중 판정: roll %.1f vs HIT %.1f%% (base %.1f + lvl_diff %d×%s) → %s" % [
        roll, final_hit, attacker.derived.hit, level_diff, cfg.hit.level_diff_coefficient,
        "명중!" if hit_success else "빗나감"
    ], Color.CYAN if hit_success else Color.GRAY)

    if not hit_success:
        _refresh_stat_panels()
        return

    # 3. 기본 데미지
    var damage_stat_key: String = _get_damage_stat(attacker.class_id)
    var damage_coefficient: float = _get_damage_coefficient(attacker.class_id, cfg)
    var base_damage: float = attacker.primary_stats[damage_stat_key] * damage_coefficient

    # 4. 치명타 판정
    var crit_roll: float = randf() * 100.0
    var is_crit: bool = crit_roll < attacker.derived.crit
    var crit_mult: float = 1.0
    if is_crit:
        crit_mult = 1.0 + cfg.crit.crit_damage_bonus / 100.0
        _log("CRIT", "치명타! roll %.1f vs CRIT %.1f%% → 배율 ×%.2f" % [
            crit_roll, attacker.derived.crit, crit_mult
        ], Color.RED)

    # 5. ARM 감소
    var arm_reduction: float = target.derived.arm / 100.0
    var final_damage: int = maxi(1, int(base_damage * (1.0 - arm_reduction) * crit_mult))

    _log("COMBAT", "데미지: base %.1f × (1 - ARM %.1f%%) × crit %.2f = %d" % [
        base_damage, target.derived.arm, crit_mult, final_damage
    ], Color.WHITE)

    # 6. HP 적용
    var old_hp: int = target.current_hp
    target.current_hp = maxi(0, target.current_hp - final_damage)

    _log("COMBAT", "%s HP %d → %d (-%d)%s" % [
        target.unit_name, old_hp, target.current_hp, final_damage,
        " ★ 사망!" if target.current_hp == 0 else ""
    ], Color.RED if target.current_hp == 0 else Color.WHITE)

    # 7. 패시브 발동 체크 (on_hit)
    _check_passives("on_hit", target, attacker)

    _refresh_stat_panels()
```

### 시나리오 3: 교전 + 턴 진행

```gdscript
func _execute_engagement_scenario(attacker: TestUnitData, target: TestUnitData) -> void:
    # 공격 실행 (시나리오 1/2와 동일)
    _execute_physical_attack(attacker, target)

    # Engagement 성립
    if target.current_hp > 0 and not attacker.is_engaged and not target.is_engaged:
        attacker.is_engaged = true
        target.is_engaged = true
        attacker.engaged_with = target
        target.engaged_with = attacker
        _log("COMBAT", "교전 성립: %s ⚔ %s" % [attacker.unit_name, target.unit_name], Color.ORANGE)

        # on_engage 패시브 체크
        _check_passives("on_engage", attacker, target)
        _check_passives("on_engage", target, attacker)

func _execute_step_turn() -> void:
    var cfg := DataLoader.get_combat_config()
    _log("SYSTEM", "──── 새 턴 시작 ────", Color.WHITE)

    for unit in _spawned_units:
        # 1. STA 회복
        var old_sta: int = unit.current_sta
        unit.current_sta = mini(unit.current_sta + cfg.stamina.sta_regen_per_turn, unit.derived.sta)
        _log("STA", "%s STA 회복 +%d → %d/%d" % [
            unit.unit_name, cfg.stamina.sta_regen_per_turn, unit.current_sta, unit.derived.sta
        ], Color.GREEN)

        # 2. 교전 유지 비용
        if unit.is_engaged:
            unit.current_sta = maxi(0, unit.current_sta - cfg.stamina.sta_cost_engagement)
            _log("STA", "%s 교전 유지비 -%d → %d/%d" % [
                unit.unit_name, cfg.stamina.sta_cost_engagement, unit.current_sta, unit.derived.sta
            ], Color.ORANGE)

        # 3. on_turn_start 패시브
        _check_passives("on_turn_start", unit, null)

        # 4. duration 만료 효과 제거
        _expire_turn_buffs(unit)

    _refresh_stat_panels()
```

### 시나리오 4: 패시브 발동 체크

```gdscript
func _execute_passive_check(unit: TestUnitData, trigger: String, context_unit: TestUnitData) -> void:
    _log("SYSTEM", "── 패시브 체크: %s [%s] ──" % [unit.unit_name, trigger], Color.WHITE)

    for passive in unit.passives:
        if passive.trigger != trigger:
            continue

        # condition 평가
        var condition_met := _evaluate_condition(passive.condition, unit, context_unit)

        _log("PASSIVE", "  %s (%s): 조건 %s → %s" % [
            passive.name, passive.trigger,
            str(passive.condition) if passive.condition != null else "무조건",
            "발동!" if condition_met else "불충족"
        ], Color.GREEN if condition_met else Color.GRAY)

        if condition_met:
            for effect in passive.effects:
                var chance_pass := true
                if effect.chance != null:
                    var roll := randf() * 100.0
                    chance_pass = roll < effect.chance
                    _log("PASSIVE", "    확률 체크: roll %.1f vs %d%% → %s" % [
                        roll, effect.chance, "성공" if chance_pass else "실패"
                    ], Color.GREEN if chance_pass else Color.GRAY)

                if chance_pass:
                    _apply_effect(unit, context_unit, effect)
                    _log("PASSIVE", "    효과 적용: %s %s %+d (duration: %s)" % [
                        effect.target, effect.stat, effect.value,
                        str(effect.duration) if effect.duration != null else "영구"
                    ], Color.YELLOW)
```

---

## A-5. LogPanel — 이벤트 로그

### 이벤트 타입 & 색상 코딩

| 타입 | 색상 | BBCode | 예시 |
|------|------|--------|------|
| SPAWN | 흰색 | `[color=white]` | 유닛 생성/삭제 |
| STAT | 라임 | `[color=lime]` | 스탯 변경 |
| COMBAT | 흰색 | `[color=white]` | 데미지 적용, 사망 |
| HIT | 시안 / 회색 | `[color=cyan]` / `[color=gray]` | 적중 성공/실패 |
| CRIT | 빨강 | `[color=red]` | 치명타 발동 |
| STA | 노랑 / 주황 | `[color=yellow]` / `[color=orange]` | 소모/회복/교전비 |
| PASSIVE | 초록 / 회색 | `[color=green]` / `[color=gray]` | 발동 성공/불충족 |

### 로그 출력 함수

```gdscript
func _log(type: String, message: String, color: Color) -> void:
    var timestamp := "T%d" % _turn_counter
    var colored_type := "[color=#%s][%s][/color]" % [color.to_html(false), type]
    var entry := "[%s] %s %s" % [timestamp, colored_type, message]

    _log_entries.append({"type": type, "text": entry, "raw_message": message})

    # 필터 적용 후 표시
    if _is_type_visible(type):
        _log_label.append_text(entry + "\n")
        # 자동 스크롤
        await get_tree().process_frame
        _log_scroll.scroll_vertical = _log_scroll.get_v_scroll_bar().max_value

### 파일 저장

```gdscript
func _save_log_to_file() -> void:
    var dir := DirAccess.open("user://")
    if not dir.dir_exists("test_logs"):
        dir.make_dir("test_logs")

    var timestamp := Time.get_datetime_string_from_system().replace(":", "-")
    var path := "user://test_logs/system_test_%s.log" % timestamp
    var file := FileAccess.open(path, FileAccess.WRITE)
    for entry in _log_entries:
        # BBCode 태그 제거한 텍스트 저장
        file.store_line(entry.raw_message)
    file.close()
    _log("SYSTEM", "로그 저장: %s" % path, Color.WHITE)
```

---

## A-6. 시나리오 7: 연속 전투 통계

```gdscript
func _execute_batch_combat(attacker: TestUnitData, target: TestUnitData, count: int) -> void:
    var stats := {
        "total": count,
        "hits": 0,
        "misses": 0,
        "crits": 0,
        "total_damage": 0,
        "max_damage": 0,
        "min_damage": 999999,
        "kills": 0,
    }

    _log("SYSTEM", "──── 연속 전투 %d회 시작 ────" % count, Color.WHITE)

    for i in range(count):
        # 매 회 HP/STA 리셋
        target.current_hp = target.derived.hp
        attacker.current_sta = attacker.derived.sta
        var result := _simulate_single_attack(attacker, target)

        if result.hit:
            stats.hits += 1
            stats.total_damage += result.damage
            stats.max_damage = maxi(stats.max_damage, result.damage)
            stats.min_damage = mini(stats.min_damage, result.damage)
            if result.crit:
                stats.crits += 1
            if result.killed:
                stats.kills += 1
        else:
            stats.misses += 1

    # 통계 출력
    var avg_dmg: float = float(stats.total_damage) / float(stats.hits) if stats.hits > 0 else 0.0
    _log("SYSTEM", "──── 결과 통계 ────", Color.WHITE)
    _log("STAT", "명중률: %d/%d (%.1f%%)" % [stats.hits, stats.total, float(stats.hits) / float(stats.total) * 100.0], Color.CYAN)
    _log("STAT", "치명타: %d/%d (%.1f%%)" % [stats.crits, stats.hits, float(stats.crits) / float(stats.hits) * 100.0 if stats.hits > 0 else 0.0], Color.RED)
    _log("STAT", "평균 데미지: %.1f (min %d / max %d)" % [avg_dmg, stats.min_damage, stats.max_damage], Color.WHITE)
    _log("STAT", "처치: %d/%d (%.1f%%)" % [stats.kills, stats.total, float(stats.kills) / float(stats.total) * 100.0], Color.RED)

    _refresh_stat_panels()
```

---

# Part B: 전투 씬 디버그 오버레이

## B-1. 씬 구조 (기존 combat_scene.tscn에 추가)

```
CombatScene
├── ... (기존 노드 전부 유지) ...
└── DebugOverlay (CanvasLayer)          ← layer = 100 (최상위)
    ├── visible = false                  ← F3 토글
    ├── UnitInspector (PanelContainer)   ← 좌측 상단
    │   └── InspectorContent (VBoxContainer)
    │       ├── InspectorTitle (Label)
    │       ├── PrimaryStats (GridContainer, columns=5)
    │       ├── DerivedStats (GridContainer, columns=2)
    │       ├── ResourceBars (VBoxContainer)
    │       ├── PassiveList (VBoxContainer)
    │       └── BuffList (VBoxContainer)
    │
    └── EventStream (PanelContainer)     ← 우측
        └── StreamContent (VBoxContainer)
            ├── StreamTitle (Label) "이벤트 스트림"
            └── StreamScroll (ScrollContainer)
                └── StreamLog (RichTextLabel)
```

## B-2. 토글 제어

```gdscript
# CombatScene.gd — 추가

@onready var _debug_overlay: CanvasLayer = $DebugOverlay
var _debug_visible: bool = false

func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventKey and event.pressed:
        if event.keycode == KEY_F3:
            _debug_visible = not _debug_visible
            _debug_overlay.visible = _debug_visible
            if _debug_visible:
                _refresh_debug_inspector()
```

## B-3. UnitInspector — 선택 유닛 상세

전투 씬에서 유닛을 클릭 선택하면, 디버그 오버레이에 해당 유닛의 전체 내부 수치가 표시된다.

### 표시 내용

```
┌─ [ALLY] Fighter Lv.3  "Aldric" ───────────┐
│                                            │
│  STR 9  DEX 4  CON 9  INT 2  WIL 2        │
│                                            │
│  HP  14/18  ████████████░░░░ 77.8%         │
│  STA 23/28  ████████████████░░ 82.1%       │
│  MP   6/6   ████████████████████ 100%      │
│                                            │
│  HIT 72.0%  MV 4  CRIT 2.0%               │
│  ARM 2.7%   RES 0.6%                      │
│                                            │
│  교전: ⚔ Enemy_001 (Fighter Lv.2)         │
│  위치: (3, 4)                              │
│                                            │
│  패시브:                                   │
│  ◆ 강인한 체력     ✓ 발동됨               │
│  ◆ 방어 본능       ○ 대기 (HP>50%)        │
│  ◆ 전열 유지       ✓ 발동됨               │
│                                            │
│  버프/디버프:                              │
│  ▲ STA +4 (강인한 체력, 영구)             │
│  ▲ STR -1→적 (전열 유지, 영구)            │
└────────────────────────────────────────────┘
```

### 갱신 시점

```gdscript
# 다음 이벤트 발생 시 자동 갱신:
# - 유닛 선택 변경
# - 공격 실행 후
# - 턴 시작 시
# - 교전 상태 변경 시
# - 패시브 발동 시

func _refresh_debug_inspector() -> void:
    if not _debug_visible:
        return

    var unit: CombatUnit = selected_unit
    if unit == null:
        _inspector_title.text = "(유닛 미선택)"
        return

    var data: UnitData = unit.unit_data
    var faction := "ALLY" if unit.is_ally else "ENEMY"

    _inspector_title.text = "[%s] %s Lv.%d  \"%s\"" % [
        faction, data.class_id, data.level, data.unit_name
    ]

    # 1차 스탯
    _update_primary_grid(data)
    # 2차 스탯 + 자원 바
    _update_derived_display(unit)
    # 교전 상태
    _update_engagement_display(unit)
    # 패시브 목록 + 발동 상태
    _update_passive_display(unit)
    # 활성 버프/디버프
    _update_buff_display(unit)
```

## B-4. EventStream — 실시간 이벤트 흐름

전투 중 발생하는 모든 시스템 이벤트를 시간순으로 스트리밍.
Part A의 LogPanel과 동일한 색상 코딩 + 필터 없이 전체 표시.

### 이벤트 소스 연결

```gdscript
# CombatScene.gd — 기존 함수에 디버그 로그 삽입

func _handle_hit_frame(event_data: Dictionary) -> void:
    # ... 기존 피해 처리 ...

    if _debug_visible:
        _debug_log("HIT", "적중 판정: roll %.1f vs HIT %.1f%% → %s" % [
            roll, final_hit, "명중" if success else "빗나감"
        ])
        if success:
            _debug_log("COMBAT", "데미지: %d (base %.1f × ARM %.1f%% × crit %.2f)" % [
                final_damage, base_damage, arm_percent, crit_mult
            ])

func _on_turn_start() -> void:
    # ... 기존 턴 처리 ...

    if _debug_visible:
        for unit in _all_units:
            _debug_log("STA", "%s: regen +%d, engage -%d → %d/%d" % [
                unit.unit_data.unit_name,
                regen, engage_cost,
                unit.current_sta, unit.max_sta
            ])
```

### 스트림 로그 함수

```gdscript
const MAX_STREAM_LINES := 50  # 최근 50개만 유지 (성능)

var _stream_entries: Array[String] = []

func _debug_log(type: String, message: String) -> void:
    if not _debug_visible:
        return

    var color := _get_type_color(type)
    var entry := "[color=#%s][%s][/color] %s" % [color, type, message]

    _stream_entries.append(entry)
    if _stream_entries.size() > MAX_STREAM_LINES:
        _stream_entries.pop_front()

    _stream_log.clear()
    _stream_log.text = ""
    for e in _stream_entries:
        _stream_log.append_text(e + "\n")
```

---

## B-5. 패시브 발동 시각 피드백

전투 중 패시브가 발동되면 UnitInspector의 해당 패시브 옆에 깜빡이는 하이라이트.

```gdscript
func _flash_passive_indicator(passive_id: String) -> void:
    var label: Label = _passive_labels.get(passive_id)
    if label == null:
        return

    var original_color: Color = label.modulate
    var flash_color := Color(1.0, 1.0, 0.3, 1.0)  # 밝은 노랑

    var tween := create_tween()
    tween.tween_property(label, "modulate", flash_color, 0.1)
    tween.tween_property(label, "modulate", original_color, 0.4)
```

---

# 공통 데이터 구조

## TestUnitData (샌드박스 전용)

```gdscript
# scripts/test/TestUnitData.gd
class_name TestUnitData
extends RefCounted

var unit_id: String
var unit_name: String
var class_id: String
var level: int
var is_ally: bool

var primary_stats: Dictionary = {}    # { str, dex, con, int_stat, wil }
var class_data: Dictionary = {}       # mv_base, crit_bonus, passives[]
var derived: Dictionary = {}          # { hp, mp, sta, hit, mv, crit, arm, res }
var formulas: Dictionary = {}         # { hp: "CON 8 × 2", ... } 표시용

var current_hp: int
var current_sta: int
var current_mp: int

var is_engaged: bool = false
var engaged_with: TestUnitData = null

var passives: Array = []              # passive_skills.json에서 로드된 데이터
var active_buffs: Array = []          # { passive_id, stat, value, remaining_turns }
```

---

# 구현 순서

| 단계 | 항목 | 의존성 | 규모 |
|------|------|--------|------|
| T-1 | `TestUnitData` 데이터 구조 | 없음 | Small |
| T-2 | SpawnPanel — 유닛 생성 + 레벨 성장 | T-1 | Medium |
| T-3 | StatPanel — 1차/2차 스탯 + 공식 표시 + 게이지 바 | T-1 | Medium |
| T-4 | LogPanel — RichTextLabel + 색상 코딩 + 필터 + 파일 저장 | 없음 | Medium |
| T-5 | ControlPanel — 시나리오 1,2 (단일 공격 물리/마법) | T-2, T-3, T-4 | Large |
| T-6 | ControlPanel — 시나리오 3 (교전 + 턴 진행) | T-5 | Medium |
| T-7 | ControlPanel — 시나리오 4 (패시브 발동 체크) | T-5 | Medium |
| T-8 | ControlPanel — 시나리오 5,6,7 (비교/소진/연속전투) | T-5 | Medium |
| T-9 | DebugOverlay — 기본 구조 + F3 토글 | 없음 | Small |
| T-10 | UnitInspector — 선택 유닛 상세 표시 | T-9 | Medium |
| T-11 | EventStream — 실시간 로그 스트리밍 | T-9, T-4 | Small |
| T-12 | 패시브 발동 시각 피드백 (깜빡이) | T-10 | Small |
| T-13 | 통합 테스트 | 전체 | — |

### 병렬 그룹

```
그룹 A (샌드박스): T-1 → T-2 → T-3 → T-5 → T-6 → T-7 → T-8
그룹 B (로그):    T-4 (독립, A에서 참조)
그룹 C (디버그):  T-9 → T-10 → T-11 → T-12
         ↓
      T-13 (통합)
```

---

# 파일 목록

## 신규 생성

| 파일 | 역할 |
|------|------|
| `scripts/test/TestUnitData.gd` | 샌드박스 유닛 데이터 |
| `scripts/test/SystemTestScene.gd` | 샌드박스 메인 로직 |
| `scenes/test/system_test.tscn` | 샌드박스 씬 |
| `scripts/debug/DebugOverlay.gd` | 디버그 오버레이 컨트롤러 |
| `scripts/debug/UnitInspector.gd` | 유닛 상세 표시 |
| `scripts/debug/EventStream.gd` | 이벤트 스트림 로그 |

## 수정

| 파일 | 변경 내용 |
|------|----------|
| `scripts/combat/CombatScene.gd` | DebugOverlay 연결, F3 토글, 이벤트 소스 훅 |
| `scenes/combat/combat_scene.tscn` | DebugOverlay CanvasLayer 노드 추가 |

---

# 설계 검증 체크리스트

- [x] 모든 스탯 계산은 combat_config.jsonc 계수 참조 — 하드코딩 없음
- [x] Part A는 전투 흐름과 완전 독립 — CombatScene 의존성 없음
- [x] Part B는 기존 전투 로직 변경 없음 — 로그 삽입만
- [x] 패시브 데이터는 passive_skills.json에서 로드
- [x] 레벨 성장 로직은 unit_data_architecture.md 확정 규칙 준수
- [x] int_stat 매핑 컨벤션 준수
- [x] 로그 파일 저장 경로 user:// 사용 (프로젝트 파일 오염 없음)
- [x] F3 토글은 프로덕션 빌드에서 비활성화 가능 (DEBUG 조건 분기 가능)
