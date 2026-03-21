# system-test-environment.md 보완 패치

- Date: 2026-03-13
- 근거: T2 시스템(StatusManager, SkillManager, ElementResolver, VP) 커버리지 갭
- 적용 대상: `2026-03-13-system-test-environment.md`
- 의존성: stat-system-revision + passive-skill-revision 먼저 적용
- 상태: 사용자 승인 대기

---

## 감사(Audit) 결과 요약

### 문제 1: T2 시나리오 누락

| 시스템 | 구현 상태 | 기존 테스트 커버 | 갭 |
|--------|----------|----------------|-----|
| 스탯 파생 | ✅ | ✅ 시나리오 1,2,5 | — |
| 전투 공식 | ✅ | ✅ 시나리오 1,2,7 | — |
| 스태미너 | ✅ | ✅ 시나리오 3,6 | — |
| 패시브 스킬 | ✅ | ✅ 시나리오 4 | — |
| 교전 시스템 | ✅ | ✅ 시나리오 3 | — |
| **ARM/RES 게이지** | ✅ (개정) | ❌ **없음** | 게이지 흡수 → HP 경로 |
| **StatusManager** | ✅ (개정) | ❌ **없음** | CC 게이팅 + DoT 틱 |
| **SkillManager** | ✅ 설계 | ❌ **없음** | 쿨다운, MP 소모, on_hit_status |
| **ElementResolver** | ✅ 설계 | ❌ **없음** | 원소 콤보 반응 |
| **VP 시스템** | ✅ 설계 | ❌ **없음** | 처치 ±VP, 이탈 조건 |

### 문제 2: 명칭 충돌

| 출처 | 파일명 | 역할 |
|------|--------|------|
| CLI 작업 서머리 (구현됨) | `DebugCombatPanel.gd` | 전투 씬 내 디버그 패널 (이미 존재) |
| 설계 문서 (Part B) | `DebugOverlay.gd` + `UnitInspector.gd` + `EventStream.gd` | 동일 목적 신규 설계 |

**결정:** `DebugCombatPanel.gd`가 이미 구현되어 있으므로, Part B는 이를 **확장**하는 방식으로 변경. 신규 파일 3개 → 기존 파일 1개 확장으로 통합.

### 문제 3: TestUnitData 누락 필드

기존 TestUnitData에 T2 연동 필드 없음:
- `current_arm` / `current_res` (게이지)
- `current_status_effects` (상태이상)
- `active_skills` (액티브 스킬)
- `vp` (사기 포인트)
- `element_states` (원소 상태)

### 문제 4: StatPanel ARM/RES 표시 누락

기존 설계의 StatPanel은 ARM/RES를 `%` 방식으로 표시하고 있으나,
개정안에서 게이지형으로 전환됨 → 게이지 바 + 소진 상태 표시 필요.

---

# 패치 내용

## 패치 1: TestUnitData 필드 추가 (섹션 "공통 데이터 구조" 교체)

```gdscript
# scripts/test/TestUnitData.gd — 전체 교체
class_name TestUnitData
extends RefCounted

# === 기본 정보 ===
var unit_id: String
var unit_name: String
var class_id: String
var level: int
var is_ally: bool

# === T1: 스탯 ===
var primary_stats: Dictionary = {}    # { str, dex, con, int_stat, wil }
var class_data: Dictionary = {}       # mv_base, crit_bonus, passives[]
var derived: Dictionary = {}          # { hp, mp, sta, hit, mv, crit, arm, res }
var formulas: Dictionary = {}         # { hp: "CON 8 × 2", ... } 표시용

# === T1: 전투 자원 (현재/최대) ===
var current_hp: int
var current_sta: int
var current_mp: int

# === T2: ARM/RES 게이지 ===
var current_arm: int = 0              # 물리 방어 게이지 (현재)
var current_res: int = 0              # 마법 저항 게이지 (현재)

func is_parm_depleted() -> bool:
    return current_arm == 0

func is_marm_depleted() -> bool:
    return current_res == 0

# === T1: 교전 ===
var is_engaged: bool = false
var engaged_with: TestUnitData = null

# === T1: 패시브 ===
var passives: Array = []              # passive_skills.json 데이터
var active_buffs: Array = []          # { passive_id, stat, value, remaining_turns }

# === T2: 상태이상 ===
var current_status_effects: Array = []
# 각 원소: { "status_id": "BLEED", "remaining_turns": 2,
#            "caster_snapshot": { str: 8, int_stat: 2 } }

func has_status(status_id: String) -> bool:
    for se in current_status_effects:
        if se.status_id == status_id:
            return true
    return false

# === T2: 액티브 스킬 ===
var active_skills: Array = []
# 각 원소: { "skill_id": "power_strike", "current_cooldown": 0 }

# === T2: VP (사기) ===
var vp: int = 0                       # 현재 사기 포인트

# === T2: 원소 상태 ===
var element_states: Array[String] = []
# 적용 중인 원소 태그: ["wet", "burning"] 등
```

