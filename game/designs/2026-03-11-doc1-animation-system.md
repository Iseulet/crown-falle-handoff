# CrownFalle — 문서 1: 애니메이션 시스템 구축

- Date: 2026-03-11
- 범위: 애니 연결 구조 / 데이터 전환 / 공격-피격 동기화 / 상황별 모션 출력
- 대상: PC + 몬스터 동일 구조

---

## 1. 전체 아키텍처

```
[ CombatUnit.gd ]
  play_action("basic_attack")
        │
        ▼
[ ActionResolver ]
  유닛의 class_data.actions 조회
  "basic_attack" → motion_name: "attack_melee"
        │
        ▼
[ AnimationConfig (Singleton) ]
  animation_config.json 에서 motion 설정 조회
  → events[], loop, speed_scale
        │
        ▼
[ UnitRenderer3D.gd ]
  play_motion(motion_name, config)
  AnimationPlayer.play("attack_melee/attack_melee")
        │
        ▼
[ AnimEventDispatcher ]
  타임라인 재생 중 events[] 순서대로 시그널 발행
  → hit_frame / spawn_fx / play_sfx / camera_effect ...
```

---

## 2. 데이터 구조

### 2-1. data/animations/animation_config.json
모션 단위 설정. 모든 유닛이 공유.

```json
{
  "motions": {
    "idle":         { "loop": true,  "speed_scale": 1.0, "events": [] },
    "move":         { "loop": true,  "speed_scale": 1.0, "events": [] },
    "attack_melee": {
      "loop": false, "speed_scale": 1.0,
      "events": [
        { "ratio": 0.30, "type": "play_sfx",      "data": { "sfx": "sword_swing" } },
        { "ratio": 0.55, "type": "hit_frame",     "data": { "damage_multiplier": 1.0 } },
        { "ratio": 0.55, "type": "spawn_fx",      "data": { "fx": "slash_impact", "bone": "hand_r" } },
        { "ratio": 0.55, "type": "camera_effect", "data": { "preset": "shake_light" } }
      ]
    },
    "attack_ranged": {
      "loop": false, "speed_scale": 1.0,
      "events": [
        { "ratio": 0.40, "type": "spawn_projectile", "data": { "type": "arrow", "bone": "hand_r" } },
        { "ratio": 0.40, "type": "play_sfx",         "data": { "sfx": "arrow_shoot" } }
      ]
    },
    "cast_spell": {
      "loop": false, "speed_scale": 1.0,
      "events": [
        { "ratio": 0.20, "type": "attach_fx",    "data": { "fx": "magic_charge", "bone": "hand_r" } },
        { "ratio": 0.80, "type": "hit_frame",    "data": { "damage_multiplier": 1.0 } },
        { "ratio": 0.80, "type": "spawn_fx",     "data": { "fx": "spell_burst",  "target": true } },
        { "ratio": 0.80, "type": "detach_fx",    "data": { "fx": "magic_charge" } },
        { "ratio": 0.80, "type": "play_sfx",     "data": { "sfx": "spell_cast" } }
      ]
    },
    "hit_light":  { "loop": false, "events": [] },
    "hit_heavy":  { "loop": false, "events": [
        { "ratio": 0.10, "type": "camera_effect", "data": { "preset": "shake_heavy" } }
    ]},
    "dead_hit":   { "loop": false, "events": [
        { "ratio": 0.30, "type": "play_sfx",  "data": { "sfx": "death_grunt" } },
        { "ratio": 1.00, "type": "on_death_complete", "data": {} }
    ]}
  }
}
```

### 2-2. data/classes/{class}.json — actions 필드
클래스별 action → motion 매핑. 코드는 action 이름만 알면 됨.

```json
// fighter.json (예시)
{
  "class_id": "fighter",
  "actions": {
    "idle":         { "motion": "idle" },
    "move":         { "motion": "move" },
    "basic_attack": { "motion": "attack_melee" },
    "skill_attack": { "motion": "attack_skill_triple" },
    "hit_light":    { "motion": "hit_light" },
    "hit_heavy":    { "motion": "hit_heavy" },
    "hit_dot":      { "motion": "hit_dot" },
    "stun_enter":   { "motion": "staggered" },
    "stun_loop":    { "motion": "stun_loop" },
    "knockback":    { "motion": "knockback" },
    "dead_hit":     { "motion": "dead_hit" },
    "dead_dot":     { "motion": "dead_dot" },
    "dead_execute": { "motion": "dead_execute" },
    "buff":         { "motion": "buff" },
    "debuff":       { "motion": "debuff" },
    "victory":      { "motion": "victory" },
    "defeat":       { "motion": "defeat" }
  }
}
```

