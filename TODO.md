# CrownFalle — TODO
> 최종 갱신: 2026-03-14
> 현황: 🔴 1 / 🟡 8 / 🟢 19 / ✅ 5 / ❌ 0

---

## 🔴 긴급 (Blocking)
> 현재 작업을 진행할 수 없게 막는 항목

(없음 — 2026-03-14 active_skills.json effect 필드 수정 완료)

---

## 🟡 다음 (Next)
> 현재 Phase 완료 후 바로 착수할 항목

- [ ] Godot에서 FBX 애니메이션 클립 이름 확인 (`CharacterArmature|Idle` vs `Idle`) `[구현:03-12]` `[2026-03-12]`
- [ ] HP_BAR_Y=1.8 높이 최종 조정 (모델 크기 확인 후) `[구현:03-12]` `[2026-03-12]`
- [ ] project-context.md / project-reference.md 싱크 갱신 `[데스크탑:03-13]` `[2026-03-13]`
- [ ] T1+T2 전체 검증 — DebugCombatPanel과 Godot 실전 전투 테스트 `[구현:03-13]` `[2026-03-13]`
- [ ] 독립 샌드박스 씬 시나리오 8~11 실제 로직 연결 `[감사:03-13]` `[2026-03-13]`
- [ ] ProjectileManager + Projectile 씬 구현 (런타임 투사체 비행) `[설계:phase-b]` `[2026-03-13]`
- [ ] data/skills/status_effects.json 삭제 (통합 완료 후) `[구현:03-14]` `[2026-03-14]`
- [ ] is_skip_turn()과 fear의 disable_actions 동작 충돌 검토 `[구현:03-14]` `[2026-03-14]`

---

## 🟢 백로그 (Backlog)
> 언젠가 해야 하지만 지금은 아닌 항목

### 시스템
- [ ] AoE 마법 (범위 공격) — 타겟 선택 UI, 별도 설계 문서 필요 `[설계:phase-b]` `[2026-03-13]`
- [ ] Archer 근접 대체 공격 — 패시브 스킬 연동 `[설계:phase-b]` `[2026-03-13]`
- [ ] 높이 차이 LOS 보정 — 3D 지형 구현 시 `[설계:phase-b]` `[2026-03-13]`
- [ ] 원거리 치명타 연출 — combat_rules.json 조건 추가 `[설계:phase-b]` `[2026-03-13]`
- [ ] dot_damage_stat/coefficient 스탯연동 DoT 구현 `[구현:03-14]` `[2026-03-14]`
- [ ] weakened debuff_stat/debuff_value 런타임 적용 로직 `[구현:03-14]` `[2026-03-14]`

### 에셋
- [ ] dodge.glb 에셋 수급 `[구현:03-12]` `[2026-03-12]`
- [ ] 투사체 GLB 에셋 교체 (프리미티브 → 실제 모델) `[설계:phase-b]` `[2026-03-13]`
      Quaternius에 화살/마법탄 에셋 있는지 확인. 없으면 Kenney 검색.
- [ ] 투사체 FX Trail 실구현 (GPUParticles3D) `[설계:phase-b]` `[2026-03-13]`
- [ ] Archer/Mage 전용 에셋 `[구현:03-12]` `[2026-03-12]`

### 폴리시
- [ ] LOS 디버그 시각화 (회색 점선) `[설계:phase-b]` `[2026-03-13]`
- [ ] 사각지대 UI 팁 메시지: "사정거리 밖 — 이동 필요" `[설계:phase-b]` `[2026-03-13]`

### 설계 미결정
- [ ] ARM/RES 게이지 전투 중 회복 여부 (없음/턴당/스킬만) `[설계:stat-revision]` `[2026-03-13]`
- [ ] arm_per_con 계수 확정 (1/2/3) `[설계:stat-revision]` `[2026-03-13]`
- [ ] res_per_wil 계수 확정 (1/2/3) `[설계:stat-revision]` `[2026-03-13]`
- [ ] level_diff_coefficient 확정 (2/3/5) `[설계:stat-system]` `[2026-03-08]`
- [ ] hit_max / hit_min 확정 (95/100, 5/10) `[설계:stat-system]` `[2026-03-08]`
- [ ] MP 회복 방식 (턴자동/전투후/스킬만) `[설계:stat-system]` `[2026-03-08]`
- [ ] crit_damage_bonus 수치 조정 `[설계:stat-system]` `[2026-03-08]`

---

## ✅ 완료 (Done)
> 완료된 항목 — 월 단위로 아카이브

- [x] active_skills.json effect 필드 수정 (🔴→✅) `[구현:03-14]` `[2026-03-14]`
- [x] 전투 HUD & 스킬 UI (UI-1~21) 구현 완료 `[구현:03-14]` `[2026-03-14]`
      신규: TopBar, UnitStatusPanel, SkillBar, SkillSlot, SkillTooltip, TargetStatusPanel, DamagePreview, DisengageModal, CombatBanners
      데이터: gauge_colors.json, status_icons.json, skill_bar_layout.json
      수정: CombatScene (signals+UI hookup), StatusManager (get_active_statuses), UnitRenderer3D (UI-10 icons+ring)
- [x] ARM/RES 게이지형 전환 설계 + 구현 `[설계:stat-revision]` `[2026-03-13]`
- [x] CC 게이팅 condition + 상태이상 7종 `[설계:passive-revision]` `[2026-03-13]`
- [x] DebugCombatPanel T2 고도화 `[구현:03-13]` `[2026-03-13]`
- [x] Phase B 투사체 설계 + GridManager/LOS/delivery 구현 `[설계:phase-b]` `[2026-03-13]`
- [x] 상태이상 데이터 단일 파일 통합 (11→13개, 캐시 리팩토링) `[구현:03-14]` `[2026-03-14]`

---

## ❌ 안 할 것 (Won't Do)
> 의도적 포기/폐기 — 사유 필수 기재

(없음)
