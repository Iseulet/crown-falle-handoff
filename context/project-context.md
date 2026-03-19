# CrownFalle Proto — 프로젝트 컨텍스트
> 생성일: 2026-03-19 | 세션: Claude Code

---

## 현재 상태

| 항목 | 내용 |
|------|------|
| Phase | Phase 0 — TBSF 데모 체험 + Unity 환경 구축 |
| 상태 | 엔진 전환 완료 (Godot → Unity 6.3 LTS). Unity 설치 완료, TBSF 임포트 완료. 데모 실행 대기. |
| 엔진 | Unity 6.3 LTS (6000.3.11f1) + C# |
| 프레임워크 | Turn Based Strategy Framework (TBSF) — 임포트 완료 |
| Godot 코드 | 전면 폐기 (설계 로직 + JSON 데이터만 이식 대상) |
| 마지막 커밋 | 351b064 [문서수정] 설정·설계문서·핸드오프 업데이트 |

---

## 2026-03-19 엔진 전환 결정사항

### 확정 결정 3가지
1. **Engine Transition** — Godot 4.6.1 + GDScript → Unity 6.3 LTS + C# + TBSF
2. **Rendering: 2D** — 3D Perspective → 2D (Dust and Salt 스타일, 다크 중세 판타지)
3. **Game Loop: 4-Layer** — Town/Camp → WorldMap → Battle → Roster Management

### 전환 이유
- 3D 에셋 소싱 병목 해소 (2D 스프라이트가 훨씬 수급 용이)
- Unity Asset Store 생태계 활용
- MCP 바이브코딩 지원

### 이식 대상 (유지)
- 전투 알고리즘: StatCalculator, AttackResolver, BonusCalculator, EngagementSystem, StatusManager, SkillManager, ElementResolver, VPManager
- JSON 데이터: combat_rules, weapon_table, skill_table, preset_config, wound_config, status_effects

### 폐기 대상
- GDScript 코드 전부, Godot 씬(.tscn) 전부
- 3D 관련: UnitRenderer3D, CameraDirector, EnvironmentBuilder, FBX 파이프라인
- Godot 전용: GridManager → TBSF로 대체, DebugCombatPanel → Unity 재구현

---

## 다음 작업 (Phase 0 — Unity 환경 구축)

우선순위 순:

1. **TBSF 데모 씬 전부 실행 + 구조 문서화** `[설계:engine-transition]`
   - TBSF 내장 그리드/턴/유닛 구조 파악
   - 어떤 기능을 그대로 쓸 수 있는지 확인
2. **TBSF 내장 대화 시스템 확인** `[설계:engine-transition]`
3. **Unity MCP 설치 + Claude Code 연동 확인** `[설계:engine-transition]`
4. **Newtonsoft Json.NET 설치** `[구현:engine-transition]`
   - JSON 데이터 이식에 필요
5. **DOTween 무료 코어 설치** `[구현:engine-transition]`
6. **플레이스홀더 2D 스프라이트 수집** `[에셋:engine-transition]`
7. **combat_config.json Unity 로드 테스트** `[구현:engine-transition]`

---

## 게임 루프 4레이어 요약

| 레이어 | 내용 |
|--------|------|
| Town/Camp | 마을 순회 + 이동식 캠프. 4대 자원: Gold/Food/Morale/Reputation |
| WorldMap | Dust and Salt식 포인트 클릭. 식량+피로도 소모. 이벤트 4종 |
| Battle | 헥스 턴제 3~10명. 기존 T1~T2 이식. XP+루팅. HP→캠프/부상→마을 |
| Roster | 5스탯 유지. 무기 기반 정체성(클래스 제거). 영구 사망 |

---

## 설계 문서

| 문서 | 경로 | 상태 |
|------|------|------|
| 게임 루프 설계 | `handoff/plans/design/2026-03-19-crownfalle-game-loop-design.md` | ✅ 확정 (Desktop 생성) |
| 에셋 요구사항 | `handoff/plans/design/2026-03-19-crownfalle-asset-requirements.md` | ✅ 확정 (Desktop 생성) |

> ⚠️ 위 2개 파일은 Claude Desktop에서 생성됨. Claude Code로 수신 완료.

---

## 이전 세션 요약 (Godot — 2026-03-17까지)

| 완료 항목 | 내용 |
|-----------|------|
| 전투 시스템 재구축 | NewCombatScene, NewTurnManager, AttackResolver, BonusCalculator 등 |
| 교전 버그 수정 | EngagementSystem _pairs 캐싱 방식 (다중 교전 방지) |
| 전투 HUD | UI-1~21 (TopBar, UnitStatusPanel, SkillBar 등) |
| Tier 1~2 | 스탯/데미지/교전/스킬/원소/VP 시스템 |
| 데이터 JSON | combat_rules, weapon_table, skill_table, preset_config, wound_config |

> 위 구현물은 모두 폐기. 설계 알고리즘만 Unity로 이식.
