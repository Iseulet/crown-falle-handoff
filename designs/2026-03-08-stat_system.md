# CrownFalle Stat System
> 크라운폴 스탯 체계 설계 문서

- Created: 2026-03-08
- Status: ✅ 확정 (미결정 항목 제외)
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-08-engagement_system.md`
  - `data/combat_config.jsonc`
  - `data/classes/class_config.json`

---

## 1. Primary Stats (1차 스탯)
> 직접 올리는 스탯 — 레벨업 시 포인트 배분 대상, 총 5개

| 스탯 | 영문 | 설명 | 파생 2차 스탯 |
|------|------|------|--------------|
| 근력 | STR (Strength)     | 물리 공격력 기반 (Fighter/Warrior)                     | -             |
| 민첩 | DEX (Dexterity)    | 물리 공격력 기반 (Archer/Rogue), 적중·이동·치명타 보정 | HIT, MV, CRIT |
| 체력 | CON (Constitution) | HP 및 물리 방어 결정                                   | HP, ARM       |
| 지능 | INT (Intelligence) | 마법 공격력 / 스킬 효과 수치                           | -             |
| 의지 | WIL (Willpower)    | MP 및 마법 저항 결정                                   | MP, RES       |

---

## 2. Secondary Stats (2차 스탯)
> 1차 스탯 + 직업 기본값에서 자동 파생 — 직접 올릴 수 없음, 총 7개

| 스탯     | 영문 | 파생 공식 | 비고 |
|----------|------|----------|------|
| 체력     | HP   | CON × `hp_per_con` | 전투 후 전량 회복 |
| 마나     | MP   | WIL × `mp_per_wil` | 스킬 사용 자원 |
| 적중률   | HIT  | `base_hit_rate` + DEX × `dex_hit_coefficient` + 레벨 편차 × `level_diff_coefficient` | %, 상한/하한 적용 |
| 이동     | MV   | 직업 기본값 + DEX × `dex_mv_coefficient`, floor() 처리 | 칸 수 (정수) |
| 치명타   | CRIT | DEX × `dex_crit_coefficient` + 직업 보정값 | % |
| 물리방어 | ARM  | CON × `con_arm_coefficient` | % 감소 방식 |
| 마법저항 | RES  | WIL × `wil_res_coefficient` | % 감소 방식 |

> 마법 공격은 HIT 판정 없음 — 항상 명중
> ARM/RES는 향후 장비 아이템 누적 적용 예정

---

## 3. 직업별 초기 스탯 ✅ 확정
> `data/classes/class_config.json` 에서 관리

| 직업    | STR | DEX | CON | INT | WIL | MV 기본값 | CRIT 보정 | 최종 MV |
|---------|-----|-----|-----|-----|-----|----------|----------|--------|
| Fighter | 8   | 4   | 8   | 2   | 2   | 4        | +0%      | **4**  |
| Archer  | 3   | 9   | 5   | 2   | 3   | 2        | +5%      | **3**  |
| Rogue   | 5   | 10  | 4   | 2   | 2   | 5        | +8%      | **7**  |
| Mage    | 2   | 3   | 4   | 9   | 8   | 1        | +0%      | **1**  |

**이동 속도 서열**: Rogue(7) > Fighter(4) > Archer(3) > Mage(1)

> [DECISION] 직업 기본 스탯은 최솟값 기준
> [DECISION] 실질 공격력 차별화는 무기 보정치로 구현 예정 (향후 무기 시스템)

---

## 4. Combat Formulas (전투 공식)

### 4-1. 적중 판정
```
최종 HIT(%) = base_hit_rate
            + DEX × dex_hit_coefficient
            + (공격자 레벨 - 피격자 레벨) × level_diff_coefficient

상한: hit_max 초과 불가
하한: hit_min 미만 불가

판정: random(0~100) < 최종 HIT  →  명중
     random(0~100) >= 최종 HIT  →  빗나감 (0 데미지)

마법 공격: HIT 판정 없음 → 항상 명중
```

### 4-2. 물리 데미지
```
기본 데미지 = STR × str_coefficient       (Fighter / Warrior)
           OR DEX × dex_dmg_coefficient   (Archer / Rogue)

최종 데미지 = 기본 데미지 × (1 - ARM / 100)
최소 데미지: 1

※ 무기 보정치 추가 예정 (향후 무기 시스템)
```

### 4-3. 마법 데미지
```
기본 데미지 = INT × int_coefficient

최종 데미지 = 기본 데미지 × (1 - RES / 100)
최소 데미지: 1
```

### 4-4. 치명타
```
치명타 확률(%) = CRIT  (2차 스탯 그대로 사용)

치명타 발동 시:
최종 데미지 = 최종 데미지 × (1 + crit_damage_bonus / 100)
```

### 4-5. 백어택
```
발동 조건:
- 교전(Engagement) 중인 적
- 교전 방향 기준 후방 3칸에서 공격 시

효과:
치명타 확률 += backattack_crit_bonus(%)
```

### 4-6. 파생 스탯 요약
```
HP   = CON × hp_per_con
MP   = WIL × mp_per_wil
HIT  = base_hit_rate + DEX × dex_hit_coefficient  (+ 레벨 편차 보정)
MV   = 직업 기본값 + DEX × dex_mv_coefficient      → floor() 처리
CRIT = DEX × dex_crit_coefficient + 직업 보정값    (%)
ARM  = CON × con_arm_coefficient                    (%)
RES  = WIL × wil_res_coefficient                    (%)
```

---

## 5. Combat Coefficients (전투 계수)
> `data/combat_config.jsonc` 에서 관리 — 코드 내 하드코딩 금지
> Godot 파싱 시 `//` 주석 제거 전처리 필요

