# passive_skill_system.md 개정안 (Delta)

- Date: 2026-03-13
- 근거: DOS2 스킬 시스템 분석 → 확정된 additive 변경사항
- 적용 대상: `handoff/plans/design/2026-03-08-passive_skill_system.md`
- 의존성: `stat_system.md` 개정안 (ARM/RES 게이지형 전환) 먼저 적용
- 상태: 사용자 승인 대기
- ⚠️ 2026-03-14 수정: 대문자 ID → 소문자 통일, `data/skills/status_effects.json` → `data/effects/status_effects.json` 경로 변경. 근거: `2026-03-14-status-data-consolidation.md`

---

## 변경 요약

| # | 항목 | 기존 | 변경 |
|---|------|------|------|
| 1 | condition 키 | 6종 | **+2종 추가** (CC 게이팅용) |
| 2 | 상태이상 | 미정의 (apply_status 이벤트 타입만 존재) | **7종 정의** + 효과/해제 규칙 |
| 3 | effects.stat 값 | 기존 스탯 키만 | **상태이상 ID 사용 가능** |

---

## 1. condition 키 추가 (섹션 5 보조)

### 기존 condition 키 (유지)

| 키 | 설명 |
|----|------|
| `is_backattack: true` | 백어택 여부 |
| `attacker_engaged: true/false` | 공격자 교전 여부 |
| `target_engaged: true/false` | 피격자 교전 여부 |
| `self_hp_below_percent: N` | 자신 HP N% 이하 |
| `self_hp_above_percent: N` | 자신 HP N% 이상 |
| `engage_just_started: true` | 교전 새로 성립 시 (1회) |

### 신규 condition 키 (+2종)

| 키 | 설명 | 용도 |
|----|------|------|
| `target_parm_depleted: true` | 대상의 ARM 게이지 == 0 | 물리 CC 적용 게이팅 |
| `target_marm_depleted: true` | 대상의 RES 게이지 == 0 | 마법 CC 적용 게이팅 |

> DOS2 영감: 상태이상(CC)은 방어 게이지가 소진된 상태에서만 적용 가능.
> ARM 있는 적에게 KNOCKDOWN(물리 CC) 시도 → 불발.
> RES 소진된 적에게 STUNNED(마법 CC) 시도 → 적용.

### CC 게이팅 원칙

```
물리 상태이상 (bleeding, knockdown, crippled, stun):
  condition: { "target_parm_depleted": true }

마법 상태이상 (burning, weakened, fear):
  condition: { "target_marm_depleted": true }

※ 일부 스킬이 게이팅 없이 상태이상을 거는 경우:
  condition에 해당 키를 넣지 않으면 됨 (JSON 레벨 제어)
```

### condition 평가 로직 수정 (PassiveManager)

```gdscript
# PassiveManager.gd — _evaluate_condition() 추가

func _evaluate_condition(condition: Dictionary, unit: TestUnitData, context_unit: TestUnitData) -> bool:
    # ... 기존 조건 평가 ...

    # 신규: CC 게이팅 조건
    if condition.has("target_parm_depleted"):
        if context_unit == null:
            return false
        if context_unit.is_parm_depleted() != condition.target_parm_depleted:
            return false

    if condition.has("target_marm_depleted"):
        if context_unit == null:
            return false
        if context_unit.is_marm_depleted() != condition.target_marm_depleted:
            return false

    return true
```

---

## 2. 상태이상 7종 정의 (신규 섹션 추가)

기존 문서에 **섹션 8. 상태이상 (Status Effects)** 을 추가한다.

### 8-1. 상태이상 분류

| 카테고리 | damage_type | CC 게이팅 | 상태이상 |
|----------|-------------|-----------|---------|
| 물리 | physical | `target_parm_depleted` | bleeding, knockdown, crippled, stun |
| 마법 | magical | `target_marm_depleted` | burning, weakened, fear |

### 8-2. 개별 상태이상 정의

