# CrownFalle — 문서 4: 인게임 캐릭터 이동 & 방향 연출

- Date: 2026-03-11
- 범위: 이동 연출 / 회전 / 지향 방향 / 카메라 반응

---

## 1. 현재 상태 & 문제점

| 항목 | 현재 | 문제 |
|------|------|------|
| 이동 | Tween 직선 이동 (0.25s/칸) | 캐릭터가 항상 정면만 봄 |
| 회전 | 없음 | 이동 방향과 무관하게 고정 방향 |
| 공격 방향 | 없음 | 적을 보지 않고 공격 |
| 피격 반응 | 모션만 재생 | 공격받은 방향으로 반응 없음 |

---

## 2. 캐릭터 회전 시스템

### 2-1. 기본 원칙
- **이동 시**: 다음 이동 칸 방향으로 즉시 회전
- **공격 시**: 타겟 방향으로 회전 후 공격 모션 시작
- **피격 시**: 공격자 방향으로 회전 후 피격 모션 (선택적)
- **대기 시**: 적이 있으면 가장 가까운 적 방향으로 유지

### 2-2. 회전 구현 방식

```gdscript
# UnitRenderer3D.gd

# 즉시 회전 (이동 중 방향 전환)
func face_direction(world_pos: Vector3) -> void:
    var dir = (world_pos - global_position)
    dir.y = 0
    if dir.length() > 0.01:
        rotation.y = atan2(dir.x, dir.z)

# 부드러운 회전 (공격/대기 시)
func face_direction_smooth(world_pos: Vector3, duration: float = 0.15) -> void:
    var dir = (world_pos - global_position)
    dir.y = 0
    if dir.length() > 0.01:
        var target_rot = atan2(dir.x, dir.z)
        var tween = create_tween()
        tween.tween_property(self, "rotation:y", target_rot, duration)
```

---

## 3. 이동 연출 상세

### 3-1. 경로 이동 흐름

```
1. 목적지 클릭
2. Dijkstra로 경로 계산 → path[] 반환
3. 첫 번째 칸 방향으로 즉시 face_direction()
4. move 모션 시작
5. 경로를 한 칸씩 Tween 이동 (0.25s/칸)
   각 칸 이동 시작 시 다음 칸 방향으로 face_direction()
6. 목적지 도착 → idle 모션 복귀
7. 가장 가까운 적 방향으로 face_direction_smooth()
```

### 3-2. 이동 속도 & 타일 크기 연동

```gdscript
# 타일 1칸 = 1.0m, 이동 시간은 거리에 비례
const MOVE_TIME_PER_METER = 0.25  # 초/m

func move_to_tween(path: Array, world_positions: Array) -> void:
    for i in range(world_positions.size()):
        var next_pos = world_positions[i]
        var dist = global_position.distance_to(next_pos)
        var duration = dist * MOVE_TIME_PER_METER

        face_direction(next_pos)   # 이동 방향으로 즉시 회전
        var tween = create_tween()
        tween.tween_property(self, "global_position", next_pos, duration)
        await tween.finished

    emit_signal("move_finished")
```

### 3-3. 지형별 이동 속도 보정

```gdscript
# mud 타일: 이동 속도 감소 (애니메이션도 느려짐)
func get_move_duration(terrain_type: String) -> float:
    match terrain_type:
        "mud":      return MOVE_TIME_PER_METER * 1.5
        "elevated": return MOVE_TIME_PER_METER * 1.3
        _:          return MOVE_TIME_PER_METER
```

---

## 4. 공격 방향 연출

### 4-1. 공격 시퀀스

```
1. 타겟 방향으로 face_direction_smooth(0.15s)
2. 회전 완료 후 attack 모션 시작
3. hit_frame 이벤트 → 피해 적용
4. 공격 모션 완료 → idle 복귀
5. 가장 가까운 적 방향으로 유지
```

### 4-2. 원거리 공격 (Archer)

```
1. 타겟 방향으로 face_direction_smooth()
2. attack_ranged 모션 시작
3. spawn_projectile 이벤트 → 투사체 생성
   - 투사체는 타겟 방향으로 직선 이동
   - 투사체 도착 시 타겟 receive_hit() 호출
4. 투사체 비행 중 공격자는 idle 복귀 가능
```

---

## 5. 피격 방향 반응

### 5-1. 피격 시 방향 처리

```gdscript
func receive_hit(damage: int, cause: String, attacker: CombatUnit = null) -> void:
    # 공격자가 있으면 공격받은 방향으로 회전
    if attacker != null:
        face_direction_smooth(attacker.global_position, 0.05)

    # 피해량에 따라 모션 선택
    var ratio = float(damage) / float(max_hp)
    if ratio >= 0.30:
        play_action("hit_heavy")
    else:
        play_action("hit_light")
```

### 5-2. 넉백 처리

**정책 확정:** knockback 모션은 **타일 이동 없이 제자리 애니메이션만 재생.**
실제 타일 이동을 동반하는 넉백은 명시적으로 타일 밀어내기 스킬에서만 별도 처리.

```gdscript
# knockback: 제자리에서 모션만 재생 (타일 이동 없음)
func apply_knockback() -> void:
    push_anim("knockback")
    # 타일 위치 변경 없음
```

타일 이동을 동반하는 넉백 스킬은 스킬 처리 로직에서 grid_manager를 통해 별도로 처리.

---

## 6. 대기 상태 지향 방향

### 6-1. 적 방향 유지

턴 대기 중 IDLE 상태에서 가장 가까운 적을 바라봄.

```gdscript
# CombatUnit.gd — 매 프레임 또는 턴 시작 시 호출
func update_idle_facing() -> void:
    var nearest = _find_nearest_enemy()
    if nearest:
        _renderer.face_direction_smooth(nearest.global_position, 0.3)
```

