# CrownFalle — Tier 1 데미지 흐름 설계서

> 작성일: 2026-03-13
> Agent: @Planner
> Status: **확정** — 구현 착수 가능
> 의존성: T1-1 (스탯 시스템), T1-2 (ARM/RES 게이지)
> 참조:
>   - `handoff/plans/design/2026-03-13-combat-system-proposal-final.md` §4
>   - `handoff/plans/design/2026-03-13-tier1-stat-full-implementation.md`

---

## 1. 설계 목표

물리/마법 공격의 전체 판정 순서를 확정하고, ARM/RES 게이지 차감 흐름을 명확히 정의.
DoT, 백어택 등 특수 케이스의 흐름도 포함.

---

## 2. 전체 데미지 흐름 순서도

### 2.1 물리 공격 순서도

```
[물리 공격 실행]
        │
        ▼
[1. HIT 판정]
  random(0~100) < 최종 HIT?
    NO  → 빗나감 (0 데미지, 이하 스킵)
    YES → 다음 단계
        │
        ▼
[2. CRIT 판정]
  random(0~100) < CRIT?
    YES → crit_multiplier 적용 플래그 설정
    NO  → 일반 데미지
        │
        ▼
[3. 백어택 판정]
  교전 중 & 후방 3칸에서 공격?
    YES → CRIT += backattack_crit_bonus (재판정 없이 확률 누적)
    NO  → 그대로
        │
        ▼
[4. 기본 데미지 계산]
  Fighter/Warrior: base_dmg = STR × str_coefficient
  Archer/Rogue:    base_dmg = DEX × dex_dmg_coefficient
  CRIT 발동 시:    base_dmg = base_dmg × (1 + crit_damage_bonus / 100)
        │
        ▼
[5. ARM 게이지 차감]
  current_arm > 0?
    YES → current_arm = max(0, current_arm - base_dmg)
          HP 변화 없음
          물리 CC 면역 유지 (ARM이 1이라도 남으면)
    NO  → HP에 직접 적용: current_hp -= max(1, base_dmg)
          물리 CC 적용 가능 (ARM = 0 상태)
        │
        ▼
[6. 사망 판정]
  current_hp <= 0?
    YES → 유닛 사망 처리
    NO  → 완료
```

### 2.2 마법 공격 순서도

```
[마법 공격 실행]
        │
        ▼
[1. HIT 판정 — 생략]
  마법 공격은 항상 명중 (HIT 판정 없음)
        │
        ▼
[2. CRIT 판정]
  (마법도 치명타 적용 여부는 스킬별 정의 — 기본값: 없음)
        │
        ▼
[3. 기본 데미지 계산]
  base_dmg = INT × int_coefficient
  (원소 보정이 있는 경우 스킬 설계서 참조 — Tier 2)
        │
        ▼
[4. RES 게이지 차감]
  current_res > 0?
    YES → current_res = max(0, current_res - base_dmg)
          HP 변화 없음
          마법 CC 면역 유지
    NO  → HP에 직접 적용: current_hp -= max(1, base_dmg)
          마법 CC 적용 가능
        │
        ▼
[5. 사망 판정]
  current_hp <= 0?
    YES → 유닛 사망 처리
    NO  → 완료
```

---

## 3. DoT (지속 피해) 흐름

DoT는 게이지를 무시하고 HP에 직접 차감한다.

```
[DoT 틱 처리 — 라운드 종료 시]
  각 DoT 상태이상 순서대로:
    current_hp -= max(1, dot_damage)
    ※ ARM/RES 게이지 차감 없음
    ※ CC 게이트 영향 없음
    ※ HIT/CRIT 판정 없음

DoT 종류 (Tier 2 상태이상 시스템):
  - 중독(Poisoned): 독 데미지, 3턴
  - 연소(Burning): 화염 데미지, 2턴
  - 출혈(Bleeding): 물리 데미지, 3턴
```

---

## 4. 백어택 보너스 적용 위치

```
백어택 적용 조건:
  - 공격자가 교전 중 (engaged_with != null)
  - 공격 위치가 교전 상대의 후방 3칸 (교전 방향 기준)

적용 방식:
  - HIT 판정 이후, CRIT 판정 이전
  - effective_crit = base_crit + backattack_crit_bonus
  - 이 effective_crit으로 CRIT 판정 수행

JSON 키: crit.backattack_crit_bonus (기본값: 20)
```

---

## 5. ARM/RES 게이지 상세 규칙

### 5.1 초과 데미지 처리

```
ARM 게이지 200, 공격 데미지 300인 경우:
  current_arm = max(0, 200 - 300) = 0
  HP 변화: 없음 (초과 100 HP 전이 안 함)

근거: DOS2 방식 — 게이지가 흡수하고 초과는 버림.
     다음 공격부터 ARM=0이므로 HP 직접 적용.
```

### 5.2 CC 게이트 판정 시점

```
CC 게이트는 데미지 적용 후 ARM/RES 현재값으로 판정.

예시:
  ARM 5 → 공격 데미지 10 → ARM = 0
  이 공격으로 기절(Stun) 시도 시:
    데미지 처리 후 current_arm = 0 → CC 발동 가능
    즉, 같은 공격에서 ARM을 0으로 만들면서 CC 발동 가능
```

### 5.3 ARM/RES 전투 후 회복

```
전투 종료 이벤트 발생 시:
  CombatUnit.restore_after_combat():
    current_arm = get_max_arm()
    current_res = get_max_res()
```

---

## 6. 전체 계수 참조

