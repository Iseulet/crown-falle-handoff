# CrownFalle Passive Skill System
> 패시브 스킬 시스템 설계 문서

- Created: 2026-03-08
- Status: ✅ 구조 확정 / 직업별 스킬 내용 PENDING
- References:
  - `handoff/plans/design/2026-03-08-stat_system.md`
  - `handoff/plans/design/2026-03-08-stamina_system.md`
  - `handoff/plans/design/2026-03-08-unit_data_architecture.md`

---

## 1. 개요

패시브 스킬은 능동적으로 사용하지 않고 조건 충족 시 자동 발동.
모든 스킬은 단일 테이블로 통합 관리, 직업 정의 파일에서 사용 가능 스킬을 참조.

---

## 2. 파일 구조

```
data/skills/
  passive_skills.json          ← 모든 패시브 스킬 통합 정의

data/classes/
  class_config.json            ← 직업 정의 + 사용 가능 스킬 ID 목록
```

---

## 3. 패시브 동작 구조

모든 패시브는 **trigger + condition + effects** 세 요소로 정의.
`condition` 유무만으로 동작 방식 결정 — 별도 type 필드 없음.

```
condition: null   → 트리거 발생 시 무조건 발동
condition: {...}  → 트리거 발생 시 조건 체크 후 발동

effects: []       → 배열 구조, 복합 효과 지원
                     단일 효과도 배열 1개 원소로 표현
```

---

## 4. 발동 타이밍

| 타이밍 | 설명 |
|--------|------|
| `on_combat_start` | 전투 시작 시 1회 |
| `on_turn_start` | 매 턴 시작 시 |
| `on_attack` | 공격 시 |
| `on_hit` | 피격 시 |
| `on_engage` | 교전 관련 이벤트 시 |

### on_engage 발동 규칙
```
condition: { "engage_just_started": true }  → 교전 성립 시 1회만 발동
condition: null (on_engage 트리거)           → 교전 중인 채로 턴 시작 시마다 발동
```
> 스택 방지: 교전 성립 1회 효과는 반드시 engage_just_started 조건 사용

---

## 5. 데이터 구조

### effects 배열 원소 필드 정의

| 필드 | 타입 | 설명 |
|------|------|------|
| `target` | string | `self` / `enemy` / `ally` |
| `stat` | string | 영향받는 스탯 키 |
| `value` | number | 적용 수치 |
| `duration` | null / `"turn"` / N | null=영구, "turn"=해당 턴 한정, N=N턴 지속 |
| `chance` | null / N | null=100% 발동, N=N% 확률 발동 |

### passive_skills.json — 통합 스킬 테이블

```json
{
  "passives": [
    {
      "id": "passive_tough_body",
      "name": "강인한 체력",
      "description": "전투 시작 시 최대 STA가 증가한다.",
      "trigger": "on_combat_start",
      "condition": null,
      "effects": [
        { "target": "self", "stat": "sta_bonus", "value": 4, "duration": null, "chance": null }
      ]
    },
    {
      "id": "passive_defense_instinct",
      "name": "방어 본능",
      "description": "HP가 50% 이하일 때 피격 시 해당 턴 물리 방어가 증가한다.",
      "trigger": "on_hit",
      "condition": { "self_hp_below_percent": 50 },
      "effects": [
        { "target": "self", "stat": "arm_bonus", "value": 10, "duration": "turn", "chance": null }
      ]
    },
    {
      "id": "passive_frontline",
      "name": "전열 유지",
      "description": "교전 성립 시 상대의 근력이 감소한다.",
      "trigger": "on_engage",
      "condition": { "engage_just_started": true },
      "effects": [
        { "target": "enemy", "stat": "str", "value": -1, "duration": null, "chance": null }
      ]
    },
    {
      "id": "passive_backstab",
      "name": "암습",
      "description": "백어택 시 데미지와 치명타 확률이 증가한다.",
      "trigger": "on_attack",
      "condition": { "is_backattack": true },
      "effects": [
        { "target": "self", "stat": "dmg_bonus_percent", "value": 20, "duration": "turn", "chance": null },
        { "target": "self", "stat": "crit_bonus", "value": 5, "duration": "turn", "chance": null }
      ]
    },
    {
      "id": "passive_afterimage",
      "name": "잔상",
      "description": "피격 시 20% 확률로 데미지를 무효화한다.",
      "trigger": "on_hit",
      "condition": null,
      "effects": [
        { "target": "self", "stat": "dmg_negate", "value": 1, "duration": "turn", "chance": 20 }
      ]
    },
    {
      "id": "passive_meditation",
      "name": "명상",
      "description": "매 턴 시작 시 MP가 회복된다.",
      "trigger": "on_turn_start",
      "condition": null,
      "effects": [
        { "target": "self", "stat": "mp_regen_bonus", "value": 1, "duration": "turn", "chance": null }
      ]
    }
  ]
}
```

### class_config.json — 직업 정의에서 스킬 참조

