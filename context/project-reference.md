# CrownFalle Proto — 프로젝트 레퍼런스
> 생성일: 2026-03-13 | 갱신: 2026-03-15 (스탯 구조 개편 반영)

---

## 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 장르 | 턴제 전술 RPG (레퍼런스: Wartales) |
| 엔진 | Godot 4.6.1 / GDScript |
| 시점 | 3D Perspective (BG3 스타일) |
| 렌더러 | Forward Plus (Vulkan/D3D12) |
| 세계관 | 중세 다크 판타지 — 용병단 "에코르셔"의 탈주자들이 이끄는 부대 |

---

## 스탯 체계

### 1차 스탯
STR / DEX / CON / INT / WIL

### 2차 스탯
| 스탯 | 공식 |
|------|------|
| HP | CON × 2 |
| MP | WIL × 3 |
| STA | 10 + CON × 2 |
| HIT | 70 + DEX × 0.5 |
| MV | class_config.move_range (고정, DEX 무관) |
| CRIT | DEX × 0.5 + 직업보정 |
| ARM | class_config.default_arm (고정, CON 무관) |
| RES | class_config.default_res (고정, WIL 무관) |

> 🇰🇷 MV/ARM/RES는 2026-03-15 개편으로 1차 스탯 파생에서 분리됨. 직업별 고정값은 class_config.json 참조.

---

## 핵심 아키텍처 원칙

### 1. 데이터 기반 설계
- 모든 수치는 JSON. 코드에 수치 직접 기재 금지.
- 카메라 효과: `data/cameras/camera_config.json` 프리셋 이름으로만 참조.
- 모션 이벤트: `data/animations/animation_config.json` ratio 기반.
- 환경: `data/environments/environment_config.json` 맵 KEY 기반.
- 레벨 데코레이션: `data/levels/level_decoration.json` 맵 KEY 기반.

### 2. 단일 진입점
- 카메라 효과 → `CameraDirector.execute(preset_name, context)` 한 곳만.
- 애니 이벤트 → `AnimEventDispatcher.event_triggered` 시그널 한 곳만.

### 3. 컴포지션 우선
- 새 모션 생성 전 기존 모션 조합(Sequence/Blend/Layer) 가능한지 먼저 검토.

### 4. 투사체 비동기
- 투사체 비행 중 공격자는 즉시 idle 복귀.
- `data/projectiles/` JSON이 투사체 속성 정의.

---

## 폴더 구조

```
crown-falle-proto/
├── data/
│   ├── animations/       ← animation_config.json, bone_masks.json, compositions/
│   ├── cameras/          ← camera_config.json, combat_rules.json
│   ├── classes/          ← fighter.json, archer.json, mage.json, rogue.json
│   ├── environments/     ← environment_config.json (신규 2026-03-12)
│   ├── levels/           ← level_decoration.json (신규 2026-03-12)
│   └── projectiles/      ← arrow.json, magic_bolt.json, throw_stone.json
├── scripts/
│   ├── singletons/       ← AnimationConfig.gd, CameraConfig.gd
│   ├── animation/        ← AnimEventDispatcher.gd, CompositionBuilder.gd
│   ├── rendering/        ← UnitRenderer3D.gd
│   ├── combat/           ← CombatUnit.gd, CombatScene.gd
│   ├── camera/           ← CameraDirector.gd
│   └── environment/      ← EnvironmentConfig.gd, EnvironmentBuilder.gd (신규 2026-03-12)
│                           LevelDecorationConfig.gd, LevelDecorationBuilder.gd
├── assets/
│   ├── _library/         ← 외부 리소스 격리 보관 (.gdignore)
│   ├── characters/       ← shared/, fighter/, archer/, mage/, rogue/, enemies/
│   └── models/units/     ← {class}_rigged.glb
├── addons/
│   └── anim_event_editor/
├── memory/
│   └── SCRATCHPAD.md     ← 에이전트 학습 공유 (절대규칙/실수/패턴)
├── skills/               ← gdscript-conventions.md, data-driven-design.md, asset-pipeline.md
├── tools/
│   └── sync_to_public.ps1 ← 공개 리포 동기화 스크립트
└── handoff/plans/design/ ← 설계 문서 (Doc 1~6 + 환경/데코레이션/환경구축/공개리포)
```

