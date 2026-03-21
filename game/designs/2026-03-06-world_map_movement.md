# Design: Phase 3-1 World Map Movement
> 🇰🇷 설계: Phase 3-1 월드맵 이동

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Agent: @Planner
- Status: Pending User Approval
- Phase: 3-1 (World Map Movement)
- References:
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-crownfalle-roadmap.md`

---

## Overview
> 🇰🇷 개요

Phase 3-1 creates the world map scene with a player icon that moves
to clicked positions via smooth Tween animation.
No grid — free pixel movement on a placeholder background.
> 🇰🇷 월드맵 씬 생성. 클릭 위치로 플레이어 아이콘이 부드럽게 이동.
> 🇰🇷 격자 없음 — 플레이스홀더 배경 위 자유 이동.

---

## 1. Scene Structure
> 🇰🇷 씬 구조

```
WorldMapScene (Node2D)  [WorldMapScene.gd]
├── Background (ColorRect)
│     color = Color(0.1, 0.12, 0.15)   ← dark placeholder
│     size = Vector2(1152, 648)
├── PlayerIcon (Node2D)  [PlayerIcon.gd]
│     position = Vector2(576, 324)     ← center start
│     └── Polygon2D  ← yellow diamond (same shape as CombatUnit)
└── WorldMapUI (CanvasLayer)
    └── HUD (VBoxContainer)
          offset_left = 10, offset_top = 10
          └── StatusLabel (Label)  text = "월드맵"
```

> 🇰🇷 배경: 어두운 단색 플레이스홀더 / 아이콘: 노란 다이아몬드

---

## 2. PlayerIcon.gd
> 🇰🇷 PlayerIcon 스크립트

**File:** `scripts/world/PlayerIcon.gd`

```gdscript
class_name PlayerIcon
extends Node2D

const MOVE_SPEED: float = 200.0  # pixels per second

var _tween: Tween = null

func move_to(target: Vector2) -> void:
    if _tween:
        _tween.kill()
    var dist := position.distance_to(target)
    var duration := dist / MOVE_SPEED
    _tween = create_tween()
    _tween.tween_property(self, "position", target, duration)
```

> 🇰🇷 이동 거리 기반 duration 계산 → 일정 속도 유지
> 🇰🇷 이동 중 재클릭 시 현재 위치에서 새 목적지로 전환

---

## 3. WorldMapScene.gd
> 🇰🇷 WorldMapScene 스크립트

**File:** `scripts/world/WorldMapScene.gd`

```gdscript
class_name WorldMapScene
extends Node2D

@onready var player_icon: PlayerIcon = $PlayerIcon
@onready var status_label: Label = $WorldMapUI/HUD/StatusLabel

func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventMouseButton \
            and event.button_index == MOUSE_BUTTON_LEFT \
            and event.pressed:
        player_icon.move_to(get_global_mouse_position())
```

> 🇰🇷 마우스 좌클릭 → PlayerIcon 이동 명령

---

## 4. Files Created
> 🇰🇷 신규 파일 목록

| File | Description |
|------|-------------|
| `scenes/world/world_map.tscn` | NEW — 월드맵 씬 |
| `scripts/world/WorldMapScene.gd` | NEW — 씬 컨트롤러 |
| `scripts/world/PlayerIcon.gd` | NEW — 아이콘 이동 로직 |

> 🇰🇷 기존 파일 수정 없음. 신규 3개.

---

## 5. Acceptance Criteria
> 🇰🇷 완료 기준

- [ ] Godot에서 world_map.tscn 실행 시 어두운 배경 + 노란 아이콘 표시
- [ ] 클릭 위치로 아이콘이 부드럽게 이동
- [ ] 이동 중 재클릭 시 현재 위치에서 새 목적지로 전환
- [ ] Godot 에러 없음

---

## 6. Next Step
> 🇰🇷 다음 단계

→ `@Implementor` implements based on this document.
→ Phase 3-2 Fatigue System 연동 예정 (이동 거리 → 피로도 누적)