```jsonc
// data/effects/status_effects.json (통합 파일 — 2026-03-14-status-data-consolidation.md 참조)
// 아래는 패시브에서 참조하는 7종만 발췌. 전체 13종은 통합 설계서 참조.
{
    "bleeding": {
        "name": "출혈",
        "type": "dot",
        "category": "physical",
        "element": "physical",
        "gate": null,
        "duration": 3,
        "dot_dmg": 2,
        "dot_damage_stat": null,
        "dot_damage_coefficient": null,
        "disable_actions": [],
        "stacks": false,
        "refresh_on_reapply": true,
        "visual": "fx_bleed_drip",
        "motion_override": null
    },
    "knockdown": {
        "name": "넉다운",
        "type": "cc",
        "category": "physical",
        "gate": "arm",
        "duration": 1,
        "effect": "skip_turn",
        "disable_actions": ["move", "attack", "skill"],
        "stacks": false,
        "refresh_on_reapply": false,
        "visual": "fx_knockdown_stars",
        "motion_override": "stun_loop"
    },
    "crippled": {
        "name": "절름발이",
        "type": "debuff",
        "category": "physical",
        "element": "physical",
        "gate": null,
        "duration": 2,
        "mv_penalty": 2,
        "disable_actions": [],
        "stacks": false,
        "refresh_on_reapply": true,
        "visual": "fx_cripple_chains",
        "motion_override": null
    },
    "burning": {
        "name": "화상",
        "type": "dot",
        "category": "magical",
        "element": "fire",
        "gate": null,
        "duration": 2,
        "dot_dmg": 4,
        "dot_damage_stat": null,
        "dot_damage_coefficient": null,
        "disable_actions": [],
        "stacks": false,
        "refresh_on_reapply": true,
        "visual": "fx_burning_flames",
        "motion_override": "burn_loop"
    },
    "stun": {
        "name": "기절",
        "type": "cc",
        "category": "magical",
        "gate": "res",
        "duration": 1,
        "effect": "skip_turn",
        "disable_actions": ["move", "attack", "skill"],
        "stacks": false,
        "refresh_on_reapply": false,
        "visual": "fx_stun_sparks",
        "motion_override": "stun_loop"
    },
    "weakened": {
        "name": "약화",
        "type": "debuff",
        "category": "magical",
        "element": "magical",
        "gate": null,
        "duration": 2,
        "mv_penalty": null,
        "debuff_stat": "dmg_multiplier",
        "debuff_value": -0.25,
        "disable_actions": [],
        "stacks": false,
        "refresh_on_reapply": true,
        "visual": "fx_weaken_aura",
        "motion_override": null
    },
    "fear": {
        "name": "공포",
        "type": "cc",
        "category": "magical",
        "gate": "res",
        "duration": 1,
        "effect": "random_move",
        "disable_actions": ["skill"],
        "stacks": false,
        "refresh_on_reapply": true,
        "visual": "fx_fear_shadow",
        "motion_override": null
    }
}
```

### 8-3. 상태이상 요약 테이블

| ID | 이름 | 분류 | 타입 | 효과 | 지속 | 행동차단 |
|----|------|------|------|------|------|---------|
| bleeding | 출혈 | 물리 | DoT | 매 턴 물리 데미지 (고정 2) | 3턴 | 없음 |
| knockdown | 넉다운 | 물리 | CC | 전체 행동 불가 | 1턴 | **전체** |
| crippled | 절름발이 | 물리 | Debuff | MV -2 | 2턴 | 없음 |
| burning | 화상 | 마법 | DoT | 매 턴 마법 데미지 (고정 4) | 2턴 | 없음 |
| stun | 기절 | 물리 | CC | 전체 행동 불가 | 1턴 | **전체** |
| weakened | 약화 | 마법 | Debuff | 데미지 -25% | 2턴 | 없음 |
| fear | 공포 | 마법 | CC | 스킬 사용 불가 | 1턴 | **스킬만** |

### 8-4. 상태이상 처리 규칙

```
적용 시:
  1. CC 게이팅 조건 확인 (condition 평가)
  2. 이미 같은 상태이상이 있고 stacks == false이면:
     - refresh_on_reapply == true: duration 리셋
     - refresh_on_reapply == false: 적용 실패 (중복 방지)
  3. motion_override가 있으면 idle 대체 루프 모션 재생

턴 시작 시:
  1. DoT 효과 발동 (dot_damage 계산 → receive_hit)
  2. duration -= 1
  3. duration == 0이면 상태이상 해제

해제 시:
  - debuff_stat 복원
  - motion_override 해제 → idle 복귀
  - visual FX 제거
```

### 8-5. DoT 데미지 계산

```
bleeding:
  dot_damage = dot_dmg (고정값 2). Tier 3 확장 시 dot_damage_stat/coefficient로 스탯연동 전환 가능.
  damage_type = "physical"
  → ARM 게이지 흡수 → 잔여 HP 적용 (stat_system.md 4-4 규칙)

burning:
  dot_damage = dot_dmg (고정값 4). Tier 3 확장 시 dot_damage_stat/coefficient로 스탯연동 전환 가능.
  damage_type = "magical"
  → RES 게이지 흡수 → 잔여 HP 적용

※ 시전자 사망 시: DoT는 적용 시점의 스탯 스냅샷 사용 (시전자 참조 끊김)
```

---

## 3. passive_skills.json 확장 — 상태이상 적용 패시브 예시

기존 패시브 구조에서 `effects[].stat`에 상태이상 ID를 사용할 수 있도록 확장.