```json
{
  "Fighter": {
    "base_stats": { "str": 8, "dex": 4, "con": 8, "int": 2, "wil": 2 },
    "mv_base": 4,
    "crit_bonus": 0,
    "passives": [
      "passive_tough_body",
      "passive_defense_instinct",
      "passive_frontline"
    ]
  },
  "Archer": {
    "base_stats": { "str": 3, "dex": 9, "con": 5, "int": 2, "wil": 3 },
    "mv_base": 2,
    "crit_bonus": 5,
    "passives": [
      "passive_keen_eye",
      "passive_ranged_focus",
      "passive_light_gear"
    ]
  },
  "Rogue": {
    "base_stats": { "str": 5, "dex": 10, "con": 4, "int": 2, "wil": 2 },
    "mv_base": 5,
    "crit_bonus": 8,
    "passives": [
      "passive_backstab",
      "passive_vital_strike",
      "passive_afterimage"
    ]
  },
  "Mage": {
    "base_stats": { "str": 2, "dex": 3, "con": 4, "int": 9, "wil": 8 },
    "mv_base": 1,
    "crit_bonus": 0,
    "passives": [
      "passive_magic_focus",
      "passive_mana_shield",
      "passive_meditation"
    ]
  }
}
```

### 주요 condition 키

| 키 | 설명 |
|----|------|
| `is_backattack: true` | 백어택 여부 |
| `attacker_engaged: true/false` | 공격자 교전 여부 |
| `target_engaged: true/false` | 피격자 교전 여부 |
| `self_hp_below_percent: N` | 자신 HP N% 이하 |
| `self_hp_above_percent: N` | 자신 HP N% 이상 |
| `engage_just_started: true` | 교전 새로 성립 시 (1회) |

---

## 6. 직업별 패시브 초안
> ⚠️ 수치 및 스킬 내용 PENDING — 확정 전 조정 가능

### Fighter
| id | 이름 | trigger | condition | effects | duration |
|----|------|---------|-----------|---------|----------|
| passive_tough_body | 강인한 체력 | on_combat_start | null | STA +4 | 영구 |
| passive_defense_instinct | 방어 본능 | on_hit | HP 50% 이하 | ARM +10% | 해당 턴 |
| passive_frontline | 전열 유지 | on_engage | 교전 성립 1회 | 상대 STR -1 | 영구 |

### Archer
| id | 이름 | trigger | condition | effects | duration |
|----|------|---------|-----------|---------|----------|
| passive_keen_eye | 예리한 눈 | on_combat_start | null | HIT +10% | 영구 |
| passive_ranged_focus | 원거리 집중 | on_attack | 자신 비교전 | 데미지 +15% | 해당 턴 |
| passive_light_gear | 경량 장비 | on_turn_start | null | STA 회복 +1 | 해당 턴 |

### Rogue
| id | 이름 | trigger | condition | effects | duration | chance |
|----|------|---------|-----------|---------|----------|--------|
| passive_backstab | 암습 | on_attack | 백어택 시 | 데미지 +20%, CRIT +5% | 해당 턴 | 100% |
| passive_vital_strike | 급소 타격 | on_attack | 교전 중인 적 공격 시 | CRIT +10% | 해당 턴 | 100% |
| passive_afterimage | 잔상 | on_hit | null | 데미지 무효 | 해당 턴 | 20% |

### Mage
| id | 이름 | trigger | condition | effects | duration |
|----|------|---------|-----------|---------|----------|
| passive_magic_focus | 마법 집중 | on_combat_start | null | INT +2 | 영구 |
| passive_mana_shield | 마력 보호막 | on_hit | null | RES +5% | 해당 턴 |
| passive_meditation | 명상 | on_turn_start | null | MP 회복 +1 | 해당 턴 |

---

## 7. Implementation Notes

```
PassiveManager.gd (신규):
  전투 시작 시:
    유닛 class_id → class_config에서 passives[] 로드
    passive_skills.json에서 id로 스킬 데이터 조회
    unit.active_passives[]에 캐싱
    unit.extra_passives[] 합산

  타이밍별 훅 함수:
    apply_on_combat_start(unit)
    apply_on_turn_start(unit)
    apply_on_attack(attacker, target, context)
    apply_on_hit(unit, attacker, context)
    apply_on_engage(unit, target, context)

  context 딕셔너리 예시:
    { "is_backattack": true, "attacker_engaged": false,
      "engage_just_started": true, "self_hp_percent": 45 }

  effects[] 처리:
    for effect in passive["effects"]:
      chance 체크 → null이면 100%, N이면 N% 확률
      stat 적용
      duration 등록 (null=영구, "turn"=턴 종료 시 제거, N=N턴 후 제거)

  duration 관리:
    턴 종료 시 duration="turn" 효과 일괄 제거
    N턴 효과는 turn_counter 감소, 0이 되면 제거

CombatScene.gd:
  각 이벤트 발생 지점에 PassiveManager 훅 호출 추가
  턴 종료 시 duration 만료 효과 일괄 정리

확장성:
  직업 외 스킬 습득 시 unit.extra_passives[]에 id append
  PassiveManager는 active_passives + extra_passives 합산 처리
```
