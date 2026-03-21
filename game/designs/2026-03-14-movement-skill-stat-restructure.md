# CrownFalle — 이동/이탈 스킬화 + 스탯 구조 개편

- Date: 2026-03-14
- Agent: @Planner
- Status: 사용자 승인 대기
- References:
  - `2026-03-13-tier1-stat-full-implementation.md` — 현행 스탯 체계
  - `2026-03-13-tier1-engagement-v2.md` — 교전 4종 이탈
  - `2026-03-13-tier2-active-skills.md` — SkillManager 구조
  - `2026-03-08-stamina_system.md` — STA 소모 체계
  - `2026-03-08-passive_skill_system.md` — 패시브 구조
  - `2026-03-13-phase-b-projectile-system.md` — 원거리/delivery 분기
  - `2026-03-14-combat-hud-skill-ui.md` — 스킬 바 8슬롯

---

## 0. 변경 목적

1. 이동/이탈을 스킬로 분류하여 스킬 바 슬롯에 매핑 — 모든 행동이 SkillManager를 경유
2. 이동 거리를 스탯 파생(MV)이 아닌 클래스별 이동 스킬 JSON range로 결정
3. ARM/RES를 1차 스탯 파생에서 분리하여 장비/패시브/버프 의존(3차 스탯)으로 전환

---

# Part 1: 이동/이탈 스킬화

## 1-1. 확정 결정사항

| 항목 | 결정 |
|------|------|
| 이동 = 스킬 | SkillManager 경유, 슬롯 1 매핑 |
| 이동 풀 | 턴당 여러 번 분할 이동 가능, 최대 거리(range)까지 |
| 최대 이동 거리 | 이동 스킬 JSON의 `range` 필드 (정수, 헥스 칸 수) |
| 이동 STA 소모 | 칸당 `sta_cost_move: 1` (현행 유지, 분할 이동도 동일) |
| 공격/스킬 | 턴당 1회 기본, 추후 횟수 증가 옵션 예정 |
| 기본 공격 = 스킬 | SkillManager 경유, 슬롯 2 매핑 |
| VP 이탈 = 스킬 | SkillManager 경유, 슬롯 8 매핑 |

## 1-2. 이동 스킬 JSON 정의

`data/skills/active_skills.json`에 추가:

```jsonc
{
  "id": "skill_move",
  "name": "이동",
  "class": "common",
  "type": "movement",
  "resource": "sta",
  "cost_per_tile": 1,
  "range": 0,
  "range_source": "class_move_range",
  "target": "tile",
  "damage": null,
  "element": null,
  "effects": [],
  "cooldown": 0,
  "animation": "walk",
  "pool_per_turn": true
}
```

> `range: 0` + `range_source` — range가 0이면 range_source 필드로 실제 값을 결정.
> `cost_per_tile: 1` — 기존 `sta_cost_move: 1`을 스킬 필드로 이전.
> `pool_per_turn: true` — 이 스킬은 턴당 사용 횟수가 아닌 잔여 거리 풀로 관리.

### 클래스별 이동 range

`data/classes/class_config.json`에서 기존 `mv_base`를 `move_range`로 **리네임**:

```jsonc
{
  "Fighter": {
    "base_stats": { "str": 8, "dex": 4, "con": 8, "int": 2, "wil": 2 },
    "move_range": 4,
    "crit_bonus": 0,
    "attack_type": "melee",
    "attack_range_max": 1,
    "attack_range_min": 1,
    "passives": ["passive_tough_body", "passive_defense_instinct", "passive_frontline"]
  },
  "Archer": {
    "base_stats": { "str": 3, "dex": 9, "con": 5, "int": 2, "wil": 3 },
    "move_range": 3,
    "crit_bonus": 5,
    "attack_type": "ranged",
    "attack_range_max": 4,
    "attack_range_min": 2,
    "passives": ["passive_keen_eye", "passive_ranged_focus", "passive_light_gear"]
  },
  "Rogue": {
    "base_stats": { "str": 5, "dex": 10, "con": 4, "int": 2, "wil": 2 },
    "move_range": 5,
    "crit_bonus": 8,
    "attack_type": "melee",
    "attack_range_max": 1,
    "attack_range_min": 1,
    "passives": ["passive_backstab", "passive_vital_strike", "passive_afterimage"]
  },
  "Mage": {
    "base_stats": { "str": 2, "dex": 3, "con": 4, "int": 9, "wil": 8 },
    "move_range": 2,
    "crit_bonus": 0,
    "attack_type": "ranged",
    "attack_range_max": 5,
    "attack_range_min": 0,
    "passives": ["passive_magic_focus", "passive_mana_shield", "passive_meditation"]
  }
}
```

