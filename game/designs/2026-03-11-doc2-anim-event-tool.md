# CrownFalle — 문서 2: 애니메이션 이벤트 구조 & Godot 툴

- Date: 2026-03-11
- 범위: AnimNotify 구조 설계 / 에디터 플러그인 제작

---

## 1. 이벤트 타입 전체 정의

UE AnimNotify에 대응하는 CrownFalle AnimEvent 시스템.

| 카테고리 | type | 설명 | data 필드 |
|----------|------|------|-----------|
| **전투** | hit_frame | 피해 적용 | damage_multiplier |
| | enable_hitbox | 히트박스 활성화 구간 시작 | radius |
| | disable_hitbox | 히트박스 비활성화 | — |
| | apply_status | 상태이상 적용 | status, duration |
| | spawn_projectile | 투사체 생성 | type, bone, speed |
| **FX** | spawn_fx | 이펙트 생성 | fx, bone, target |
| | attach_fx | 본에 FX 부착 (루프) | fx, bone |
| | detach_fx | 부착 FX 제거 | fx |
| **사운드** | play_sfx | 효과음 재생 | sfx, volume |
| **카메라** | camera_effect | 카메라 효과 (Doc 5 프리셋 참조) | preset |
| **이동** | root_motion | 유닛 위치 강제 이동 | direction, distance |
| **시스템** | on_death_complete | 사망 모션 완료 알림 | — |
| | anim_complete | 모션 완료 알림 (일반) | — |

---

## 2. 이벤트 데이터 구조

```json
// animation_config.json 내 events 배열
{
  "ratio": 0.55,          // 모션 전체 길이 대비 발생 시점 (0.0 ~ 1.0)
  "type": "hit_frame",    // 이벤트 타입
  "data": {               // 타입별 추가 데이터
    "damage_multiplier": 1.0
  }
}
```

### ratio 계산
```
ratio = 이벤트 발생 시간(초) / 모션 전체 길이(초)
예) 전체 1.2초 모션, 0.66초 지점 타격 → ratio = 0.55
```

---

## 2-1. 투사체 데이터 구조

`spawn_projectile` 이벤트의 `type` 필드가 참조하는 데이터.

```json
// data/projectiles/arrow.json
{
  "type": "arrow",
  "speed": 12.0,
  "arc": false,
  "fx_trail": "fx_arrow_trail",
  "fx_hit":   "fx_arrow_impact",
  "sfx_hit":  "sfx_arrow_hit"
}

// data/projectiles/magic_bolt.json
{
  "type": "magic_bolt",
  "speed": 10.0,
  "arc": false,
  "fx_trail": "fx_magic_trail",
  "fx_hit":   "fx_magic_burst",
  "sfx_hit":  "sfx_magic_hit"
}

// data/projectiles/throw_stone.json
{
  "type": "throw_stone",
  "speed": 8.0,
  "arc": true,
  "arc_height": 2.0,
  "fx_trail": "",
  "fx_hit":   "fx_dust_hit",
  "sfx_hit":  "sfx_stone_hit"
}
```

**spawn_projectile 이벤트 예시:**
```json
{
  "ratio": 0.55,
  "type": "spawn_projectile",
  "data": {
    "type": "arrow",
    "bone": "RightHand",
    "speed_override": 0.0
  }
}
```

투사체 비행이 완료되어 타겟에 도달하면 `CombatScene`이 타겟의 `receive_hit()` 호출.
투사체 비행 중 공격자는 즉시 idle 복귀 가능 (Doc 4 섹션 4-2 참조).

```
data/projectiles/
  arrow.json
  magic_bolt.json
  throw_stone.json
```

---

```gdscript
# scripts/animation/AnimEventDispatcher.gd
class_name AnimEventDispatcher
extends Node

signal event_triggered(event: Dictionary)

var _motion_config: Dictionary = {}
var _anim_player: AnimationPlayer
var _pending_events: Array = []
var _elapsed: float = 0.0
var _total_length: float = 0.0
var _is_playing: bool = false

func setup(anim_player: AnimationPlayer, motion_name: String) -> void:
    _anim_player = anim_player
    _motion_config = AnimationConfig.get_motion(motion_name)
    _pending_events = _motion_config.get("events", []).duplicate()
    _pending_events.sort_custom(func(a, b): return a.ratio < b.ratio)
    _elapsed = 0.0
    _total_length = anim_player.current_animation_length
    _is_playing = true

func _process(delta: float) -> void:
    if not _is_playing: return
    _elapsed += delta
    var current_ratio = _elapsed / _total_length

    while not _pending_events.is_empty():
        var next = _pending_events[0]
        if current_ratio >= next.ratio:
            event_triggered.emit(next)
            _pending_events.pop_front()
        else:
            break

func stop() -> void:
    _is_playing = false
    _pending_events.clear()
```