---

## 패치 2: StatPanel ARM/RES 게이지 표시 추가

기존 StatPanel 표시 구조(섹션 A-3)에서 ARM/RES 행을 교체.

### 기존 (삭제)
```
│  ARM  2.7% (CON 9 × 0.3)                 │
│  RES  0.6% (WIL 2 × 0.3)                 │
```

### 변경 (대체)
```
│  ── 방어 게이지 ──                          │
│  ARM ██████████░░░░░░░░░░  8/8   정상      │
│  RES ██░░░░░░░░░░░░░░░░░░  2/2   정상      │
│                                            │
│  (ARM 0 = ⚠ 물리방어 소진 → 물리 CC 가능)  │
│  (RES 0 = ⚠ 마법저항 소진 → 마법 CC 가능)  │
```

### StatPanel `_recalculate_derived_stats()` 수정

```gdscript
# ARM/RES 게이지 계산 추가
var arm_base: int = cfg.stat_derivation.get("arm_base", 0)
var arm_per_con: int = cfg.stat_derivation.get("arm_per_con", 1)
var res_base: int = cfg.stat_derivation.get("res_base", 0)
var res_per_wil: int = cfg.stat_derivation.get("res_per_wil", 1)

unit.derived.arm = arm_base + s.con * arm_per_con
unit.derived.res = res_base + s.wil * res_per_wil
unit.current_arm = unit.derived.arm
unit.current_res = unit.derived.res

unit.formulas.arm = "%d + CON %d × %d" % [arm_base, s.con, arm_per_con]
unit.formulas.res = "%d + WIL %d × %d" % [res_base, s.wil, res_per_wil]
```

### StatPanel 상태이상 / 원소 표시 추가

기존 "활성 버프/디버프" 영역 아래에 추가:

```
│  ── 상태이상 ──                             │
│  🩸 BLEED (2턴 남음, 시전자 STR 8)          │
│  ⚡ STUNNED (1턴 남음)                       │
│                                            │
│  ── 원소 상태 ──                             │
│  💧 wet                                     │
│                                            │
│  ── VP (사기) ──                             │
│  VP: 3 / 4 (최대)                           │
```

---

## 패치 3: 시나리오 8~11 추가 (섹션 A-4 ControlPanel 확장)

### 시나리오 목록 (기존 7종 + 신규 4종)

| # | 시나리오 | 설명 | 필요 유닛 | 의존 시스템 |
|---|---------|------|----------|-----------|
| 1~7 | (기존 유지) | | | T1 |
| **8** | **상태이상 (CC 게이팅)** | ARM/RES 게이지 깎기 → 소진 → CC 적용 | 2 | StatusManager |
| **9** | **액티브 스킬** | 쿨다운, MP/STA 소모, on_hit_status 확률 | 2 | SkillManager |
| **10** | **원소 상성** | 원소 부여 → 콤보 반응 체크 | 2 | ElementResolver |
| **11** | **VP (사기)** | 처치 시 VP 변동, VP 0 이탈 조건 | 4+ | VP 시스템 |

### 시나리오 8: 상태이상 (CC 게이팅) 상세