> `mv_base` → `move_range` 리네임. 값은 기존과 동일.
> Mage `move_range: 2` (기존 `mv_base: 1` + DEX파생 → 이제 고정 2로 상향 조정. DEX 파생 삭제 보상).

### SkillManager range 해석 규칙

```
스킬 실행 시 range 결정:
  1. skill.range > 0 → 스킬 자체 range 사용 (예: skill_shield_bash.range = 1)
  2. skill.range == 0 → range_source 필드로 분기:
     - "class_move_range"   → class_config.move_range
     - "class_attack_range" → class_config.attack_range_max / attack_range_min
  3. range_source 미지정 + range == 0 → 에러 (잘못된 스킬 정의)
```

## 1-3. 기본 공격 스킬 JSON 정의

`data/skills/active_skills.json`에 추가:

```jsonc
{
  "id": "skill_basic_attack",
  "name": "기본 공격",
  "class": "common",
  "type": "active_physical",
  "resource": "sta",
  "cost": 3,
  "range": 0,
  "range_source": "class_attack_range",
  "target": "enemy",
  "damage": { "base_stat": "auto", "coefficient": 1.0 },
  "element": null,
  "effects": [],
  "cooldown": 0,
  "animation": "auto"
}
```

> `range: 0` + `range_source: "class_attack_range"` — class_config의 `attack_range_max`/`attack_range_min` 참조.
> `damage.base_stat: "auto"` — class_config의 `attack_type`에 따라 str(melee) / dex(ranged-projectile) / int_stat(ranged-instant) 자동 결정.
> `animation: "auto"` — class_config의 delivery에 따라 `attack_melee`/`attack_ranged`/`cast_instant` 자동 결정.
> `cost: 3` — 기존 `sta_cost_attack: 3` 이전.

## 1-4. VP 이탈 스킬 JSON 정의

`data/skills/active_skills.json`에 추가:

```jsonc
{
  "id": "skill_vp_disengage",
  "name": "VP 이탈",
  "class": "common",
  "type": "utility",
  "resource": "vp",
  "cost": 1,
  "range": 0,
  "target": "self",
  "damage": null,
  "element": null,
  "effects": [
    { "type": "disengage", "safe": true }
  ],
  "cooldown": 0,
  "animation": null,
  "condition": "self_engaged"
}
```

> `condition: "self_engaged"` — 교전 중일 때만 활성화.
> `effects.safe: true` — 기회 공격 없음.

## 1-5. 스킬 바 슬롯 매핑 (수정)

`data/ui/skill_bar_layout.json` 업데이트:

```jsonc
{
  "slot_count": 8,
  "slots": {
    "1": { "type": "common_skill", "skill_id": "skill_move", "color_group": "teal" },
    "2": { "type": "common_skill", "skill_id": "skill_basic_attack", "color_group": "red" },
    "3": { "type": "class_skill", "skill_index": 0, "color_group": "blue" },
    "4": { "type": "class_skill", "skill_index": 1, "color_group": "blue" },
    "5": { "type": "class_skill", "skill_index": 2, "color_group": "blue" },
    "6": { "type": "class_skill", "skill_index": 3, "color_group": "blue" },
    "7": { "type": "ultimate", "skill_index": 0, "color_group": "amber" },
    "8": { "type": "common_skill", "skill_id": "skill_vp_disengage", "color_group": "red" }
  },
  "dividers_after": [2, 6]
}
```

> 기존 `"action": "move"` → `"skill_id": "skill_move"` 변경.
> 슬롯 1, 2, 8 모두 스킬 ID 참조로 통일.

## 1-6. CombatUnit 변경

### 폐기 변수

```gdscript
# 삭제
var move_range: int           # → class_config.move_range에서 직접 참조
var has_moved: bool           # → move_remaining 풀로 대체
```

### 신규 변수

```gdscript
# 추가
var move_remaining: int = 0   # 턴당 남은 이동 풀 (턴 시작 시 class move_range로 리셋)
var has_attacked: bool = false # 유지 — 턴당 1회 공격 제한
```

