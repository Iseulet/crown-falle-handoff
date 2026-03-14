# CrownFalle Proto — 프로젝트 컨텍스트
> 생성일: 2026-03-15 | 세션: Claude Code

---

## 현재 상태

| 항목 | 내용 |
|------|------|
| Phase | Phase A-2 완료 / Tier 1+2 전투 시스템 완료 / 전투 HUD UI-1~21 구현 완료 |
| 상태 | Godot 런타임 테스트 대기 — 스킬 로드 미해결 중 |
| 마지막 작업 | 전투 HUD & 스킬 UI 커밋 완료 (30파일, 2858줄+) |

---

## 2026-03-15 세션 작업 내용

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

**신규 데이터 (data/ui/)**
- `gauge_colors.json` — HP/ARM/RES/STA/MP 색상 + 임계값 + 슬롯 상태 색상
- `status_icons.json` — 13종 상태이상 아이콘/색상/카테고리
- `skill_bar_layout.json` — 8슬롯 레이아웃, 구분선, 클래스별 슬롯 매핑

**수정 파일**
- `CombatScene.gd` — unit_selected/vp_changed 시그널, get_skill_manager/get_status_manager 접근자, _setup_new_ui(), 스킬 타겟팅 파이프라인, DisengageModal 연동
- `StatusManager.gd` — get_active_statuses() 추가 (UI 패널용)
- `UnitRenderer3D.gd` — update_action_state/update_action_ring, 💤 아이콘, 파벌 서클 색상

### 직업별 스킬 확장

**추가된 스킬 (6개)**
| 직업 | 스킬 | 효과 |
|------|------|------|
| Fighter | skill_war_cry | 전투 함성 — 약화 100% (STA 4, CD 3) |
| Fighter | skill_heavy_strike | 강격 STR×1.3 + 출혈 50% (STA 6, CD 3) |
| Archer | skill_poison_arrow | 독화살 DEX×0.9 + 중독 75% (STA 4, CD 2) |
| Archer | skill_crippling_shot | 다리 저격 DEX×0.8 + 절름발이 100% (STA 3, CD 3) |
| Rogue | skill_hemorrhage | 출혈 공격 DEX×1.0 + 출혈 100% (STA 3, CD 2) |
| Rogue | skill_shadow_strike | 암습 DEX×1.5 고피해 (STA 6, CD 4) |

**skill_bar_layout.json 슬롯 확장**
- Fighter: [2,3] → [2,3,4,5] (4슬롯)
- Archer/Rogue: [2,3] → [2,3,4] (3슬롯)
- Mage: [2,3,4,5] 유지

### 버그 수정

**ResourceLoader.exists() 오류 (5개 파일)**
- 원인: `ResourceLoader.exists()`는 Godot 임포트 리소스(.tres/.res)만 인식. FileAccess로 읽는 원시 JSON에는 항상 `false` → 모든 UI 데이터 로드 실패
- 수정: `SkillBar.gd`, `SkillSlot.gd`, `UnitStatusPanel.gd`, `SkillTooltip.gd`, `TargetStatusPanel.gd` — `ResourceLoader.exists()` 가드 제거, `DataLoader.load_json()` 직접 호출

---

## 미해결 사항

### 🟡 테스트 대기
- 스킬 로드 확인 필요 (ResourceLoader 버그 수정 후 재테스트)
- 전체 UI 패널 Godot 표시 확인
  1. 유닛 선택 → UnitStatusPanel HP/ARM/RES/STA/MP 갱신
  2. SkillSlot 클릭 → 사거리 하이라이트 → 타겟 클릭 → 효과 발동
  3. 교전 이동 → DisengageModal 3버튼
  4. VP 변동 → TopBar 다이아몬드 갱신
  5. 적 턴 → EnemyActionBanner
  6. STA 0 → UI-21 경고 + 💤 아이콘

### 🟡 잔여 TODO (handoff/TODO.md 참조)
- Godot에서 FBX 애니메이션 클립 이름 확인
- HP_BAR_Y=1.8 높이 최종 조정
- project-context.md / project-reference.md 싱크 갱신
- T1+T2 전체 검증
- 샌드박스 씬 시나리오 8~11 연결
- ProjectileManager + Projectile 씬 구현
- data/skills/status_effects.json 삭제
- is_skip_turn()과 fear 동작 충돌 검토

---

## 전체 스킬 현황

### 액티브 스킬 (13개)
| 직업 | 스킬 ID | 이름 | 효과 요약 |
|------|---------|------|----------|
| Fighter | skill_shield_bash | 방패 강타 | STR×0.8 + 기절 100%, STA 5 |
| Fighter | skill_taunt | 도발 | 강제 교전, STA 3 |
| Fighter | skill_war_cry | 전투 함성 | 약화 100%, STA 4 |
| Fighter | skill_heavy_strike | 강격 | STR×1.3 + 출혈 50%, STA 6 |
| Archer | skill_aimed_shot | 조준 사격 | DEX×1.5, STA 4 |
| Archer | skill_poison_arrow | 독화살 | DEX×0.9 + 중독 75%, STA 4 |
| Archer | skill_crippling_shot | 다리 저격 | DEX×0.8 + 절름발이 100%, STA 3 |
| Rogue | skill_smoke_screen | 연막 | 교전 해제, STA 4 |
| Rogue | skill_hemorrhage | 출혈 공격 | DEX×1.0 + 출혈 100%, STA 3 |
| Rogue | skill_shadow_strike | 암습 | DEX×1.5, STA 6 |
| Mage | skill_fireball | 화염구 | INT×1.5 + 화상 50%, MP 8 |
| Mage | skill_lightning_bolt | 번개 볼트 | INT×1.2 + 감전 75%, MP 7 |
| Mage | skill_water_bolt | 물 화살 | INT×0.8 + 젖음 100%, MP 5 |

### 상태이상 (13개)
| 타입 | 효과 |
|------|------|
| CC (5) | stun, knockdown, frozen, charm, fear |
| DoT (3) | poisoned, burning, bleeding |
| Debuff (5) | shocked, wet, chilled, crippled, weakened |

---

## 최근 커밋 이력

| 커밋 | 내용 |
|------|------|
| 3ca422e | [기능추가] 전투 HUD & 스킬 UI (UI-1~21) + 직업별 스킬 확장 (30파일) |
| 4864ee2 | [문서수정] 데스크탑 싱크 컨텍스트 갱신 — TODO 구현 완료 반영 |
| 4c6d54d | [문서수정] 데스크탑 싱크 컨텍스트 갱신 — 03-13 reference + 03-14 context |
| 3eda4e6 | [기능추가] TODO 프로세스 — TODO.md 신규, 14개 커맨드, CLAUDE/GEMINI 규칙 |