각 직업별 차이:
| action | fighter | archer | mage | rogue |
|--------|---------|--------|------|-------|
| basic_attack | attack_melee | attack_ranged | cast_spell | attack_melee |
| skill_attack | attack_skill_triple | attack_ranged | cast_aoe | attack_melee_heavy |

### 2-3. data/enemies/{enemy_id}.json — 동일 구조
몬스터도 actions 필드 동일하게 적용. 코드 수정 없이 JSON만 추가.

```json
// enemy_fighter.json
{
  "enemy_id": "enemy_fighter",
  "actions": {
    "basic_attack": { "motion": "attack_melee" },
    "hit_light":    { "motion": "hit_light" },
    "dead_hit":     { "motion": "dead_hit" }
    // ...동일 구조
  }
}
```

---

## 3. AnimState 전체 정의

```gdscript
enum AnimState {
    IDLE,
    MOVE,
    ATTACK_MELEE,
    ATTACK_RANGED,
    ATTACK_SKILL,
    CAST,
    HIT_LIGHT,
    HIT_HEAVY,
    HIT_DOT,
    STAGGERED,
    STUN_LOOP,
    KNOCKBACK,
    BUFF,
    DEBUFF,
    DEAD_HIT,
    DEAD_DOT,
    DEAD_EXECUTE,
    VICTORY,
    DEFEAT,
}
```

---

## 4. 공격-피격 동기화

### 4-1. 단타 공격
```
공격자                              피격자
  │                                    │
  ├─ play_action("basic_attack")       │
  │                                    │
  │  [ratio 0.30] play_sfx ─────────► 효과음 재생
  │  [ratio 0.55] hit_frame ─────────► receive_hit(damage, "attack")
  │               spawn_fx  ─────────►   └─ HP 잔량에 따라
  │               camera_effect             hit_light or hit_heavy
  │                                        HP 0이면 → dead_hit
  │                                    │
  ├─ 모션 완료 → IDLE                  ├─ HIT 완료 → IDLE
  ▼                                    ▼
```

### 4-2. 멀티히트 스킬
```json
"attack_skill_triple": {
  "events": [
    { "ratio": 0.30, "type": "hit_frame", "data": { "damage_multiplier": 0.4 } },
    { "ratio": 0.55, "type": "hit_frame", "data": { "damage_multiplier": 0.4 } },
    { "ratio": 0.75, "type": "hit_frame", "data": { "damage_multiplier": 0.6 } }
  ]
}
```

### 4-3. AoE 스킬
hit_frame 이벤트 수신 시 범위 내 모든 유닛에 동시 receive_hit() 호출.
```
시전자 [ratio 0.80 hit_frame] → CombatScene이 범위 내 타겟 목록 순회
                               → 각 타겟.receive_hit() 동시 호출
```

---

## 5. 상황별 모션 출력 규칙

### 5-1. 피격 모션 자동 선택
```gdscript
func receive_hit(damage: int, cause: String) -> void:
    var ratio = float(damage) / float(max_hp)
    if cause == "dot":
        play_action("hit_dot")
    elif ratio >= 0.30:
        play_action("hit_heavy")
    else:
        play_action("hit_light")
```

### 5-2. 사망 모션 자동 선택
```gdscript
func die(cause: String) -> void:
    match cause:
        "attack":  play_action("dead_hit")
        "dot":     play_action("dead_dot")
        "execute": play_action("dead_execute")
```

### 5-3. 상태이상 루프
상태이상은 IDLE을 대체하는 루프 모션으로 처리.
도트 틱마다 hit_dot를 큐에 삽입 후 루프 복귀.
```
정상:  [IDLE loop]
기절:  [STUN_LOOP] ── 해제 ──► [IDLE loop]
화염:  [BURN_LOOP] + 매 틱 [HIT_DOT 큐 삽입]
```

