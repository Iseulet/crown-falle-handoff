# CrownFalle Proto — 프로젝트 컨텍스트
> 생성일: 2026-03-15 | 갱신: 2026-03-15 세션2 | 세션: Claude Code

---

## 현재 상태

| 항목 | 내용 |
|------|------|
| Phase | Phase A-2 완료 / Tier 1+2 전투 시스템 완료 / 전투 HUD UI-1~21 완료 |
| 상태 | 이동 스킬화 + 스탯 구조 개편 완료 — Godot 런타임 테스트 대기 |
| 마지막 작업 | [기능추가] move_remaining / ARM·RES 분리 커밋 (369066a) |

---

## 2026-03-15 세션2 작업 내용

### 이동 스킬화 + 스탯 구조 개편

**변경 목적 3가지:**
1. `has_moved` 불리언 → `move_remaining` 풀 전환 (분할 이동 지원)
2. ARM/RES를 CON/WIL 1차 파생에서 분리 → `class_config.default_arm/default_res` 고정값
3. MV 스탯 폐기 → `class_config.move_range` 직접 사용

**JSON 변경 (4개)**
| 파일 | 변경 |
|------|------|
| `class_config.json` | `mv_base`→`move_range`, Mage 1→2, `default_arm/default_res` 추가 |
| `active_skills.json` | `skill_move`, `skill_basic_attack`, `skill_vp_disengage` 3개 추가 |
| `skill_bar_layout.json` | 슬롯 0/1/7에 `skill_id` 연결, type `"common_skill"` |
| `combat_config.jsonc` | `dex_mv_coefficient` 삭제, `armor` 섹션 전체 삭제 |

**GDScript 변경 (7개)**
| 파일 | 변경 |
|------|------|
| `CombatUnit.gd` | `has_moved`/`move_range` 삭제, `move_remaining` 추가, ARM/RES→class_cfg, `get_max_move_range()`, `reset_for_new_turn()` |
| `CombatScene.gd` | 이동 로직 전환, post-move highlight 재활성, 음수 방지, VP이탈 중복 제거 |
| `SkillBar.gd` | 이동 슬롯 `move_remaining`/교전 체크, VP이탈 슬롯 교전 조건 |
| `UnitStatusPanel.gd` | 이동 뱃지 `move_remaining` 표시 |
| `DebugCombatPanel.gd` | `MV:%d` 표시 추가 |
| `TestCombatScene.gd` | 레거시 참조 전수 교체 |
| `SystemTestScene.gd` | ARM/RES/MV 수식 갱신 |

**직업별 수치 확정**
| 직업 | move_range | default_arm | default_res |
|------|-----------|-------------|-------------|
| Fighter | 4 | 20 | 5 |
| Archer | 3 | 10 | 8 |
| Rogue | 5 | 8 | 5 |
| Mage | 2 | 5 | 18 |

**버그 수정 (리뷰 후)**
- `move_remaining` 음수 방지: `maxi(..., 0)`
- VP이탈 후 `highlight_move_range` 중복 호출 제거

---

## 2026-03-15 세션1 작업 내용

### 전투 HUD & 스킬 UI (UI-1~21) 구현 완료

**신규 스크립트 (scripts/ui/)**
| 파일 | 담당 |
|------|------|
| `TopBar.gd` | 상단 Turn/VP 다이아몬드 표시 |
| `UnitStatusPanel.gd` | 좌하단 아군 HUD (HP/ARM/RES/STA/MP + 상태이상 + 패시브) |
| `SkillBar.gd` | 하단 8슬롯 스킬 바 (클래스별 동적 슬롯) |
| `SkillSlot.gd` | 개별 슬롯 (6가지 상태, 비용 표시, CD 오버레이) |
| `SkillTooltip.gd` | 호버 0.3초 후 스킬 상세 팝업 |
| `TargetStatusPanel.gd` | 우하단 적 유닛 게이지 |
| `DamagePreview.gd` | 예상 피해/적중/크리티컬/ARM흡수 팝업 |
| `DisengageModal.gd` | VP이탈/강제이탈/취소 3버튼 모달 |
| `CombatBanners.gd` | 적 행동 배너(UI-19), 기회공격 경고(UI-20), STA소진 경고(UI-21) |

---

## 미해결 사항

### 🟡 런타임 테스트 대기
1. 유닛 선택 → `move_remaining` 반영 이동 하이라이트 확인
2. 이동 후 잔여 `move_remaining` 하이라이트 재표시 확인
3. 교전 성립 → 이동 슬롯 비활성 확인
4. VP이탈 → 잔여 이동력으로 이동 가능 확인
5. Fighter ARM=20, Mage RES=18 DebugPanel 확인
6. 턴 시작 → `move_remaining` 리셋 확인
7. 전체 UI 패널 표시 (UnitStatusPanel, SkillBar, DisengageModal)

### 🟡 잔여 TODO (handoff/TODO.md 참조)
- Godot에서 FBX 애니메이션 클립 이름 확인
- HP_BAR_Y=1.8 높이 최종 조정
- T1+T2 전체 검증
- 샌드박스 씬 시나리오 8~11 연결
- ProjectileManager + Projectile 씬 구현

---

## 액티브 스킬 현황 (16개)

| 분류 | 스킬 ID | 이름 |
|------|---------|------|
| common | skill_move | 이동 |
| common | skill_basic_attack | 기본 공격 |
| common | skill_vp_disengage | VP 이탈 |
| Fighter | skill_shield_bash | 방패 강타 |
| Fighter | skill_taunt | 도발 |
| Fighter | skill_war_cry | 전투 함성 |
| Fighter | skill_heavy_strike | 강격 |
| Archer | skill_aimed_shot | 조준 사격 |
| Archer | skill_poison_arrow | 독화살 |
| Archer | skill_crippling_shot | 다리 저격 |
| Rogue | skill_smoke_screen | 연막 |
| Rogue | skill_hemorrhage | 출혈 공격 |
| Rogue | skill_shadow_strike | 암습 |
| Mage | skill_fireball | 화염구 |
| Mage | skill_lightning_bolt | 번개 볼트 |
| Mage | skill_water_bolt | 물 화살 |

---

## 최근 커밋 이력

| 커밋 | 내용 |
|------|------|
| 369066a | [기능추가] 이동 스킬화 + 스탯 구조 개편 (move_remaining / ARM·RES 분리) |
| 3ca422e | [기능추가] 전투 HUD & 스킬 UI (UI-1~21) + 직업별 스킬 확장 (30파일) |
| 4864ee2 | [문서수정] 데스크탑 싱크 컨텍스트 갱신 — TODO 구현 완료 반영 |
| 4c6d54d | [문서수정] 데스크탑 싱크 컨텍스트 갱신 — 03-13 reference + 03-14 context |