### 6-2. 팀별 기본 방향

적이 없을 때 (전투 시작 전 등):
- 아군 팀: 오른쪽 방향 (맵 중앙을 향해)
- 적 팀: 왼쪽 방향 (맵 중앙을 향해)

---

## 7. 카메라 연출 연동

### 7-1. CameraDirector 위임 방식 (Doc 5 기준)

모든 카메라 효과는 **CameraDirector.execute()** 에 위임.
Doc 4에서 수치를 직접 정의하지 않음 — camera_config.json 프리셋 이름으로만 참조.

```gdscript
# AnimEventDispatcher → CombatScene → CameraDirector 흐름
# Doc 4는 "어떤 상황에 어떤 프리셋을 쓸지" 만 정의

# 피격 시 camera_effect 이벤트 발행 (animation_config.json에 정의)
# → CameraDirector가 프리셋 찾아 실행
#
# 예: attack_melee 모션
# { "ratio": 0.55, "type": "camera_effect", "data": { "preset": "shake_light" } }
#
# 예: attack_melee_heavy 모션
# { "ratio": 0.55, "type": "camera_effect", "data": { "preset": "shake_heavy" } }
```

### 7-2. 상황별 프리셋 매핑 (참조용)

실제 수치는 camera_config.json 에 있음. 여기선 "어떤 상황 → 어떤 프리셋" 만 정의.

| 상황 | camera_effect 프리셋 | 발동 경로 |
|------|----------------------|-----------|
| 일반 공격 피격 | shake_light | 경로 B (animation_config 이벤트) |
| 강타 피격 | shake_heavy + fov_pulse_heavy | 경로 B |
| 사망 (일반) | cinematic_closeup | 경로 C (combat_rules 조건) |
| 사망 (강타) | cinematic_execute | 경로 C (combat_rules 조건) |
| AoE 스킬 피격 | shake_aoe + fov_pulse_aoe | 경로 B |
| 공격 시 추적 | soft_track_melee / ranged / aoe | 경로 B |

→ 프리셋 수치 변경은 `data/cameras/camera_config.json` 에서만 수행.

### 7-3. 공격 시 카메라 소프트 추적

공격 대상 방향으로 살짝 추적하는 연출.
animation_config.json 의 모션 이벤트에 soft_track 프리셋으로 정의.

```json
// attack_melee 이벤트 예시
{ "ratio": 0.55, "type": "camera_effect", "data": { "preset": "soft_track_melee" } }
```

CameraDirector._apply_soft_track() 이 target_pos context를 받아 처리.
context 는 CombatScene 이 hit_frame 이벤트 수신 시 채워서 전달.

---

## 8. 구현 순서

| 순위 | 항목 |
|------|------|
| 1 | face_direction() / face_direction_smooth() 구현 |
| 2 | 이동 시 방향 전환 (move_to_tween 수정) |
| 3 | 공격 전 타겟 방향 회전 |
| 4 | 피격 시 공격자 방향 반응 |
| 5 | 대기 시 적 방향 유지 |
| 6 | 지형별 이동 속도 보정 |
| 7 | 카메라 셰이크 연동 |
| 8 | 원거리 투사체 연출 |
| 9 | 넉백 처리 |
| 10 | 카메라 소프트 추적 |

---

## 9. 방향 연출 구현 계획 (2026-03-11 추가)

### 9-1. 현재 상태 (2026-03-11 구현 완료)

`face_direction()` / `face_direction_smooth()`는 UnitRenderer3D에 구현 완료.
CombatScene.gd 4개 호출 지점 모두 구현 완료.

### 9-2. 트리거 시점별 처리 계획

| 시점 | 대상 | 방식 | 호출 위치 | 상태 |
|------|------|------|-----------|------|
| 공격 시작 | 공격자 → 대상 방향 | 즉시 (`face_direction`) | `CombatScene._attack_target()` | ✅ 구현 |
| 공격 시작 | 대상 → 공격자 방향 | 부드럽게 (`face_direction_smooth`) | `CombatScene._attack_target()` | ✅ 구현 |
| hit_frame 처리 | 대상 → 공격자 방향 재확정 | 즉시 | `CombatScene._handle_hit_frame()` | ✅ 구현 |
| 이동 완료 | 유닛 → 가장 가까운 적 방향 | 즉시 | `CombatScene._move_selected_unit()` | ✅ 구현 |
| 교전 성립 | 공격자 ↔ 대상 서로 마주보기 | 즉시 | `CombatScene._attack_target()` 교전 성립 블록 | ✅ 구현 |

### 9-3. 구현 코드 (CombatScene 수정 위치)

**`_attack_target()` — 애니메이션 시작 직전:**
```gdscript
attacker._renderer.face_direction(target.global_position)
target._renderer.face_direction_smooth(attacker.global_position)
```

**`_handle_hit_frame()` — 대미지 적용 직전:**
```gdscript
if is_instance_valid(_dispatch_target):
    _dispatch_target._renderer.face_direction(_dispatch_attacker.global_position)
```

**`_move_selected_unit()` — `await unit.move_finished` 이후:**
```gdscript
var targets = _get_attack_targets(unit)
if not targets.is_empty():
    var nearest_world := grid_manager.grid_to_world(targets[0])
    unit._renderer.face_direction(nearest_world)
```

**`_attack_target()` — 교전 성립 블록:**
```gdscript
attacker._renderer.face_direction(target.global_position)
target._renderer.face_direction(attacker.global_position)
```

### 9-4. 미구현 항목 (Phase B 이후)

- 대기 시 매 턴 가장 가까운 적 방향 유지 (`update_idle_facing`)
- 넉백 방향 처리
- 원거리 투사체 비행 중 방향 전환