---

## 핵심 스크립트 역할

| 스크립트 | 역할 |
|---------|------|
| `AnimationConfig.gd` | `animation_config.json` 파싱, loop/speed/event 쿼리 (static class) |
| `CameraConfig.gd` | `camera_config.json` + `combat_rules.json` 파싱 (static class) |
| `EnvironmentConfig.gd` | `environment_config.json` 파싱 — 맵 KEY별 조명/안개/하늘 설정 |
| `AnimEventDispatcher.gd` | ratio 기반 이벤트 발화 (hit_frame/sfx/fx/camera_effect) |
| `CompositionBuilder.gd` | AnimationTree sequence/blend/layer 빌더 |
| `UnitRenderer3D.gd` | 모든 3D 비주얼 담당 (모델 로드, 애니, HP바, 파벌 서클, 레이어) |
| `CameraDirector.gd` | execute(preset, ctx) 단일 진입점, 4종 시네마틱 |
| `EnvironmentBuilder.gd` | 맵 KEY에 따라 조명/환경 노드 빌드 |
| `LevelDecorationBuilder.gd` | exterior/interior 에셋 자동 배치 |
| `CombatUnit.gd` | 전투 유닛 상태/스탯/행동 |
| `CombatScene.gd` | 전투 씬 진행 (TurnManager, AI, 이벤트 연결) |

---

## 카메라 3경로

| 경로 | 트리거 | 예시 |
|------|--------|------|
| 경로 A | SkillSystem → CameraDirector | 스킬 시전 시 FOV 변경 |
| 경로 B | AnimEventDispatcher → CameraDirector | hit_frame shake |
| 경로 C | CombatScene (combat_rules 평가) → CameraDirector | 致命打 시 슬로모션 |

---

## 에셋 관리 정책 요약

```
assets/_library/          ← 외부 리소스 격리 (수정 금지, gdignore)
assets/characters/{class}/ ← 승격된 실사용 에셋 (copy, not move)
```

승격 절차:
1. `_library`에서 대상 선정
2. `tools/verify_skeleton.py` 본 이름 확인
3. 정식 경로에 복사 (원본 유지)
4. Godot Import 탭 설정
5. `ASSET_LOG.md` 기록

현재 사용 모델:
- **캐릭터**: Quaternius RPG Classes FBX (CC0) — warrior/ranger/rogue/wizard
  - 한 FBX에 모든 클립 내장
  - `ANIM_PATHS`: `{"path": "res://..fbx", "clip": "ClipName"}` Dict 포맷
- **레벨 데코레이션**: FBX 24종 — 성채/마을/자연/야외 테마별 (2026-03-12 추가)

---

## Claude Code 운영 환경 (2026-03-13 신규)

### 슬래시 커맨드 (`.claude/commands/`)
| 커맨드 | 역할 |
|--------|------|
| `/plan [request]` | @Planner 전환, 컨텍스트 로드, 설계 시작 |
| `/implement [target]` | @Implementor 전환, 체크포인트, 구현 시작 |
| `/review [target]` | @Reviewer 전환, 설계문서+컨벤션 재읽기, 리뷰 |
| `/commit [notes]` | 5단계 체크리스트 게이트 → 커밋 메시지 제안 → 승인 대기 |
| `/sync [scope]` | 싱크 패키지 생성 + 공개 리포 동기화 |

### 스킬 파일 (`skills/`)
| 파일 | 내용 |
|------|------|
| `gdscript-conventions.md` | 네이밍, JSON↔GDScript 매핑, 수정 금지 파일 |
| `data-driven-design.md` | 데이터 파일 맵, 신규 수치 추가 절차, 싱글톤 목록 |
| `asset-pipeline.md` | 라이브러리 규칙, 승격 절차, FBX 로딩 패턴 |

