# stat_system.md 개정안 (Delta)

- Date: 2026-03-13
- 근거: DOS2 스킬 시스템 분석 → 확정된 additive 변경사항
- 적용 대상: `handoff/plans/design/2026-03-08-stat_system.md`
- 상태: 사용자 승인 대기

---

## 변경 요약

| # | 항목 | 기존 | 변경 |
|---|------|------|------|
| 1 | ARM/RES 타입 | % 감소 방식 | **게이지형 (HP처럼 소진되는 풀)** |
| 2 | ARM/RES 공식 | `CON × 0.3%` / `WIL × 0.3%` | `CON × arm_per_con` / `WIL × res_per_wil` (정수 게이지) |
| 3 | 데미지 분기 | 물리/마법 공식이 별도 섹션 | `damage_type` 필드로 통합 분기 |
| 4 | ARM/RES 소진 상태 | 개념 없음 | `parm_depleted` / `marm_depleted` 상태 추가 |
| 5 | 데미지→HP 경로 | 항상 ARM/RES 비율 감소 후 HP 적용 | **게이지 먼저 흡수 → 잔여만 HP에 적용** |

---

## 1. 섹션 2 (2차 스탯) 변경

### 기존 (삭제)

```
| 물리방어 | ARM  | CON × `con_arm_coefficient` | % 감소 방식 |
| 마법저항 | RES  | WIL × `wil_res_coefficient` | % 감소 방식 |
```

### 변경 (대체)

```
| 물리방어 | ARM  | CON × `arm_per_con` + `arm_base` | 게이지 (정수), 물리 데미지 흡수 풀 |
| 마법저항 | RES  | WIL × `res_per_wil` + `res_base` | 게이지 (정수), 마법 데미지 흡수 풀 |
```

> ARM/RES는 **HP와 동일한 게이지 타입**. 해당 방어 게이지가 0이 되면 "소진(depleted)" 상태.
> 전투 후 전량 회복 (HP와 동일).

---

## 2. 파생 스탯 요약 (섹션 4-6) 변경

### 기존 (삭제)

```
ARM  = CON × con_arm_coefficient                    (%)
RES  = WIL × wil_res_coefficient                    (%)
```

### 변경 (대체)

```
ARM  = arm_base + CON × arm_per_con                 (정수 게이지)
RES  = res_base + WIL × res_per_wil                 (정수 게이지)
```

---

## 3. 전투 공식 변경 (섹션 4-2, 4-3 통합 재작성)

### 기존 섹션 4-2 물리 데미지 + 4-3 마법 데미지 → 삭제 후 아래로 대체

### 신규 섹션 4-2. damage_type 분기 및 데미지 적용

```
모든 공격/스킬은 damage_type 필드를 가진다:
  damage_type: "physical"  → HIT 판정 있음, ARM 게이지에 먼저 적용
  damage_type: "magical"   → HIT 판정 없음 (항상 명중), RES 게이지에 먼저 적용
```

### 신규 섹션 4-3. 기본 데미지 계산

```
damage_type == "physical":
  기본 데미지 = STR × str_coefficient       (Fighter / Warrior)
             OR DEX × dex_dmg_coefficient   (Archer / Rogue)

damage_type == "magical":
  기본 데미지 = INT × int_coefficient

※ 무기 보정치 추가 예정 (향후 무기 시스템)
※ 스킬별 damage_multiplier는 기본 데미지에 곱셈 적용
```

### 신규 섹션 4-4. 게이지 흡수 → HP 적용 흐름

```
최종 데미지 = 기본 데미지 × crit_multiplier (치명타 시)

if damage_type == "physical":
  shield = target.current_arm
else:
  shield = target.current_res

if shield > 0:
  absorbed = min(shield, 최종 데미지)
  overflow = 최종 데미지 - absorbed

  if damage_type == "physical":
    target.current_arm -= absorbed
  else:
    target.current_res -= absorbed

  hp_damage = overflow
else:
  hp_damage = 최종 데미지

target.current_hp -= max(hp_damage, 0)
최소 총 데미지: 1 (게이지 + HP 합산)
```

**핵심 변경: 기존 `(1 - ARM/100)` 비율 감소 → 게이지 직접 차감 방식으로 전환.**

### 신규 섹션 4-5. ARM/RES 소진 상태

```
소진(depleted) 조건:
  parm_depleted: target.current_arm == 0
  marm_depleted: target.current_res == 0

소진 상태의 의미:
  1. 해당 타입 데미지가 HP에 직접 적용됨 (흡수 0)
  2. CC(상태이상) 적용 가능 조건으로 사용됨
     → passive_skill_system.md의 condition 키로 참조

소진 상태 해제:
  ARM/RES가 1 이상으로 회복되면 자동 해제
  (장비/버프/스킬에 의한 게이지 회복)
```