모든 수치는 `data/combat_config.jsonc`에서 읽음.

| 계산 항목 | JSON 경로 | 기본값 |
|-----------|-----------|--------|
| HIT 기본값 | `hit.base_hit_rate` | 70 |
| HIT 상한 | `hit.hit_max` | 95 |
| HIT 하한 | `hit.hit_min` | 5 |
| 레벨 편차 보정 | `hit.level_diff_coefficient` | 3 |
| STR 계수 | `damage.str_coefficient` | 1.0 |
| DEX 계수 | `damage.dex_dmg_coefficient` | 1.0 |
| INT 계수 | `damage.int_coefficient` | 1.0 |
| CRIT 데미지 보너스 | `crit.crit_damage_bonus` | 50 |
| 백어택 CRIT 보너스 | `crit.backattack_crit_bonus` | 20 |
| ARM 게이지 계수 | `armor.arm_per_con` | 3 |
| RES 게이지 계수 | `armor.res_per_wil` | 3 |

---

## 7. Implementation Notes

### 7.1 GDScript 구현 골격

```gdscript
# CombatScene.gd 또는 CombatCalculator.gd

func calculate_physical_damage(attacker: CombatUnit, target: CombatUnit) -> Dictionary:
    var result = {"hit": false, "crit": false, "backattack": false, "damage": 0, "arm_absorbed": 0}

    # 1. HIT 판정
    var hit_rate = attacker.get_hit()
    # 레벨 편차 보정 추가 (⚠️ PENDING 항목, 현재 임시값 3 사용)
    var level_diff = attacker.level - target.level
    hit_rate += level_diff * float(CombatConfig.get("hit.level_diff_coefficient"))
    hit_rate = clamp(hit_rate, float(CombatConfig.get("hit.hit_min")), float(CombatConfig.get("hit.hit_max")))

    if randf() * 100.0 >= hit_rate:
        return result  # 빗나감

    result["hit"] = true

    # 2. CRIT 판정
    var effective_crit = attacker.get_crit()

    # 3. 백어택 판정
    if attacker.engaged_with == target and _is_backattack(attacker, target):
        effective_crit += float(CombatConfig.get("crit.backattack_crit_bonus"))
        result["backattack"] = true

    if randf() * 100.0 < effective_crit:
        result["crit"] = true

    # 4. 기본 데미지 계산
    var base_dmg: float
    match attacker.unit_data.unit_class:
        "fighter": base_dmg = attacker.unit_data.str * float(CombatConfig.get("damage.str_coefficient"))
        "archer", "rogue": base_dmg = attacker.unit_data.dex * float(CombatConfig.get("damage.dex_dmg_coefficient"))
        _: base_dmg = attacker.unit_data.str * float(CombatConfig.get("damage.str_coefficient"))

    if result["crit"]:
        base_dmg *= 1.0 + float(CombatConfig.get("crit.crit_damage_bonus")) / 100.0

    var final_dmg = int(max(1, base_dmg))
    result["damage"] = final_dmg

    # 5. ARM 게이지 차감
    if target.current_arm > 0:
        var absorbed = min(target.current_arm, final_dmg)
        target.current_arm -= absorbed
        result["arm_absorbed"] = absorbed
        # HP 변화 없음
    else:
        target.current_hp -= final_dmg

    return result


func calculate_magic_damage(attacker: CombatUnit, target: CombatUnit, skill_data: Dictionary) -> Dictionary:
    var result = {"damage": 0, "res_absorbed": 0}

    # 1. HIT 없음 (마법은 항상 명중)

    # 2. 기본 데미지 계산
    var base_dmg = attacker.unit_data.int_stat * float(CombatConfig.get("damage.int_coefficient"))
    var final_dmg = int(max(1, base_dmg))
    result["damage"] = final_dmg

    # 3. RES 게이지 차감
    if target.current_res > 0:
        var absorbed = min(target.current_res, final_dmg)
        target.current_res -= absorbed
        result["res_absorbed"] = absorbed
        # HP 변화 없음
    else:
        target.current_hp -= final_dmg

    return result


func apply_dot_damage(target: CombatUnit, dmg: int) -> void:
    # DoT는 게이지 무시, HP 직접 차감
    target.current_hp -= max(1, dmg)
```

### 7.2 주의사항

```
- ARM/RES = 0일 때만 HP에 적용 — "초과 데미지 HP 전이 없음" 원칙
- CC 게이트: 데미지 적용 후 current_arm/current_res 값으로 판정
- DoT는 apply_dot_damage() 별도 함수 사용 — 게이지 차감 로직과 분리
- 마법 공격에 CRIT 적용 여부는 스킬별 정의 (Tier 2 스킬 시스템에서 처리)
- int_stat 변수명 확인 (int 예약어 충돌)
```

### 7.3 검증 기준

```
시나리오 1: Fighter(ARM=24)가 10 데미지 물리 공격 받은 경우
  current_arm: 24 → 14 (차감)
  current_hp: 변화 없음 ✅

시나리오 2: Fighter(ARM=5)가 10 데미지 물리 공격 받은 경우
  current_arm: 5 → 0 (차감)
  current_hp: 변화 없음 (초과 5 버림) ✅

시나리오 3: Fighter(ARM=0)가 10 데미지 물리 공격 받은 경우
  current_arm: 0 (변화 없음)
  current_hp: HP -= 10 ✅
  물리 CC 적용 가능 ✅

시나리오 4: DoT 5 데미지 (중독)
  current_arm: 변화 없음 ✅
  current_hp: HP -= 5 ✅
```