### 턴 시작 리셋

```gdscript
func reset_for_new_turn() -> void:
    move_remaining = _class_move_range  # class_config.move_range
    has_attacked = false
    disengaged_this_turn = false
```

## 1-7. CombatScene 이동 로직 변경

### 현행 흐름 (폐기)

```
_select_unit() → get_reachable_cells(unit.grid_pos, unit.move_range)
_move_selected_unit() → unit.has_moved = true → 이동 끝
```

### 신규 흐름

```
_select_unit():
  → move_remaining > 0 AND 교전 중 아님 → 이동 하이라이트 표시
  → move_remaining > 0 AND 교전 중 → 이동 하이라이트 없음 (이동 슬롯 비활성)

_execute_move_skill(unit, target_pos):
  1. 교전 중이면 → 이동 불가 (DisengageModal로 분기)
  2. 경로 계산 → 칸 수(path_length) 산출
  3. STA 체크: unit.current_sta >= path_length × sta_cost_move(1)
  4. STA 소모: unit.consume_sta(path_length)
  5. 이동 실행: _execute_move(unit, target_pos)
  6. 풀 차감: unit.move_remaining -= path_length
  7. 유닛 선택 유지 (move_remaining > 0이면 추가 이동 가능)
  8. 하이라이트 갱신: get_reachable_cells(new_pos, move_remaining)
```

### 이동 → 공격 → 이동 흐름

```
유닛 선택 (move_remaining=4)
  → 이동 2칸 (move_remaining=2, STA -2)
  → 공격 선택 → 스킬 실행 (has_attacked=true, STA -3)
  → 교전 성립 여부 확인:
      교전 성립(⚔) → 이동 슬롯 비활성, 하이라이트 제거 (봉쇄)
      교전 미성립   → 이동 하이라이트 갱신 (move_remaining=2)
  → (교전 미성립 시) 추가 이동 1칸 (move_remaining=1, STA -1)
  → 턴 종료 (또는 추가 이동)
```

### 분할 이동 중 교전 성립 규칙

```
공격 실행 후 교전이 성립되면:
  1. move_remaining 값 자체는 유지 (소멸 아님)
  2. 이동 슬롯 비활성화 (교전 봉쇄)
  3. 이동 하이라이트 제거
  4. 이동하려면 이탈 필요 (VP/강제/스킬)
  5. 이탈 성공 시 잔여 move_remaining으로 이동 가능
```

## 1-8. 교전 이탈 처리 변경

### 현행 (독립 함수)

```
_attempt_move() → _can_vp_disengage() → _execute_vp_disengage()
                                       → _execute_forced_disengage()
```

### 신규 (스킬 경유)

```
교전 중 이동 시도:
  → DisengageModal 표시
    [VP 이탈] → _execute_skill("skill_vp_disengage") → 교전 해제 → 이동 가능 상태
    [강제 이탈] → _execute_forced_disengage() → 기회 공격 → 교전 해제 → 이동 실행

교전 중 스킬 이탈 (연막 등):
  → 스킬 바에서 직접 실행 → effects.disengage → 교전 해제
```

VP 이탈만 스킬화. 강제 이탈은 스킬이 아닌 시스템 동작(이동 시도의 결과)으로 유지.

## 1-9. GridManager 변경

함수 자체는 유지. 호출자만 변경:

```gdscript
# 변경 전
get_reachable_cells(unit.grid_pos, unit.move_range)

# 변경 후
get_reachable_cells(unit.grid_pos, unit.move_remaining)
```

## 1-10. 교전 중 이동 풀 규칙

```
교전 중:
  - move_remaining 값은 유지 (0으로 리셋하지 않음)
  - 이동 슬롯 비활성화 (교전 봉쇄)
  - 이동 하이라이트 표시 안 함

이탈 성공 후 (VP/강제/스킬):
  - 이동 슬롯 재활성화
  - 잔여 move_remaining만큼 이동 가능
  - 하이라이트 갱신: get_reachable_cells(unit.grid_pos, unit.move_remaining)

강제 이탈 후:
  - 기회 공격 → 이동 실행 (이동 칸수만큼 move_remaining 차감)
  - 잔여 move_remaining으로 추가 이동 가능 (STA 허용 시)

VP 이탈 후:
  - VP 1 소모 → 교전 해제 → 잔여 move_remaining 전부 사용 가능
```

