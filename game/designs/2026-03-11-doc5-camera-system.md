# CrownFalle — 문서 5: 카메라 시스템 (데이터 기반)

- Date: 2026-03-11
- 범위: 기본 시점 구성 / 데이터 기반 카메라 효과 / 트리거 구조 / 연출 시점

---

## 1. 설계 원칙

```
기존 방식 (하드코딩)          데이터 기반 방식
─────────────────────         ──────────────────────────────────────────
shake(0.5, 0.25)         →   camera_config.json 의 "shake_heavy" 참조
fov_pulse(-8, 0.12)      →   skill 데이터의 camera_preset 필드로 지정
play_execute_cinematic() →   combat_rules.json — 사망 판정 시 자동 선택
```

**코드는 "어떻게 실행하는가"만 담당.**
**"언제, 무엇을, 어떤 조건으로"는 전부 데이터.**

---

## 2. 카메라 노드 구조

```
CombatScene
├── CameraRig              ← 피벗 (WASD 패닝, 회전 기준)
│   └── CameraArm          ← SpringArm3D (줌 거리 / 벽 충돌 자동 처리)
│       └── Camera3D       ← 렌더 카메라 (FOV / 셰이크 오프셋 전담)
│
├── CinematicCamera3D      ← 연출 전용 카메라 (평소 비활성)
│
└── CameraDirector         ← 데이터 로드 + 효과 실행 단일 진입점
```

- `CameraRig` : 월드 이동/회전. 플레이어 조작 및 소프트 트래킹.
- `CameraArm` : SpringArm3D. 지형 충돌 자동 처리. 연출 줌에만 length 변경.
- `Camera3D` : FOV, 셰이크 오프셋. 직접 이동하지 않음.
- `CameraDirector` : 모든 카메라 효과의 **단일 진입점**. 데이터 로드 및 실행.

---

## 3. 데이터 구조

### 3-1. camera_config.json — 효과 프리셋 정의

모든 카메라 효과의 원본 데이터. 코드에서 수치를 직접 쓰지 않음.

```json
// data/cameras/camera_config.json
{
  "presets": {

    // ── 기본 시점 ──────────────────────────────────────────────
    "default": {
      "type": "base",
      "fov": 45.0,  "pitch": -28.0,  "yaw": 0.0,
      "position": [0.0, 8.0, 14.0],
      "transition": { "mode": "lerp", "duration": 0.4 }
    },

    // ── 셰이크 ────────────────────────────────────────────────
    "shake_light":   { "type": "shake", "intensity": 0.20, "duration": 0.15, "decay": "ease_out" },
    "shake_heavy":   { "type": "shake", "intensity": 0.50, "duration": 0.25, "decay": "ease_out" },
    "shake_aoe":     { "type": "shake", "intensity": 0.60, "duration": 0.30, "decay": "ease_out" },
    "shake_execute": { "type": "shake", "intensity": 0.80, "duration": 0.40, "decay": "linear"   },

    // ── FOV 펄스 ──────────────────────────────────────────────
    "fov_pulse_heavy":   { "type": "fov_pulse", "delta_fov": -6.0,  "duration": 0.10 },
    "fov_pulse_execute": { "type": "fov_pulse", "delta_fov": -12.0, "duration": 0.20 },
    "fov_pulse_aoe":     { "type": "fov_pulse", "delta_fov": -8.0,  "duration": 0.15 },

    // ── 방향 틸트 ─────────────────────────────────────────────
    "tilt_hit": {
      "type": "direction_tilt",
      "intensity": 1.5, "in_duration": 0.06, "out_duration": 0.15
    },

    // ── 소프트 트래킹 ─────────────────────────────────────────
    "soft_track_melee":  { "type": "soft_track", "weight": 0.20, "in_duration": 0.15, "hold_duration": 0.20, "out_duration": 0.30 },
    "soft_track_ranged": { "type": "soft_track", "weight": 0.30, "in_duration": 0.15, "hold_duration": 0.40, "out_duration": 0.30 },
    "soft_track_aoe":    { "type": "soft_track", "weight": 0.25, "in_duration": 0.15, "hold_duration": 0.50, "out_duration": 0.30 },

    // ── 스킬용 기본 시점 변경 ──────────────────────────────────
    "skill_base_melee": {
      "type": "base",
      "fov": 40.0, "pitch": -22.0, "arm_length_delta": -1.5,
      "transition": { "mode": "ease_out", "duration": 0.20 },
      "restore_on_end": true
    },
    "skill_base_ranged": {
      "type": "base",
      "fov": 50.0, "pitch": -30.0, "arm_length_delta": 2.0,
      "transition": { "mode": "ease_out", "duration": 0.25 },
      "restore_on_end": true
    },
    "skill_base_aoe": {
      "type": "base",
      "fov": 55.0, "pitch": -38.0, "arm_length_delta": 3.5,
      "transition": { "mode": "ease_out", "duration": 0.30 },
      "restore_on_end": true
    },

    // ── 연출 시점 ──────────────────────────────────────────────
    "cinematic_closeup": {
      "type": "cinematic", "mode": "closeup",
      "zoom_offset": [0.0, 2.5, 3.5], "watch_offset": [0.0, 3.0, 5.0],
      "zoom_duration": 0.25, "watch_duration": 0.30, "hold_duration": 0.60,
      "blend_in": 0.20, "blend_out": 0.30
    },
    "cinematic_execute": {
      "type": "cinematic", "mode": "drift",
      "start_offset": [3.5, 2.0, 2.5], "end_offset": [1.5, 1.5, 4.0],
      "drift_duration": 1.80, "hold_duration": 0.50,
      "time_scale": 0.30, "time_scale_dur": 1.50,
      "blend_in": 0.15, "blend_out": 0.40
    },
    "cinematic_battle_intro": {
      "type": "cinematic", "mode": "drop",
      "start_position": [0.0, 20.0, 10.0], "start_pitch": -70.0,
      "drop_duration": 1.80, "blend_out": 0.30
    },
    "cinematic_victory": {
      "type": "cinematic", "mode": "orbit",
      "orbit_radius": 4.50, "orbit_speed": 0.40, "orbit_height": 2.00,
      "blend_in": 0.50
    }
  }
}
```

