# CrownFalle — Tier 1 교전 시스템 v2 설계서

> 작성일: 2026-03-13
> Agent: @Planner
> Status: **확정** — 구현 착수 가능
> 의존성: T1-1 (스탯), T1-4 (STA 시스템)
> 이전 문서: `handoff/plans/design/2026-03-08-engagement_system.md` (v3 final)
> 참조: `handoff/plans/design/2026-03-13-combat-system-proposal-final.md` §5

---

## 1. 설계 목표

기존 교전 시스템(v3)에 **기회 공격**과 **4종 이탈** 메커니즘을 추가.
v3의 기본 구조(Engagement 형성/해제, 이동 봉쇄, 공격 대상 제한)는 변경 없음.

---

## 2. 기존 확정 사항 (v3 유지)

| 항목 | 현재 설계 | 변경 |
|------|----------|------|
| 교전 발동 | 자유로운 적에게 근접 공격 시 | 없음 |
| 이동 봉쇄 | engaged_with != null → BFS 생략 | 없음 |
| 공격 대상 제한 | 교전 상대만 | 없음 |
| 기본 해제 | 교전 상대 사망 시 자동 | 없음 |
| STA 교전 유지 비용 | 턴당 5 (회복량 5과 상쇄 → 현상 유지) | 없음 |
| 1:1 교전 제한 | 이미 교전 중인 유닛에 재교전 불가 | 없음 |
| 반격 | 없음 (스킬 시스템으로 이관) | 없음 |

---

## 3. 교전 상태 전이 다이어그램 (v2)

```
[자유 상태]
    │
    │ 자유로운 적에게 근접 공격
    ▼
[교전 상태] ◄────────────────────────────────────────────────
    │                                                         │
    │ 이탈 방법 4종:                                          │
    │                                                         │
    ├─ 강제 이탈 ────→ [기회 공격 발생] ──→ [자유 상태]      │
    │  (비용 없음)                                            │
    │                                                         │
    ├─ VP 이탈 ──────────────────────────→ [자유 상태]       │
    │  (VP 1 소모)                                            │
    │                                                         │
    ├─ 스킬 이탈 ────────────────────────→ [자유 상태]       │
    │  (스킬 비용)                                            │
    │                                                         │
    ├─ 아군 지원 이탈 ───────────────────→ [자유 상태]       │
    │  (아군 스킬 비용)                                       │
    │                                                         │
    └─ 교전 상대 사망 ───────────────────→ [자유 상태]       │
       (자동 해제)                                            │
                                                              │
         (도발 스킬 사용 시 재교전 형성 가능) ───────────────┘
```

---

## 4. 기회 공격 (Opportunity Attack)

### 4.1 발동 조건

```
강제 이탈 시에만 발생:
  - 교전 중인 유닛이 이동 명령 실행 (VP/스킬/아군 지원 이탈 아님)
  - STA >= sta_cost_move (최소 1칸 이동 가능)
```

### 4.2 처리 순서

```
1. 이탈 유닛의 이동 실행 (목적지로 이동 완료)
2. 교전 상대가 기회 공격 실행:
   - 일반 물리 공격과 동일한 공식
   - HIT 판정 적용 (빗나갈 수 있음)
   - CRIT 판정 적용
   - ARM 게이지 차감 흐름 동일
3. 교전 해제 (engaged_with = null 양방향)
4. 이탈 유닛의 이동 선택 불가 (이미 이동 완료)
```

### 4.3 기회 공격 데미지 공식

```
기회 공격은 일반 물리 공격과 동일:
  base_dmg = STR × str_coefficient  (Fighter)
           OR DEX × dex_dmg_coefficient  (Archer/Rogue)
  HIT 판정 → CRIT 판정 → ARM 게이지 차감
```

> ⚠️ 기회 공격은 STA 소모 없음 (반응 공격이므로)
> ⚠️ 기회 공격에 백어택 보너스 없음

---

## 5. 조건부 이탈 4종 상세

### 5.1 강제 이탈

```
조건:  교전 중 이동 명령 실행
비용:  없음 (STA는 이동 칸 수만큼 소모)
효과:  기회 공격 발생 후 교전 해제
UI:    "강제 이탈 — 기회 공격을 받습니다." 확인 팝업 표시 (Tier 3 UI)
```

### 5.2 VP 이탈

```
조건:  교전 중 + 파티 VP >= 1
비용:  파티 VP -= 1
효과:  기회 공격 없이 교전 해제 → 이동 가능
처리 순서:
  1. VP 이탈 선택
  2. party_vp >= 1 확인
     → 부족 시: "VP 부족 (현재 X/X)" 안내, 이탈 취소
  3. party_vp -= 1
  4. engaged_with = null (양방향)
  5. 이동 범위 재계산 표시
```

### 5.3 스킬 이탈

```
조건:  이탈 전용 스킬 보유 + 자원(STA/MP) 충분
예시:  연막(Smoke), 순간이동, 그림자 이동
처리:  스킬 효과 발동 → 교전 해제 (스킬 설계서에서 상세 정의)
비고:  Tier 2 스킬 시스템에서 구현 (현재는 플레이스홀더)
```

### 5.4 아군 지원 이탈

```
조건:  아군이 도발/밀치기 스킬 사용 가능 + 자원 충분
예시:  Fighter 도발 → 교전 대상을 Fighter로 전환
       아군 밀치기 → 교전 상대를 밀어내 거리 벌림
처리:  아군 스킬 효과 발동 → 교전 해제/전환 (스킬 설계서에서 상세 정의)
비고:  Tier 2~3 스킬 시스템에서 구현 (현재는 플레이스홀더)
```

