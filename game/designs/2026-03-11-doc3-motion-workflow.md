# CrownFalle — 문서 3: 모션 제작 워크플로우

- Date: 2026-03-11
- 범위: Meshy AI 공통 스켈레톤 / 모션 소스 재사용 파이프라인

---

## 1. 핵심 목표

**"Meshy AI로 생성된 모든 캐릭터가 동일한 스켈레톤 구조를 공유"**
→ 한 번 생성된 모션 GLB를 다른 캐릭터에 그대로 적용 가능

---

## 2. 공통 스켈레톤 전략

### 2-1. 문제
Meshy API의 자동 리깅은 캐릭터마다 본 이름/계층이 달라질 수 있음.
→ fighter의 "hand_r" 본이 archer에선 "RightHand"로 생성되면 모션 공유 불가.

### 2-2. 해결: Meshy 리깅 파라미터 고정

```python
# tools/generate_units.py — 리깅 요청 시 공통 파라미터
rigging_payload = {
    "model_url": model_url,
    "height_meters": 1.75,      # 모든 캐릭터 동일 고정
    "topology": "humanoid",     # 휴머노이드 고정
}
```

Meshy 자동 리깅은 휴머노이드 표준 본 구조(Mixamo 호환)를 생성함.
동일 height_meters + humanoid topology → 본 이름 일치율 높음.

### 2-3. 표준 본 이름 (Meshy 휴머노이드 기준)

```
Hips
├── Spine
│   └── Chest
│       └── Neck
│           └── Head
├── LeftShoulder → LeftArm → LeftForeArm → LeftHand
├── RightShoulder → RightArm → RightForeArm → RightHand
├── LeftUpLeg → LeftLeg → LeftFoot
└── RightUpLeg → RightLeg → RightFoot
```

### 2-4. 본 호환성 검증 스크립트

```python
# tools/verify_skeleton.py
# 생성된 GLB의 본 이름을 표준 목록과 비교
STANDARD_BONES = [
    "Hips", "Spine", "Chest", "Neck", "Head",
    "LeftArm", "RightArm", "LeftForeArm", "RightForeArm",
    "LeftHand", "RightHand",
    "LeftUpLeg", "RightUpLeg", "LeftLeg", "RightLeg",
    "LeftFoot", "RightFoot"
]

def verify(glb_path: str) -> dict:
    # GLB 파싱 → 본 목록 추출 → 표준 목록과 비교
    # 불일치 본 목록 반환
```

---

## 3. 모션 컴포지션 시스템

### 3-0. 핵심 개념

스킬/피격 모션을 매번 새로 생성하지 않고,
**기존 리소스 모션을 레이어/블렌드/시퀀스로 조합**해서 새로운 모션을 만드는 구조.

```
리소스 모션 (소스)          컴포지션 정의 (데이터)        출력 모션
─────────────────           ────────────────────────      ────────────
attack_melee.glb    ─┐
attack_ranged.glb   ─┤  blend_tree.json            →   skill_dual_strike
cast_spell.glb      ─┤  sequence_config.json        →   skill_charge_slash
hit_light.glb       ─┘  layer_mask_config.json      →   hit_while_casting
```

---

### 3-1. 컴포지션 방식 3종

#### A. Sequence (순차 재생)
여러 모션을 순서대로 이어 붙임. 콤보 공격에 적합.

```json
// data/animations/compositions/skill_fighter_charge_slash.json
{
  "type": "sequence",
  "motions": [
    { "motion": "move",         "duration": 0.3,  "trim_end": 0.7 },
    { "motion": "attack_melee", "duration": 1.0,  "trim_start": 0.0 }
  ],
  "transition": "blend",   // cut / blend / match_pose
  "blend_time": 0.1
}
```

```
[move 앞 30%] → (0.1s 블렌드) → [attack_melee 전체]
     돌진 이동                      타격
```

#### B. Blend (가중치 혼합)
두 모션을 비율로 섞음. 상태 변화 표현에 적합.

```json
// data/animations/compositions/hit_staggered_cast.json
{
  "type": "blend",
  "motions": [
    { "motion": "cast_spell", "weight": 0.6 },
    { "motion": "hit_light",  "weight": 0.4 }
  ]
}
```

```
cast_spell ──60%──┐
                  ├── 블렌드 → 시전 중 피격 비틀림 표현
hit_light  ──40%──┘
```

#### C. Layer (본 마스크 레이어)
상체/하체를 분리해 다른 모션을 동시 재생.
이동하면서 공격하거나, 걸으면서 마법 시전 등.