### 신규 effect 타입: `apply_status`

```jsonc
// effects 배열에 추가 가능한 신규 구조
{
    "target": "enemy",
    "stat": "apply_status",
    "status_id": "bleeding",
    "duration_override": null,
    "chance": 50
}
```

| 필드 | 설명 |
|------|------|
| `stat: "apply_status"` | 상태이상 적용 효과임을 표시 |
| `status_id` | status_effects.json의 ID 참조 |
| `duration_override` | null이면 default_duration, 숫자면 덮어쓰기 |
| `chance` | 확률 (기존 chance 필드와 동일) |

### 예시: Fighter 패시브 — 출혈 일격

```jsonc
{
    "id": "passive_bleeding_strike",
    "name": "출혈 일격",
    "description": "공격 시 적의 ARM이 소진되어 있으면 50% 확률로 출혈을 부여한다.",
    "trigger": "on_attack",
    "condition": { "target_parm_depleted": true },
    "effects": [
        {
            "target": "enemy",
            "stat": "apply_status",
            "status_id": "bleeding",
            "duration_override": null,
            "chance": 50
        }
    ]
}
```

### 예시: Mage 패시브 — 마비 주문

```jsonc
{
    "id": "passive_paralyzing_spell",
    "name": "마비 주문",
    "description": "마법 공격 시 적의 RES가 소진되어 있으면 30% 확률로 기절을 부여한다.",
    "trigger": "on_attack",
    "condition": { "target_marm_depleted": true },
    "effects": [
        {
            "target": "enemy",
            "stat": "apply_status",
            "status_id": "stun",
            "duration_override": null,
            "chance": 30
        }
    ]
}
```

---

## 4. data 파일 추가

| 파일 | 설명 |
|------|------|
| `data/effects/status_effects.json` | 통합 파일 — 상태이상 13종 정의 (기존 11 + crippled, weakened 추가) |
| ~~`data/skills/status_effects.json`~~ | **폐기 예정** — `data/effects/status_effects.json`으로 통합 |

data-driven-design.md의 Data File Map 갱신 필요:

```
| Status effects | `data/effects/status_effects.json` | 상태이상 정의 (13종) |
```

---

## 5. Implementation Notes 추가

```
StatusManager.gd (기존 — 통합 후 확장):
  data/effects/status_effects.json 파싱/캐싱 (단일 파일)
  상태이상 적용: apply_status(target, status_id, caster, duration_override)
  턴 시작: process_tick(unit) — DoT 발동 + duration 감소 + 해제
  상태 확인: has_status(unit, status_id) → bool
  행동 차단: is_action_disabled(unit, action_type) → bool — disable_actions 필드 기반

CombatUnit.gd:
  변수 추가: active_statuses: Array[Dictionary]
    # { "status_id": "bleeding", "remaining_turns": 2, "caster_snapshot": {...} }
  함수 추가:
    add_status(status_data, caster_snapshot, duration)
    remove_status(status_id)
    has_status(status_id) → bool

CombatScene.gd:
  턴 시작 시: StatusEffectManager.process_turn_start(unit) 호출
  행동 전: StatusEffectManager.is_action_disabled(unit, action) 체크
  공격 시 패시브 apply_status 효과 처리

PassiveManager.gd:
  _apply_effect() 내 stat == "apply_status" 분기 추가
  → StatusEffectManager.apply_status() 위임
```

---

## 6. 변경하지 않는 항목 (명시)

| 항목 | 상태 |
|------|------|
| 기존 패시브 구조 (trigger/condition/effects) | 변경 없음 |
| 기존 6종 condition 키 | 변경 없음 |
| 기존 effects 필드 (target/stat/value/duration/chance) | 변경 없음 |
| 기존 직업별 패시브 (12종) | 변경 없음 (수치 PENDING 상태 유지) |
| 발동 타이밍 5종 | 변경 없음 |
| on_engage 스택 방지 규칙 | 변경 없음 |

---

## 적용 지침 (CLI용)

```
1. passive_skill_system.md 섹션 5 condition 키 테이블:
   target_parm_depleted, target_marm_depleted 2행 추가

2. passive_skill_system.md에 섹션 8 전체 추가:
   8-1 분류, 8-2 JSON 정의, 8-3 요약 테이블, 8-4 처리 규칙, 8-5 DoT 계산

3. passive_skill_system.md 섹션 5 데이터 구조:
   effects 배열에 apply_status 타입 설명 추가

4. passive_skill_system.md 섹션 7 Implementation Notes:
   StatusEffectManager.gd 신규 내용 추가

5. data/skills/status_effects.json 파일 신규 생성

6. data-driven-design.md Data File Map에 status_effects 행 추가
```