### 시나리오 예시

```
Fighter (move_remaining=4, STA=20)
  → 이동 1칸 (move_remaining=3, STA=19)
  → 인접 적 공격 → 교전 성립(⚔)
  → move_remaining=3이지만 이동 봉쇄
  → VP 이탈 (VP -1, 교전 해제)
  → 잔여 move_remaining=3으로 후퇴 가능 (STA -3)
```

## 1-11. 적 유닛 이동 풀 적용

적 유닛도 아군과 동일한 이동 풀 규칙을 사용한다:

```
- 적 턴 시작 시: 모든 적 유닛 reset_for_new_turn() 호출
  → move_remaining = class_config.move_range
  → has_attacked = false
- 적 AI가 이동 결정 시: move_remaining과 current_sta를 확인
- 이동 실행 시: move_remaining 차감 + STA 칸당 소모
- 교전 봉쇄: 아군과 동일 (engaged_with != null → 이동 불가)
```

> 현재 적 AI는 auto-skip이므로 즉시 영향 없음.
> AI 구현 시 move_remaining 기반으로 이동 판단하면 됨.

---

# Part 2: 스탯 구조 개편

## 2-1. 변경 요약

### ARM/RES: 2차 스탯 → 3차 스탯

```
변경 전:
  ARM = CON × arm_per_con (3)    ← 1차 스탯에서 파생
  RES = WIL × res_per_wil (3)    ← 1차 스탯에서 파생

변경 후:
  ARM = 장비 ARM + 패시브 ARM + 버프 ARM   ← 3차 스탯, 1차 무관
  RES = 장비 RES + 패시브 RES + 버프 RES   ← 3차 스탯, 1차 무관
```

### MV: 2차 스탯 폐기

```
변경 전:
  MV = mv_base + floor(DEX × dex_mv_coefficient)

변경 후:
  MV 스탯 폐기 — class_config.move_range로 대체 (Part 1 참조)
```

## 2-2. 1차 스탯 역할 변경

| 스탯 | 변경 전 | 변경 후 | 삭제된 파생 |
|------|---------|---------|-----------|
| **STR** | 물리 데미지(근접) | 동일 | — |
| **DEX** | HIT, CRIT, DEX 데미지, MV | HIT, CRIT, DEX 데미지 | MV 파생 삭제 |
| **CON** | HP, STA, ARM | HP, STA | ARM 파생 삭제 |
| **INT** | 마법 데미지 | 동일 | — |
| **WIL** | MP, RES | MP | RES 파생 삭제 |

## 2-3. 2차 스탯 테이블 (개편 후)

| 2차 스탯 | 파생 공식 | 변경 |
|---------|----------|------|
| HP | CON × hp_per_con (2) | 유지 |
| STA | sta_base (10) + CON × sta_per_con (2) | 유지 |
| MP | WIL × mp_per_wil (3) | 유지 |
| HIT | base_hit_rate (70) + DEX × dex_hit_coefficient (0.5) + 레벨차 보정 | 유지 |
| CRIT | DEX × dex_crit_coefficient (0.5) + crit_bonus | 유지 |

### 삭제되는 2차 스탯

| 삭제 스탯 | 이유 | 대체 |
|----------|------|------|
| MV | 이동 스킬 range로 대체 | class_config.move_range |
| ARM | 3차 스탯으로 이전 | 장비 + 패시브 + 버프 |
| RES | 3차 스탯으로 이전 | 장비 + 패시브 + 버프 |

## 2-4. 3차 스탯 (신규 계층)

```
3차 스탯 = 장비 기본값 + 패시브 보너스 + 전투 중 버프/디버프

ARM_total = equip_arm + passive_arm_bonus + buff_arm
RES_total = equip_res + passive_res_bonus + buff_res
```

### ARM/RES 소스 우선순위

| 소스 | 적용 시점 | 예시 |
|------|----------|------|
| 장비 (equip) | 전투 시작 시 로드 | 판금갑옷 ARM +20, 로브 RES +15 |
| 패시브 (passive) | 전투 시작 시 적용 | passive_mana_shield: RES +5 |
| 버프/디버프 (buff) | 전투 중 동적 | 방패 강타로 적 ARM -5 |

### 프로토타입 처리 (장비 미구현)

장비 시스템이 아직 없으므로, 프로토타입에서는 class_config에 **임시 기본값**을 둔다:

```jsonc
// class_config.json — 임시 필드 (장비 시스템 구현 시 삭제)
{
  "Fighter": {
    "default_arm": 20,
    "default_res": 5
  },
  "Archer": {
    "default_arm": 10,
    "default_res": 8
  },
  "Rogue": {
    "default_arm": 8,
    "default_res": 5
  },
  "Mage": {
    "default_arm": 5,
    "default_res": 18
  }
}
```

> 기존 CON/WIL 파생값과 다른 수치 — 장비 기반 밸런스 시작점.
> Fighter: 높은 ARM(중갑), 낮은 RES. Mage: 낮은 ARM, 높은 RES(마법 로브).

## 2-5. 직업별 파생값 전체 테이블 (개편 후, Lv.1)

| 스탯 | Fighter | Archer | Rogue | Mage |
|------|---------|--------|-------|------|
| **1차** | | | | |
| STR | 8 | 3 | 5 | 2 |
| DEX | 4 | 9 | 10 | 3 |
| CON | 8 | 5 | 4 | 4 |
| INT | 2 | 2 | 2 | 9 |
| WIL | 2 | 3 | 2 | 8 |
| **2차 (1차 파생)** | | | | |
| HP | 16 | 10 | 8 | 8 |
| STA | 26 | 20 | 18 | 18 |
| MP | 6 | 9 | 6 | 24 |
| HIT | 72% | 74.5% | 75% | 71.5% |
| CRIT | 2% | 9.5% | 13% | 1.5% |
| **3차 (장비/패시브/버프)** | | | | |
| ARM | 20 | 10 | 8 | 5 |
| RES | 5 | 8 | 5 | 18 |
| **스킬 기반** | | | | |
| 이동 range | 4칸 | 3칸 | 5칸 | 2칸 |

> MV 행 제거. ARM/RES가 3차로 이동. 이동 range는 스킬 기반.

## 2-6. combat_config.jsonc 변경

### 삭제 키

```jsonc
// stat_derivation 섹션에서 삭제:
"dex_mv_coefficient": 0.2    // MV 스탯 폐기

// armor 섹션 전체 삭제:
"armor": {
  "arm_per_con": 3,          // ARM CON 파생 삭제
  "res_per_wil": 3           // RES WIL 파생 삭제
}
```

### 유지 키

```jsonc
{
  "stat_derivation": {
    "hp_per_con": 2,            // 유지
    "mp_per_wil": 3,            // 유지
    "dex_hit_coefficient": 0.5, // 유지
    "dex_crit_coefficient": 0.5 // 유지
    // dex_mv_coefficient 삭제
  },

  // armor 섹션 삭제

  "stamina": {
    "sta_base": 10,             // 유지
    "sta_per_con": 2,           // 유지
    "sta_regen_per_turn": 5,    // 유지
    "sta_cost_move": 1,         // 유지 (이동 스킬 cost_per_tile과 동기화)
    "sta_cost_attack": 3,       // 유지 (기본 공격 스킬 cost와 동기화)
    "sta_cost_engagement": 5    // 유지
  },

  "hit": { /* 전체 유지 */ },
  "damage": { /* 전체 유지 */ },
  "crit": { /* 전체 유지 */ },
  "levelup": { /* 아래 참조 */ }
}
```

## 2-7. levelup_config.json 변경

### 삭제 키

```jsonc
// 직업별 레벨업 선택지에서 삭제:
"mv_choice": 1    // MV 선택지 폐기
```

### 변경 후 구조

```jsonc
{
  "Fighter": {
    "stat_choices": ["str", "dex", "con", "wil"],
    "hp_choice": 2,
    "sta_choice": 2
  },
  "Archer": {
    "stat_choices": ["str", "dex", "con", "int_stat"],
    "hp_choice": 2,
    "sta_choice": 2
  },
  "Rogue": {
    "stat_choices": ["str", "dex", "con", "int_stat"],
    "hp_choice": 2,
    "sta_choice": 2
  },
  "Mage": {
    "stat_choices": ["int_stat", "wil", "dex", "con"],
    "hp_choice": 2,
    "sta_choice": 2
  }
}
```

> mv_choice 삭제. 1차 스탯에만 투자, 2차는 자동 파생.

## 2-8. CombatUnit 스탯 초기화 변경

### 변경 전

