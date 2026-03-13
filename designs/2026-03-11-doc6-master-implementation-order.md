# CrownFalle — 문서 6: 통합 구현 착수 순서

- Date: 2026-03-11
- 범위: Doc 1~5 의존 관계 정리 + 전체 구현 순서 마스터 플랜

---

## 1. 의존 관계 맵

```
[Doc 3] 스켈레톤 & 에셋 파이프라인
    ↓
    bone_masks.json
    animation_config.json (path 필드)
    assets/animations/ 구조

[Doc 1] AnimationConfig 싱글톤
    ↓ 읽음
    animation_config.json + compositions/ 폴더

[Doc 1] UnitRenderer3D.play_motion()
    ↓ 단순/컴포지션 분기
    AnimationPlayer  ←── 단순 모션
    AnimationTree    ←── 컴포지션 (CompositionBuilder)

[Doc 2] AnimEventDispatcher
    ↓ setup_simple / setup_composition
    animation_config.json events[] 읽음
    ↓ event_triggered 시그널 발행

[Doc 4] UnitRenderer3D (이동/회전)
    ↓ 이동/공격/피격 흐름 처리
    face_direction / face_direction_smooth

[Doc 5] CameraDirector
    ↓ execute(preset_name, context)
    camera_config.json 프리셋 읽음
    combat_rules.json 규칙 읽음

[Doc 5] CameraDirector ← AnimEventDispatcher (경로 B)
[Doc 5] CameraDirector ← SkillSystem (경로 A)
[Doc 5] CameraDirector ← CombatScene hit_frame (경로 C)
```

---

## 2. 구현 페이즈

### Phase 0 — 데이터 파일 먼저 (코드 없음)

모든 코드가 참조하는 JSON을 먼저 만든다.

| # | 작업 | 파일 |
|---|------|------|
| 0-1 | 공통 모션 설정 | `data/animations/animation_config.json` |
| 0-2 | 본 마스크 정의 | `data/animations/bone_masks.json` |
| 0-3 | 카메라 프리셋 정의 | `data/cameras/camera_config.json` |
| 0-4 | 전투 카메라 규칙 | `data/cameras/combat_rules.json` |
| 0-5 | 직업 데이터에 actions 필드 추가 | `data/classes/fighter.json` 등 4개 |
| 0-6 | 투사체 데이터 | `data/projectiles/arrow.json` 등 |

---

### Phase 1 — 기반 싱글톤

| # | 작업 | 스크립트 | 의존 |
|---|------|----------|------|
| 1-1 | AnimationConfig 싱글톤 | `scripts/singletons/AnimationConfig.gd` | 0-1 |
| 1-2 | CameraConfig 싱글톤 | `scripts/singletons/CameraConfig.gd` | 0-3, 0-4 |

```gdscript
# 검증: 싱글톤이 제대로 로드됐는지 확인
func _ready():
    print(AnimationConfig.get_motion("attack_melee"))   # {} 아닌 dict 출력 확인
    print(CameraConfig.get_presets().keys())             # 프리셋 목록 출력 확인
```

---

### Phase 2 — 렌더러 & 이동

| # | 작업 | 스크립트 | 의존 |
|---|------|----------|------|
| 2-1 | UnitRenderer3D.play_motion() 단순/컴포지션 분기 | `scripts/rendering/UnitRenderer3D.gd` | 1-1 |
| 2-2 | face_direction() / face_direction_smooth() | 위 동일 | — |
| 2-3 | AnimationQueue (AnimPlayer + AnimTree 완료 감지) | 위 동일 | 2-1 |
| 2-4 | 이동 시 방향 전환 (move_to_tween 수정) | 위 동일 | 2-2 |
| 2-5 | 공격 전 타겟 방향 회전 | `scripts/combat/CombatUnit.gd` | 2-2 |
| 2-6 | 피격 시 공격자 방향 반응 | 위 동일 | 2-2 |

---

### Phase 3 — 애니메이션 이벤트

| # | 작업 | 스크립트 | 의존 |
|---|------|----------|------|
| 3-1 | AnimEventDispatcher setup_simple() | `scripts/animation/AnimEventDispatcher.gd` | 1-1 |
| 3-2 | hit_frame 이벤트 핸들러 | `scripts/combat/CombatScene.gd` | 3-1 |
| 3-3 | play_sfx 이벤트 핸들러 | 위 동일 | 3-1 |
| 3-4 | spawn_fx 이벤트 핸들러 | 위 동일 | 3-1 |
| 3-5 | spawn_projectile 이벤트 핸들러 | 위 동일 | 3-1 |
| 3-6 | AnimEventDispatcher setup_composition() | 위 AnimEventDispatcher.gd | 1-1 |

---

### Phase 4 — 카메라

