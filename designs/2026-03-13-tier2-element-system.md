# CrownFalle — Tier 2 원소 속성 시스템 설계서 (T2-4)

> 작성일: 2026-03-13
> Agent: @Planner
> Status: **확정** — 구현 착수 가능
> 의존성: T2-1 (StatusManager), T2-2 (SkillManager)
> 참조: `2026-03-13-combat-system-proposal-final.md` §7

---

## 1. 설계 목표

스킬에 원소 속성을 부여하고, 기존 상태(Wet/Chilled)와 교차하여 단계적 CC 전환을 구현.
DOS2의 원소 상성 시스템을 단순화한 버전.

---

## 2. 원소 종류 (Tier 2 구현 대상)

| 원소 | 레이어 | 기본 효과 | 상태 전이 |
|------|--------|----------|----------|
| `fire` | 스킬 | 데미지 | → 연소(burning) DoT |
| `lightning` | 스킬 | 데미지 | → 감전(shocked) 디버프 |
| `water` | 스킬 | 데미지 | → 젖음(wet) 디버프 |
| `ice` | 스킬(냉기) | 데미지 | → 냉기(chilled) 디버프 |
| `poison` | 장비 | DoT | → 중독(poisoned) |
| `dark` | 지형 | 시야 제한 | — (Tier 3) |

---

## 3. 원소 상성 규칙

### 3.1 상성 테이블

```
Wet(젖음) + Fire(화염) → 데미지 × 0.5, Wet 제거
Wet(젖음) + Lightning(전기) → 데미지 × 1.5, Shocked 적용
Wet(젖음) + Ice(냉기) → Wet 제거, Frozen 전환 [RES 게이트]
Chilled(냉기) + Ice(냉기) → Chilled 제거, Frozen 전환 [RES 게이트]
```

### 3.2 처리 흐름

```
공격자가 원소 스킬 사용
→ ElementResolver.resolve(target, element, status_manager)
  → 상성 규칙 테이블에서 매칭
  → 결과 반환: { damage_multiplier, apply_statuses, remove_statuses, convert_statuses }
→ CombatScene에서 결과 적용:
  1. damage_multiplier 적용 (이미 계산된 final_dmg_int에 곱셈)
  2. remove_statuses: 해당 상태 제거
  3. apply_statuses: 새 상태 적용 (게이트 없이)
  4. convert_statuses: from 제거 → to 적용 (게이트 있음)
```

---

## 4. ElementResolver 구조

### 파일 위치
`scripts/combat/ElementResolver.gd`

### 핵심 API

```gdscript
class_name ElementResolver
extends RefCounted

func setup(data_path: String) -> void

# 원소 상호작용을 해석하여 수정자 반환
func resolve(target: CombatUnit, incoming_element: String,
        status_manager: StatusManager) -> Dictionary
# 반환:
# {
#   "damage_multiplier": float,
#   "apply_statuses": Array[String],
#   "remove_statuses": Array[String],
#   "convert_statuses": Array[{from, to, gate}]
# }
```

---

## 5. element_table.json 구조

```json
{
  "interactions": {
    "wet_fire": {
      "condition": { "status_on_target": "wet", "incoming_element": "fire" },
      "effect": "damage_multiplier",
      "multiplier": 0.5,
      "removes_status": "wet"
    },
    "wet_lightning": {
      "condition": { "status_on_target": "wet", "incoming_element": "lightning" },
      "effect": "damage_multiplier",
      "multiplier": 1.5,
      "apply_status": "shocked"
    },
    "wet_ice": {
      "condition": { "status_on_target": "wet", "incoming_element": "ice" },
      "effect": "convert_status",
      "from": "wet",
      "to": "frozen",
      "gate": "res"
    },
    "chilled_ice": {
      "condition": { "status_on_target": "chilled", "incoming_element": "ice" },
      "effect": "convert_status",
      "from": "chilled",
      "to": "frozen",
      "gate": "res"
    }
  }
}
```

---

## 6. CombatScene 연동

### 초기화
```gdscript
var _element_resolver: ElementResolver = ElementResolver.new()
_element_resolver.setup("res://data/effects/element_table.json")
```

### _resolve_attack() 원소 적용
```gdscript
# 데미지 계산 후:
var el_modifier: Dictionary = {}
if not element.is_empty():
    el_modifier = _element_resolver.resolve(target, element, _status_manager)
    final_dmg_int = max(1, int(float(final_dmg_int) * el_modifier.get("damage_multiplier", 1.0)))

# dispatch context에 저장
_dispatch_atk_ctx["element"] = element
_dispatch_atk_ctx["element_modifier"] = el_modifier
```

### _handle_hit_frame() 원소 상태 적용
```gdscript
func _apply_element_effects(target: CombatUnit, el_modifier: Dictionary,
        source: CombatUnit) -> void:
    for s in el_modifier.get("remove_statuses", []):
        _status_manager.remove_status(target, s)
    for s in el_modifier.get("apply_statuses", []):
        _status_manager.apply_status(target, s, source)
    for conv in el_modifier.get("convert_statuses", []):
        if _status_manager.has_status(target, conv["from"]):
            _status_manager.remove_status(target, conv["from"])
            _status_manager.apply_status(target, conv["to"], source)
            # apply_status 내부에서 gate 체크 (RES = 0 필요)
```

---

## 7. 검증 시나리오 (디버그 모듈)

```
시나리오 1: Wet + Fire → 반감
  water_bolt → wet 적용 → fireball 공격
  → 데미지 × 0.5 확인, wet 제거 확인 ✅

시나리오 2: Wet + Lightning → 증폭
  water_bolt → wet 적용 → lightning_bolt 공격
  → 데미지 × 1.5 확인, shocked 추가 확인 ✅

시나리오 3: Wet + Ice → 빙결 전환
  water_bolt → wet 적용 → ice 스킬 공격 (RES = 0 선행 필요)
  → wet 제거, frozen CC 적용 확인 ✅
```
