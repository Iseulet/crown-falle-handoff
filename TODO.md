# CrownFalle — TODO
> 최종 갱신: 2026-03-19 (엔진 전환 반영 — Godot 항목 정리, Unity Phase 0 등록)
> 현황: 🔴 0 / 🟡 7 / 🟢 9 / ✅ 6 / ❌ 11

---

## 🔴 긴급 (Blocking)
> 현재 작업을 진행할 수 없게 막는 항목

(없음 — 2026-03-19 엔진 전환 완료. Unity 환경 구축 진행 중.)

---

## 🟡 다음 (Next)
> 현재 Phase 완료 후 바로 착수할 항목

### Unity Phase 0 — 환경 구축 `[설계:engine-transition]` `[2026-03-19]`
- [ ] TBSF 데모 씬 전부 실행 + 구조 문서화 `[설계:engine-transition]` `[2026-03-19]`
- [ ] TBSF 내장 대화 시스템 확인 `[설계:engine-transition]` `[2026-03-19]`
- [ ] Unity MCP 설치 + Claude Code 연동 확인 `[설계:engine-transition]` `[2026-03-19]`
- [ ] Newtonsoft Json.NET 설치 `[구현:engine-transition]` `[2026-03-19]`
- [ ] DOTween 무료 코어 설치 `[구현:engine-transition]` `[2026-03-19]`
- [ ] 플레이스홀더 2D 스프라이트 수집 `[에셋:engine-transition]` `[2026-03-19]`
- [ ] combat_config.json Unity 로드 테스트 `[구현:engine-transition]` `[2026-03-19]`

### 레이어 기획 세션 (Phase 0 병행) `[설계:layer-design]` `[2026-03-19]`
- [ ] Town/Camp 기획 세션 (12개 결정사항 확정) `[설계:layer-design]` `[2026-03-19]`
- [ ] WorldMap 기획 세션 (11개 결정사항 확정) `[설계:layer-design]` `[2026-03-19]`
- [ ] Roster 기획 세션 (14개 결정사항 확정) `[설계:layer-design]` `[2026-03-19]`
- [ ] 자원 흐름 통합 설계 (4대 자원 × 4레이어) `[설계:layer-design]` `[2026-03-19]`
- [ ] 세계관/스토리 기초 설정 `[설계:worldbuilding]` `[2026-03-19]`

### 이식 대상 (전투 로직 — Unity 재구현 시 착수)
- [ ] data/skills/status_effects.json 정리 (Unity 이식 시 경로 정비) `[구현:03-14]` `[2026-03-14]`
- [ ] is_skip_turn / fear disable_actions 충돌 — 전투 로직 이식 시 C# 재검토 `[구현:03-14]` `[2026-03-14]`

---

## 🟢 백로그 (Backlog)
> 언젠가 해야 하지만 지금은 아닌 항목

### 시스템 (Unity 이식 후 구현)
- [ ] 조직력 설정 방식 설계 — TBSF 스폰 시스템 파악 후 설계 `[구현:03-17]` `[2026-03-17]`
- [ ] AoE 마법 (범위 공격) — 타겟 선택 UI, 별도 설계 문서 필요 `[설계:phase-b]` `[2026-03-13]`
- [ ] Archer 근접 대체 공격 — 패시브 스킬 연동 `[설계:phase-b]` `[2026-03-13]`
- [ ] 원거리 치명타 연출 — combat_rules.json 조건 추가 `[설계:phase-b]` `[2026-03-13]`
- [ ] dot_damage_stat/coefficient 스탯연동 DoT 구현 `[구현:03-14]` `[2026-03-14]`
- [ ] weakened debuff_stat/debuff_value 런타임 적용 로직 `[구현:03-14]` `[2026-03-14]`

### 설계 미결정 (Unity 이식 시 유효)
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

- [x] 엔진 전환 확정 및 핸드오프 문서 처리 `[설계:engine-transition]` `[2026-03-19]`
      DECISIONS.md 3개 신규 + 3개 수정됨, INDEX.md 생성, desktop-sync/2026-03-19/ 생성
- [x] project-context.md / project-reference.md 싱크 갱신 `[데스크탑:03-13]` `[2026-03-19]`
- [x] active_skills.json effect 필드 수정 (🔴→✅) `[구현:03-14]` `[2026-03-14]`
- [x] 전투 HUD & 스킬 UI (UI-1~21) 구현 완료 `[구현:03-14]` `[2026-03-14]`
      신규: TopBar, UnitStatusPanel, SkillBar, SkillSlot, SkillTooltip, TargetStatusPanel, DamagePreview, DisengageModal, CombatBanners
      데이터: gauge_colors.json, status_icons.json, skill_bar_layout.json
- [x] ARM/RES 게이지형 전환 설계 + 구현 `[설계:stat-revision]` `[2026-03-13]`
- [x] CC 게이팅 condition + 상태이상 7종 `[설계:passive-revision]` `[2026-03-13]`
- [x] 상태이상 데이터 단일 파일 통합 (11→13개, 캐시 리팩토링) `[구현:03-14]` `[2026-03-14]`

---

## ❌ 안 할 것 (Won't Do)
> 의도적 포기/폐기 — 사유 필수 기재

### Godot 전용 구현 — 엔진 전환으로 폐기 `[2026-03-19]`
- [x] Godot에서 FBX 애니메이션 클립 이름 확인 (`CharacterArmature|Idle` vs `Idle`) — ❌ Godot 폐기
- [x] HP_BAR_Y=1.8 높이 최종 조정 — ❌ Godot 3D UnitRenderer3D 폐기
- [x] T1+T2 전체 검증 — DebugCombatPanel과 Godot 실전 전투 테스트 — ❌ Godot 폐기
- [x] 독립 샌드박스 씬 시나리오 8~11 실제 로직 연결 — ❌ Godot 씬 폐기
- [x] ProjectileManager + Projectile 씬 구현 (런타임 투사체 비행) — ❌ Godot 폐기 (Unity 재구현)
- [x] DebugCombatPanel T2 고도화 — ❌ Godot 폐기 (Unity 재구현)
- [x] Phase B 투사체 설계 + GridManager/LOS/delivery 구현 — ❌ Godot 폐기 (TBSF로 대체)
- [x] LOS 디버그 시각화 (회색 점선) — ❌ Godot 씬 폐기
- [x] 사각지대 UI 팁 메시지 — ❌ Godot 씬 폐기
- [x] 높이 차이 LOS 보정 — 3D 지형 구현 시 — ❌ 2D 전환으로 불필요

### 3D 에셋 — 2D 전환으로 불필요 `[2026-03-19]`
- [x] dodge.glb 에셋 수급 — ❌ 3D 에셋 폐기
- [x] 투사체 GLB 에셋 교체 (프리미티브 → 실제 모델) — ❌ 3D 에셋 폐기
- [x] 투사체 FX Trail 실구현 (GPUParticles3D) — ❌ Godot 3D 폐기
- [x] Archer/Mage 전용 에셋 (3D FBX/GLB) — ❌ 3D 에셋 폐기