---

### 3-2. 트리거 경로 3종

카메라 효과 발동 경로는 세 가지. 모두 프리셋 이름(string)으로만 참조.

```
경로 A │ 스킬 데이터         → 스킬 발동 시 기본 시점 변경
경로 B │ animation_config    → 모션 타임라인 이벤트로 효과 호출
경로 C │ combat_rules        → 전투 결과 조건부 연출 분기
```

---

#### 경로 A — 스킬 데이터의 camera_preset 필드

스킬이 발동되는 순간 기본 시점(FOV/피치/거리)을 변경.

```json
// data/skills/fighter_shield_bash.json
{
  "skill_id": "fighter_shield_bash",
  "camera_preset": "skill_base_melee",
  ...
}

// data/skills/mage_meteor.json
{
  "skill_id": "mage_meteor",
  "camera_preset": "skill_base_aoe",
  ...
}
```

`restore_on_end: true` 프리셋은 `anim_complete` 이벤트 수신 시 자동으로 `default` 복귀.

---

#### 경로 B — animation_config.json 의 camera_effect 이벤트

Doc 2의 AnimEvent 시스템과 동일 구조. 모션 ratio 지점에서 카메라 효과 발동.

```json
// data/animations/animation_config.json
{
  "motions": {
    "attack_melee": {
      "events": [
        { "ratio": 0.55, "type": "hit_frame",     "data": { "damage_multiplier": 1.0 } },
        { "ratio": 0.55, "type": "camera_effect", "data": { "preset": "shake_light" } },
        { "ratio": 0.55, "type": "camera_effect", "data": { "preset": "soft_track_melee" } }
      ]
    },
    "attack_melee_heavy": {
      "events": [
        { "ratio": 0.55, "type": "hit_frame",     "data": { "damage_multiplier": 1.8 } },
        { "ratio": 0.55, "type": "camera_effect", "data": { "preset": "shake_heavy" } },
        { "ratio": 0.55, "type": "camera_effect", "data": { "preset": "fov_pulse_heavy" } }
      ]
    },
    "cast_aoe": {
      "events": [
        { "ratio": 0.80, "type": "hit_frame",     "data": { "damage_multiplier": 1.0 } },
        { "ratio": 0.80, "type": "camera_effect", "data": { "preset": "shake_aoe" } },
        { "ratio": 0.80, "type": "camera_effect", "data": { "preset": "fov_pulse_aoe" } }
      ]
    }
  }
}
```

---