```gdscript
func _execute_status_effect_scenario(attacker: TestUnitData, target: TestUnitData) -> void:
    _log("SYSTEM", "──── 시나리오 8: 상태이상 테스트 ────", Color.WHITE)

    # Phase 1: ARM 게이지 상태 확인
    _log("STAT", "%s ARM: %d/%d | RES: %d/%d" % [
        target.unit_name,
        target.current_arm, target.derived.arm,
        target.current_res, target.derived.res
    ], Color.CYAN)

    # Phase 2: 물리 공격으로 ARM 게이지 깎기
    _log("SYSTEM", "── ARM 게이지 공격 시작 ──", Color.WHITE)
    while target.current_arm > 0 and target.current_hp > 0:
        _execute_physical_attack_with_gauge(attacker, target)

    # Phase 3: ARM 소진 확인
    if target.is_parm_depleted():
        _log("STAT", "⚠ %s ARM 소진 (parm_depleted)" % target.unit_name, Color.RED)

        # Phase 4: CC 적용 시도 (KNOCKDOWN)
        _log("SYSTEM", "── CC 적용 시도: KNOCKDOWN ──", Color.WHITE)
        var condition := { "target_parm_depleted": true }
        var condition_met := _evaluate_condition(condition, attacker, target)
        _log("PASSIVE", "CC 게이팅 조건: target_parm_depleted = %s → %s" % [
            str(target.is_parm_depleted()),
            "통과!" if condition_met else "차단"
        ], Color.GREEN if condition_met else Color.RED)

        if condition_met:
            _apply_status_effect(target, "KNOCKDOWN", attacker)

    # Phase 5: 턴 진행 → DoT 틱 + duration 감소
    if target.has_status("BLEED") or target.has_status("BURNING"):
        _log("SYSTEM", "── 턴 진행: DoT 틱 확인 ──", Color.WHITE)
        _process_status_turn_start(target)

    _refresh_stat_panels()


func _execute_physical_attack_with_gauge(attacker: TestUnitData, target: TestUnitData) -> void:
    """게이지 흡수 로직이 포함된 물리 공격"""
    var cfg := DataLoader.get_combat_config()

    # 기본 데미지 계산 (기존 시나리오 1과 동일)
    var damage_stat_key := _get_damage_stat(attacker.class_id)
    var base_damage: float = attacker.primary_stats[damage_stat_key] * _get_damage_coefficient(attacker.class_id, cfg)
    var final_damage: int = maxi(1, int(base_damage))

    # 게이지 흡수
    var absorbed: int = mini(target.current_arm, final_damage)
    var overflow: int = final_damage - absorbed
    target.current_arm -= absorbed

    _log("COMBAT", "데미지 %d → ARM 흡수 %d (ARM %d→%d), HP 피해 %d" % [
        final_damage, absorbed,
        target.current_arm + absorbed, target.current_arm,
        overflow
    ], Color.WHITE)

    if overflow > 0:
        target.current_hp = maxi(0, target.current_hp - overflow)
        _log("COMBAT", "%s HP %d (-%d)" % [target.unit_name, target.current_hp, overflow],
            Color.RED if target.current_hp == 0 else Color.YELLOW)


func _apply_status_effect(target: TestUnitData, status_id: String, caster: TestUnitData) -> void:
    """상태이상 적용"""
    var status_data := DataLoader.get_status_effect(status_id)
    if status_data.is_empty():
        _log("SYSTEM", "상태이상 '%s' 데이터 없음" % status_id, Color.RED)
        return

    # 중복 체크
    if target.has_status(status_id):
        if not status_data.get("refresh_on_reapply", false):
            _log("PASSIVE", "%s 이미 %s 상태 — 중복 적용 불가" % [target.unit_name, status_id], Color.GRAY)
            return
        # refresh: 기존 제거 후 재적용
        _remove_status(target, status_id)

    var entry := {
        "status_id": status_id,
        "remaining_turns": status_data.get("default_duration", 1),
        "caster_snapshot": {
            "str": caster.primary_stats.get("str", 0),
            "int_stat": caster.primary_stats.get("int_stat", 0),
        }
    }
    target.current_status_effects.append(entry)

    _log("PASSIVE", "✓ %s에 %s 적용 (%d턴)" % [
        target.unit_name, status_data.get("name", status_id), entry.remaining_turns
    ], Color.GREEN)

    # Disable 효과 즉시 표시
    if status_data.get("prevents_action", false):
        _log("PASSIVE", "  → 행동 불가 상태!" , Color.RED)


func _process_status_turn_start(unit: TestUnitData) -> void:
    """턴 시작 시 상태이상 처리: DoT → duration 감소 → 해제"""
    var to_remove: Array[String] = []

    for se in unit.current_status_effects:
        var status_data := DataLoader.get_status_effect(se.status_id)

        # DoT 처리
        if status_data.get("effect_type", "") == "dot":
            var dot_stat: String = status_data.get("dot_damage_stat", "str")
            var dot_coeff: float = status_data.get("dot_damage_coefficient", 0.3)
            var dot_base: float = se.caster_snapshot.get(dot_stat, 0) * dot_coeff
            var dot_damage: int = maxi(1, int(dot_base))
            var dot_type: String = status_data.get("dot_damage_type", "physical")

            # 게이지 흡수 적용
            var shield: int = unit.current_arm if dot_type == "physical" else unit.current_res
            var absorbed: int = mini(shield, dot_damage)
            var hp_damage: int = dot_damage - absorbed

            if dot_type == "physical":
                unit.current_arm -= absorbed
            else:
                unit.current_res -= absorbed
            unit.current_hp = maxi(0, unit.current_hp - hp_damage)

            _log("STA", "%s DoT [%s]: %d 데미지 (게이지 흡수 %d, HP -%d)" % [
                unit.unit_name, se.status_id, dot_damage, absorbed, hp_damage
            ], Color.ORANGE)

        # duration 감소
        se.remaining_turns -= 1
        if se.remaining_turns <= 0:
            to_remove.append(se.status_id)
            _log("PASSIVE", "%s [%s] 해제 (duration 만료)" % [unit.unit_name, se.status_id], Color.GRAY)

    for sid in to_remove:
        _remove_status(unit, sid)
```