```json
// data/animations/compositions/move_while_cast.json
{
  "type": "layer",
  "layers": [
    {
      "motion": "move",
      "bone_mask": "lower_body",  // Hips, LeftUpLeg, RightUpLeg, ...
      "weight": 1.0
    },
    {
      "motion": "cast_spell",
      "bone_mask": "upper_body",  // Spine, Chest, LeftArm, RightArm, ...
      "weight": 1.0
    }
  ]
}
```

```
하체: [move loop]          → 걷는 발/다리
상체: [cast_spell]         → 팔/상체 시전 모션
     ─────────────────────────────────────────
결과: 걸으면서 마법 시전
```

---

### 3-2-1. 모션 간 블렌드 처리

컴포지션 타입과 상황에 따라 블렌드 방식이 달라짐.

#### 블렌드 방식 4종

| 방식 | 설명 | 적합한 상황 |
|------|------|-------------|
| **cut** | 즉시 전환, 블렌드 없음 | 빠른 반응이 필요한 피격 |
| **linear** | 균일한 속도로 교차 페이드 | 일반적인 모션 전환 |
| **ease_in_out** | 시작/끝 느리고 중간 빠름 | 자연스러운 동작 연결 |
| **match_pose** | 전환 시점의 포즈를 맞춰 끊김 최소화 | 콤보 공격, 이동→공격 |

```json
// Sequence 블렌드 설정 예시
{
  "type": "sequence",
  "motions": [
    { "motion": "move",               "trim_end": 0.4 },
    { "motion": "attack_melee_heavy", "trim_start": 0.0 }
  ],
  "transitions": [
    {
      "from": 0, "to": 1,
      "blend_time": 0.10,
      "blend_mode": "match_pose"
    }
  ]
}
```

#### match_pose 처리 방식

전환 시점에 두 모션의 포즈를 보간하여 관절 위치 불연속 제거.

```gdscript
# CompositionBuilder.gd

func _apply_match_pose(
    from_anim: Animation,
    to_anim: Animation,
    from_ratio: float,    # 전환 시작 시점 (from 모션 기준)
    to_ratio: float,      # 전환 시작 시점 (to 모션 기준)
    blend_time: float
) -> AnimationNodeBlend2:

    # from 모션의 전환 시점 포즈 샘플링
    var from_pose = _sample_pose(from_anim, from_ratio)
    # to 모션의 시작 포즈 샘플링
    var to_pose   = _sample_pose(to_anim, to_ratio)

    # 두 포즈 사이를 blend_time 동안 선형 보간
    var blend_node = AnimationNodeBlend2.new()
    blend_node.blend_amount = 0.0  # 0 = from, 1 = to
    # blend_time 동안 0 → 1 Tween
    return blend_node
```

#### Blend 타입 내 가중치 보간

단순 고정 weight가 아니라 **시간에 따라 가중치가 변하는 커브 지원**:

```json
{
  "type": "blend",
  "motions": [
    { "motion": "cast_spell", "weight_curve": "ease_out" },
    { "motion": "hit_light",  "weight_curve": "ease_in"  }
  ],
  "blend_duration": 0.3
}
```

```
cast_spell:  1.0 ─────────╲___  (ease_out: 점점 줄어듦)
hit_light:   0.0 ___╱─────────  (ease_in:  점점 늘어남)
                    ↑ 블렌드 구간
```

```gdscript
enum WeightCurve { LINEAR, EASE_IN, EASE_OUT, EASE_IN_OUT }

func _evaluate_curve(curve: WeightCurve, t: float) -> float:
    match curve:
        WeightCurve.LINEAR:      return t
        WeightCurve.EASE_IN:     return t * t
        WeightCurve.EASE_OUT:    return 1.0 - (1.0 - t) * (1.0 - t)
        WeightCurve.EASE_IN_OUT: return t * t * (3.0 - 2.0 * t)
        _:                       return t
```

#### Layer 타입 블렌드 (상하체 전환 시)

레이어 전환 시 본 마스크 경계(허리 관절)에서 끊김 방지:

```json
{
  "type": "layer",
  "layers": [
    { "motion": "move",          "bone_mask": "lower_body", "weight": 1.0 },
    { "motion": "attack_ranged", "bone_mask": "upper_body",
      "blend_in_time": 0.1,   // 상체 모션 시작 시 서서히 적용
      "blend_out_time": 0.1   // 상체 모션 종료 시 서서히 해제
    }
  ],
  "spine_blend": {
    "bone": "Spine",          // 경계 관절
    "lower_weight": 0.3,      // 하체 모션이 Spine에 미치는 영향
    "upper_weight": 0.7       // 상체 모션이 Spine에 미치는 영향
  }
}
```