### SCRATCHPAD (`memory/SCRATCHPAD.md`)
모든 에이전트가 작업 시작 시 읽는 학습 파일:
- 🔴 절대 규칙: `int`→`int_stat`, JSONC `//`만, `assets/` 수정 금지
- ⚠️ 자주 발생하는 실수: passive skill effects[], FBX 클립명, HP바 look_at 방향
- 💡 유용한 패턴: 카메라 이벤트 타이밍, 파벌 서클 조건부 표시

### 공개 핸드오프 리포
- `Iseulet/crown-falle-handoff` — Claude Desktop raw URL 접근
- 동기화: `tools/sync_to_public.ps1` (proto→handoff 단방향)
- Raw base: `https://raw.githubusercontent.com/Iseulet/crown-falle-handoff/main/`

---

## 코딩 컨벤션

| 항목 | 규칙 |
|------|------|
| Scene 파일 | `snake_case` (e.g. `combat_map.tscn`) |
| Script 파일 | `PascalCase` (e.g. `CombatManager.gd`) |
| class_name | `PascalCase` |
| 함수/변수 | `snake_case` |
| 상수 | `UPPER_SNAKE_CASE` |
| Private | `_underscore_prefix` |
| Signal | `snake_case` 과거형 (e.g. `unit_moved`) |
| JSON 키 | `snake_case` |
| 수치 | JSON에만 — 코드에 매직 넘버 기재 금지 |
| JSON int 키 | GDScript에서 `int_stat`으로 매핑 (예약어 충돌) |

수정 금지:
- `assets/` — Godot 에디터 전용 관리
- `project.godot` — 명시적 승인 없이 수정 금지

---

## 에이전트 구조

| 에이전트 | 역할 | 출력 경로 |
|---------|------|----------|
| @Planner | 아키텍처 설계, 기능 기획 | `handoff/plans/design/YYYY-MM-DD-기능명.md` |
| @Implementor | 코드 구현, 버그 수정 | — |
| @Reviewer | 코드 리뷰, 품질 감시 | `handoff/plans/review/YYYY-MM-DD-기능명.md` |

충돌 우선순위: DECISIONS.md → @Reviewer → @Planner → 사용자 확인

---

## 설계 문서 요약 (Doc 1~6 + 신규)

### Doc 1~6 — 애니메이션/카메라 시스템 (완료)
- Phase 0~6 전체 구현 완료 (2026-03-11)
- 상세 내용: `handoff/plans/design/2026-03-11-doc*.md` 참조

### Doc 7 — 환경 시스템 (2026-03-12, 완료)
- `EnvironmentConfig.gd` + `EnvironmentBuilder.gd`
- 맵 KEY별 조명/안개/하늘 자동 적용

### Doc 8 — 레벨 데코레이션 시스템 (2026-03-12, Godot 테스트 대기)
- FBX 24종: 성채/마을/자연/야외 4테마
- exterior(그리드 외) / interior(그리드 내, 스폰 제외) 분리 배치

---

## 확정 결정사항 이력 (주요)

| 날짜 | 결정 |
|------|------|
| 2026-03-09 | 렌더러 Forward Plus 전환 (GL Compatibility TDR 크래시 해소) |
| 2026-03-09 | 2D 아이소메트릭 → 3D BG3 스타일 원근 시점 |
| 2026-03-11 | 데이터 기반 애니메이션 파이프라인 설계 완료 (Doc 1~6) |
| 2026-03-11 | 에셋 라이브러리 시스템: `_library/` 격리, `ASSET_LOG.md` 이력 관리 |
| 2026-03-11 | FBX 한 파일에 모든 클립 내장 패턴 확립 |
| 2026-03-12 | HP 게이지 QuadMesh, 파벌 서클 CylinderMesh, `y=0.06` |
| 2026-03-12 | 카메라 사망 이벤트: 즉시 발화 → animation_config ratio 0.45 |
| 2026-03-12 | Git–Drive 에셋 분리: 바이너리는 gitignore, Drive가 백업 |