### 시나리오 9: 액티브 스킬

```gdscript
func _execute_active_skill_scenario(attacker: TestUnitData, target: TestUnitData) -> void:
    _log("SYSTEM", "──── 시나리오 9: 액티브 스킬 테스트 ────", Color.WHITE)

    # 스킬 선택 (ControlPanel의 SkillPicker에서 선택)
    var skill_id: String = _skill_picker.get_item_text(_skill_picker.selected)
    var skill_data := DataLoader.get_skill(skill_id)

    if skill_data.is_empty():
        _log("SYSTEM", "스킬 '%s' 데이터 없음" % skill_id, Color.RED)
        return

    # 1. 쿨다운 체크
    var skill_entry := _find_skill_entry(attacker, skill_id)
    if skill_entry != null and skill_entry.current_cooldown > 0:
        _log("COMBAT", "쿨다운 %d턴 남음 — 사용 불가" % skill_entry.current_cooldown, Color.ORANGE)
        return

    # 2. 자원 소모 체크
    var resource_type: String = skill_data.get("resource", "sta")  # "sta" | "mp"
    var resource_cost: int = skill_data.get("cost", 0)

    match resource_type:
        "sta":
            if attacker.current_sta < resource_cost:
                _log("STA", "STA 부족 (%d < %d)" % [attacker.current_sta, resource_cost], Color.ORANGE)
                return
            attacker.current_sta -= resource_cost
            _log("STA", "%s STA -%d → %d" % [attacker.unit_name, resource_cost, attacker.current_sta], Color.YELLOW)
        "mp":
            if attacker.current_mp < resource_cost:
                _log("STA", "MP 부족 (%d < %d)" % [attacker.current_mp, resource_cost], Color.ORANGE)
                return
            attacker.current_mp -= resource_cost
            _log("STA", "%s MP -%d → %d" % [attacker.unit_name, resource_cost, attacker.current_mp], Color.BLUE)

    # 3. damage_type 분기
    var damage_type: String = skill_data.get("damage_type", "physical")
    var damage_multiplier: float = skill_data.get("damage_multiplier", 1.0)

    _log("COMBAT", "스킬 [%s] 사용: damage_type=%s, multiplier=%.2f" % [
        skill_data.get("name", skill_id), damage_type, damage_multiplier
    ], Color.CYAN)

    # 4. 데미지 계산 + 게이지 흡수 (damage_type 기반)
    var base_damage: float
    if damage_type == "physical":
        var stat_key := _get_damage_stat(attacker.class_id)
        base_damage = attacker.primary_stats[stat_key] * damage_multiplier
    else:
        base_damage = attacker.primary_stats.get("int_stat", 0) * damage_multiplier

    var final_damage: int = maxi(1, int(base_damage))

    # 게이지 분기
    var shield_key: String = "current_arm" if damage_type == "physical" else "current_res"
    var shield: int = target.get(shield_key)
    var absorbed: int = mini(shield, final_damage)
    var hp_damage: int = final_damage - absorbed
    target.set(shield_key, shield - absorbed)
    target.current_hp = maxi(0, target.current_hp - hp_damage)

    var gauge_name := "ARM" if damage_type == "physical" else "RES"
    _log("COMBAT", "데미지 %d → %s 흡수 %d, HP -%d | %s HP %d" % [
        final_damage, gauge_name, absorbed, hp_damage, target.unit_name, target.current_hp
    ], Color.WHITE)

    # 5. on_hit_status 확률 체크
    var on_hit_status: Dictionary = skill_data.get("on_hit_status", {})
    if not on_hit_status.is_empty():
        var status_id: String = on_hit_status.get("status_id", "")
        var chance: int = on_hit_status.get("chance", 0)
        var roll: float = randf() * 100.0
        var success := roll < chance

        # CC 게이팅 체크
        var gating_key := "target_parm_depleted" if damage_type == "physical" else "target_marm_depleted"
        var gate_passed: bool = target.is_parm_depleted() if damage_type == "physical" else target.is_marm_depleted()

        _log("PASSIVE", "on_hit_status [%s]: 확률 roll %.1f vs %d%% → %s | 게이팅(%s) → %s" % [
            status_id, roll, chance,
            "통과" if success else "실패",
            gating_key,
            "열림" if gate_passed else "차단"
        ], Color.GREEN if (success and gate_passed) else Color.GRAY)

        if success and gate_passed:
            _apply_status_effect(target, status_id, attacker)

    # 6. 쿨다운 설정
    var cooldown: int = skill_data.get("cooldown", 0)
    if cooldown > 0:
        if skill_entry == null:
            skill_entry = { "skill_id": skill_id, "current_cooldown": cooldown }
            attacker.active_skills.append(skill_entry)
        else:
            skill_entry.current_cooldown = cooldown
        _log("COMBAT", "쿨다운 설정: %d턴" % cooldown, Color.YELLOW)

    _refresh_stat_panels()
```

