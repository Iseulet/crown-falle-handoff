# Design: Engagement System (v3 — Final)
> 🇰🇷 설계: 교전(Engagement) 시스템 v3 최종 확정

- **Agent:** @Planner
- **Date:** 2026-03-08 (v3 — 전면 재작성)
- **Status:** Design Complete — Pending @Implementor
- **References:**
  - `handoff/plans/design/2026-03-06-crownfalle-concept.md`
  - `handoff/plans/design/2026-03-06-wartales-grid-analysis.md`

---

## 확정된 시스템 개요
> System Overview

두 가지 상태를 구분하되, **현재 게임플레이 효과는 Engagement만 적용**.
Proximity는 함수만 구현해두고 효과 미적용.

| 상태 | 현재 효과 | 구현 상태 |
|------|-----------|-----------|
| Proximity (인접) | 없음 | `is_in_proximity()` 함수만 구현, 추후 활용 |
| Engagement (교전) | 이동 봉쇄 + 공격 제한 + ⚔ 표시 | 완전 구현 |

---

## 1. Proximity — 보류 상태
> Proximity — Deferred

### 현재 사양

| 항목 | 내용 |
|------|------|
| 발동 | 적군과 6방향 인접 시 자동 감지 |
| **게임플레이 효과** | **없음 (현재 비적용)** |
| 구현 | `GridManager.is_in_proximity(pos, is_ally)` 함수만 존재 |
| 향후 | 다른 게임 요소(스킬, AI 등)에서 활용 예정 |

### 구현 (GridManager.gd — 기존 유지)

```gdscript
func is_in_proximity(pos: Vector2i, is_ally: bool) -> bool:
    for neighbor in get_attack_neighbors(pos):
        var cu := get_unit_at(neighbor) as CombatUnit
        if cu != null and cu.is_ally != is_ally:
            return true
    return false
```

- CombatScene에서 호출하지 않음
- 함수 자체는 GridManager에 보존

---

## 2. Engagement — 교전 상태
> Engagement — Active System

### 발동 조건

| 조건 | 결과 |
|------|------|
| 자유로운 적(target.engaged_with == null)에게 인접 공격 | Engagement 형성 (양방향) |
| 이미 교전 중인 적(target.engaged_with != null)에게 공격 | 데미지만 적용, Engagement 미형성 |
| 자신이 이미 교전 중(attacker.engaged_with != null) | `_get_attack_targets()`가 교전 상대만 반환 → 해당 없음 |

### 효과

| 효과 | 설명 |
|------|------|
| 이동 봉쇄 | `engaged_with != null` → BFS 생략, 주황 타일 표시 |
| 공격 제한 | `_get_attack_targets()`가 교전 상대만 반환 |
| ⚔ 아이콘 | `set_engaged(true)` → `_engage_label.visible = true` |

### 해제 조건

| 조건 | 처리 |
|------|------|
| 교전 상대 사망 | `_on_unit_died()` → 파트너의 `engaged_with = null` |

### 반격

- **없음** — 현재 반격 미구현
- 추후 스킬 시스템을 통해 반격 스킬로 구현 예정

### 상태 전이 다이어그램

```
[일반 상태]
    │ 자유로운 적 공격 (target.engaged_with == null)
    ▼
[Engagement]  ◄──── 교전 상대 사망 시 해제
```

---

## 3. 공격 처리 흐름
> Attack Flow

### `_attack_target()` 처리 순서

```
아군 A가 적 B를 공격:
  1. B.take_damage(A.unit_str)          ← 정상 공격

  2-a. B 사망:
       A.gain_xp(xp_per_kill)
       → Engagement 미형성

  2-b. B 생존:
       B.engaged_with == null?
         YES → A.engaged_with = B
               B.engaged_with = A      ← Engagement 형성
         NO  → 데미지만 적용, 상태 변화 없음

  ※ 반격 없음
```

### GDScript 구조

```gdscript
func _attack_target(target_pos: Vector2i) -> void:
    var target := grid_manager.get_unit_at(target_pos) as CombatUnit
    if target == null:
        return
    var attacker := selected_unit
    attacker.has_attacked = true

    # 1. 정상 공격
    target.take_damage(attacker.unit_str)

    if target.is_queued_for_deletion():
        # 적군 사망 — XP 획득
        attacker.gain_xp(int(_config.get("xp_per_kill", 10)))
    elif target.engaged_with == null and attacker.engaged_with == null:
        # 양측 자유 → Engagement 형성 (1:1 보장)
        attacker.engaged_with = target
        target.engaged_with = attacker
    # else: target 이미 교전 중 → 데미지만, 상태 변화 없음

    _deselect_unit()
    _refresh_all_engagement()
```

---

## 4. 이동 봉쇄 구현
> Movement Blocking

Engagement 상태만 이동 봉쇄. Proximity는 이동에 영향 없음.