---

## 4. Godot 에디터 플러그인 — HitFrame Editor

### 4-1. 플러그인 구조

```
addons/anim_event_editor/
  plugin.cfg
  plugin.gd               ← EditorPlugin 등록
  anim_event_panel.gd     ← 패널 메인 로직
  anim_event_panel.tscn   ← UI 레이아웃
  event_row.gd            ← 이벤트 행 컴포넌트
  event_row.tscn
```

### 4-2. UI 레이아웃

```
┌─ Anim Event Editor ────────────────────────────────────────────┐
│                                                                 │
│  Class: [fighter ▼]    Motion: [attack_melee ▼]               │
│                                                                 │
│  ┌─ 미리보기 ─────────────────────────────────────────────┐    │
│  │  [◀◀] [▶] [■]   ──────●──────────────  0.66s / 1.20s  │    │
│  │  ratio: 0.550          speed: [1.0 ▼]                  │    │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  이벤트 목록                          [+ 현재 위치에 추가]      │
│  ┌──────┬──────────────┬─────────────────────────┬───────┐     │
│  │ratio │ type         │ data                    │       │     │
│  ├──────┼──────────────┼─────────────────────────┼───────┤     │
│  │ 0.30 │ play_sfx     │ sfx: sword_swing        │ [삭제]│     │
│  │ 0.55 │ hit_frame    │ damage_multiplier: 1.0  │ [삭제]│     │
│  │ 0.55 │ spawn_fx     │ fx: slash_impact        │ [삭제]│     │
│  │ 0.55 │ camera_effect│ preset: shake_light     │ [삭제]│     │
│  └──────┴──────────────┴─────────────────────────┴───────┘     │
│                                                                 │
│  [저장 → animation_config.json]                                │
└─────────────────────────────────────────────────────────────────┘
```

### 4-3. 이벤트 타입 Enum (드롭다운)

type 필드는 string 직접 입력이 아니라 **OptionButton 드롭다운**으로 표시.
코드에서 enum으로 관리하고, 드롭다운 선택 시 data 입력 영역 자동 전환.

```gdscript
# anim_event_panel.gd

enum EventType {
    HIT_FRAME,          # 전투 — 피해 적용
    ENABLE_HITBOX,      # 전투 — 히트박스 활성화
    DISABLE_HITBOX,     # 전투 — 히트박스 비활성화
    APPLY_STATUS,       # 전투 — 상태이상 적용
    SPAWN_PROJECTILE,   # 전투 — 투사체 생성
    SPAWN_FX,           # FX — 이펙트 생성
    ATTACH_FX,          # FX — 본에 FX 부착
    DETACH_FX,          # FX — 부착 FX 제거
    PLAY_SFX,           # 사운드 — 효과음 재생
    CAMERA_EFFECT,      # 카메라 — Doc 5 프리셋 참조 (camera_config.json)
    ROOT_MOTION,        # 이동 — 강제 이동
    ON_DEATH_COMPLETE,  # 시스템 — 사망 완료
    ANIM_COMPLETE,      # 시스템 — 모션 완료
}

const EVENT_TYPE_LABELS = {
    EventType.HIT_FRAME:         "hit_frame",
    EventType.ENABLE_HITBOX:     "enable_hitbox",
    EventType.DISABLE_HITBOX:    "disable_hitbox",
    EventType.APPLY_STATUS:      "apply_status",
    EventType.SPAWN_PROJECTILE:  "spawn_projectile",
    EventType.SPAWN_FX:          "spawn_fx",
    EventType.ATTACH_FX:         "attach_fx",
    EventType.DETACH_FX:         "detach_fx",
    EventType.PLAY_SFX:          "play_sfx",
    EventType.CAMERA_EFFECT:     "camera_effect",
    EventType.ROOT_MOTION:       "root_motion",
    EventType.ON_DEATH_COMPLETE: "on_death_complete",
    EventType.ANIM_COMPLETE:     "anim_complete",
}
```

