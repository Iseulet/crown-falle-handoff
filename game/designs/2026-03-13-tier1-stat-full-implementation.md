# CrownFalle — Tier 1 스탯 시스템 풀구현 설계서

> 작성일: 2026-03-13
> Agent: @Planner
> Status: **확정** — 구현 착수 가능
> 의존성: 없음 (T1-1, 전체 전투 시스템의 기반)
> 참조:
>   - `handoff/plans/design/2026-03-08-stat_system.md`
>   - `handoff/plans/design/2026-03-13-combat-system-proposal-final.md` §4, §10

---

## 1. 설계 목표

5차 스탯(STR/DEX/CON/INT/WIL) → 7개 2차 스탯 파생 공식 전체를 완결된 상태로 구현.
ARM/RES를 게이지형으로 전환하여 데미지 흐름과 CC 게이트의 기반을 확립.

---

## 2. 1차 스탯 (Primary Stats)

직접 올리는 스탯. 레벨업 시 포인트 배분 대상. 총 5개.

| 스탯 | 코드 키 | 설명 | 파생 |
|------|---------|------|------|
| STR | `str` | 물리 공격력 기반 (Fighter) | 물리 데미지 계산 |
| DEX | `dex` | 물리 공격력 기반 (Archer/Rogue), 적중·이동·치명타 | HIT, MV, CRIT |
| CON | `con` | 내구도 — HP, STA, ARM 담당 | HP, STA, ARM |
| INT | `int_stat` | 마법 공격력 / 스킬 효과 | 마법 데미지 계산 |
| WIL | `wil` | MP, RES 담당 | MP, RES |

> ⚠️ GDScript 예약어 충돌: JSON의 `"int"` 키 → GDScript에서 `int_stat`으로 매핑 필수

---

## 3. 2차 스탯 (Secondary Stats)

1차 스탯 + 직업 기본값에서 자동 파생. 직접 올릴 수 없음.

### 3.1 파생 공식 전체

| 스탯 | 공식 | JSON 키 | 비고 |
|------|------|---------|------|
| HP | CON × `hp_per_con` | `stat_derivation.hp_per_con` | 전투 후 전량 회복 |
| MP | WIL × `mp_per_wil` | `stat_derivation.mp_per_wil` | 마법 스킬 자원 |
| STA | `sta_base` + CON × `sta_per_con` | `stamina.sta_base`, `stamina.sta_per_con` | 물리 행동 자원 |
| HIT | `base_hit_rate` + DEX × `dex_hit_coefficient` + 레벨 편차 × `level_diff_coefficient` | `hit.*` | %, 상한/하한 적용 |
| MV | 직업 `mv_base` + DEX × `dex_mv_coefficient`, floor() | `stat_derivation.dex_mv_coefficient` | 칸 수 (정수) |
| CRIT | DEX × `dex_crit_coefficient` + 직업 `crit_bonus` | `stat_derivation.dex_crit_coefficient` | %, 치명타 확률 |
| ARM | CON × `arm_per_con` | `armor.arm_per_con` | **게이지** (피격 시 차감) |
| RES | WIL × `res_per_wil` | `armor.res_per_wil` | **게이지** (피격 시 차감) |

### 3.2 ARM/RES 게이지형 상세

```
ARM 게이지 (물리 방어):
  - 초기값: CON × arm_per_con
  - 역할: 물리 공격이 먼저 ARM에서 차감
  - CC 게이트: ARM ≥ 1이면 물리 CC(기절/넘어짐) 면역
  - 전투 후: 전량 회복 (HP/STA와 동일)

RES 게이지 (마법 저항):
  - 초기값: WIL × res_per_wil
  - 역할: 마법 공격이 먼저 RES에서 차감
  - CC 게이트: RES ≥ 1이면 마법 CC(빙결/매혹/공포) 면역
  - 전투 후: 전량 회복
```

> ⚠️ 마법 공격은 HIT 판정 없음 — 항상 명중
> ⚠️ DoT(중독/연소/출혈)는 게이지 무시, HP 직접 차감

---

## 4. 직업별 초기 스탯 + 파생값 전체 테이블

### 4.1 직업별 기본 스탯 (`data/classes/class_config.json`)

| 직업 | STR | DEX | CON | INT | WIL | mv_base | crit_bonus |
|------|-----|-----|-----|-----|-----|---------|------------|
| Fighter | 8 | 4 | 8 | 2 | 2 | 4 | +0% |
| Archer | 3 | 9 | 5 | 2 | 3 | 2 | +5% |
| Rogue | 5 | 10 | 4 | 2 | 2 | 5 | +8% |
| Mage | 2 | 3 | 4 | 9 | 8 | 1 | +0% |

