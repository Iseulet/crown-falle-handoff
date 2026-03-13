# CrownFalle — Tier 2 액티브 스킬 프레임워크 설계서 (T2-2)

> 작성일: 2026-03-13
> Agent: @Planner
> Status: **확정** — 구현 착수 가능
> 의존성: T1-4 (STA), T2-1 (StatusManager), T2-4 (ElementResolver)
> 참조: `2026-03-13-combat-system-proposal-final.md` §6

---

## 1. 설계 목표

직업별 1~3개 액티브 스킬. STA/MP 자원 소모, 쿨다운 관리, 효과(데미지+CC+DoT+유틸) 실행.
기존 PassiveManager 구조와 평행하게 SkillManager로 관리.

---

## 2. 스킬 JSON 구조

```json
{
  "id": "skill_shield_bash",
  "name": "방패 강타",
  "class": "Fighter",
  "type": "active_physical",
  "resource": "sta",
  "cost": 5,
  "range": 1,
  "target": "enemy",
  "damage": { "base_stat": "str", "coefficient": 0.8 },
  "element": null,
  "effects": [
    { "type": "cc", "cc_type": "stun", "gate": "arm", "duration": 1, "chance": 100 }
  ],
  "cooldown": 2,
  "animation": "attack_skill_triple"
}
```

**핵심 필드:**
- `"type"`: `active_physical` / `active_magic` / `utility`
- `"resource"`: `sta` (물리 스킬) / `mp` (마법 스킬)
- `"target"`: `enemy` / `self` / `ally`
- `"damage"`: null이면 데미지 없음 (유틸 스킬)
- `"element"`: null / `fire` / `lightning` / `water` / `ice` / `poison`
- `"effects"`: 효과 배열 (CC, DoT, 디버프, 유틸)
- `"cooldown"`: 사용 후 쿨다운 턴 수

---

## 3. 직업별 스킬 목록

### Fighter (STA 기반)
| 스킬 ID | 이름 | 비용 | 효과 |
|---------|------|------|------|
| `skill_shield_bash` | 방패 강타 | STA 5 | STR × 0.8 데미지 + 기절(ARM 게이트, CD 2) |
| `skill_taunt` | 도발 | STA 3 | 적과 강제 교전 형성 (CD 3) |

### Archer (STA 기반)
| 스킬 ID | 이름 | 비용 | 효과 |
|---------|------|------|------|
| `skill_aimed_shot` | 조준 사격 | STA 4 | DEX × 1.5 데미지, 사거리 5 (CD 2) |

### Rogue (STA 기반)
| 스킬 ID | 이름 | 비용 | 효과 |
|---------|------|------|------|
| `skill_smoke_screen` | 연막 | STA 4 | 교전 해제 + 이탈 (CD 3) |

### Mage (MP 기반)
| 스킬 ID | 이름 | 비용 | 원소 | 효과 |
|---------|------|------|------|------|
| `skill_fireball` | 화염구 | MP 8 | fire | INT × 1.5 + 연소 50% (CD 2) |
| `skill_lightning_bolt` | 번개 볼트 | MP 7 | lightning | INT × 1.2 + 감전 75% (CD 2) |
| `skill_water_bolt` | 물 화살 | MP 5 | water | INT × 0.8 + 젖음 100% (CD 1) |

---

## 4. SkillManager 구조

### 파일 위치
`scripts/combat/SkillManager.gd`

### 핵심 API

```gdscript
class_name SkillManager
extends RefCounted

func setup(data_path: String) -> void
func register_unit(unit: CombatUnit) -> void
func unregister_unit(unit: CombatUnit) -> void

func get_skill(skill_id: String) -> Dictionary
func get_skills_for_class(class_id: String) -> Array[String]

func can_use(unit: CombatUnit, skill_id: String) -> bool       # 쿨다운 + 자원 확인
func consume_resource(unit: CombatUnit, skill_id: String) -> bool
func start_cooldown(unit: CombatUnit, skill_id: String) -> void
func tick_cooldowns(unit: CombatUnit) -> void                  # 턴 시작 시 호출
func get_cooldown(unit: CombatUnit, skill_id: String) -> int

func is_on_cooldown(unit: CombatUnit, skill_id: String) -> bool
func clear_all() -> void
```

### 내부 구조
```gdscript
var _skill_data: Dictionary       # skill_id → JSON 정의
var _unit_cooldowns: Dictionary   # CombatUnit → {skill_id: turns_remaining}
```

---

## 5. CombatScene 연동

### 스킬 실행 함수

```gdscript
func _execute_skill(user: CombatUnit, skill_id: String, target: CombatUnit) -> void:
    var skill: Dictionary = _skill_manager.get_skill(skill_id)
    if skill.is_empty() or not _skill_manager.can_use(user, skill_id):
        return

    if not _skill_manager.consume_resource(user, skill_id):
        return
    _skill_manager.start_cooldown(user, skill_id)

    # Damage
    var element: String = skill.get("element", "")
    if skill.has("damage") and skill["damage"] != null:
        await _resolve_attack(user, target, false, element)
        # (damage formula uses skill coefficient override — Tier 3 UI)

    # Effects: CC, DoT, 유틸
    for fx in skill.get("effects", []):
        _apply_skill_effect(user, target, fx)

    user.play_action(skill.get("animation", "basic_attack"))
```

### 효과 적용

```gdscript
func _apply_skill_effect(user: CombatUnit, target: CombatUnit, fx: Dictionary) -> void:
    var chance: int = int(fx.get("chance", 100))
    if randf_range(0, 100) > chance:
        return
    match fx.get("type", ""):
        "cc":
            var applied := _status_manager.apply_status(target, fx["cc_type"], user)
            if applied:
                _add_log_entry("💢 %s → %s CC: %s" % [...], Color.ORANGE)
        "dot":
            _status_manager.apply_status(target, fx["dot_type"], user)
        "disengage":
            if is_instance_valid(user.engaged_with):
                _execute_vp_disengage(user)  # 스킬 이탈: 기회 공격 없음
        "force_engage":
            if user.engaged_with == null and target.engaged_with == null:
                user.engaged_with = target
                target.engaged_with = user
                _refresh_all_engagement()
```

---

## 6. 쿨다운 처리

턴 시작 `_regen_all_sta()` 내부에서 `_skill_manager.tick_cooldowns(unit)` 호출.

---

## 7. 검증 시나리오 (디버그 모듈)

```
시나리오 1: skill_fireball 사용
  Mage MP >= 8 → 화염구 실행 → INT × 1.5 데미지 + 연소 50% 발동 확인 ✅

시나리오 2: 쿨다운 적용
  skill_shield_bash 사용 → 쿨다운 2턴 → 2턴 후 재사용 가능 ✅

시나리오 3: 연막 이탈
  Rogue 교전 중 smoke_screen → 기회 공격 없이 교전 해제 ✅
```
