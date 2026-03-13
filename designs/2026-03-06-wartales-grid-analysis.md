# Wartales Grid System Analysis
> 🇰🇷 워테일즈 그리드 시스템 분석 문서

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Status: Reference Document (Future Implementation)
- Purpose: Wartales grid system analysis for CrownFalle post-prototype implementation
> 🇰🇷 생성일: 2026-03-06
> 🇰🇷 상태: 참조 문서 (향후 구현 예정)
> 🇰🇷 목적: 프로토타입 이후 CrownFalle 그리드 고도화를 위한 워테일즈 분석

---

## 1. Grid Topology
> 🇰🇷 타일 도형 구조

### Base Structure: Square Grid
> 🇰🇷 기본 구조: 사각형 그리드

Wartales uses a Square Grid (not hex).
> 🇰🇷 워테일즈는 정사각형 기반 격자 구조 채택

### Movement Direction
> 🇰🇷 이동 방향

- Basic: 4-direction (up/down/left/right)
- Diagonal movement also allowed
- Distance calculation options:
  * Diagonal = 1.414 (√2) cost
  * OR Diagonal = 1 (same as cardinal)
- Final system: Radius-based movement
  (unit moves freely within movement range value)
> 🇰🇷 기본 4방향 + 대각선 이동 허용
> 🇰🇷 대각선 거리 계산: √2 또는 1칸으로 처리 (설정에 따라 상이)
> 🇰🇷 최종: Radius(반경) 기반 이동 방식
> 🇰🇷 유닛의 이동 수치 내에서 자유로운 궤적으로 이동

---

## 2. Tile Data Structure (Layered Architecture)
> 🇰🇷 타일 데이터 구조 (레이어드 아키텍처)

Each tile is NOT just a position — it holds multiple layers of data.
> 🇰🇷 타일은 단순 위치 정보가 아닌 다층 레이어 구조를 가짐

### Layer Table
> 🇰🇷 레이어 구성표

| Layer | Description | Godot Implementation |
|-------|-------------|---------------------|
| Base Layer | Terrain height + walkable status | TileMapLayer (GroundLayer) |
| Surface Layer | Mud/snow/sand — movement cost modifier | TileMapLayer (SurfaceLayer) |
| Effect Layer | Fire/poison/trap — per-turn damage | Node2D (dynamic actors) |
| Occupant | Current unit pointer on tile | Dictionary in GridManager |

> 🇰🇷 Base Layer: 지형 고도 + 이동 가능 여부
> 🇰🇷 Surface Layer: 진흙/눈/모래 등 이동 속도에 영향
> 🇰🇷 Effect Layer: 불/독/함정 등 턴마다 데미지
> 🇰🇷 Occupant: 현재 타일을 점유 중인 유닛 포인터

### Dynamic Layer Behavior
> 🇰🇷 동적 레이어 작동 방식

```
Example: Fire spreads to tile (2,2)
→ BaseLayer remains "Grass" (unchanged)
→ EffectLayer adds "Fire" overlay independently
→ Two layers coexist without conflict
```

> 🇰🇷 예시: (2,2) 타일에 불이 붙으면
> 🇰🇷 BaseLayer는 여전히 'Grass' 유지
> 🇰🇷 EffectLayer에만 'Fire'가 오버레이로 추가됨
> 🇰🇷 두 레이어가 독립적으로 공존

### Effect Propagation
> 🇰🇷 영향력 전파

When explosion occurs at tile (x,y):
→ All 8 adjacent tiles' EffectLayer updated simultaneously
> 🇰🇷 특정 타일 폭발 시
> 🇰🇷 인접 8방향 타일의 EffectLayer 상태 일괄 변경

---

## 3. Unit Occupancy
> 🇰🇷 유닛 점유 시스템

### Size Types
> 🇰🇷 유닛 크기 종류

| Type | Tiles | Example |
|------|-------|---------|
| Standard | 1x1 | Human units (ally, rogue, etc.) |
| Large | 2x2+ | Bear, boar, giant monster |

> 🇰🇷 표준 유닛: 1x1 (아군, 도적 등 인간형)
> 🇰🇷 대형 유닛: 2x2 이상 (곰, 멧돼지, 거대 괴수)

### Large Unit Pathfinding Rule
> 🇰🇷 대형 유닛 경로탐색 규칙