### 4.2 2차 스탯 파생값 전체 (기본 계수 기준)

> 계수: hp_per_con=2, mp_per_wil=3, dex_hit=0.5, dex_mv=0.2, dex_crit=0.5, arm_per_con=3, res_per_wil=3, sta_base=10, sta_per_con=2

| 직업 | HP | MP | STA | HIT | MV | CRIT | ARM | RES |
|------|----|----|-----|-----|----|------|-----|-----|
| Fighter | **16** | **6** | **26** | 72% | **4** | 2% | **24** | **6** |
| Archer | **10** | **9** | **20** | 74.5% | **3** | 9.5% | **15** | **9** |
| Rogue | **8** | **6** | **18** | 75% | **7** | 13% | **12** | **6** |
| Mage | **8** | **24** | **18** | 71.5% | **1** | 1.5% | **12** | **24** |

> HIT 계산 예시 (Fighter): 70 + 4 × 0.5 = 72%
> MV 계산 예시 (Rogue): floor(5 + 10 × 0.2) = floor(7) = 7
> ARM 계산 예시 (Fighter): 8 × 3 = 24
> CRIT 계산 예시 (Rogue): 10 × 0.5 + 8 = 13%

---

## 5. combat_config.jsonc 최종 구조

> `data/combat_config.jsonc` — 모든 수치는 이 파일에서 관리. 코드 하드코딩 금지.

```jsonc
{
  // ============================================================
  // CrownFalle Combat Config
  // 모든 전투 계수는 이 파일에서 관리 — 코드 내 하드코딩 금지
  // ============================================================

  // 파생 스탯 계수 — 1차 스탯 → 2차 스탯 변환 시 적용
  "stat_derivation": {
    // HP = CON × hp_per_con
    "hp_per_con": 2,
    // MP = WIL × mp_per_wil
    "mp_per_wil": 3,
    // HIT(%) += DEX × 계수
    "dex_hit_coefficient": 0.5,
    // MV(칸) = mv_base + DEX × 계수, floor()
    "dex_mv_coefficient": 0.2,
    // CRIT(%) = DEX × 계수 + 직업 보정값
    "dex_crit_coefficient": 0.5
  },

  // ARM/RES 게이지 계수 — 게이지형 방어. %형 폐기.
  "armor": {
    // ARM 게이지 = CON × arm_per_con
    "arm_per_con": 3,
    // RES 게이지 = WIL × res_per_wil
    "res_per_wil": 3
  },

  // 스태미너 계수
  "stamina": {
    // STA 최대치 = sta_base + CON × sta_per_con
    "sta_base": 10,
    "sta_per_con": 2,
    // 턴 시작 시 STA 회복량
    "sta_regen_per_turn": 5,
    // 이동 1칸당 STA 소모
    "sta_cost_move": 1,
    // 일반 공격 1회당 STA 소모
    "sta_cost_attack": 3,
    // 교전 유지 턴당 STA 소모 (회복량과 동일 → 교전 중 STA 현상 유지)
    "sta_cost_engagement": 5
  },

  // 적중률 계수
  "hit": {
    // 기본 명중률 (DEX 0 기준)
    "base_hit_rate": 70,
    // 레벨 편차 보정: (공격자 - 피격자) × 계수 | ⚠️ PENDING: 2/3/5 미결정
    "level_diff_coefficient": 3,
    // HIT 상한 | ⚠️ PENDING: 95% or 100%
    "hit_max": 95,
    // HIT 하한 | ⚠️ PENDING: 5% or 10%
    "hit_min": 5
  },

  // 데미지 계수
  "damage": {
    // 물리 데미지(근접) = STR × 계수
    "str_coefficient": 1.0,
    // 물리 데미지(원거리) = DEX × 계수
    "dex_dmg_coefficient": 1.0,
    // 마법 데미지 = INT × 계수
    "int_coefficient": 1.0
  },

  // 치명타 계수
  "crit": {
    // 치명타 데미지 보너스(%) | ⚠️ PENDING: 수치 미결정
    "crit_damage_bonus": 50,
    // 백어택 치명타 확률 추가 보너스(%)
    "backattack_crit_bonus": 20
  },

  // 레벨업 계수
  "levelup": {
    "exp_per_level": 100,
    "stat_points_per_level": 1
  }
}
```

---

## 6. DataLoader / CombatUnit 계산 흐름

### 6.1 데이터 로딩 흐름

