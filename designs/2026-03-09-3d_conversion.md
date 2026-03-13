# CrownFalle 3D 전환 설계
> 2D → 3D 환경 전환 설계 문서

- Created: 2026-03-09
- Status: ✅ 확정
- References:
  - `handoff/plans/design/2026-03-06-staggered_grid.md`
  - `handoff/plans/design/2026-03-08-unit_data_architecture.md`

---

## 1. 전환 방향

### 목표 스타일
- **B안**: 3D 메시 평면 바닥 + 유닛만 3D
- 레퍼런스: Wartales — 오블리크 카메라, 평탄한 전투 지형, 3D 유닛

### 전환 방식
- 기존 코드 베이스 유지, 점진적 전환
- 그리드 로직(GridManager.gd) 좌표 계산은 유지
- 렌더링 레이어만 2D → 3D로 교체

### 에셋
- 유닛 플레이스홀더: Kenney 3D 에셋 (무료)
  - https://kenney.nl/assets/mini-characters-1
  - https://kenney.nl/assets/animated-characters-2 (애니메이션 포함 버전)

---

## 2. 렌더링 구조 변경

### 현재 (2D)
```
Node2D (CombatScene)
  TileMapLayer    ← 스태거드 그리드
  Node2D units    ← Sprite2D 유닛
  Camera2D
```

### 변경 후 (3D)
```
Node3D (CombatScene)
  Node3D GridRoot
    MeshInstance3D tiles[]  ← 평면 메시 타일 (텍스처로 그리드 표현)
  Node3D UnitRoot
    CharacterBody3D units[] ← 3D 유닛
  Camera3D                  ← 오블리크 고정 카메라
```

---

## 3. 카메라 설정 (오블리크)

```
Camera3D:
  projection: PROJECTION_ORTHOGONAL (직교 투영)
  size: 10 (줌 레벨 — 조정 가능)

Transform:
  position: (0, 12, 8)       ← 높이 12, 뒤로 8
  rotation_degrees: (-55, 0, 0)  ← X축 -55도 (Wartales 근사값)

환경 조명:
  DirectionalLight3D: 오블리크 방향에 맞게 그림자 설정
  WorldEnvironment: 간단한 ambient 조명
```

---

## 4. 그리드 타일 구현

### 타일 메시
```
각 타일 = MeshInstance3D (PlaneMesh)
크기: 1.0 × 1.0 (기존 타일 크기에 대응)
머티리얼: 격자선 텍스처 or ShaderMaterial

스태거드 그리드 오프셋:
  현재 2D 오프셋 공식 유지 → 3D XZ 평면으로 변환
  y축 고정 (0.0) — 평탄 지형
```

### 좌표 변환
```
2D (col, row) → 3D (x, 0, z)

짝수 행: x = col × tile_width
홀수 행: x = col × tile_width + tile_width * 0.5
z = row × tile_height * 0.75

기존 GridManager.gd의 grid_to_world() 함수 수정:
  Vector2 반환 → Vector3 반환 (y=0 고정)
```

### 타일 클릭 판정
```
2D: 마우스 위치 → TileMap 좌표 변환
3D: Camera3D.project_ray_origin() + project_ray_normal()
    → PhysicsRayQueryParameters3D
    → 타일 CollisionShape3D와 교차 검사
```

---

## 5. 유닛 표현

### Kenney Mini Characters 사용
```
다운로드: https://kenney.nl/assets/mini-characters
포맷: GLTF/GLB

직업별 모델 매핑 (예시):
  Fighter → character_soldier.glb
  Archer  → character_archer.glb
  Rogue   → character_rogue.glb
  Mage    → character_mage.glb

배치:
  position.y = 0.0 (타일 평면 위)
  scale: 타일 크기에 맞게 조정
```

### 팀 구분
```
플레이어 팀: 기본 머티리얼
적 팀: 붉은 색조 오버레이 (ShaderMaterial albedo_mix)
선택된 유닛: 발 아래 원형 하이라이트 (Decal 또는 MeshInstance3D)
```

---

## 6. 점진 전환 계획

### Phase 1 — 카메라 + 그리드 (우선)
- Camera3D 오블리크 설정
- 기존 타일을 MeshInstance3D로 교체
- GridManager 좌표 변환 Vector3 대응
- 타일 Raycast 클릭 판정

### Phase 2 — 유닛
- Sprite2D → CharacterBody3D + Kenney GLB
- 유닛 위치/이동 3D 대응
- 팀 구분 머티리얼

### Phase 3 — 조명 + 마무리
- DirectionalLight3D 설정
- 선택/하이라이트 이펙트
- UI 위치 조정 (3D 씬에 맞게)

---

## 7. 유지되는 코드

```
GridManager.gd     — 그리드 좌표 계산 로직 (좌표 반환 타입만 수정)
CombatUnit.gd      — 전투 로직 전체 유지
CombatScene.gd     — 전투 흐름 유지, 렌더링 부분만 교체
PassiveManager.gd  — 변경 없음
DataLoader.gd      — 변경 없음
```

---

## 8. 신규/수정 파일

```
수정:
  scenes/combat/combat.tscn       Node2D → Node3D 구조로 재구성
  scripts/combat/CombatScene.gd   렌더링 레이어 3D 대응
  scripts/grid/GridManager.gd     grid_to_world() Vector3 반환

신규:
  scripts/rendering/TileRenderer3D.gd   3D 타일 생성/관리
  scripts/rendering/UnitRenderer3D.gd   3D 유닛 메시 로드/배치
  assets/models/units/                  Kenney GLB 파일
  assets/materials/tile_grid.material   타일 격자 머티리얼
```
