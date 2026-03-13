# CrownFalle Proto — 현재 세션 컨텍스트
> 생성일: 2026-03-13 | 갱신 주기: 세션마다

---

## Sync Metadata
> 🇰🇷 싱크 메타데이터

| Field | Value |
|-------|-------|
| Generated | 2026-03-13 (3차 갱신) |
| Source commit | (이 커밋 직후 갱신) |
| Uncommitted changes | No |
| Sessions since last sync | 1 |

⚠️ If reading this after the generated date, information may be outdated.
> 🇰🇷 생성일 이후에 읽는 경우 정보가 오래되었을 수 있습니다.

---

## 현재 상태

| 항목 | 내용 |
|------|------|
| 현재 Phase | **Phase B — Tier 1 + Tier 2 구현 완료, 디버그 모듈 고도화 예정** |
| 세션 상태 | 🟡 Active |
| Git | `main` branch |
| 마지막 작업 | 2026-03-13 |

---

## 구현 완료 목록

| 시스템 | 상태 | 주요 파일 |
|--------|------|----------|
| T1-1 스탯 시스템 | ✅ | CombatUnit.gd |
| T1-2 ARM/RES 게이지 | ✅ | CombatUnit.gd |
| T1-3 HIT/CRIT/데미지 공식 | ✅ | CombatScene.gd |
| T1-4 STA 시스템 | ✅ | CombatUnit.gd, CombatScene.gd |
| T1-5 교전 v2 (강제이탈/기회공격) | ✅ | CombatScene.gd |
| T2-1 상태이상 CC+DoT | ✅ | StatusManager.gd, status_effects.json |
| T2-2 액티브 스킬 프레임워크 | ✅ | SkillManager.gd, active_skills.json |
| T2-3 패시브 스킬 | ✅ | PassiveManager.gd (이전 완료) |
| T2-4 원소 속성 시스템 | ✅ | ElementResolver.gd, element_table.json |
| T2-5 VP 시스템 | ✅ | CombatScene.gd |
| T2-6 조건부 이탈 | ✅ | CombatScene.gd (_can_vp_disengage) |
| 투사체 비동기 시스템 | ✅ | Projectile.gd, ProjectileManager.gd |
| AnimEvent / CameraDirector | ✅ | — |
| 환경/데코레이션 시스템 | ✅ | — |
| DebugCombatPanel (기본 골격) | ✅ | DebugCombatPanel.gd |

---

## 다음 작업

### 🔵 디버그 모듈 고도화 (다음 세션)

`DebugCombatPanel.gd`는 기본 골격(스탯 표시/VP 조작/상태이상 적용/스킬 실행) 완료.
다음 세션에서 추가 기능 개발 후 전체 T1+T2 시스템 일괄 검증 예정.

**검증 시나리오:**
1. T1-5: 교전 중 이동 → VP 이탈 vs 강제 이탈 + 기회 공격 분기
2. T2-5: 적 처치/치명타 → VP +1 로그
3. T2-6: VP=0 상태 → 강제 이탈 + 기회 공격
4. T2-1: stun 적용 → ARM=0 게이트 확인 / poisoned → 턴마다 HP 차감
5. T2-4: wet 상태 + lightning → ×1.5 + shocked
6. T2-2: skill_fireball 실행 → MP 소모 + burning 확률 적용

---

## 핵심 아키텍처 결정

### ARM/RES 게이지형 방어
- CON × 3 = ARM (물리 흡수), WIL × 3 = RES (마법 흡수)
- CC 게이트: ARM > 0 → 물리 CC 면역, RES > 0 → 마법 CC 면역

### VP 시스템
- 초기값: 파티 인원 수 (4인 = 4VP)
- 획득: 적 처치 +1, 치명타 +1
- 소모: VP 이탈 -1
- `_can_vp_disengage()`: `party_vp >= 1 and unit.is_ally`

### 교전 시스템 v2
- VP 이탈: `spend_vp(1)` → 기회 공격 없음
- 강제 이탈: `_execute_opportunity_attack()` → STA 비용 없는 물리 공격
- 기회 공격: `_resolve_attack(attacker, target, is_opportunity=true)` → 교전 재형성 없음

### 원소 시스템
- 원소는 스킬/아이템에 귀속 (캐릭터 속성 없음)
- wet+fire→×0.5+wet제거 / wet+lightning→×1.5+shocked / wet+ice→frozen / chilled+ice→frozen
- `ElementResolver.resolve()` → `{damage_multiplier, apply_statuses, remove_statuses, convert_statuses}`

### 상태이상 게이트
- 물리 CC (stun/knockdown): `current_arm > 0` 이면 차단
- 마법 CC (frozen/charm/fear): `current_res > 0` 이면 차단
- DoT (poisoned/burning/bleeding): 게이트 없음, HP 직접

### 스킬 실행 흐름
`_execute_skill()` → `can_use()` → `consume_resource()` → `start_cooldown()` → `_apply_skill_effect()`

---

## 주요 파일 경로

```
scripts/combat/
├── CombatScene.gd     ← 메인 전투 씬 (T1-5/T2-5/T2-6 로직 포함)
├── CombatUnit.gd      ← 유닛 스탯/전투 (disengaged_this_turn, current_mp 추가)
├── StatusManager.gd   ← CC+DoT 상태이상 관리 (신규)
├── SkillManager.gd    ← 스킬 정의/쿨다운/자원 관리 (신규)
└── ElementResolver.gd ← 원소 상성 해석 (신규)

scripts/test/
└── DebugCombatPanel.gd ← 디버그 UI (신규, 고도화 예정)

data/effects/
├── status_effects.json ← CC/DoT/debuff 11종
└── element_table.json  ← 원소 상성 4종

data/skills/
└── active_skills.json  ← 직업별 스킬 7종 (Fighter/Archer/Rogue/Mage)
```
