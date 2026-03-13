# CrownFalle — Tier 2 상태이상 시스템 설계서 (T2-1)

> 작성일: 2026-03-13
> Agent: @Planner
> Status: **확정** — 구현 착수 가능
> 의존성: T1-2 (ARM/RES 게이지), T1-3 (전투 공식)
> 참조: `2026-03-13-combat-system-proposal-final.md` §8

---

## 1. 설계 목표

DOS2 방식의 **게이지 게이트 CC** + **게이지 무시 DoT** 이원화.
ARM/RES 게이지가 0이 될 때 CC가 활성화되는 구조로 "점사 전략"을 자연 유도.

---

## 2. 상태이상 분류

### CC (군중 제어) — ARM 또는 RES = 0 게이트

| ID | 이름 | 게이트 | 효과 | 지속 |
|----|------|--------|------|------|
| `stun` | 기절 | ARM = 0 | 다음 턴 행동 불능 | 1턴 |
| `knockdown` | 넘어짐 | ARM = 0 | 행동 불능, 기상에 STA 소모 | 1턴 |
| `frozen` | 빙결 | RES = 0 | 행동 불능, 물리 피격 시 추가 데미지 | 1턴 |
| `charm` | 매혹 | RES = 0 | 가장 가까운 아군 공격 | 1턴 |
| `fear` | 공포 | RES = 0 | 무작위 방향 이동 후 턴 종료 | 1턴 |

> CC는 game 에서 "effect": "skip_turn" / "attack_ally" / "random_move" 로 구분

### DoT — 게이트 없음 (방어 게이지 무시, HP 직접)

| ID | 이름 | 원소 | DoT 데미지/턴 | 지속 |
|----|------|------|---------------|------|
| `poisoned` | 중독 | 독 | 3 | 3턴 |
| `burning` | 연소 | 화염 | 4 | 2턴 |
| `bleeding` | 출혈 | 물리 | 2 | 3턴 |

### 디버프 — 게이트 없음 (상태 수정)

| ID | 이름 | 원소 | 효과 | 지속 |
|----|------|------|------|------|
| `shocked` | 감전 | 전기 | MV -1 | 2턴 |
| `wet` | 젖음 | 물 | 화염 반감, 전기 증폭, 냉기→빙결 전환 | 2턴 |
| `chilled` | 냉기 | 냉기 | MV -1, 재적용 시 빙결 전환 | 2턴 |

---

## 3. StatusManager 구조

### 파일 위치
`scripts/combat/StatusManager.gd`

### 핵심 API

```gdscript
class_name StatusManager
extends RefCounted

func setup(data_path: String) -> void
func register_unit(unit: CombatUnit) -> void
func unregister_unit(unit: CombatUnit) -> void

# 상태이상 적용 (게이트 실패 시 false 반환)
func apply_status(unit: CombatUnit, effect_id: String, source: CombatUnit = null) -> bool

func has_status(unit: CombatUnit, effect_id: String) -> bool
func remove_status(unit: CombatUnit, effect_id: String) -> void
func get_statuses(unit: CombatUnit) -> Array  # [{effect_id, turns_remaining}]

# 턴 종료 시 DoT 틱 처리. 반환: [{unit, effect_id, damage}]
func process_tick(unit: CombatUnit) -> Array

func is_skip_turn(unit: CombatUnit) -> bool      # stun/knockdown/frozen/charm/fear
func get_mv_penalty(unit: CombatUnit) -> int      # shocked/chilled MV 페널티 합산
func clear_all() -> void
```

### 내부 구조

```gdscript
var _status_data: Dictionary  # effect_id → JSON 정의
var _unit_statuses: Dictionary  # CombatUnit → Array[{effect_id, turns_remaining}]
```

---

## 4. status_effects.json 구조

```json
{
  "stun": {
    "type": "cc",
    "gate": "arm",
    "duration": 1,
    "effect": "skip_turn"
  },
  "knockdown": {
    "type": "cc",
    "gate": "arm",
    "duration": 1,
    "effect": "skip_turn"
  },
  "frozen": {
    "type": "cc",
    "gate": "res",
    "duration": 1,
    "effect": "skip_turn",
    "physical_bonus_dmg_pct": 30
  },
  "poisoned": {
    "type": "dot",
    "element": "poison",
    "gate": null,
    "duration": 3,
    "dot_dmg": 3
  }
}
```

**핵심 필드:**
- `"gate"`: `"arm"` / `"res"` / `null` — CC 발동 조건
- `"effect"`: `"skip_turn"` / `"attack_ally"` / `"random_move"` — CC 행동 치환
- `"dot_dmg"`: DoT 1턴당 HP 직접 차감량
- `"mv_penalty"`: 이동력 감소량

---

## 5. CombatScene 연동

### 초기화 (ready)
```gdscript
var _status_manager: StatusManager = StatusManager.new()
_status_manager.setup("res://data/effects/status_effects.json")
```

### 유닛 등록/해제
```gdscript
# _spawn_unit() 내부
_status_manager.register_unit(unit)

# _on_unit_died() 내부
_status_manager.unregister_unit(unit)
```

### DoT 틱 처리 (턴 시작)
```gdscript
# _on_turn_changed() 에서 호출
func _process_all_dot_ticks() -> void:
    for unit in ally_container.get_children() + enemy_container.get_children():
        var cu := unit as CombatUnit
        if cu == null or not cu.is_alive():
            continue
        for event in _status_manager.process_tick(cu):
            cu.take_damage(event["damage"])
            _add_log_entry("☠ %s [%s] %d" % [...], Color.PURPLE)
            if not cu.is_alive():
                _on_unit_died(cu)
```

### 상태이상 적용 (공격 히트 시)
```gdscript
# _handle_hit_frame() 에서 element_modifier 기반으로 호출
for status_id in el_modifier.get("apply_statuses", []):
    _status_manager.apply_status(target, status_id, attacker)
```

---

## 6. CC 행동 치환 (Tier 2 플레이스홀더)

CC 행동 치환(`skip_turn`, `attack_ally`, `random_move`)은 `_on_turn_changed` 또는 AI 루프에서 처리.
Tier 2에서는 **is_skip_turn() 체크로 턴 스킵**만 구현. 완전 행동 치환은 Tier 3.

```gdscript
# _run_enemy_ai() 또는 플레이어 턴 시작 시
if _status_manager.is_skip_turn(unit):
    _add_log_entry("💤 %s 행동 불능" % unit.unit_name, Color.GRAY)
    continue  # 해당 유닛 행동 스킵
```

---

## 7. 검증 시나리오 (디버그 모듈)

```
시나리오 1: ARM = 0 → 기절 적용 가능
  Fighter ARM 24 → 물리 공격 24+로 ARM 고갈 → stun 적용 시도
  → ARM > 0이면 BLOCKED, ARM = 0이면 OK ✅

시나리오 2: DoT 게이지 무시
  중독(poisoned) 적용 → ARM > 0이어도 매 턴 HP -3 ✅

시나리오 3: RES = 0 → 빙결 적용
  Mage 공격 → RES 고갈 → frozen 적용 → 다음 턴 행동 불능 ✅
```