### 기존 섹션 4-4 치명타 → 번호 조정: 4-6으로 이동

### 기존 섹션 4-5 백어택 → 번호 조정: 4-7로 이동

---

## 4. combat_config.jsonc 변경 (섹션 5)

### stat_derivation — 삭제 항목

```jsonc
// 삭제
"con_arm_coefficient": 0.3,
"wil_res_coefficient": 0.3
```

### stat_derivation — 추가 항목

```jsonc
// ARM(게이지) = arm_base + CON × arm_per_con | 적용: 유닛 초기화 / 레벨업 시
"arm_base": 0,
"arm_per_con": 1,

// RES(게이지) = res_base + WIL × res_per_wil | 적용: 유닛 초기화 / 레벨업 시
"res_base": 0,
"res_per_wil": 1
```

### damage — 변경 항목

```jsonc
// 삭제
"arm_max": 80,
"res_max": 80

// 대체 (게이지는 최대값 = 파생 공식 결과, 상한 개념 불필요)
// arm_max / res_max 삭제 — 게이지 최대치는 스탯에서 파생
```

---

## 5. 직업별 초기 ARM/RES (섹션 3 보조)

| 직업 | CON | ARM 공식 | 초기 ARM | WIL | RES 공식 | 초기 RES |
|------|-----|---------|---------|-----|---------|---------|
| Fighter | 8 | 0 + 8×1 | **8** | 2 | 0 + 2×1 | **2** |
| Archer | 5 | 0 + 5×1 | **5** | 3 | 0 + 3×1 | **3** |
| Rogue | 4 | 0 + 4×1 | **4** | 2 | 0 + 2×1 | **2** |
| Mage | 4 | 0 + 4×1 | **4** | 8 | 0 + 8×1 | **8** |

> arm_base=0, arm_per_con=1 기준 (PENDING — 밸런스 테스트 후 조정)
> 장비 시스템 도입 시 arm_base/res_base에 장비 보정 누적 예정

---

## 6. Implementation Notes 변경 (섹션 7)

### 추가

```
CombatUnit.gd:
  변수 추가: current_arm: int, current_res: int
  함수 추가:
    get_max_arm() → arm_base + CON × arm_per_con
    get_max_res() → res_base + WIL × res_per_wil
    is_parm_depleted() → current_arm == 0
    is_marm_depleted() → current_res == 0

  receive_hit() 수정:
    damage_type 파라미터 추가
    게이지 흡수 → overflow → HP 적용 순서

UI:
  HP 바 위에 ARM/RES 게이지 바 추가 (또는 HP 바 옆 별도 표시)
  ARM = 회색/은색 바, RES = 보라색/파란색 바
  소진(depleted) 시 깨진 방패 아이콘 또는 빨간 테두리
```

---

## 7. Pending Decisions 변경 (섹션 6)

### 추가 항목

| # | 항목 | 선택지 | 현재 임시값 |
|---|------|--------|------------|
| 6 | arm_per_con | 1, 2, 3 | 1 |
| 7 | res_per_wil | 1, 2, 3 | 1 |
| 8 | arm_base / res_base | 0 또는 직업별 차등 | 0 |
| 9 | ARM/RES 게이지 전투 중 회복 여부 | A) 회복 없음 B) 턴당 소량 C) 스킬만 | A |

---

## 8. 변경하지 않는 항목 (명시)

| 항목 | 상태 |
|------|------|
| 1차 스탯 5종 (STR/DEX/CON/INT/WIL) | 변경 없음 |
| HP, MP, STA, HIT, MV, CRIT 공식 | 변경 없음 |
| 직업별 초기 1차 스탯 | 변경 없음 |
| 적중 판정 (HIT) 공식 | 변경 없음 |
| 치명타 공식 | 변경 없음 (번호만 4-4→4-6) |
| 백어택 공식 | 변경 없음 (번호만 4-5→4-7) |
| 레벨업 계수 | 변경 없음 |
| class_config.json 구조 | 변경 없음 |

---

## 적용 지침 (CLI용)

```
1. stat_system.md 섹션 2: ARM/RES 행 교체 (% → 게이지)
2. stat_system.md 섹션 4-2, 4-3: 삭제 → 4-2(damage_type), 4-3(기본 데미지), 4-4(게이지 흡수), 4-5(소진 상태) 삽입
3. stat_system.md 섹션 4-4(치명타) → 4-6, 4-5(백어택) → 4-7 번호 조정
4. stat_system.md 섹션 4-6(파생 요약): ARM/RES 행 교체
5. stat_system.md 섹션 5 combat_config.jsonc: 계수 삭제/추가
6. stat_system.md 섹션 6 Pending: 항목 6~9 추가
7. stat_system.md 섹션 7 Implementation: CombatUnit 변수/함수 추가
```