For 2x2+ units, pathfinding must check
the FULL occupancy area (not just center point)
for obstacle collision.
> 🇰🇷 2x2 유닛은 중심점이 아닌
> 🇰🇷 점유 면적 전체가 장애물에 걸리는지 체크 필요

### Occupant Pointer Update Rule
> 🇰🇷 점유 포인터 업데이트 규칙

```
Unit moves from (0,0) → (0,1):
1. (0,0).Occupant = null  ← clear old tile
2. (0,1).Occupant = self  ← register new tile
→ Engagement check uses this pointer
```

> 🇰🇷 유닛 이동 시:
> 🇰🇷 1. 이전 타일 Occupant = null
> 🇰🇷 2. 새 타일 Occupant = 자기 자신
> 🇰🇷 교전 판정은 이 포인터 기준으로 작동

---

## 4. Adjacency & Engagement System
> 🇰🇷 인접성 및 교전 시스템

### Core Mechanic
> 🇰🇷 핵심 메커니즘

When two units occupy adjacent tiles sharing an edge:
→ Both enter "Engagement" state
→ Movement is blocked for both units
→ This is the core tactical constraint of Wartales
> 🇰🇷 두 유닛이 변(Edge)을 공유하는 인접 타일에 위치 시
> 🇰🇷 양쪽 모두 '교전(Engagement)' 상태 진입
> 🇰🇷 이동 봉쇄 — 워테일즈 핵심 전술 메커니즘

### Edge vs Vertex
> 🇰🇷 변 vs 꼭짓점

Engagement is triggered by EDGE sharing (4-direction adjacency).
Diagonal (vertex-only) contact does NOT trigger engagement.
> 🇰🇷 교전은 변(Edge) 공유 시 발동 (4방향 인접)
> 🇰🇷 대각선(꼭짓점만 공유)은 교전 미발동

---

## 5. Grid Data Example (5x5)
> 🇰🇷 그리드 데이터 예시 (5x5)

### Tile Legend
> 🇰🇷 타일 범례

| Symbol | Meaning | Movement Cost |
|--------|---------|---------------|
| P | Player Unit (1x1) — placeable in deployment zone | 1 |
| E | Enemy Unit (1x1) — fixed initial placement | 1 |
| B | Boss/Large Unit (2x2) — 4 tiles simultaneously | 1 |
| M | Mud/Hazard — movement cost x2 | 2 |
| F | Fire/Effect — dot damage on turn end | 1 |
| O | Obstacle (rock/wall) — impassable, blocks LOS | ∞ |

> 🇰🇷 P: 아군 유닛 (1x1) — 배치 구역 내 위치 변경 가능
> 🇰🇷 E: 적군 유닛 (1x1) — 초기 배치 고정
> 🇰🇷 B: 보스/대형 유닛 (2x2) — 4타일 동시 점유
> 🇰🇷 M: 진흙 — 이동력 소모 2배
> 🇰🇷 F: 화염 — 턴 종료 시 도트 데미지
> 🇰🇷 O: 장애물 — 진입 불가, 시야 차단

### Data Table Structure
> 🇰🇷 데이터 테이블 구조

| Index (X,Y) | Base Terrain | Effect | Occupant | Move Cost |
|-------------|-------------|--------|----------|-----------|
| (0,0) | Grass | None | Player_1 | 1 |
| (1,2) | Mud | None | None | 2 |
| (2,2) | Rock | None | Obstacle | ∞ |
| (3,3) | Grass | Fire | Enemy_1 | 1 |
| (4,4) | Grass | None | Boss (Part) | 1 |

> 🇰🇷 각 타일은 지형/효과/점유/이동비용 4가지 데이터 보유

---

## 6. CrownFalle Implementation Plan
> 🇰🇷 크라운폴 구현 계획

### Current Prototype State
> 🇰🇷 현재 프로토타입 상태

```
✅ Implemented:
- Square isometric grid (10x10)
- 3 layers: Ground / Obstacle / Highlight
- 4-direction movement (BFS)
- 1x1 unit occupancy
- Occupant pointer (GridManager)

❌ Not yet implemented:
- Diagonal + Radius-based movement
- Surface layer (movement cost modifier)
- Effect layer (fire/poison dot damage)
- 2x2+ large units
- Engagement system (movement block)
- Deployment zone (pre-battle placement)
```

