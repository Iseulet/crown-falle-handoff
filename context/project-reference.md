# CrownFalle Proto — 프로젝트 레퍼런스
> 생성일: 2026-03-12 | 갱신 주기: 아키텍처 변경 시만

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
| MV | 직업기본 + DEX × 0.2 |
| CRIT | DEX × 0.5 + 직업보정 |
| ARM | CON × 0.3% |
| RES | WIL × 0.3% |

---

## 핵심 아키텍처 원칙

### 1. 데이터 기반 설계
- 모든 수치는 JSON. 코드에 수치 직접 기재 금지.
- 카메라 효과: `data/cameras/camera_config.json` 프리셋 이름으로만 참조.
- 모션 이벤트: `data/animations/animation_config.json` ratio 기반.

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
│   └── projectiles/      ← arrow.json, magic_bolt.json, throw_stone.json
├── scripts/
│   ├── singletons/       ← AnimationConfig.gd, CameraConfig.gd
│   ├── animation/        ← AnimEventDispatcher.gd, CompositionBuilder.gd
│   ├── rendering/        ← UnitRenderer3D.gd
│   ├── combat/           ← CombatUnit.gd, CombatScene.gd
│   └── camera/           ← CameraDirector.gd
├── assets/
│   ├── _library/         ← 외부 리소스 격리 보관 (.gdignore)
│   ├── characters/       ← shared/, fighter/, archer/, mage/, rogue/, enemies/
│   └── models/units/     ← {class}_rigged.glb
├── addons/
│   └── anim_event_editor/
└── handoff/plans/design/ ← 설계 문서 (Doc 1~6)
```

---

## 핵심 스크립트 역할

| 스크립트 | 역할 |
|---------|------|
| `AnimationConfig.gd` | `animation_config.json` 파싱, loop/speed/event 쿼리 (static class) |
| `CameraConfig.gd` | `camera_config.json` + `combat_rules.json` 파싱 (static class) |
| `AnimEventDispatcher.gd` | ratio 기반 이벤트 발화 (hit_frame/sfx/fx/camera_effect) |
| `CompositionBuilder.gd` | AnimationTree sequence/blend/layer 빌더 |
| `UnitRenderer3D.gd` | 모든 3D 비주얼 담당 (모델 로드, 애니, HP바, 파벌 서클) |
| `CameraDirector.gd` | execute(preset, ctx) 단일 진입점, 4종 시네마틱 |
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

현재 사용 모델: **Quaternius RPG Classes FBX (CC0)** — warrior/ranger/rogue/wizard
- 한 FBX에 모든 클립 내장
- `ANIM_PATHS`: `{"path": "res://..fbx", "clip": "ClipName"}` Dict 포맷

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

## 설계 문서 요약 (Doc 1~6)

### Doc 1 — 애니메이션 시스템 아키텍처
- AnimationConfig (static class) — JSON 파싱
- UnitRenderer3D.play_motion(name) — AnimPlayer(단순) / AnimTree(컴포지션) 분기
- AnimEventDispatcher — ratio 기반 이벤트 발화

### Doc 2 — AnimEvent 구조 & 에디터 플러그인
- 이벤트 타입: `hit_frame`, `sfx`, `fx`, `spawn_projectile`, `camera_effect`
- `addons/anim_event_editor/` — 타임라인 UI, JSON 저장, composition 편집

### Doc 3 — 모션 제작 워크플로우 & 컴포지션
- Sequence: 연속 재생 (A → B → C)
- Blend: 가중치 혼합
- Layer: 본 마스크로 상/하체 분리
- 새 모션 전에 기존 모션 조합 우선 검토

### Doc 4 — 이동 & 방향 연출
- `face_direction(target_pos)` — 즉시 회전
- `face_direction_smooth(target_pos, duration)` — Tween 보간
- 이동 Tween: 각 waypoint에서 방향 전환 → 이동 시작
- 투사체: 비동기 — 공격자 즉시 idle 복귀

### Doc 5 — 카메라 시스템 (데이터 기반)
- `CameraDirector.execute(preset_name, context)` 단일 진입점
- 프리셋 타입: base, shake, fov_pulse, tilt, soft_track, cinematic
- 시네마틱 4종: drop(낙하), closeup(클로즈업), drift(드리프트), orbit(궤도)
- combat_rules.json: 致命打/승리/패배 조건 및 연출 매핑

### Doc 6 — 통합 구현 착수 순서
- Phase 0: 데이터 JSON (완료)
- Phase 1: AnimationConfig / CameraConfig 싱글톤 (완료)
- Phase 2: UnitRenderer3D play_motion / AnimQueue / 방향 (완료)
- Phase 3: AnimEventDispatcher + 이벤트 핸들러 (완료)
- Phase 4: CameraDirector (경로 A/B/C, 시네마틱 4종) (완료)
- Phase 5: CompositionBuilder + AnimationTree (완료)
- Phase 6: 에디터 플러그인 (완료)
- **다음: Phase B** — 원거리 공격 사거리, 투사체 비행, 스태미너

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