### 시나리오 10: 원소 상성

```gdscript
func _execute_element_scenario(attacker: TestUnitData, target: TestUnitData) -> void:
    _log("SYSTEM", "──── 시나리오 10: 원소 상성 테스트 ────", Color.WHITE)

    # 원소 부여 (ElementPicker에서 선택)
    var element: String = _element_picker.get_item_text(_element_picker.selected)

    _log("STAT", "%s 현재 원소 상태: %s" % [target.unit_name, str(target.element_states)], Color.CYAN)

    # 원소 부여
    if element not in target.element_states:
        target.element_states.append(element)
        _log("STAT", "%s에 [%s] 원소 부여" % [target.unit_name, element], Color.GREEN)

    # 콤보 반응 체크
    var reaction := _check_element_reaction(target)
    if reaction.is_empty():
        _log("STAT", "콤보 반응: 없음", Color.GRAY)
    else:
        _log("STAT", "⚡ 콤보 반응 발동: %s → %s!" % [
            reaction.get("trigger_combo", ""),
            reaction.get("result", "")
        ], Color.RED)

        # 반응 효과 적용
        _apply_element_reaction(target, reaction)

        # 소모된 원소 제거
        for consumed in reaction.get("consumed_elements", []):
            target.element_states.erase(consumed)

    _refresh_stat_panels()


func _check_element_reaction(unit: TestUnitData) -> Dictionary:
    """
    원소 콤보 테이블 (combat-system-proposal-final 기준)
    6원소 3레이어: 장비(poison) / 스킬(fire, lightning, water) / 지형(darkness)
    """
    var states := unit.element_states
    var reactions := DataLoader.get_element_reactions()  # data/combat/element_reactions.json

    for reaction in reactions:
        var required: Array = reaction.get("required_elements", [])
        var all_present := true
        for req in required:
            if req not in states:
                all_present = false
                break
        if all_present:
            return reaction

    return {}
```

### 시나리오 11: VP (사기)