#### 경로 C — combat_rules.json 의 조건부 분기

**전투 결과에 따른 카메라 분기.** 조건 충족 시 경로 B의 camera_effect를 덮어씀.

```json
// data/cameras/combat_rules.json
{
  "rules": [
    {
      "id": "execute_on_death_heavy",
      "trigger": "hit_frame",
      "priority": 10,
      "conditions": [
        { "check": "target_hp_after",   "op": "<=", "value": 0 },
        { "check": "damage_multiplier", "op": ">=", "value": 1.5 }
      ],
      "camera": "cinematic_execute",
      "suppress_base_event": true      // B 경로 camera_effect 억제
    },
    {
      "id": "closeup_on_death_normal",
      "trigger": "hit_frame",
      "priority": 9,
      "conditions": [
        { "check": "target_hp_after",   "op": "<=", "value": 0 },
        { "check": "damage_multiplier", "op": "<",  "value": 1.5 }
      ],
      "camera": "cinematic_closeup",
      "suppress_base_event": true
    },
    {
      "id": "heavy_shake_on_heavy_hit",
      "trigger": "hit_frame",
      "priority": 5,
      "conditions": [
        { "check": "target_hp_after",   "op": ">",  "value": 0 },
        { "check": "damage_multiplier", "op": ">=", "value": 1.5 }
      ],
      "camera": "shake_heavy",
      "suppress_base_event": false     // B 경로와 병행
    },
    {
      "id": "battle_intro",
      "trigger": "battle_start",
      "priority": 10,
      "conditions": [],
      "camera": "cinematic_battle_intro"
    },
    {
      "id": "victory_cinematic",
      "trigger": "battle_end",
      "priority": 10,
      "conditions": [
        { "check": "result", "op": "==", "value": "victory" }
      ],
      "camera": "cinematic_victory"
    }
  ]
}
```

---

## 4. CameraDirector — 단일 실행 진입점

```gdscript
# scripts/camera/CameraDirector.gd
class_name CameraDirector
extends Node

var _presets: Dictionary = {}
var _rules: Array = []
var _active_base_preset: String = "default"

func _ready() -> void:
    _presets = CameraConfig.get_presets()   # camera_config.json
    _rules   = CameraConfig.get_rules()    # combat_rules.json

# ── 외부 호출 3종 ──────────────────────────────────────────────

# 경로 A: 스킬 발동 시
func on_skill_activated(skill_data: Dictionary) -> void:
    var preset = skill_data.get("camera_preset", "")
    if preset != "": execute(preset, {})

# 경로 B: AnimEvent camera_effect 이벤트 수신
func on_anim_event(event: Dictionary, context: Dictionary) -> void:
    if event.type != "camera_effect": return
    if context.get("suppressed", false): return
    execute(event.data.get("preset", ""), context)

# 경로 C: 전투 결과 평가
func on_combat_event(trigger: String, context: Dictionary) -> void:
    var rule = _evaluate_rules(trigger, context)
    if not rule.is_empty():
        if rule.get("suppress_base_event", false):
            context["suppressed"] = true
        execute(rule.camera, context)

# ── 실행 분기 ─────────────────────────────────────────────────

func execute(preset_name: String, context: Dictionary) -> void:
    var p = _presets.get(preset_name, {})
    if p.is_empty():
        push_warning("[CameraDirector] 미등록 프리셋: %s" % preset_name)
        return
    match p.type:
        "base":           _apply_base(p)
        "shake":          _apply_shake(p)
        "fov_pulse":      _apply_fov_pulse(p)
        "direction_tilt": _apply_tilt(p, context)
        "soft_track":     _apply_soft_track(p, context)
        "cinematic":      _apply_cinematic(p, context)

# ── 규칙 평가 ─────────────────────────────────────────────────

func _evaluate_rules(trigger: String, ctx: Dictionary) -> Dictionary:
    var candidates = _rules.filter(func(r): return r.trigger == trigger)
    candidates.sort_custom(func(a, b): return a.priority > b.priority)
    for rule in candidates:
        if _check_conditions(rule.conditions, ctx):
            return rule
    return {}

func _check_conditions(conditions: Array, ctx: Dictionary) -> bool:
    for c in conditions:
        var val = ctx.get(c.check, null)
        if val == null: return false
        match c.op:
            "==": if not (val == c.value): return false
            "!=": if not (val != c.value): return false
            ">=": if not (val >= c.value): return false
            "<=": if not (val <= c.value): return false
            ">":  if not (val >  c.value): return false
            "<":  if not (val <  c.value): return false
    return true

# ── anim_complete 시 기본 시점 복귀 ──────────────────────────

func on_anim_complete(_motion_name: String) -> void:
    var active = _presets.get(_active_base_preset, {})
    if active.get("restore_on_end", false):
        execute("default", {})
        _active_base_preset = "default"
```