### 5-4. AnimationQueue
DEAD 상태는 모든 큐 무시. 그 외 재생 중이면 큐에 삽입 후 순차 처리.
```gdscript
func push_anim(state: AnimState) -> void:
    if _is_dead(): return
    if _current_state == AnimState.IDLE:
        set_state(state)
    else:
        _anim_queue.append(state)

func _on_animation_finished() -> void:
    if _anim_queue.is_empty():
        set_state(AnimState.IDLE)
    else:
        set_state(_anim_queue.pop_front())
```

---

---

## 6. 컴포지션 시스템(Doc 3) 영향으로 인한 수정 사항

### 6-1. 아키텍처 분기 추가 ⚠️

현재 아키텍처는 `AnimationConfig → UnitRenderer3D → AnimationPlayer.play()` 단일 경로.
컴포지션은 `AnimationTree`를 통해 재생해야 하므로 **분기가 필요**.

```
[ AnimationConfig (Singleton) ]
  animation_config.json 조회
        │
        ├─ 단순 모션 ──────────────────────────────────┐
        │   (motions 키에 존재)                         ▼
        │                               [ UnitRenderer3D ]
        │                               AnimationPlayer.play()
        │
        └─ 컴포지션 ──────────────────────────────────┐
            (compositions 키에 존재)                    ▼
            [ CompositionBuilder ]       [ UnitRenderer3D ]
            AnimationTree 동적 생성  →  AnimationTree.active = true
```

```gdscript
# UnitRenderer3D.gd — play_motion() 수정 필요
func play_motion(motion_name: String) -> void:
    if AnimationConfig.is_composition(motion_name):
        var tree = CompositionBuilder.build(
            AnimationConfig.get_composition(motion_name)
        )
        _anim_tree.tree_root = tree
        _anim_tree.active = true
        _anim_player.active = false   # AnimationPlayer 비활성화
    else:
        _anim_player.active = true
        _anim_tree.active = false     # AnimationTree 비활성화
        _anim_player.play(motion_name)
```

### 6-2. AnimationQueue — 완료 감지 방식 분기 ⚠️

현재 `_on_animation_finished`는 `AnimationPlayer.animation_finished` 시그널에만 연결.
`AnimationTree` 사용 시 완료 감지가 다름.

```gdscript
# UnitRenderer3D.gd — 초기화 시 양쪽 연결
func _ready() -> void:
    _anim_player.animation_finished.connect(_on_motion_complete)
    _anim_tree.animation_finished.connect(_on_motion_complete)  # 추가

func _on_motion_complete(_anim_name: String) -> void:
    if _anim_queue.is_empty():
        play_motion("idle")
    else:
        play_motion(_anim_queue.pop_front())
```

### 6-3. AnimState enum — SKILL 상태 범용화 ⚠️

스킬이 늘어날수록 `ATTACK_SKILL_A`, `ATTACK_SKILL_B` ... enum이 폭발.
컴포지션 기반 스킬은 `COMPOSITION` 단일 상태로 통합하고 motion 이름을 별도 보관.

```gdscript
# 수정 전
enum AnimState { ..., ATTACK_SKILL, ... }

# 수정 후
enum AnimState {
    IDLE, MOVE,
    ATTACK_MELEE, ATTACK_RANGED, CAST,   # 기본 단일 모션
    COMPOSITION,                          # 컴포지션 모션 전체 통합
    HIT_LIGHT, HIT_HEAVY, HIT_DOT,
    STAGGERED, STUN_LOOP, KNOCKBACK,
    BUFF, DEBUFF,
    DEAD_HIT, DEAD_DOT, DEAD_EXECUTE,
    VICTORY, DEFEAT,
}

# AnimState와 별도로 현재 재생 중인 motion 이름 보관
var _current_motion_name: String = ""

func set_state(state: AnimState, motion: String = "") -> void:
    _current_state = state
    _current_motion_name = motion
    play_motion(motion if motion != "" else _state_to_default_motion(state))
```

### 6-4. hit_frame ratio — Sequence 구간 재계산 ⚠️

`attack_skill_triple` 같은 멀티히트 스킬이 Sequence 컴포지션으로 전환되면,
`animation_config.json`에 정의된 `ratio`가 **소스 모션 기준**인지 **컴포지션 전체 기준**인지 불일치 발생.