| # | 작업 | 스크립트 | 의존 |
|---|------|----------|------|
| 4-1 | CameraDirector 기본 구조 + execute() 분기 | `scripts/camera/CameraDirector.gd` | 1-2 |
| 4-2 | _apply_base() | 위 동일 | 4-1 |
| 4-3 | _apply_shake() | 위 동일 | 4-1 |
| 4-4 | _apply_fov_pulse() | 위 동일 | 4-1 |
| 4-5 | _apply_tilt() / _apply_soft_track() | 위 동일 | 4-1 |
| 4-6 | 경로 B 연결: AnimEventDispatcher → CameraDirector | CombatScene.gd | 3-1, 4-1 |
| 4-7 | 경로 C 연결: combat_rules 평가 + suppress 처리 | CombatScene.gd | 4-6 |
| 4-8 | 경로 A 연결: SkillSystem → CameraDirector | SkillSystem.gd | 4-1 |
| 4-9 | _apply_cinematic() 4종 | CameraDirector.gd | 4-1 |
| 4-10 | restore_on_end 복귀 처리 | 위 동일 | 4-9 |
| 4-11 | 카메라 상태 머신 완성 | 위 동일 | 4-1~4-10 |

---

### Phase 5 — 컴포지션 시스템

| # | 작업 | 스크립트 | 의존 |
|---|------|----------|------|
| 5-1 | CompositionBuilder.gd | `scripts/animation/CompositionBuilder.gd` | 1-1, 0-2 |
| 5-2 | UnitRenderer3D AnimationTree 연결 | UnitRenderer3D.gd | 5-1, 2-1 |
| 5-3 | 스킬별 composition JSON 작성 | `data/animations/compositions/` | — |
| 5-4 | AnimEventDispatcher setup_composition() 연결 | AnimEventDispatcher.gd | 3-6, 5-1 |

---

### Phase 6 — 에디터 플러그인

| # | 작업 | 위치 | 의존 |
|---|------|------|------|
| 6-1 | plugin.cfg + plugin.gd 등록 | `addons/anim_event_editor/` | — |
| 6-2 | 미리보기 뷰포트 + 타임라인 슬라이더 | anim_event_panel.gd | 1-1 |
| 6-3 | 이벤트 목록 편집 UI + JSON 저장 | 위 동일 | 6-2 |
| 6-4 | 이벤트 타입별 data 입력 UI | 위 동일 | 6-3 |
| 6-5 | Composition Editor 탭 | composition_panel.gd | 5-1 |
| 6-6 | 블렌드 커브 미리보기 | blend_curve_preview.gd | 6-5 |

---

## 3. 인게임 전투 흐름 (완성 후 검증 시나리오)

구현 완료 후 아래 흐름이 동작하는지 순서대로 검증.

```
1. 전투 씬 진입
   → cinematic_battle_intro 재생 (경로 C: battle_start)

2. Fighter가 기본 공격 실행
   → face_direction_smooth(타겟, 0.15s)
   → attack_melee 모션 재생
   → ratio 0.30: play_sfx "sword_swing"
   → ratio 0.55: hit_frame → 피해 적용
   → ratio 0.55: camera_effect "shake_light" (경로 B)
   → ratio 0.55: camera_effect "soft_track_melee" (경로 B)
   → 모션 완료 → idle 복귀

3. 피격 유닛 반응
   → face_direction_smooth(공격자, 0.05s)
   → 피해량 < 30% HP: hit_light 모션
   → 피해량 ≥ 30% HP: hit_heavy 모션

4. 致命打 (피격 유닛 사망 + damage_multiplier ≥ 1.5)
   → combat_rules 평가 (경로 C)
   → cinematic_execute 연출 (슬로모션 + 드리프트 카메라)
   → suppress_base_event = true → B 경로 camera_effect 억제

5. Archer가 스킬 시전
   → skill_data.camera_preset = "skill_base_ranged" (경로 A)
   → FOV/피치 변경
   → attack_ranged 모션 → spawn_projectile 이벤트
   → 투사체 비행 → 타겟 receive_hit()
   → anim_complete → restore_on_end → default 시점 복귀

6. 전투 승리
   → combat_rules battle_end + result == "victory"
   → cinematic_victory 재생
```

---

## 4. 파일 구조 전체 요약

```
crown-falle-proto/
├── data/
│   ├── animations/
│   │   ├── animation_config.json       ← Phase 0-1
│   │   ├── bone_masks.json             ← Phase 0-2
│   │   └── compositions/               ← Phase 5-3
│   │       ├── skill_fighter_charge_slash.json
│   │       └── ...
│   ├── cameras/
│   │   ├── camera_config.json          ← Phase 0-3
│   │   └── combat_rules.json           ← Phase 0-4
│   ├── classes/
│   │   ├── fighter.json                ← Phase 0-5 (actions 필드 추가)
│   │   └── ...
│   └── projectiles/                    ← Phase 0-6
│       ├── arrow.json
│       └── ...
│
├── scripts/
│   ├── singletons/
│   │   ├── AnimationConfig.gd          ← Phase 1-1
│   │   └── CameraConfig.gd             ← Phase 1-2
│   ├── animation/
│   │   ├── AnimEventDispatcher.gd      ← Phase 3-1, 3-6
│   │   └── CompositionBuilder.gd       ← Phase 5-1
│   ├── rendering/
│   │   └── UnitRenderer3D.gd           ← Phase 2-1~2-4, 5-2
│   ├── combat/
│   │   ├── CombatUnit.gd               ← Phase 2-5~2-6
│   │   └── CombatScene.gd              ← Phase 3-2~3-5, 4-6~4-7
│   └── camera/
│       └── CameraDirector.gd           ← Phase 4-1~4-11
│
└── addons/
    └── anim_event_editor/              ← Phase 6-1~6-6
```
