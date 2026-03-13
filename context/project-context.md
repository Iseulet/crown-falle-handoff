# CrownFalle Proto — 현재 세션 컨텍스트
> 생성일: 2026-03-13 | 갱신 주기: 세션마다

---

## Sync Metadata
> 🇰🇷 싱크 메타데이터

| Field | Value |
|-------|-------|
| Generated | 2026-03-13 (재갱신) |
| Source commit | bc0d5b7 |
| Uncommitted changes | No |
| Sessions since last sync | 3 (2026-03-12, 2026-03-13 x2) |

⚠️ If reading this after the generated date, information may be outdated.
> 🇰🇷 생성일 이후에 읽는 경우 정보가 오래되었을 수 있습니다.

---

## 현재 상태

| 항목 | 내용 |
|------|------|
| 현재 Phase | **Phase B — Tier 1 완료, Tier 2 구현 진행 예정** |
| 세션 상태 | 🟡 Active (T2 구현 작업 중) |
| Git | `main` branch, last commit `bc0d5b7` |
| 마지막 작업 | 2026-03-13 |

---

## 최근 세션 완료 작업

### 2026-03-13 — Tier 1 전투 시스템 구현 완료 (커밋 bc0d5b7)

**T1-1 스탯 시스템:**
- `combat_config.jsonc` — stat_derivation / armor / hit / crit / stamina 계수 전면 재구성
- `CombatUnit.initialize_stats()` — 5차 스탯 → HP/MP/STA/ARM/RES 게이지 파생
- `CombatUnit.get_hit() / get_crit() / get_mv()` — 2차 스탯 공식 구현

**T1-2 ARM/RES 게이지형 방어:**
- `take_physical_damage()` — ARM 먼저 차감 후 잔여만 HP 전이
- `take_magic_damage()` — RES 먼저 차감 후 잔여만 HP 전이
- CC 게이트: ARM ≥ 1 → 물리 CC 면역 / RES ≥ 1 → 마법 CC 면역 (T2-1에서 활용)

**T1-3 전투 공식:**
- HIT 판정: base 70 + DEX × 0.5 + 레벨 편차 × 3, 클램프 5~95%
- CRIT 판정: DEX × 0.5 + 직업보정, 백어택 +20%
- 데미지: STR/DEX/INT × 계수 (직업별), CRIT 시 1.5배
- `CombatScene._is_backattack()` 백어택 판정

**T1-4 STA 시스템:**
- `consume_sta(amount)` — 부족 시 false 반환
- `regen_sta()` — 턴 시작 +5 회복, 교전 중 -5 차감 (교전 유지 비용 현상 유지)
- `CombatScene._regen_all_sta()` — 아군/적군 전체 턴 시작 처리

**투사체 시스템 (Phase B-1):**
- `ProjectileConfig.gd` — `data/projectiles/*.json` 기반 투사체 속성 관리
- `Projectile.gd` — 비동기 비행 (공격자 즉시 idle 복귀), hit_arrived 시그널
- `ProjectileManager.gd` — spawn/clear_all, 동시 투사체 격리
- `CombatScene._handle_spawn_projectile()` + `_on_projectile_hit()` 연동

**GridManager 확장:**
- `grid_distance()` — 스태거드 오프셋 → 큐브 좌표 변환으로 정확한 거리 계산
- `get_cells_in_range()` — 원거리 공격 범위 내 셀 조회

**설계 문서:**
- `handoff/plans/design/2026-03-13-combat-system-proposal-final.md` — T1~T4 전체 로드맵 확정
- `handoff/plans/design/2026-03-13-tier1-engagement-v2.md` — 교전 v2 상세 설계 (T1-5 + T2-5 + T2-6)
- `handoff/plans/design/2026-03-13-tier1-stat-full-implementation.md` — T1-1~T1-4 구현 스펙
- `handoff/plans/design/2026-03-13-phase-b-projectile-system.md` — 투사체 설계
- `handoff/plans/design/2026-03-13-tier1-damage-flow.md` — 데미지 흐름 설계

---

## 미완료 / 다음 작업

### 🔴 T2 구현 예정 (현재 세션 진행 중)

**다음 구현 순서 (의존성 기반):**

| 순서 | 시스템 | 주요 파일 | 설계 문서 |
|------|--------|----------|----------|
| 1 | T1-5 교전 v2 (기회 공격 + 강제 이탈) | CombatScene.gd, CombatUnit.gd | tier1-engagement-v2.md |
| 2 | T2-5 VP 시스템 | CombatScene.gd | tier1-engagement-v2.md §7.3 |
| 3 | T2-6 VP 이탈 | CombatScene.gd | tier1-engagement-v2.md §5.2 |
| 4 | T2-1 상태이상 (CC+DoT) | StatusManager.gd (신규), status_effects.json | tier2-status-effects.md (미작성) |
| 5 | T2-2 액티브 스킬 | SkillManager.gd (신규), active_skills.json | tier2-active-skills.md (미작성) |
| 6 | T2-4 원소 시스템 | ElementResolver.gd (신규), element_table.json | tier2-element-system.md (미작성) |
| 7 | 디버그 모듈 | DebugCombatPanel.gd (신규) | — |

**신규 생성 예정 파일:**
- `scripts/combat/StatusManager.gd`
- `scripts/combat/SkillManager.gd`
- `scripts/combat/ElementResolver.gd`
- `scripts/test/DebugCombatPanel.gd`
- `data/effects/status_effects.json`
- `data/effects/element_table.json`
- `data/skills/active_skills.json`
- `handoff/plans/design/2026-03-13-tier2-status-effects.md`
- `handoff/plans/design/2026-03-13-tier2-active-skills.md`
- `handoff/plans/design/2026-03-13-tier2-element-system.md`

### ✅ 이미 완료된 작업

| 시스템 | 상태 |
|--------|------|
| T1-1 스탯 시스템 | ✅ 완료 |
| T1-2 ARM/RES 게이지 | ✅ 완료 |
| T1-3 전투 공식 HIT/CRIT | ✅ 완료 |
| T1-4 STA 시스템 | ✅ 완료 |
| T2-3 패시브 스킬 (PassiveManager) | ✅ 완료 |
| 투사체 비동기 시스템 | ✅ 완료 |
| AnimEvent / CameraDirector | ✅ 완료 |
| 환경/데코레이션 시스템 | ✅ 완료 |

---

## 핵심 아키텍처 결정 (T1 이후 확정)

### ARM/RES 게이지형 방어 (D1)
- **%형 폐기** → 게이지형 (CON × 3 = ARM, WIL × 3 = RES)
- 공격이 ARM/RES를 먼저 깎고, 0이 되면 HP 직접 차감
- CC 게이트: ARM > 0 → 물리 CC 면역, RES > 0 → 마법 CC 면역

### VP 초기값 (D2)
- 파티 인원 × 1 (4인 = 4VP)
- 적 처치 +1, 치명타 +1 / VP 이탈 -1 / 궁극기 -2~3

### 원소 시스템 (D5)
- 원소는 캐릭터가 아닌 **스킬/아이템에 귀속**
- Mage 적성 없음 — 자유 조합
- Wet + 냉기 → 빙결(RES 게이트), Wet + 전기 → 감전 증폭

### 구현 테스트 방식
- Godot 에디터 실행 없이 코드만 작성 (디버그 모듈로 일괄 검증 예정)