```
게임 시작 시:
  DataLoader.gd → combat_config.jsonc 파싱
    → strip_comments(text)   // "//" 이후 제거
    → JSON.parse(stripped)
    → CombatConfig 싱글톤에 저장

  DataLoader.gd → class_config.json 파싱
    → 각 직업의 base_stats(str/dex/con/int_stat/wil), mv_base, crit_bonus 저장
```

### 6.2 CombatUnit 파생 스탯 계산 함수

```gdscript
# CombatUnit.gd

# --- ARM/RES 게이지 상태 변수 ---
var current_arm: int   # 현재 ARM 게이지
var current_res: int   # 현재 RES 게이지

# --- 2차 스탯 계산 (combat_config 참조) ---

func get_max_hp() -> int:
    return unit_data.con * int(CombatConfig.get("stat_derivation.hp_per_con"))

func get_max_mp() -> int:
    return unit_data.wil * int(CombatConfig.get("stat_derivation.mp_per_wil"))

func get_max_sta() -> int:
    var cfg = CombatConfig.get("stamina")
    return int(cfg["sta_base"]) + unit_data.con * int(cfg["sta_per_con"])

func get_hit() -> float:
    var cfg = CombatConfig.get("hit")
    return float(cfg["base_hit_rate"]) + unit_data.dex * float(CombatConfig.get("stat_derivation.dex_hit_coefficient"))

func get_mv() -> int:
    return int(floor(unit_data.mv_base + unit_data.dex * float(CombatConfig.get("stat_derivation.dex_mv_coefficient"))))

func get_crit() -> float:
    return unit_data.dex * float(CombatConfig.get("stat_derivation.dex_crit_coefficient")) + unit_data.crit_bonus

func get_max_arm() -> int:
    return unit_data.con * int(CombatConfig.get("armor.arm_per_con"))

func get_max_res() -> int:
    return unit_data.wil * int(CombatConfig.get("armor.res_per_wil"))

# --- 유닛 초기화 시 ---
func initialize_stats() -> void:
    current_hp = get_max_hp()
    current_mp = get_max_mp()
    current_sta = get_max_sta()
    current_arm = get_max_arm()    # ARM 게이지 초기화
    current_res = get_max_res()    # RES 게이지 초기화

# --- 전투 후 전량 회복 ---
func restore_after_combat() -> void:
    current_hp = get_max_hp()
    current_sta = get_max_sta()
    current_arm = get_max_arm()
    current_res = get_max_res()
```

### 6.3 MercenaryData.gd 저장 필드

```gdscript
# MercenaryData.gd (Resource)
# 저장: 1차 스탯만 (5개)
var str: int       # 근력
var dex: int       # 민첩
var con: int       # 체력
var int_stat: int  # 지능 (int 예약어 충돌 → int_stat)
var wil: int       # 의지

# 직업 기본값 (class_config.json에서 로드)
var mv_base: int
var crit_bonus: float

# 미저장 (런타임 계산):
# hp, mp, sta, hit, mv, crit, arm, res
```

---

## 7. Implementation Notes

### 7.1 구현 순서

```
1. CombatConfig 싱글톤 — combat_config.jsonc 파싱 + 쿼리 함수
2. MercenaryData.gd — int_stat 필드명 확정
3. CombatUnit.gd — 파생 스탯 계산 함수 8개 + current_arm/current_res 변수
4. CombatUnit.initialize_stats() — 전투 시작 시 호출
5. CombatUnit.restore_after_combat() — 전투 종료 후 호출
6. class_config.json — mv_base, crit_bonus 직업별 정의 확인
```

### 7.2 주의사항

```
- JSON 키 "int" → GDScript var int_stat: int 로 매핑 필수 (절대 규칙)
- JSONC 파싱: "//" 주석 제거 전처리 후 JSON.parse() 호출
- floor() 처리: MV 계산 시 반드시 적용 (정수 칸 수)
- ARM/RES는 current_arm/current_res 별도 변수로 추적 (get_max_arm()과 구분)
- 전투 후 restore_after_combat()에서 ARM/RES도 함께 회복 필수
```

### 7.3 검증 기준

```
Fighter 기준 (STR=8, DEX=4, CON=8, INT=2, WIL=2):
  HP   = 8 × 2 = 16     ✅
  MP   = 2 × 3 = 6      ✅
  STA  = 10 + 8×2 = 26  ✅
  HIT  = 70 + 4×0.5 = 72%  ✅
  MV   = floor(4 + 4×0.2) = 4  ✅
  CRIT = 4×0.5 + 0 = 2%  ✅
  ARM  = 8 × 3 = 24     ✅
  RES  = 2 × 3 = 6      ✅
```