```
하체 [move]      ─── Hips~Thigh 100% ──────────────────────
                                       Spine: 하체 30% + 상체 70%
상체 [attack]    ─── Chest~Hand 100% ──────────────────────
```

Spine 관절을 양쪽 모션이 일정 비율로 공유 → 어색한 허리 꺾임 방지.

---

### 3-2-2. 블렌드 관련 데이터 전체 필드 정리

```json
{
  "type": "sequence | blend | layer",

  // sequence 전용
  "transitions": [
    {
      "from": 0,                  // 소스 모션 인덱스
      "to": 1,                    // 타겟 모션 인덱스
      "blend_time": 0.10,         // 블렌드 지속 시간 (초)
      "blend_mode": "match_pose", // cut / linear / ease_in_out / match_pose
      "from_trim": 0.85,          // from 모션의 블렌드 시작 지점 (ratio)
      "to_trim": 0.05             // to 모션의 블렌드 시작 지점 (ratio)
    }
  ],

  // blend 전용
  "blend_duration": 0.3,
  "motions": [
    {
      "motion": "cast_spell",
      "weight": 0.6,              // 고정 가중치 (weight_curve와 택1)
      "weight_curve": "ease_out"  // 동적 가중치 커브
    }
  ],

  // layer 전용
  "spine_blend": {
    "bone": "Spine",
    "lower_weight": 0.3,
    "upper_weight": 0.7
  },
  "layers": [
    {
      "motion": "move",
      "bone_mask": "lower_body",
      "weight": 1.0,
      "blend_in_time": 0.1,       // 레이어 페이드 인
      "blend_out_time": 0.1       // 레이어 페이드 아웃
    }
  ]
}
```

```json
// data/animations/bone_masks.json
{
  "upper_body": [
    "Spine", "Chest", "Neck", "Head",
    "LeftShoulder", "LeftArm", "LeftForeArm", "LeftHand",
    "RightShoulder", "RightArm", "RightForeArm", "RightHand"
  ],
  "lower_body": [
    "Hips",
    "LeftUpLeg", "LeftLeg", "LeftFoot",
    "RightUpLeg", "RightLeg", "RightFoot"
  ],
  "right_arm": [
    "RightShoulder", "RightArm", "RightForeArm", "RightHand"
  ],
  "left_arm": [
    "LeftShoulder", "LeftArm", "LeftForeArm", "LeftHand"
  ],
  "full_body": []   // 빈 배열 = 전체 본
}
```

---

### 3-3. 컴포지션 → Godot AnimationTree 변환

런타임에서 composition JSON을 읽어 AnimationTree 노드를 동적 생성.

```gdscript
# scripts/animation/CompositionBuilder.gd

func build(composition: Dictionary) -> AnimationNode:
    match composition.type:
        "sequence": return _build_sequence(composition)
        "blend":    return _build_blend(composition)
        "layer":    return _build_layer(composition)

func _build_sequence(comp: Dictionary) -> AnimationNodeStateMachine:
    var sm = AnimationNodeStateMachine.new()
    var prev_name = ""
    for i in comp.motions.size():
        var m = comp.motions[i]
        var clip_node = AnimationNodeAnimation.new()
        clip_node.animation = m.motion
        sm.add_node("step_%d" % i, clip_node)
        if prev_name != "":
            var trans = AnimationNodeStateMachineTransition.new()
            trans.xfade_time = comp.get("blend_time", 0.1)
            sm.add_transition(prev_name, "step_%d" % i, trans)
        prev_name = "step_%d" % i
    return sm

func _build_layer(comp: Dictionary) -> AnimationNodeBlendTree:
    var tree = AnimationNodeBlendTree.new()
    var masks = _load_bone_masks()
    for i in comp.layers.size():
        var layer = comp.layers[i]
        var clip_node = AnimationNodeAnimation.new()
        clip_node.animation = layer.motion
        # 본 마스크 필터 적용
        var filter_node = _apply_bone_filter(clip_node, masks[layer.bone_mask])
        tree.add_node("layer_%d" % i, filter_node)
    # 레이어 Add 노드로 합산
    return tree
```

---

### 3-4. 스킬별 컴포지션 예시

#### Fighter — Shield Bash (돌진 + 강타)
```json
{
  "type": "sequence",
  "motions": [
    { "motion": "move",               "trim_end": 0.4  },
    { "motion": "attack_melee_heavy", "trim_start": 0.0 }
  ],
  "blend_time": 0.08
}
```