> 🇰🇷 완료: 아이소메트릭 격자, 3레이어, 4방향 이동, 1x1 점유
> 🇰🇷 미완료: 대각선+Radius, 표면 레이어, 효과 레이어, 대형 유닛, 교전 시스템

### Phase 5+ Implementation Priority
> 🇰🇷 Phase 5 이후 구현 우선순위

```
Priority 1 (Most impactful):
→ Diagonal + Radius-based movement
→ Engagement system

Priority 2 (Adds depth):
→ Surface layer (mud cost x2)
→ Effect layer (fire dot damage)

Priority 3 (Advanced):
→ 2x2 large units
→ Deployment zone system
→ Line of sight (LOS) blocking
```

> 🇰🇷 1순위: 대각선+Radius 이동, 교전 시스템
> 🇰🇷 2순위: 표면 레이어 (진흙), 효과 레이어 (화염)
> 🇰🇷 3순위: 대형 유닛, 배치 구역, 시야 차단

---

## 7. Distance-Based Movement System
> 🇰🇷 거리 기반 이동 시스템

### Overview
> 🇰🇷 개요

Wartales expresses movement range in meters (m), not tile count.
Units move freely within a radius — the boundary is an irregular polygon,
not a discrete set of highlighted tiles.
> 🇰🇷 워테일즈는 이동 범위를 칸 수가 아닌 미터(m) 단위로 표시
> 🇰🇷 유닛은 반경 내에서 자유롭게 이동
> 🇰🇷 경계선은 이산 타일 집합이 아닌 불규칙 폴리곤

### Key Properties
> 🇰🇷 핵심 특성

| Property | Description |
|----------|-------------|
| Unit: meters (m) | Movement range displayed as a real-world distance value |
| Circular base radius | Clean circle when no obstacles present |
| Irregular boundary | Obstacles, terrain, and occupied tiles deform the boundary |
| Free path within range | No per-tile step limit — continuous movement inside radius |
| Polygon outline visualization | Teal/cyan polygon outline marks the reachable boundary |
| Tiles still exist | Grid remains for occupancy tracking and adjacency; movement is distance-based |

> 🇰🇷 단위: 미터(m) — 이동 범위를 실제 거리값으로 표시
> 🇰🇷 기본 형태: 장애물 없을 때 원형 반경
> 🇰🇷 불규칙 경계: 장애물/지형/점유 유닛에 따라 경계선 변형
> 🇰🇷 반경 내 자유 이동: 칸 단위 제약 없이 연속 이동
> 🇰🇷 시각화: 청록색 폴리곤 외곽선으로 도달 가능 구역 표시
> 🇰🇷 타일 유지: 점유 추적 및 인접 판정용 격자는 그대로 존재

### Boundary Deformation Rules
> 🇰🇷 경계선 변형 규칙

- **Wall / Impassable tile:** radius is cut at the obstacle edge
- **Occupied tile (enemy):** that tile is excluded from reachable area
- **Occupied tile (ally):** passable for path, excluded as destination
- **Diagonal passage:** allowed unless both adjacent tiles are blocked (corner cut rule)
> 🇰🇷 벽/이동 불가 타일: 반경이 장애물 경계에서 잘림
> 🇰🇷 적 점유 타일: 도달 가능 구역에서 제외
> 🇰🇷 아군 점유 타일: 경로 통과 가능, 목적지 제외
> 🇰🇷 대각선 통과: 양쪽 인접 타일이 모두 막힌 경우 차단 (코너 컷 규칙)

### Visualization
> 🇰🇷 시각화 방식

```
청록색(Teal) 폴리곤 외곽선만 표시 — 내부 타일 하이라이트 없음
장애물에 의해 경계선이 안쪽으로 꺾임
클릭한 지점이 폴리곤 내부면 이동 실행
```

---

## Reference
> 🇰🇷 참조

- Wartales (Shiro Games) — direct gameplay analysis
- Concept document: handoff/plans/design/2026-03-06-crownfalle-concept.md
- Roadmap: handoff/plans/design/2026-03-06-crownfalle-roadmap.md
- Distance movement design: handoff/plans/design/2026-03-08-distance_movement.md