### 4-4. data 입력 필드 — 타입별 자동 전환

type 드롭다운 선택 시 data 영역을 해당 타입에 맞는 위젯으로 교체.
각 필드는 **입력 성격에 따라 위젯 타입이 다름.**

```
┌─ 이벤트 편집 ──────────────────────────────────────────────────┐
│  ratio: [0.550]   type: [ spawn_fx           ▼]               │
│                                                                 │
│  data:                                                          │
│    fx:     [ slash_impact        ▼ 🔍 ]  ← FX 리소스 드롭다운 │
│    bone:   [ hand_r              ▼ ]     ← 본 이름 드롭다운    │
│    target: [ ○ self  ● target ]          ← 라디오 버튼         │
└─────────────────────────────────────────────────────────────────┘
```

#### 필드 타입 정의표

| type | 필드명 | 위젯 | 데이터 소스 |
|------|--------|------|-------------|
| hit_frame | damage_multiplier | SpinBox (float) | — |
| enable_hitbox | radius | SpinBox (float) | — |
| apply_status | status | OptionButton + 🔍 | **StatusEffect enum** |
| | duration | SpinBox (float) | — |
| spawn_projectile | type | OptionButton + 🔍 | **Projectile 리소스 목록** |
| | bone | OptionButton | **현재 모델 본 이름 목록** |
| | speed | SpinBox (float) | — |
| spawn_fx | fx | OptionButton + 🔍 | **FX 리소스 목록** |
| | bone | OptionButton | **현재 모델 본 이름 목록** |
| | target | OptionButton | self / target / position |
| attach_fx | fx | OptionButton + 🔍 | **FX 리소스 목록** |
| | bone | OptionButton | **현재 모델 본 이름 목록** |
| detach_fx | fx | OptionButton + 🔍 | **FX 리소스 목록** (attach된 것만) |
| play_sfx | sfx | OptionButton + 🔍 | **SFX 리소스 목록** |
| | volume | SpinBox (float, 0~1) | — |
| camera_effect | preset | OptionButton + 🔍 | **camera_config.json presets 키 목록** |
| root_motion | direction | Vector3 입력 (x/y/z) | — |
| | distance | SpinBox (float) | — |

#### 드롭다운 + 검색 (🔍) 위젯 동작

리소스 필드는 **LineEdit + OptionButton 조합**으로 검색 가능하게 구현:

```
[ slash_impact    🔍 ]
  ↓ 타이핑 시 필터링
  ┌──────────────────┐
  │ slash_impact     │ ← 현재 선택
  │ slash_trail      │
  │ slash_heavy      │
  └──────────────────┘
```

```gdscript
# 검색 가능한 드롭다운 구현
func _build_searchable_dropdown(
    parent: Control,
    items: Array[String],
    on_selected: Callable
) -> HBoxContainer:
    var hbox = HBoxContainer.new()
    var line_edit = LineEdit.new()
    var popup = PopupMenu.new()

    line_edit.text_changed.connect(func(text):
        popup.clear()
        for item in items:
            if text.is_empty() or item.contains(text):
                popup.add_item(item)
        popup.popup()
    )
    popup.id_pressed.connect(func(id):
        line_edit.text = popup.get_item_text(id)
        on_selected.call(line_edit.text)
    )
    hbox.add_child(line_edit)
    hbox.add_child(popup)
    return hbox
```

### 4-5. Composition Editor 탭 — 블렌드 파라미터 편집 UI