```gdscript
func _select_unit(unit: CombatUnit) -> void:
    selected_unit = unit
    reachable_cells = []
    grid_manager.clear_highlights()

    if not unit.has_moved:
        if unit.engaged_with != null:
            # 교전 중 → 이동 봉쇄, 주황 타일
            grid_manager.highlight_proximity(unit.grid_pos)
        else:
            # 자유 → 이동 범위 표시
            reachable_cells = grid_manager.get_reachable_cells(unit.grid_pos, unit.move_range)
            grid_manager.highlight_move_range(reachable_cells)
    _refresh_attack_highlights()
```

BFS(`get_reachable_cells`) 변경 없음.

---

## 5. 공격 대상 제한
> Attack Target Restriction

```gdscript
func _get_attack_targets(unit: CombatUnit) -> Array[Vector2i]:
    # Engagement 상태: 교전 상대만
    if unit.engaged_with != null and not unit.engaged_with.is_queued_for_deletion():
        return [unit.engaged_with.grid_pos]
    # 자유 상태: 인접 적 전부
    var targets: Array[Vector2i] = []
    for cell in grid_manager.get_attack_neighbors(unit.grid_pos):
        var enemy := grid_manager.get_unit_at(cell) as CombatUnit
        if enemy != null and not enemy.is_ally and not enemy.is_queued_for_deletion():
            targets.append(cell)
    return targets
```

---

## 6. 교전 해제 처리
> Disengagement

```gdscript
func _on_unit_died(unit: CombatUnit) -> void:
    if unit.engaged_with != null and not unit.engaged_with.is_queued_for_deletion():
        unit.engaged_with.engaged_with = null
    grid_manager.clear_occupied(unit.grid_pos)
    unit.queue_free()
    _refresh_all_engagement()
    _check_combat_end()
```

---

## 7. 시각적 표시
> Visual Indication

| 상태 | 본인 타일 | 공격 하이라이트(빨간) | 이동 범위 |
|------|----------|--------------------|-----------|
| 자유 | 없음 | 인접 적 전부 | 파란 하이라이트 |
| Engagement | **주황** | 교전 상대 1명만 | 없음 (봉쇄) |

- Proximity: 별도 하이라이트 없음 (현재 효과 없으므로)
- Engagement ⚔ 아이콘: `set_engaged(true)` → 상시 표시

---

## 8. 변경 파일 목록
> Expected Changed Files

### GridManager.gd
| 항목 | 내용 |
|------|------|
| `is_in_proximity()` | 기존 유지 (호출 없음, 함수만 보존) |
| `highlight_proximity()` | 기존 유지 (Engagement 이동 봉쇄 표시에 재활용) |
| `TILE_HIGHLIGHT_PROXIMITY` | 기존 유지 |
| **변경 없음** | 파일 수정 불필요 |

### CombatUnit.gd
| 항목 | 내용 |
|------|------|
| **변경 없음** | `engaged_with`, `_engage_label`, `set_engaged()` 이미 구현 완료 |

### CombatScene.gd
| 항목 | 변경 내용 |
|------|-----------|
| `_select_unit()` | Proximity 분기 제거 → `engaged_with != null` 단독 봉쇄 판정 |
| `_attack_target()` | 반격 로직 제거 + Engagement 형성 조건 단순화 |
| `_get_attack_targets()` | 변경 없음 (기존 로직 일치) |
| `_on_unit_died()` | 변경 없음 |
| `_refresh_all_engagement()` | 변경 없음 |

**실질 변경:** CombatScene.gd 1개 파일, 2개 함수

---

## 9. 1:1 교전 제한
> One-to-One Constraint

- `attacker.engaged_with == null` 조건이 `_attack_target()` 분기에 포함
- `target.engaged_with == null` 조건으로 이미 교전 중인 적에게 Engagement 미형성
- 한 유닛이 동시에 복수 Engagement를 보유하는 경우 원천 차단

---

## 10. v2 대비 변경점
> Changes from v2

| 항목 | v2 | v3 (최종) |
|------|----|----|
| Proximity 효과 | 이동 봉쇄 + 모든 인접 적 공격 | **효과 없음** (함수만 보존) |
| Engagement 발동 | 공격 + 반격 (상호 생존) 후 형성 | **공격 시 즉시 형성** (target 자유 상태면) |
| 반격 | 즉시 반격 처리 | **없음** (스킬 시스템으로 이관 예정) |
| 이동 봉쇄 조건 | Proximity OR Engagement | **Engagement만** |
| 공격 흐름 복잡도 | 고 (공격 → 반격 → 상태 체크) | 저 (공격 → 상태 체크) |

---

## 구현 체크리스트 (for @Implementor)

- [x] GridManager.gd — `is_in_proximity()` 구현 (완료, 변경 없음)
- [x] GridManager.gd — `highlight_proximity()` 구현 (완료, 변경 없음)
- [x] CombatUnit.gd — `engaged_with`, `_engage_label`, `set_engaged()` (완료, 변경 없음)
- [ ] CombatScene.gd — `_select_unit()`: Proximity 분기 제거, Engagement 단독 봉쇄
- [ ] CombatScene.gd — `_attack_target()`: 반격 제거, Engagement 형성 단순화
- [ ] 동작 확인: 자유 적 공격 → ⚔ 형성, 교전 중 적 공격 → 데미지만, 사망 시 해제