#### Archer — Moving Shot (이동 중 사격)
```json
{
  "type": "layer",
  "layers": [
    { "motion": "move",          "bone_mask": "lower_body", "weight": 1.0 },
    { "motion": "attack_ranged", "bone_mask": "upper_body", "weight": 1.0 }
  ]
}
```

#### Mage — Charged Burst (차징 + 폭발)
```json
{
  "type": "sequence",
  "motions": [
    { "motion": "cast_spell",   "trim_end": 0.8  },
    { "motion": "cast_aoe",     "trim_start": 0.2 }
  ],
  "blend_time": 0.15
}
```

#### Rogue — Backstab Hit (피격 중 반격)
```json
{
  "type": "blend",
  "motions": [
    { "motion": "attack_melee", "weight": 0.7 },
    { "motion": "hit_light",    "weight": 0.3 }
  ]
}
```

---

### 3-5. Godot 컴포지션 에디터 (플러그인 확장)

Doc 2의 Anim Event Editor 플러그인에 **Composition 탭 추가**.

```
┌─ Anim Event Editor ──────────────────────────────────────────────┐
│  [Event Editor 탭]  [Composition Editor 탭]                      │
├──────────────────────────────────────────────────────────────────┤
│  Composition: [ skill_fighter_charge_slash ▼ ]  [+신규] [삭제]   │
│  Type:        [ sequence ▼ ]                                     │
│                                                                   │
│  ┌─ 레이어/시퀀스 목록 ────────────────────────────────────────┐  │
│  │ #  motion            bone_mask     weight  trim_s  trim_e  │  │
│  │ 1  [move       ▼]   [lower_body▼] [1.0 ] [0.00]  [0.40]  │  │
│  │ 2  [attack_melee▼]  [full_body ▼] [1.0 ] [0.00]  [1.00]  │  │
│  │ [+ 레이어 추가]                                  [미리보기] │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  blend_time: [0.10]                                              │
│                                                                   │
│  ┌─ 미리보기 뷰포트 ──────────────────────────────────────────┐  │
│  │  [◀◀] [▶] [■]   ─────●──────────────   0.66s / 1.50s     │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  [저장 → compositions/{name}.json]                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3-6. 데이터 파일 구조

```
data/animations/
  animation_config.json         ← 기본 모션 설정
  bone_masks.json                ← 본 마스크 정의
  compositions/                  ← 컴포지션 정의
    skill_fighter_charge_slash.json
    skill_archer_moving_shot.json
    skill_mage_charged_burst.json
    skill_rogue_backstab.json
    hit_staggered_cast.json
    ...
```

class_config의 actions에서 컴포지션도 동일하게 참조:

```json
// fighter.json
{
  "actions": {
    "skill_attack": { "motion": "skill_fighter_charge_slash" }
                                  ↑ compositions/ 파일명과 일치
  }
}
```

AnimationConfig 싱글톤이 로드 시 motions와 compositions를 통합 관리.
UnitRenderer3D는 motion 이름만 받으면 단순 모션인지 컴포지션인지 내부에서 판단.

---

## 3-7. 구현 순서

| 순위 | 항목 |
|------|------|
| 1 | bone_masks.json 정의 |
| 2 | CompositionBuilder.gd — sequence/blend/layer 빌드 |
| 3 | AnimationConfig 싱글톤에 compositions 통합 로드 |
| 4 | UnitRenderer3D에서 컴포지션 판별 후 AnimationTree 적용 |
| 5 | 스킬별 composition JSON 작성 (4개 직업) |
| 6 | 에디터 플러그인 Composition 탭 추가 |
| 7 | 컴포지션 미리보기 뷰포트 |

---

## 3-8. 모션 소스 재사용 구조

### 3-1. 모션 GLB는 캐릭터 독립적으로 저장

```
assets/animations/
  shared/                     ← 공통 모션 (모든 캐릭터 재사용)
    idle.glb
    move.glb
    hit_light.glb
    hit_heavy.glb
    dead_hit.glb
    dead_dot.glb
    ...

  fighter/                    ← fighter 전용 모션
    attack_melee.glb
    attack_skill_triple.glb

  archer/                     ← archer 전용 모션
    attack_ranged.glb

  mage/                       ← mage 전용 모션
    cast_spell.glb
    cast_aoe.glb

  rogue/                      ← rogue 전용 모션
    attack_melee_heavy.glb

  enemies/
    shared/                   ← 몬스터 공통 모션 (PC shared 재사용 가능)
    goblin/                   ← 고블린 전용
    orc/