```gdscript
func _derive_secondary_stats() -> void:
    max_hp = con * hp_per_con
    max_sta = sta_base + con * sta_per_con
    max_mp = wil * mp_per_wil
    current_arm = con * arm_per_con      # ← 삭제
    max_arm = current_arm                 # ← 삭제
    current_res = wil * res_per_wil      # ← 삭제
    max_res = current_res                 # ← 삭제
    move_range = mv_base + floor(dex * dex_mv_coefficient)  # ← 삭제
    hit_rate = base_hit_rate + dex * dex_hit_coefficient
    crit_rate = dex * dex_crit_coefficient + crit_bonus
```

### 변경 후

```gdscript
func _derive_secondary_stats() -> void:
    max_hp = con * hp_per_con
    max_sta = sta_base + con * sta_per_con
    max_mp = wil * mp_per_wil
    hit_rate = base_hit_rate + dex * dex_hit_coefficient
    crit_rate = dex * dex_crit_coefficient + crit_bonus
    # ARM/RES: _apply_equipment()에서 별도 설정
    # move_range: class_config.move_range에서 직접 참조

func _apply_equipment() -> void:
    # 장비 시스템 구현 전: class_config.default_arm/default_res 사용
    max_arm = _class_default_arm
    current_arm = max_arm
    max_res = _class_default_res
    current_res = max_res

func _apply_passive_bonuses() -> void:
    # 패시브 중 ARM/RES 보너스 합산
    for passive in active_passives:
        for effect in passive.effects:
            if effect.stat == "arm":
                max_arm += effect.value
                current_arm += effect.value
            elif effect.stat == "res":
                max_res += effect.value
                current_res += effect.value
```

## 2-9. 패시브 스킬 영향

기존 패시브 중 ARM/RES 관련 항목:

| 패시브 | 현행 | 변경 |
|--------|------|------|
| `passive_defense_instinct` | on_hit, HP≤50% → ARM +10% | **ARM +N 절대값으로 변경** (% → 절대값. 장비 의존이므로 % 기반 부적절) |
| `passive_mana_shield` | on_hit → RES +5% | **RES +N 절대값으로 변경** |

> ARM/RES가 더 이상 1차 스탯 비례가 아니므로, % 보너스의 기준이 불명확.
> 절대값 보너스로 통일하는 것이 안전. 수치는 밸런스 테스트 후 결정.

---

# Part 3: 폐기/수정 파일 목록

## 3-1. combat_config.jsonc

| 키 | 조치 |
|----|------|
| `stat_derivation.dex_mv_coefficient` | **삭제** |
| `armor.arm_per_con` | **삭제** |
| `armor.res_per_wil` | **삭제** |
| `armor` 섹션 | **섹션 전체 삭제** |
| `stamina.sta_cost_move` | 유지 (이동 스킬 cost_per_tile 참조) |
| `stamina.sta_cost_attack` | 유지 (기본 공격 스킬 cost 참조) |
| 나머지 | 유지 |

## 3-2. class_config.json

| 키 | 조치 |
|----|------|
| `mv_base` | **리네임** → `move_range` (값 유지, Mage만 1→2 상향) |
| `default_arm` | **추가** (임시, 장비 시스템 전까지) |
| `default_res` | **추가** (임시, 장비 시스템 전까지) |
| `attack_type`, `attack_range_*` | 유지 (Phase B에서 추가됨) |
| `base_stats`, `crit_bonus`, `passives` | 유지 |

## 3-3. levelup_config.json

| 키 | 조치 |
|----|------|
| `mv_choice` | **삭제** |
| `stat_choices` | MV 관련 선택지 제거 (있었다면) |

## 3-4. active_skills.json

| 항목 | 조치 |
|------|------|
| `skill_move` | **추가** — 이동 스킬 정의 |
| `skill_basic_attack` | **추가** — 기본 공격 스킬 정의 |
| `skill_vp_disengage` | **추가** — VP 이탈 스킬 정의 |
| 기존 스킬 (skill_shield_bash 등) | 유지 |

## 3-5. skill_bar_layout.json

| 슬롯 | 조치 |
|------|------|
| 1 | `type: "common"` → `type: "common_skill"`, `skill_id: "skill_move"` |
| 2 | `type: "common"` → `type: "common_skill"`, `skill_id: "skill_basic_attack"` |
| 8 | `type: "vp_disengage"` → `type: "common_skill"`, `skill_id: "skill_vp_disengage"` |
| 3~7 | 유지 |