```
┌─ Anim Event Editor ──────────────────────────────────────────────┐
│  [Event Editor 탭]  [Composition Editor 탭]                      │
├──────────────────────────────────────────────────────────────────┤
│  Composition: [ skill_fighter_charge_slash ▼ ]  [+신규] [삭제]   │
│  Type:        [ sequence ▼ ]                                     │
│                                                                   │
│  ┌─ 모션 목록 ─────────────────────────────────────────────────┐  │
│  │  #  motion            bone_mask      weight  trim_s  trim_e │  │
│  │  1  [move       ▼]   [full_body ▼]  [1.0 ]  [0.00]  [0.40]│  │
│  │  2  [attack_melee▼]  [full_body ▼]  [1.0 ]  [0.00]  [1.00]│  │
│  │  [+ 모션 추가]                                              │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌─ 트랜지션 (모션 간 블렌드) ──────────────────────────────────┐  │
│  │  from → to  blend_time  blend_mode       from_trim  to_trim │  │
│  │  1 → 2      [0.10s   ]  [match_pose ▼]   [0.85   ]  [0.05]│  │
│  │                                                              │  │
│  │  ┌─ 블렌드 커브 미리보기 ─────────────────────────────────┐ │  │
│  │  │                                                         │ │  │
│  │  │   1.0 ─ from ╲                                         │ │  │
│  │  │               ╲___                                     │ │  │
│  │  │   0.0 ─      ___╱─ to                                  │ │  │
│  │  │         blend_time                                      │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌─ 미리보기 ────────────────────────────────────────────────┐    │
│  │  [◀◀] [▶] [■]   ─────●──────────────   0.66s / 1.50s    │    │
│  └───────────────────────────────────────────────────────────┘    │
│                                                                   │
│  [저장 → compositions/{name}.json]                               │
└──────────────────────────────────────────────────────────────────┘
```

#### 타입별 표시 파라미터

**Sequence 타입 선택 시:**
```
┌─ 트랜지션 [1 → 2] ────────────────────────────────────────────┐
│                                                                 │
│  blend_time:   [0.10] 초                                        │
│  blend_mode:   [ match_pose ▼ ]  cut / linear / ease_in_out /  │
│                                  match_pose                     │
│  from_trim:    [0.85]   ← from 모션의 블렌드 시작 지점 (ratio) │
│  to_trim:      [0.05]   ← to 모션의 블렌드 시작 지점 (ratio)   │
│                                                                 │
│  ┌─ 커브 미리보기 ─────────────────────────────────────────┐   │
│  │  (blend_mode에 따라 실시간 커브 그래프 표시)            │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Blend 타입 선택 시:**
```
┌─ 블렌드 파라미터 ──────────────────────────────────────────────┐
│                                                                 │
│  blend_duration:  [0.30] 초                                     │
│                                                                 │
│  모션별 가중치:                                                  │
│  [cast_spell]  weight: [0.60]  curve: [ ease_out ▼ ]           │
│                ──────────────────────────────────────           │
│                ↑ 고정 weight 또는 weight_curve 중 선택          │
│                (curve 선택 시 weight 비활성화)                   │
│                                                                 │
│  [hit_light ]  weight: [0.40]  curve: [ ease_in  ▼ ]           │
│                                                                 │
│  ┌─ 가중치 커브 미리보기 ──────────────────────────────────┐   │
│  │  cast_spell: 1.0 ──╲___                                 │   │
│  │  hit_light:  0.0 ___╱──                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Layer 타입 선택 시:**
```
┌─ 레이어 파라미터 ──────────────────────────────────────────────┐
│                                                                 │
│  Spine 블렌드:                                                   │
│    spine_bone:     [ Spine ▼ ]  ← 현재 모델 본 이름 드롭다운   │
│    lower_weight:   [0.30]                                       │
│    upper_weight:   [0.70]                                       │
│    (합계 자동 표시: 1.00 ✅ / 1.00 초과 시 ⚠️ 경고)            │
│                                                                 │
│  레이어별:                                                       │
│  [lower_body / move      ]  blend_in: [0.10]  blend_out: [0.10]│
│  [upper_body / attack    ]  blend_in: [0.10]  blend_out: [0.10]│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 블렌드 커브 미리보기 구현

```gdscript
# blend_curve_preview.gd — Control 노드 상속, _draw() 오버라이드

func _draw() -> void:
    var w = size.x
    var h = size.y
    var steps = 60

    # from 모션 가중치 곡선 (1→0)
    var from_points = PackedVector2Array()
    # to 모션 가중치 곡선 (0→1)
    var to_points = PackedVector2Array()

    for i in steps:
        var t = float(i) / float(steps - 1)
        var from_w = 1.0 - _evaluate_curve(_blend_mode, t)
        var to_w   = _evaluate_curve(_blend_mode, t)

        from_points.append(Vector2(t * w, h - from_w * h))
        to_points.append(Vector2(t * w, h - to_w * h))

    draw_polyline(from_points, Color(0.4, 0.7, 1.0), 2.0)  # 파랑: from
    draw_polyline(to_points,   Color(1.0, 0.6, 0.2), 2.0)  # 주황: to

    # blend_time 구간 표시 (세로선)
    var blend_x = _from_trim * w
    draw_line(Vector2(blend_x, 0), Vector2(blend_x, h),
              Color(1, 1, 0, 0.5), 1.0)