```

### 3-2. animation_config.json 에 경로 명시

```json
{
  "motions": {
    "idle":         { "path": "shared/idle.glb",          "loop": true  },
    "move":         { "path": "shared/move.glb",          "loop": true  },
    "attack_melee": { "path": "fighter/attack_melee.glb", "loop": false },
    "attack_ranged":{ "path": "archer/attack_ranged.glb", "loop": false },
    "hit_light":    { "path": "shared/hit_light.glb",     "loop": false },
    "dead_hit":     { "path": "shared/dead_hit.glb",      "loop": false }
  }
}
```

UnitRenderer3D는 경로를 보고 로드. 클래스를 알 필요 없음.

---

## 4. Meshy AI 생성 파이프라인 (tools/generate_units.py 확장)

### 4-1. 전체 플로우

```
1. Text-to-3D (preview)
   └─ POST /openapi/v2/text-to-3d (mode=preview)

2. Refine
   └─ POST /openapi/v2/text-to-3d (mode=refine)
   └─ 저장: assets/models/units/{class}.glb

3. 리깅
   └─ POST /openapi/v1/rigging (height_meters=1.75, topology=humanoid)
   └─ 저장: assets/models/units/{class}_rigged.glb

4. 스켈레톤 검증
   └─ tools/verify_skeleton.py 실행
   └─ 불일치 시 경고 출력 (수동 확인 필요)

5. 공통 모션 생성 (shared — 최초 1회만)
   └─ action_ids: [0, 21, 178, 7, 173, 187, 189, 188, 180]
   └─ 저장: assets/animations/shared/

6. 직업 전용 모션 생성
   └─ fighter: [92, 105]
   └─ archer:  [224]
   └─ mage:    [129, 132]
   └─ rogue:   [128]
   └─ 저장: assets/animations/{class}/
```

### 4-2. 공통 모션 action_id 목록

| 모션명 | action_id | 설명 |
|--------|-----------|------|
| idle | 0 | Idle |
| move | 21 | Walk_Fight_Forward |
| hit_light | 178 | Hit_Reaction |
| hit_heavy | 7 | BeHit_FlyUp |
| hit_dot | 173 | Slap_Reaction |
| dead_hit | 187 | Knock_Down |
| dead_dot | 189 | dying_backwards |
| dead_execute | 188 | Fall_Dead_from_Abdominal_Injury |
| knockback | 180 | Shot_in_the_Back_and_Fall |
| staggered | 178 | Hit_Reaction (재사용) |
| buff | 59 | Victory_Cheer |
| debuff | 391 | Head_Hold_in_Pain |
| victory | 59 | Victory_Cheer (재사용) |
| defeat | 391 | Head_Hold_in_Pain (재사용) |

### 4-3. 직업 전용 action_id 목록

| 직업 | 모션명 | action_id |
|------|--------|-----------|
| fighter | attack_melee | 92 |
| fighter | attack_skill_triple | 105 |
| archer | attack_ranged | 224 |
| mage | cast_spell | 129 |
| mage | cast_aoe | 132 |
| rogue | attack_melee_heavy | 128 |

---

## 5. 몬스터 모션 전략

### 5-1. PC shared 모션 재사용
몬스터의 hit_light, hit_heavy, dead_hit 등은 shared/ 그대로 사용.
→ 별도 생성 불필요.

### 5-2. 몬스터 전용 공격 모션
enemy_fighter는 fighter의 attack_melee 재사용.
고유 몬스터(goblin 등) 추가 시 별도 모션 생성.

```json
// enemy_fighter.json actions
{
  "basic_attack": { "motion": "attack_melee" }  // fighter 모션 재사용
}
```

---

## 6. 워크플로우 체크리스트

새 캐릭터 추가 시:
```
[ ] 1. Text-to-3D 프롬프트 작성 (치비 스타일 가이드 참조)
[ ] 2. generate_units.py 실행 (preview → refine → rig)
[ ] 3. verify_skeleton.py 실행 → 본 호환성 확인
[ ] 4. 전용 모션 필요 여부 판단
    [ ] 필요 시: generate_units.py 모션 생성 단계 실행
    [ ] 불필요 시: shared/ 또는 기존 직업 모션 재사용
[ ] 5. data/classes(enemies)/{id}.json actions 필드 작성
[ ] 6. animation_config.json 에 신규 모션 path 추가 (전용 모션인 경우)
[ ] 7. Anim Event Editor 플러그인으로 hit_frame 및 이벤트 설정
[ ] 8. 인게임 테스트
```
