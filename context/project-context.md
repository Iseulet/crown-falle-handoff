# CrownFalle Proto — 현재 세션 컨텍스트
> 생성일: 2026-03-13 | 갱신 주기: 세션마다

---

## Sync Metadata
> 🇰🇷 싱크 메타데이터

| Field | Value |
|-------|-------|
| Generated | 2026-03-13 |
| Source commit | f733cc3 |
| Uncommitted changes | No |
| Sessions since last sync | 2 (2026-03-12, 2026-03-13) |
| DECISIONS.md entries since last sync | 0 (신규 없음) |

⚠️ If reading this after the generated date, information may be outdated.
> 🇰🇷 생성일 이후에 읽는 경우 정보가 오래되었을 수 있습니다.

---

## 현재 상태

| 항목 | 내용 |
|------|------|
| 현재 Phase | Doc 6 Phase 0~6 완료 + 환경/데코레이션 시스템 완료 — Phase B 대기 |
| 세션 상태 | 🔴 Closed |
| Git | `main` branch, last commit `f733cc3` |
| 마지막 작업 | 2026-03-13 |
| 공개 핸드오프 리포 | `Iseulet/crown-falle-handoff` 생성 완료 |

---

## 최근 세션 완료 작업

### 2026-03-13 — Claude Code 환경 구축 + 공개 리포

**Claude Code 환경 (2026-03-13-claude-code-environment-final.md Step 1~8):**
- `memory/SCRATCHPAD.md` — 절대규칙/자주발생실수/유용한패턴 학습 저장소
- `.claude/commands/` — 5개 슬래시 커맨드 (plan/implement/review/commit/sync)
- `skills/` — 3개 스킬 파일 (gdscript-conventions/data-driven-design/asset-pipeline)
- `agents/` Skill References 추가 (PLANNER/IMPLEMENTOR/REVIEWER)
- `handoff/DECISIONS.md` — Category/Status/Superseded By 메타데이터 포맷 마이그레이션

**공개 핸드오프 리포 (`2026-03-13-handoff-public-repo.md`):**
- `Iseulet/crown-falle-handoff` 공개 GitHub 리포 생성
- Claude Desktop이 raw URL로 최신 컨텍스트 직접 읽기 가능 (수동 업로드 불필요)
- `tools/sync_to_public.ps1` — proto→handoff 단방향 동기화 스크립트
- `/sync` 커맨드 Step 5~6 추가 (공개 리포 자동 동기화)

### 2026-03-12 — 레벨 데코레이션 + 환경 시스템

**환경 시스템:**
- `EnvironmentConfig.gd` (static class) — `data/environments/environment_config.json` 파싱
- `EnvironmentBuilder.gd` — 맵별 DirectionalLight3D/WorldEnvironment 적용
- 캐릭터 라이트: `character_light` (레이어 2), `UnitRenderer3D._set_character_layers()` 재귀 적용

**레벨 데코레이션 시스템:**
- FBX 24종 승격 (성채/마을/자연/야외 테마별)
- `data/levels/level_decoration.json` — 4맵별 exterior/interior 배치 정의
- `LevelDecorationConfig.gd` / `LevelDecorationBuilder.gd` 신규
- `CombatScene.gd` 연동 — 전투 시작 시 자동 배치

**이전 세션 연속 작업:**
- Quaternius RPG Classes FBX 교체 (warrior/ranger/rogue/wizard)
- HP 게이지 + 파벌 서클 시스템
- 스폰 방향 초기화 (`_face_units_toward_opponents()`)
- 카메라 사망 연출 개선 (ratio 0.45 이벤트 기반)

---

## 미완료 / 다음 작업

### 즉시 확인 필요 (Godot 테스트 — 코드 없음)
- [ ] 레벨 데코레이션: 전투 시작 시 ExteriorDecoration + InteriorDecoration 노드 씬 트리 확인
- [ ] 외곽 건물/소품이 그리드 밖에 정확히 배치되는지 확인
- [ ] 내부 소품이 스폰 영역 제외하고 eligible_terrain에만 배치되는지 확인
- [ ] 4개 맵(KEY 1~4) 각각 테마별 에셋 표시 확인
- [ ] FBX import 경고 여부 확인
- [ ] warrior_rpg.fbx AnimationPlayer 클립 이름 확인 (`CharacterArmature|Idle` vs `Idle`)
- [ ] HP_BAR_Y=1.8 높이 최종 조정 (모델 크기 확인 후)

### 미진행 계획 논의
- [ ] Phase B 개발 계획 정리 (이번 세션 /plan 시작 중 종료 — 다음 세션 재개)
  - 논의 필요: 범위(Phase B만 vs 전체 로드맵), 목표(연출 우선 vs 게임플레이 루프 우선)

### Phase B (다음 단계)
- [ ] 원거리 공격 사거리 시스템
- [ ] 투사체 비행 (비동기, 공격자 즉시 idle 복귀)
- [ ] Archer/Mage/Rogue 전용 에셋 완성
- [ ] dodge.glb 에셋 수급

---

## 최근 아키텍처 결정 (신규 — 2026-03-12~13)

### 환경 시스템 (2026-03-12)
- 맵 테마: castle/village/forest/field (4종)
- `EnvironmentBuilder` — 조명 + 안개 + 하늘을 맵 KEY에 따라 교체
- 캐릭터 전용 라이트: 레이어 2 (배경 조명과 분리)

### 레벨 데코레이션 (2026-03-12)
- exterior: 그리드 외곽 배치 (건물/울타리 등)
- interior: 그리드 내부 배치 (스폰 영역 제외, eligible_terrain만)
- 맵별 JSON 데이터 주도 — 에셋 추가 = JSON 수정만

### Claude Code 운영 환경 (2026-03-13)
- SCRATCHPAD.md: 에이전트 간 학습 공유 (절대규칙/실수/패턴)
- 슬래시 커맨드: /plan, /implement, /review, /commit, /sync
- 공개 리포: raw URL 기반 Claude Desktop 자동 읽기