---

## 5. 이벤트 흐름 전체

```
┌─ 경로 A ──────────────────────────────────────────────────────┐
│  SkillSystem.activate(skill_data)                              │
│    └─ CameraDirector.on_skill_activated()                      │
│         └─ execute("skill_base_aoe") → FOV/피치/거리 변경     │
└───────────────────────────────────────────────────────────────┘

┌─ 경로 B + C (hit_frame 타이밍) ───────────────────────────────┐
│  AnimEventDispatcher.event_triggered(event)                    │
│    └─ CombatScene._on_anim_event(event, ctx)                  │
│         │                                                      │
│         ├─ [경로 C] on_combat_event("hit_frame", ctx)         │
│         │    └─ rules 평가                                     │
│         │         ├─ 사망 + 강타 → cinematic_execute          │
│         │         │   suppress = true → B 억제                │
│         │         ├─ 사망 + 일반 → cinematic_closeup          │
│         │         │   suppress = true → B 억제                │
│         │         └─ 생존         → suppress = false          │
│         │                                                      │
│         └─ [경로 B] on_anim_event(event, ctx)                 │
│              suppress = false 일 때만 실행                     │
│              └─ execute("shake_light") → 셰이크               │
└───────────────────────────────────────────────────────────────┘

┌─ 경로 C (전투 시작/종료) ─────────────────────────────────────┐
│  CombatScene._on_battle_start()                                │
│    └─ on_combat_event("battle_start", {})                     │
│         └─ execute("cinematic_battle_intro")                   │
└───────────────────────────────────────────────────────────────┘
```

---

## 6. 기본 조작 (플레이어 입력)

수치는 camera_config.json `"default"` 프리셋에서 로드.

| 입력 | 동작 | 제한 |
|------|------|------|
| 우클릭 드래그 수직 | 피치 조절 | -50° ~ -15° |
| 우클릭 드래그 수평 | 요 회전 | 무제한 |
| 마우스 휠 | FOV 줌 | 25 ~ 65 |
| WASD | 패닝 | 중심 ±15 유닛 |
| R | 기본값 리셋 | — |

`CameraState != NORMAL`일 때 입력 차단.

---

## 7. 카메라 상태 머신

```
NORMAL
  ├─ on_skill_activated  → BASE_CHANGED ──(anim_complete)──► NORMAL
  ├─ soft_track 발동     → SOFT_TRACK   ──(완료)──────────► NORMAL
  ├─ cinematic 발동      → CINEMATIC    ──(완료)──────────► NORMAL
  └─ victory cinematic   → ORBIT        ──(씬 종료)
```

---

## 8. 데이터 파일 구조

```
data/cameras/
  camera_config.json    ← 모든 프리셋 수치 정의
  combat_rules.json     ← 전투 조건부 카메라 분기 규칙
```

새 효과 추가 → `camera_config.json`에 프리셋 항목 추가.
새 조건 추가 → `combat_rules.json`에 rule 항목 추가.
**코드 수정 불필요.**

---

## 9. 구현 순서

| 순위 | 항목 |
|------|------|
| 1 | camera_config.json 기본 프리셋 작성 |
| 2 | combat_rules.json 기본 규칙 작성 |
| 3 | CameraConfig 싱글톤 (JSON 로드) |
| 4 | CameraDirector 기본 구조 + execute() 분기 |
| 5 | _apply_base / _apply_shake / _apply_fov_pulse 구현 |
| 6 | _apply_tilt / _apply_soft_track 구현 |
| 7 | 경로 B 연결: AnimEventDispatcher → CombatScene → CameraDirector |
| 8 | 경로 C 연결: combat_rules 평가 + suppress 처리 |
| 9 | 경로 A 연결: SkillSystem → CameraDirector |
| 10 | _apply_cinematic (battle_intro / closeup / execute / victory) |
| 11 | restore_on_end 복귀 처리 |
| 12 | 카메라 상태 머신 완성 |