```gdscript
func _execute_vp_scenario() -> void:
    _log("SYSTEM", "──── 시나리오 11: VP (사기) 테스트 ────", Color.WHITE)

    # VP 초기화: 인원 × 1
    var allies := _spawned_units.filter(func(u): return u.is_ally)
    var enemies := _spawned_units.filter(func(u): return not u.is_ally)

    var ally_vp: int = allies.size()
    var enemy_vp: int = enemies.size()

    _log("STAT", "아군 VP: %d (인원 %d × 1) | 적군 VP: %d (인원 %d × 1)" % [
        ally_vp, allies.size(), enemy_vp, enemies.size()
    ], Color.CYAN)

    for ally in allies:
        ally.vp = ally_vp
    for enemy in enemies:
        enemy.vp = enemy_vp

    # 처치 시뮬레이션
    if enemies.size() >= 1:
        var killed := enemies[0]
        killed.current_hp = 0
        _log("COMBAT", "★ %s 처치!" % killed.unit_name, Color.RED)

        # VP 변동
        enemy_vp -= 1
        _log("STAT", "적군 VP: %d → %d (-1 처치)" % [enemy_vp + 1, enemy_vp], Color.ORANGE)

        # VP 0 이탈 체크
        if enemy_vp <= 0:
            _log("STAT", "⚠ 적군 VP 0 → 이탈 조건 충족!", Color.RED)
        else:
            # VP 이탈 임계값 체크 (50% 이하)
            var max_vp: int = enemies.size()
            var vp_ratio: float = float(enemy_vp) / float(max_vp) * 100.0
            _log("STAT", "VP 비율: %.0f%% (임계값 이하 시 이탈 판정)" % vp_ratio, Color.YELLOW)

        for enemy in enemies:
            enemy.vp = enemy_vp

    _refresh_stat_panels()
```

---

## 패치 4: ControlPanel 씬 구조 확장

기존 ControlContent에 시나리오 8~11 관련 UI 위젯 추가:

```
ControlContent (VBoxContainer)
    ├── ... (기존 위젯 유지) ...
    ├── HSeparator                              ← 추가
    ├── T2Label (Label) "── T2 시스템 ──"       ← 추가
    ├── StatusPicker (OptionButton)              ← 추가
    │     ← "BLEED"/"KNOCKDOWN"/"CRIPPLED"/"BURNING"/"STUNNED"/"WEAKENED"/"FRIGHTENED"
    ├── SkillPicker (OptionButton)               ← 추가
    │     ← 선택된 유닛의 class_id 기반 스킬 목록 동적 로드
    ├── ElementPicker (OptionButton)             ← 추가
    │     ← "poison"/"fire"/"lightning"/"water"/"darkness"
    └── ApplyStatusButton (Button) "상태이상 수동 부여"  ← 추가
```

---

## 패치 5: Part B 명칭 통합 (DebugOverlay → DebugCombatPanel 확장)

### 변경 전 (기존 설계)

```
신규 파일 3개:
  scripts/debug/DebugOverlay.gd
  scripts/debug/UnitInspector.gd
  scripts/debug/EventStream.gd
```

### 변경 후

```
기존 파일 1개 확장:
  scripts/debug/DebugCombatPanel.gd  ← 이미 구현됨, 아래 기능 추가

추가 내용:
  - UnitInspector 영역: ARM/RES 게이지 바 + 소진 상태 표시
  - UnitInspector 영역: 상태이상 목록 + 남은 턴수
  - UnitInspector 영역: VP 표시
  - EventStream 영역: T2 이벤트 로그 (CC 게이팅, DoT, 원소 반응, VP 변동)
```

### DebugCombatPanel.gd에 추가할 표시 항목

```
┌─ [ALLY] Fighter Lv.3 ─────────────────────┐
│                                            │
│  ... (기존 1차/2차 스탯 표시 유지) ...       │
│                                            │
│  ── 방어 게이지 ──                          │  ← 추가
│  ARM ████████░░░░  5/8                     │
│  RES ██░░░░░░░░░░  2/2                     │
│                                            │
│  ── 상태이상 ──                              │  ← 추가
│  🩸 BLEED (1턴)  ⚡ STUNNED (1턴)          │
│                                            │
│  ── 원소 상태 ──                             │  ← 추가
│  💧 wet  🔥 burning                        │
│                                            │
│  ── VP ──                                   │  ← 추가
│  아군 VP: 3/4  적군 VP: 2/3                 │
│                                            │
│  ── 스킬 쿨다운 ──                           │  ← 추가
│  Power Strike: 사용 가능                    │
│  Fireball: 쿨다운 2턴                       │
└────────────────────────────────────────────┘
```

---

## 패치 6: 구현 순서 수정

기존 T-1~T-13에 T2 관련 단계 추가:

| 단계 | 항목 | 의존성 | 규모 | 변경 |
|------|------|--------|------|------|
| T-1 | TestUnitData (T2 필드 포함) | 없음 | Small | **수정** |
| T-2 | SpawnPanel | T-1 | Medium | 유지 |
| T-3 | StatPanel (ARM/RES 게이지 + 상태이상 표시) | T-1 | Medium | **수정** |
| T-4 | LogPanel | 없음 | Medium | 유지 |
| T-5 | 시나리오 1,2 (게이지 흡수 반영) | T-2,3,4 | Large | **수정** |
| T-6 | 시나리오 3 (교전) | T-5 | Medium | 유지 |
| T-7 | 시나리오 4 (패시브) | T-5 | Medium | 유지 |
| T-8 | 시나리오 5,6,7 (비교/소진/연속전투) | T-5 | Medium | 유지 |
| **T-9** | **시나리오 8 (상태이상 + CC 게이팅)** | T-5 | **Medium** | **신규** |
| **T-10** | **시나리오 9 (액티브 스킬)** | T-5 | **Medium** | **신규** |
| **T-11** | **시나리오 10 (원소 상성)** | T-5 | **Medium** | **신규** |
| **T-12** | **시나리오 11 (VP)** | T-1 | **Small** | **신규** |
| T-13 | DebugCombatPanel 확장 (명칭 통합) | 없음 | Medium | **수정** |
| T-14 | DebugCombatPanel T2 표시 | T-13 | Medium | **신규** |
| T-15 | 패시브 발동 시각 피드백 | T-13 | Small | 유지 (번호 변경) |
| T-16 | 통합 테스트 | 전체 | — | 유지 (번호 변경) |

### 병렬 그룹 (수정)

```
그룹 A (샌드박스 T1):  T-1 → T-2 → T-3 → T-5 → T-6 → T-7 → T-8
그룹 B (샌드박스 T2):  T-9 → T-10 → T-11 → T-12  (T-5 완료 후)
그룹 C (로그):         T-4 (독립)
그룹 D (디버그):       T-13 → T-14 → T-15
                ↓
             T-16 (통합)
```

---

## 패치 7: 신규 파일 목록 수정

### 신규 생성 (수정)

| 파일 | 역할 | 변경 |
|------|------|------|
| `scripts/test/TestUnitData.gd` | 샌드박스 유닛 데이터 (T2 필드 포함) | **수정** |
| `scripts/test/SystemTestScene.gd` | 샌드박스 메인 (시나리오 8~11 포함) | **수정** |
| `scenes/test/system_test.tscn` | 샌드박스 씬 (T2 위젯 포함) | **수정** |
| ~~`scripts/debug/DebugOverlay.gd`~~ | ~~삭제~~ | **삭제** |
| ~~`scripts/debug/UnitInspector.gd`~~ | ~~삭제~~ | **삭제** |
| ~~`scripts/debug/EventStream.gd`~~ | ~~삭제~~ | **삭제** |

### 수정 (수정)

| 파일 | 변경 내용 | 변경 |
|------|----------|------|
| `scripts/debug/DebugCombatPanel.gd` | T2 표시 추가 (ARM/RES 게이지, 상태이상, 원소, VP, 스킬 쿨다운) | **기존 확장** |
| `scripts/combat/CombatScene.gd` | DebugCombatPanel 이벤트 훅 확장 (T2 이벤트) | 유지 |
| `scenes/combat/combat_scene.tscn` | DebugCombatPanel 노드 T2 위젯 추가 | 유지 |

### 신규 데이터 파일 (테스트용)

| 파일 | 역할 |
|------|------|
| `data/skills/test_skills.json` | 시나리오 9용 테스트 스킬 (쿨다운/damage_type/on_hit_status 예시) |
| `data/combat/element_reactions.json` | 시나리오 10용 원소 반응 테이블 |

---

## 검증 체크리스트 (추가)

- [x] TestUnitData에 T2 필드 전체 포함 (ARM/RES 게이지, 상태이상, 스킬, VP, 원소)
- [x] StatPanel에 ARM/RES 게이지 바 + 소진 상태 표시
- [x] 시나리오 8: ARM/RES 깎기 → 소진 → CC 게이팅 → 상태이상 적용 → DoT 틱 검증
- [x] 시나리오 9: MP/STA 소모 → damage_type 분기 → 게이지 흡수 → on_hit_status → 쿨다운
- [x] 시나리오 10: 원소 부여 → 콤보 반응 체크 → 반응 효과 적용
- [x] 시나리오 11: VP 초기화(인원×1) → 처치 시 VP -1 → 이탈 조건 체크
- [x] DebugCombatPanel 명칭 통합 — 신규 파일 3개 삭제, 기존 1개 확장
- [x] 기존 시나리오 1,2의 데미지 흐름이 게이지 흡수 방식으로 갱신
