# CrownFalle Stamina System
> 스태미너 시스템 설계 문서

- Created: 2026-03-08
- Status: ✅ 확정
- References:
  - `handoff/plans/design/2026-03-08-stat_system.md`
  - `handoff/plans/design/2026-03-08-unit_data_architecture.md`
  - `data/combat_config.jsonc`

---

## 1. 개요

스태미너(STA)는 전투 중 물리적 행동의 자원.
마법 행동은 MP 소모, 물리 행동은 STA 소모로 자원 이원화.

| 자원 | 파생 공식 | 사용처 |
|------|----------|--------|
| HP   | CON × 2 | 생존 |
| STA  | sta_base + CON × sta_per_con | 물리 행동 (이동/공격/행동스킬/교전) |
| MP   | WIL × 3 | 마법 행동 (마법스킬) |
| VP   | 파티 인원 × 1 (파티 공유) | 궁극기, VP 이탈, 전투 중 부상 회복 [2026-03-13 추가] |

> CON이 HP / STA / ARM 세 가지를 담당 → "육체적 내구도" 스탯으로 확립
> [2026-03-13 변경] STA/MP 이원화 → **STA/MP/VP 삼원화** 구조로 확장.
> VP(사기, Morale)는 파티 전체 공유 자원. 초기값 파티 인원 × 1 (4인 파티 = 4VP).
> 참조: `handoff/plans/design/2026-03-13-combat-system-proposal-final.md` §3

---

## 2. 파생 공식

```
STA(최대) = sta_base + CON × sta_per_con

직업별 초기 STA:
  Fighter: 10 + 8 × 2 = STA 26
  Archer:  10 + 5 × 2 = STA 20
  Rogue:   10 + 4 × 2 = STA 18
  Mage:    10 + 4 × 2 = STA 18
```

---

## 3. 회복

```
턴 시작 처리 순서:
  1. 현재 STA += sta_regen_per_turn (5)    ← 회복 먼저
  2. 교전 중이면 STA -= sta_cost_engagement (5)  ← 교전 비용 차감
  3. 상한: 최대 STA 초과 불가

전투 후: 전량 회복 (HP와 동일)
```

> 교전 유지 비용(5) = 회복량(5) → 교전 중 STA 현상 유지
> 교전 중에도 이동/공격 시에만 STA 소모됨

---

## 4. 소모 행동

| 행동 | 소모량 | 비고 |
|------|--------|------|
| 이동 (1칸) | 1 | 이동한 칸 수만큼 누적 소모 |
| 일반 공격 | 3 | 1회 공격당 |
| 행동 스킬 사용 | 스킬별 개별 정의 | `data/skills/` 에서 관리 |
| 교전 유지 (턴당) | 5 | 턴 시작 시 회복 후 차감 |

---

## 5. STA 0 처리

```
STA == 0:
  이동 불가
  일반 공격 불가
  행동 스킬 사용 불가
  → 강제 대기 (턴 종료만 가능)

예외:
  마법 스킬은 MP 자원이므로 STA 무관 → 사용 가능

※ 교전 해제는 능동적으로 불가 (상대 사망 시 자동 해제만 존재)
```

---

## 6. 전투 계수
> `data/combat_config.jsonc` 의 `stamina` 섹션

```jsonc
"stamina": {
  // STA 최대치 = sta_base + CON × sta_per_con | 적용: 유닛 초기화 / 레벨업 시
  "sta_base": 10,
  "sta_per_con": 2,

  // 턴 시작 시 STA 회복량 (회복 후 교전 비용 차감)
  "sta_regen_per_turn": 5,

  // 이동 1칸당 STA 소모
  "sta_cost_move": 1,

  // 일반 공격 1회당 STA 소모
  "sta_cost_attack": 3,

  // 교전 유지 턴당 STA 소모 (회복 후 차감 → 회복량과 동일하여 교전 중 STA 현상 유지)
  "sta_cost_engagement": 5
}
```

---

## 7. Implementation Notes

```
UnitData.gd / CombatUnit.gd:
  변수 추가: current_sta: int
  함수 추가:
    get_max_sta() → sta_base + CON × sta_per_con
    consume_sta(amount: int) → bool  # 부족 시 false 반환, 소모 안 함
    regen_sta() → 턴 시작 시 호출 (회복 후 교전 비용 순서 내부 처리)

CombatScene.gd:
  턴 시작 시: regen_sta() 호출
  이동 전: consume_sta(칸 수 × sta_cost_move) → false면 이동 차단
  공격 전: consume_sta(sta_cost_attack) → false면 공격 차단

UI:
  스탯 패널에 STA 현재치 / 최대치 추가
  전투 로그에 STA 소모 이벤트 추가 권장
```