```

#### 파라미터 변경 시 JSON 실시간 반영

```gdscript
# 어떤 파라미터가 변경돼도 즉시 내부 composition dict 업데이트
func _on_any_param_changed(_value) -> void:
    _composition["transitions"][_selected_transition_idx] = {
        "from":       _from_spin.value,
        "to":         _to_spin.value,
        "blend_time": _blend_time_spin.value,
        "blend_mode": _blend_mode_names[_blend_mode_option.selected],
        "from_trim":  _from_trim_spin.value,
        "to_trim":    _to_trim_spin.value,
    }
    _curve_preview.set_params(_blend_mode_option.selected,
                               _from_trim_spin.value)
    _curve_preview.queue_redraw()  # 커브 미리보기 즉시 갱신
```

각 드롭다운이 참조하는 데이터 소스:

| 필드 | 소스 | 로드 방법 |
|------|------|-----------|
| fx | `assets/fx/` 폴더 내 .tscn 파일 목록 | `DirAccess.get_files_at()` |
| sfx | `assets/sfx/` 폴더 내 .wav/.ogg 파일 목록 | `DirAccess.get_files_at()` |
| type (projectile) | `data/projectiles/` JSON 키 목록 | JSON 파싱 |
| status | `StatusEffect enum` 키 목록 (GDScript) | enum 반영 |
| bone | 현재 선택된 모델 GLB의 본 이름 목록 | GLB Skeleton3D 파싱 |
| motion (모션 카테고리) | `animation_config.json` motions 키 목록 | JSON 파싱 |

```gdscript
# 본 이름 목록 추출 (GLB Skeleton3D에서)
func _get_bone_names_from_model(class_name: String) -> Array[String]:
    var path = "res://assets/models/units/%s_rigged.glb" % class_name
    var scene = load(path).instantiate()
    var skeleton = _find_skeleton(scene)
    var bones: Array[String] = []
    for i in skeleton.get_bone_count():
        bones.append(skeleton.get_bone_name(i))
    scene.queue_free()
    return bones
```

### 4-6. 핵심 기능 구현 가이드

```gdscript
# type 드롭다운 선택 시 data 영역 재빌드
func _on_event_type_selected(type_idx: int) -> void:
    _clear_data_area()
    match type_idx:
        EventType.HIT_FRAME:
            _add_spinbox("damage_multiplier", 1.0, 0.0, 10.0)
        EventType.SPAWN_FX:
            _add_searchable_dropdown("fx", _fx_list)
            _add_option_button("bone", _bone_list)
            _add_option_button("target", ["self", "target", "position"])
        EventType.PLAY_SFX:
            _add_searchable_dropdown("sfx", _sfx_list)
            _add_spinbox("volume", 1.0, 0.0, 1.0)
        EventType.APPLY_STATUS:
            _add_searchable_dropdown("status", _status_list)
            _add_spinbox("duration", 1.0, 0.1, 10.0)
        EventType.CAMERA_EFFECT:
            _add_searchable_dropdown("preset", _camera_preset_list)
            # _camera_preset_list: camera_config.json presets 키 목록
        # ...

# animation_config.json 저장
func _on_save_pressed() -> void:
    var config = _load_json("res://data/animations/animation_config.json")
    var motion_name = _motion_dropdown.get_selected_text()
    config["motions"][motion_name]["events"] = _collect_events()
    _save_json("res://data/animations/animation_config.json", config)
    print("[AnimEventEditor] 저장 완료: ", motion_name)
```

---

---

## 5. 컴포지션 시스템(Doc 3) 영향으로 인한 수정 사항

### 5-1. AnimEventDispatcher — 구간 ratio 재계산 ⚠️

현재 구현은 `_total_length = anim_player.current_animation_length` 단일 기준.
Sequence 컴포지션 재생 시 `AnimationPlayer`는 전체 합산 길이를 모르므로,
**구간(segment) 정보를 별도로 받아서 소스 모션 기준 ratio로 변환** 필요.

```gdscript
# AnimEventDispatcher.gd — 수정