---

## 6. STA 교전 유지 비용 (기존 확정)

```
턴 시작 처리 순서 (CombatScene — 아군 턴 시작 시):
  1. current_sta += sta_regen_per_turn (5)    ← 회복 먼저
  2. engaged_with != null?
       YES → current_sta -= sta_cost_engagement (5)   ← 교전 비용 차감
  3. 상한: max_sta 초과 불가

결과:
  - 교전 중: 회복(+5) - 비용(-5) = 현상 유지
  - 교전 없음: 회복(+5)만
  - 교전 중 STA=0: 이동/공격 불가. 강제 이탈도 STA 필요.

JSON 키: stamina.sta_regen_per_turn, stamina.sta_cost_engagement
```

---

## 7. Implementation Notes

### 7.1 CombatUnit.gd 추가 변수

```gdscript
# 기존 유지
var engaged_with: CombatUnit = null

# v2 신규: VP 이탈 후 이 턴에 이탈했는지 추적 (중복 이탈 방지)
var disengaged_this_turn: bool = false
```

### 7.2 CombatScene.gd — 이탈 분기 처리

```gdscript
# _select_unit() 에서 이동 명령 처리 시

func _attempt_move(unit: CombatUnit, target_pos: Vector2i) -> void:
    if unit.engaged_with == null:
        # 자유 상태 — 일반 이동
        _execute_move(unit, target_pos)
        return

    # 교전 중 이동 시도 — 이탈 종류 판정
    if _can_vp_disengage(unit):
        # VP 이탈 가능 → UI에서 선택지 제시 (Tier 3 UI 전까지는 자동 VP 이탈)
        _execute_vp_disengage(unit, target_pos)
    else:
        # 강제 이탈 → 기회 공격 발생
        _execute_forced_disengage(unit, target_pos)


func _can_vp_disengage(unit: CombatUnit) -> bool:
    return CombatState.party_vp >= 1  # CombatState 싱글톤 or 씬 변수


func _execute_vp_disengage(unit: CombatUnit, target_pos: Vector2i) -> void:
    CombatState.party_vp -= 1
    var partner = unit.engaged_with
    unit.engaged_with = null
    partner.engaged_with = null
    unit.disengaged_this_turn = true
    _execute_move(unit, target_pos)
    _refresh_all_engagement()


func _execute_forced_disengage(unit: CombatUnit, target_pos: Vector2i) -> void:
    var attacker = unit.engaged_with  # 기회 공격자
    # 1. 이동 실행
    _execute_move(unit, target_pos)
    # 2. 교전 해제
    var partner = attacker
    unit.engaged_with = null
    partner.engaged_with = null
    unit.disengaged_this_turn = true
    # 3. 기회 공격 실행
    _execute_opportunity_attack(attacker, unit)
    _refresh_all_engagement()


func _execute_opportunity_attack(attacker: CombatUnit, target: CombatUnit) -> void:
    # 일반 물리 공격과 동일 (STA 소모 없음)
    var result = calculate_physical_damage(attacker, target)
    if result["hit"]:
        # 데미지 적용 (calculate_physical_damage 내부에서 처리됨)
        _show_damage_number(target, result["damage"], result["crit"])
    else:
        _show_miss(target)
    if target.current_hp <= 0:
        _on_unit_died(target)
```

### 7.3 파티 VP 관리

```gdscript
# CombatState.gd (싱글톤 또는 CombatScene 변수)

var party_vp: int = 0

func initialize_vp(party_size: int) -> void:
    # VP 초기값: 파티 인원 × 1
    party_vp = party_size

func gain_vp(amount: int) -> void:
    party_vp += amount
    # 상한 없음 (현재 설계)

func spend_vp(amount: int) -> bool:
    if party_vp < amount:
        return false
    party_vp -= amount
    return true

func reset_after_combat() -> void:
    party_vp = 0  # 전투 후 잔여 VP 소멸 (캠프 휴식으로 복구)
```

### 7.4 구현 우선순위

```
Tier 1 (지금 구현):
  1. 강제 이탈 + 기회 공격
     - _execute_forced_disengage()
     - _execute_opportunity_attack()
  2. 교전 유지 STA 비용 확인 (기존 STA 시스템 연동)

Tier 2 (VP 시스템 구현 후):
  3. VP 이탈
     - CombatState.party_vp 관리
     - _execute_vp_disengage()

Tier 2~3 (스킬 시스템 구현 후):
  4. 스킬 이탈 (연막, 순간이동)
  5. 아군 지원 이탈 (도발, 밀치기)
```

### 7.5 검증 기준

```
시나리오 1: 강제 이탈
  Fighter(ARM=24, HP=16)가 교전 중 이동 → 적이 기회 공격
  → 기회 공격 HIT 판정 후 ARM 차감 또는 빗나감
  → 이동 완료, 교전 해제 ✅

시나리오 2: VP 이탈
  party_vp=2, 교전 중 VP 이탈 시도
  → party_vp=1, 기회 공격 없음, 이동 가능 ✅

시나리오 3: VP 이탈 불가
  party_vp=0, VP 이탈 시도
  → "VP 부족" 안내, 강제 이탈로 전환 ✅

시나리오 4: 교전 상대 사망 후 자동 해제
  engaged_with 사망 → _on_unit_died() → 파트너 engaged_with = null ✅
```