```jsonc
{
  // ============================================================
  // CrownFalle Combat Config
  // 모든 전투 계수는 이 파일에서 관리 — 코드 내 하드코딩 금지
  // ============================================================

  // 파생 스탯 계수 — 1차 스탯 → 2차 스탯 변환 시 적용
  "stat_derivation": {
    // HP = CON × hp_per_con | 적용: 유닛 초기화 / 레벨업 시
    "hp_per_con": 2,
    // MP = WIL × mp_per_wil | 적용: 유닛 초기화 / 레벨업 시
    "mp_per_wil": 3,
    // HIT(%) += DEX × 계수 | 적용: 물리 공격 명중 판정 시 (매 공격마다)
    "dex_hit_coefficient": 0.5,
    // MV(칸) = 직업 기본값 + DEX × 계수, floor() | 적용: 턴 시작 시 이동 범위 계산
    "dex_mv_coefficient": 0.2,
    // CRIT(%) = DEX × 계수 + 직업 보정값 | 적용: 치명타 확률 계산 시 (매 공격마다)
    "dex_crit_coefficient": 0.5,
    // ARM(%) = CON × 계수 | 적용: 물리 피격 시 데미지 감소 계산
    "con_arm_coefficient": 0.3,
    // RES(%) = WIL × 계수 | 적용: 마법 피격 시 데미지 감소 계산
    "wil_res_coefficient": 0.3
  },

  // 적중률 계수 — 물리 공격 명중 판정 시 적용 (마법 공격은 항상 명중)
  "hit": {
    // 모든 유닛의 기본 명중률 (DEX 0 기준)
    "base_hit_rate": 70,
    // 레벨 편차 보정: (공격자 레벨 - 피격자 레벨) × 계수 | ⚠️ PENDING: 2/3/5 미결정
    "level_diff_coefficient": 3,
    // 최종 HIT 상한 — 초과 불가 (필중 방지) | ⚠️ PENDING: 95% or 100% 미결정
    "hit_max": 95,
    // 최종 HIT 하한 — 미만 불가 (완전 회피 방지) | ⚠️ PENDING: 5% or 10% 미결정
    "hit_min": 5
  },

  // 데미지 계수 — 각 공격 유형의 기본 데미지 계산 시 적용
  "damage": {
    // 물리 데미지(근접) = STR × 계수 | 적용: Fighter / Warrior 공격 시
    "str_coefficient": 1.0,
    // 물리 데미지(원거리) = DEX × 계수 | 적용: Archer / Rogue 공격 시
    "dex_dmg_coefficient": 1.0,
    // 마법 데미지 = INT × 계수 | 적용: Mage 스킬/공격 시
    "int_coefficient": 1.0,
    // ARM 상한 — 초과 불가 (물리 데미지 완전 무효화 방지)
    "arm_max": 80,
    // RES 상한 — 초과 불가 (마법 데미지 완전 무효화 방지)
    "res_max": 80
  },

  // 치명타 계수 — 명중 판정 이후, 치명타 발동 및 데미지 계산 시 적용
  "crit": {
    // 치명타 데미지 보너스(%) | 최종 데미지 × (1 + 계수/100) | ⚠️ PENDING: 수치 미결정
    "crit_damage_bonus": 50,
    // 백어택 치명타 확률 추가 보너스(%) | 적용: 교전 중인 적의 후방 3칸에서 공격 시
    "backattack_crit_bonus": 20
  },

  // 레벨업 계수 — 전투 후 경험치 획득 및 레벨업 처리 시 적용
  "levelup": {
    // 레벨업에 필요한 경험치량
    "exp_per_level": 100,
    // 레벨업 시 획득하는 1차 스탯 포인트 수
    "stat_points_per_level": 1
  }
}
```

---

## 6. Pending Decisions (미결정 항목)

| # | 항목 | 선택지 | 현재 임시값 |
|---|------|--------|------------|
| 1 | level_diff_coefficient | 1당 +/-2%, 3%, 5% | 3 |
| 2 | hit_max / hit_min | 상한 95% or 100% / 하한 5% or 10% | 95 / 5 |
| 3 | MP 회복 방식 | A) 턴마다 자동회복  B) 전투 후 전량 회복  C) 아이템/스킬만 | 미정 |
| 4 | crit_damage_bonus | 수치 조정 필요 | 50% |
| 5 | dex_mv_coefficient | DEX 5당 1칸 증가(0.2) 적절한지 | 0.2 |

---

## 7. Implementation Notes

```
MercenaryData.gd:
  저장: str, dex, con, int, wil  (1차 스탯만)
  미저장: hp, mp, hit, mv, crit, arm, res  (런타임 계산)

CombatUnit.gd:
  get_hp(), get_mp(), get_hit(), get_mv(), get_crit(), get_arm(), get_res()
  → combat_config.jsonc 계수 기반 계산 함수 추가 필요

DataLoader.gd:
  .jsonc 파싱 시 // 주석 전처리 필요
  func strip_comments(text: String) -> String:
      줄 단위로 // 이후 제거 후 JSON 파싱

data/classes/class_config.json:
  직업별 base_stats (1차 스탯 5개), mv_base, crit_bonus 정의 완료
```