# 단순 모션용 setup (기존)
func setup_simple(anim_player: AnimationPlayer, motion_name: String) -> void:
    _segments = [{
        "motion": motion_name,
        "start_time": 0.0,
        "length": anim_player.get_animation(motion_name).length,
        "events": AnimationConfig.get_motion(motion_name).get("events", [])
    }]
    _total_length = _segments[0].length
    _is_playing = true

# 컴포지션용 setup (신규) — CompositionBuilder에서 구간 정보 전달
func setup_composition(segments: Array) -> void:
    # segments = [
    #   { "motion": "move",         "start_time": 0.0,  "length": 0.4, "events": [...] },
    #   { "motion": "attack_melee", "start_time": 0.35, "length": 1.2, "events": [...] }
    # ]  ※ start_time은 블렌드 구간 고려한 실제 시작점
    _segments = segments
    _total_length = segments[-1].start_time + segments[-1].length
    _is_playing = true

func _process(delta: float) -> void:
    if not _is_playing: return
    _elapsed += delta

    for seg in _segments:
        var seg_elapsed = _elapsed - seg.start_time
        if seg_elapsed < 0.0 or seg_elapsed > seg.length: continue

        var current_ratio = seg_elapsed / seg.length  # 소스 모션 기준 ratio

        while not seg._pending.is_empty():
            var evt = seg._pending[0]
            if current_ratio >= evt.ratio:
                event_triggered.emit(evt)
                seg._pending.pop_front()
            else:
                break
```

### 5-2. Event Editor — 컴포지션 모션의 events 귀속 정책 ⚠️

**정책 확정: events는 항상 소스 모션(단순 모션)에 귀속.**
컴포지션 모션을 Event Editor에서 선택 시 소스 모션 각각의 events를 편집하도록 유도.

```
Event Editor에서 모션 선택:

  [ skill_fighter_charge_slash ▼ ]  ← 컴포지션 모션 선택

  ⚠️ 이 모션은 컴포지션입니다.
     구성 소스 모션을 선택하세요:
     [ move ▼ ]  또는  [ attack_melee ▼]
     ↓
  선택한 소스 모션의 events 편집
```

컴포지션 자체에 events를 따로 정의하지 않음 → 소스 모션 재사용 시 events도 자동 재사용.

### 5-3. Event Editor — 모션 드롭다운에 컴포지션 구분 표시 ⚠️

현재 `animation_config.json`의 motions 키만 나열.
컴포지션 모션도 드롭다운에 표시하되 **[C] 접두어로 구분**.

```
Motion: [  attack_melee              ▼ ]
        ─────────────────────────────────
        ── 단순 모션 ──────────────────
          idle
          move
          attack_melee
          attack_ranged
          cast_spell
          hit_light
          ...
        ── 컴포지션 [C] ───────────────
          [C] skill_fighter_charge_slash
          [C] skill_archer_moving_shot
          [C] skill_mage_charged_burst
          ...
```

컴포지션 선택 시 → 소스 모션 선택 안내 UI로 자동 전환 (5-2 정책 적용).

### 5-4. 플러그인 구조 확장

```
addons/anim_event_editor/
  plugin.cfg
  plugin.gd
  anim_event_panel.gd       ← 기존
  anim_event_panel.tscn
  event_row.gd
  event_row.tscn
  composition_panel.gd      ← 신규 (Doc 3 Composition Editor 탭)
  composition_panel.tscn
  blend_curve_preview.gd    ← 신규 (블렌드 커브 그래프)
  segment_info.gd           ← 신규 (구간 정보 → Dispatcher 전달용)
```

---

## 6. 구현 순서 (수정)

| 순위 | 항목 |
|------|------|
| 1 | AnimEventDispatcher — setup_simple / setup_composition 분리 |
| 2 | UnitRenderer3D에 Dispatcher 연결 (단순/컴포지션 분기) |
| 3 | 각 이벤트 타입 핸들러 구현 (hit_frame, play_sfx 우선) |
| 4 | 에디터 플러그인 기본 구조 (plugin.cfg, 패널 등록) |
| 5 | 미리보기 뷰포트 + 타임라인 슬라이더 |
| 6 | 이벤트 목록 편집 UI (단순 모션) |
| 7 | JSON 저장 기능 |
| 8 | 이벤트 타입별 data 입력 UI (드롭다운 + 검색) |
| 9 | Composition Editor 탭 추가 |
| 10 | 컴포지션 선택 시 소스 모션 전환 안내 UI |
| 11 | 블렌드 커브 미리보기 |