**정책 확정: events는 항상 소스 모션(단순 모션) 기준의 ratio로 정의.**
컴포지션 재생 시 `AnimEventDispatcher`가 현재 구간의 소스 모션 기준 ratio로 변환해서 처리.

```
컴포지션 전체:  [─ move 0.4s ─][블렌드 0.1s][─ attack_melee 1.2s ─]
                ↑ 이 구간 내 elapsed를 각 소스 모션 길이로 나눠 ratio 계산

move 구간:       elapsed / 0.4  = 소스 ratio
attack 구간:     elapsed / 1.2  = 소스 ratio  → hit_frame 0.55 비교
```

→ Doc 2 `AnimEventDispatcher`도 구간 정보를 받아야 함 (영향 있음, Doc 2 참조).

### 6-5. AnimationConfig 싱글톤 — compositions 통합 로드

```gdscript
# AnimationConfig.gd — 수정 필요
var _motions: Dictionary = {}
var _compositions: Dictionary = {}

func _ready() -> void:
    var config = _load_json("res://data/animations/animation_config.json")
    _motions = config.get("motions", {})

    # compositions 폴더 전체 스캔해서 로드
    var dir = DirAccess.open("res://data/animations/compositions/")
    for fname in dir.get_files():
        var name = fname.replace(".json", "")
        _compositions[name] = _load_json(
            "res://data/animations/compositions/" + fname
        )

func is_composition(motion_name: String) -> bool:
    return _compositions.has(motion_name)

func get_motion(motion_name: String) -> Dictionary:
    return _motions.get(motion_name, {})

func get_composition(motion_name: String) -> Dictionary:
    return _compositions.get(motion_name, {})
```

---

## 7. ANIM_PATHS 키 네이밍 규칙 (2026-03-11 추가)

### 7-1. 규칙

`UnitRenderer3D.ANIM_PATHS`의 키 이름은 **animation_config.json의 motion 이름과 완전히 일치**해야 한다.
GLB 파일 이름과 무관하게 키 이름이 AnimationPlayer 라이브러리 이름으로 등록됨.

### 7-2. 올바른 매핑 (확정)

| ANIM_PATHS 키 (= AnimationPlayer 라이브러리명) | GLB 파일 | animation_config.json motion |
|----------------------------------------------|----------|-------------------------------|
| `idle` | `idle.glb` | `idle` |
| `move` | `move.glb` | `move` |
| `attack_melee` | `attack.glb` | `attack_melee` |
| `hit_light` | `hitted.glb` | `hit_light` |
| `dead_hit` | `death.glb` | `dead_hit` |

### 7-3. 잘못된 패턴 (금지)

```gdscript
# ❌ GLB 파일명 그대로 키로 쓰면 안됨
"attack": "res://assets/animations/fighter/attack.glb"   # attack/attack 으로 등록됨
"hitted": "res://assets/animations/fighter/hitted.glb"  # hitted/hitted 으로 등록됨
"death":  "res://assets/animations/fighter/death.glb"   # death/death 으로 등록됨

# ✅ animation_config.json motion 이름과 일치시켜야 함
"attack_melee": "res://assets/animations/fighter/attack.glb"
"hit_light":    "res://assets/animations/fighter/hitted.glb"
"dead_hit":     "res://assets/animations/fighter/death.glb"
```

---

## 8. 구현 순서 (수정)

| 순위 | 항목 |
|------|------|
| 1 | animation_config.json 생성 |
| 2 | class_config / enemy_data 에 actions 필드 추가 |
| 3 | AnimationConfig 싱글톤 (motions + compositions 통합 로드) |
| 4 | CombatUnit.play_action() — action → motion 해석 |
| 5 | UnitRenderer3D.play_motion() — **단순/컴포지션 분기** |
| 6 | AnimationQueue — **AnimationPlayer + AnimationTree 양쪽 완료 감지** |
| 7 | 공격-피격 비동기 타이밍 연동 (CombatScene) |
| 8 | receive_hit() 피해량 분기 (light/heavy/dot) |
| 9 | AnimState enum COMPOSITION 통합 |
| 10 | 상태이상 루프 (Phase B 이후) |