## 3-6. GDScript 파일

| 파일 | 변경 내용 |
|------|----------|
| `CombatUnit.gd` | `move_range` 삭제, `has_moved` → `move_remaining`으로 대체, `_derive_secondary_stats()` ARM/RES/MV 파생 삭제, `_apply_equipment()` 추가 |
| `CombatScene.gd` | `_attempt_move()` → `_execute_move_skill()`, VP이탈을 SkillManager 경유, `get_reachable_cells` 파라미터를 `move_remaining`으로 |
| `SkillManager.gd` | `skill_move`/`skill_basic_attack`/`skill_vp_disengage` 처리 로직, 이동 풀 관리 추가 |
| `StatusManager.gd` | 변경 없음 |
| `ElementResolver.gd` | 변경 없음 |
| `DebugCombatPanel.gd` | MV 표시 제거, move_remaining 표시 추가, ARM/RES 출처 표시 변경 |
| `UnitData.gd` | move_range 필드 제거 (class_config에서 직접 참조) |
| `DataLoader.gd` | ARM/RES 초기화 로직 변경, mv_base 참조 → move_range |

## 3-7. 설계 문서 영향

| 문서 | 영향 | 조치 |
|------|------|------|
| `stat_system.md` | MV 행 삭제, ARM/RES → 3차 이전 | 개정 필요 |
| `stamina_system.md` | sta_cost_move 유지, 이동 스킬 참조 추가 | 소폭 개정 |
| `passive_skill_system.md` | ARM/RES % 보너스 → 절대값 변경 | 개정 필요 |
| `combat-hud-skill-ui.md` | 슬롯 1/2/8이 스킬 ID 참조로 변경 | 소폭 개정 |
| `engagement-v2.md` | VP 이탈이 스킬 경유, 강제 이탈은 시스템 유지 | 소폭 개정 |
| `phase-b-projectile-system.md` | 변경 없음 (delivery 분기 유지) | — |
| `tier2-active-skills.md` | 공통 스킬 3종 추가 반영 | 소폭 개정 |
| `tier2-status-effects.md` | 변경 없음 | — |
| `tier2-element-system.md` | 변경 없음 | — |

---

# Part 4: 검증 체크리스트

- [ ] 이동 스킬: 분할 이동 (이동 → 공격 → 이동) 정상 동작
- [ ] 이동 풀: 턴 시작 시 리셋, 칸 이동마다 차감, 0이면 이동 불가
- [ ] 이동 STA: 칸당 1 STA 소모, STA 부족 시 이동 제한
- [ ] 교전 중 이동 봉쇄: move_remaining 유지, 이동 슬롯 비활성
- [ ] 이탈 후 잔여 이동: VP/강제 이탈 성공 후 move_remaining만큼 이동 가능
- [ ] 분할 이동 중 교전 성립: 공격 후 교전 성립 시 잔여 이동 봉쇄
- [ ] range 해석: range > 0은 직접 사용, range == 0은 range_source 참조
- [ ] 적 유닛 이동 풀: 적 턴 시작 시 reset_for_new_turn() 호출, move_remaining 리셋
- [ ] 기본 공격: SkillManager 경유, class_config delivery 분기 유지
- [ ] VP 이탈: 교전 중만 활성, VP 1 소모, 기회 공격 없음
- [ ] 강제 이탈: 스킬 아님, 교전 중 이동 시도 시 기회 공격 발생
- [ ] ARM: CON 파생 없음, class_config.default_arm에서 초기화
- [ ] RES: WIL 파생 없음, class_config.default_res에서 초기화
- [ ] ARM/RES 게이지 흡수 + CC 게이트: 기존 메커니즘 유지
- [ ] MV 스탯: 존재하지 않음, DEX에서 MV 파생 없음
- [ ] HIT/CRIT: DEX 파생 유지 (변경 없음)
- [ ] HP/STA: CON 파생 유지 (변경 없음)
- [ ] MP: WIL 파생 유지 (변경 없음)
- [ ] 레벨업: MV 선택지 없음, 1차 스탯만 투자
- [ ] 패시브: ARM/RES 보너스 절대값 적용
- [ ] DebugCombatPanel: move_remaining 표시, ARM/RES 출처 정확
