# CrownFalle Proto — 현재 세션 컨텍스트
> 생성일: 2026-03-12 | 갱신 주기: 세션마다

---

## 현재 상태

| 항목 | 내용 |
|------|------|
| 현재 Phase | Doc 6 Phase 0~6 완료 — Phase B 대기 |
| 세션 상태 | 🟡 In Progress (미커밋) |
| Git | `main` branch, last commit `72c3597` |
| 마지막 작업 | 2026-03-12 |

---

## 최근 세션 완료 작업 (2026-03-12)

### 1. 에셋 시스템 구축
- `assets/_library/` 구조 생성 + `.gdignore` (Godot 임포트 차단)
- Quaternius 4팩 + Synty 1팩 수급 → `_library/` 정리
- `ASSET_LOG.md` 생성 (승격 이력 기록)
- `tools/verify_skeleton.py` 생성 (GLB/GLTF 본 이름 검증)

### 2. Quaternius RPG Classes FBX 교체
- Meshy AI GLB → `_library/characters/backup_meshy/` 백업
- warrior/ranger/rogue/wizard_rpg.fbx → `assets/characters/{class}/` 승격
- **UnitRenderer3D.gd 변경:**
  - `ANIM_PATHS` Dict 포맷: `{"path": ..., "clip": ...}` 지원
  - `_import_anim(clip_name)` — 정확 매칭 + `|suffix` 폴백
  - `_apply_png_textures()` 신규 — FBX 외부 텍스처 경로 깨짐 해결
  - idle 루프: `anim.loop_mode = LOOP_LINEAR` 명시 설정

### 3. HP 게이지 + 클래스 레이블 시스템
- `UnitRenderer3D._create_hp_bar(class_id, is_ally)` — QuadMesh 배경+채움
- `update_hp(current, max)` — HP% 기반 녹색→노랑→빨강 색상 전환
- `_process()` — `look_at(pos - to_cam)` 으로 QuadMesh 앞면 카메라 향함
- `Label3D` billboard — 클래스명 항상 카메라 정면
- `CombatUnit._hp_label` 제거 → `_renderer.update_hp()` 연결

### 4. 파벌 서클 (Faction Disc)
- 선택 링(TorusMesh) → CylinderMesh 평면 디스크로 교체
- 기본 `visible = false` — 조건부 표시:
  - **아군**: 클릭 선택 시만 표시 (파란색)
  - **적군**: 행동 중에만 표시 (빨간색)
- `position.y = 0.06` — 타일 상단(0.05) 위에 위치
- 적군 빨간 재질 오버레이 제거

### 5. 스폰 방향 초기화
- `_face_units_toward_opponents()` 함수 신규
- 스폰 완료 후 각 유닛이 가장 가까운 적 방향을 바라봄

### 6. 카메라 사망 연출 개선
- `DEBUG_FORCE_CRIT = false` (테스트 모드 해제)
- 사망 시 즉시 셰이크 제거 → `animation_config.json` dead_hit ratio 0.45 이벤트로 이동
- `shake_execute`: intensity 0.80→0.35, duration 0.40s→0.22s, decay ease_out
- `soft_track_melee` weight: 0.20→0.12
- 카메라 포커스 y 오프셋 +0.7 (발→몸통 중심)

---

## 수정된 파일 목록 (미커밋)

| 파일 | 변경 내용 |
|------|----------|
| `scripts/rendering/UnitRenderer3D.gd` | 텍스처, HP바, 파벌 서클, 방향 |
| `scripts/combat/CombatScene.gd` | 스폰방향, 카메라 ctx, 적 행동 disc |
| `scripts/combat/CombatUnit.gd` | _hp_label 제거, update_hp 연결 |
| `data/animations/animation_config.json` | dead_hit camera_effect ratio 0.45 추가 |
| `data/cameras/camera_config.json` | shake_execute, soft_track_melee 조정 |
| `assets/characters/` | FBX + 텍스처 파일 추가 |
| `assets/_library/` | 라이브러리 구조 생성 |
| `ASSET_LOG.md` | 승격 이력 |
| `tools/verify_skeleton.py` | 신규 생성 |

---

## 미완료 / 다음 작업

### 즉시 확인 필요
- [ ] Godot 에디터에서 FBX 애니메이션 클립 이름 확인 (`CharacterArmature|Idle` vs `Idle`)
- [ ] HP_BAR_Y=1.8 높이 최종 조정 (모델 크기 확인 후)

### Phase B (다음 단계)
- [ ] 원거리 공격 사거리 시스템
- [ ] 투사체 비행 (비동기, 공격자 즉시 idle 복귀)
- [ ] Archer/Mage 전용 에셋
- [ ] dodge.glb 에셋 수급

---

## 최근 아키텍처 결정 (2026-03-12)

### Unit Visual System
- HP 게이지: QuadMesh 2개 (배경+채움) — `HP_BAR_Y=1.8`
- `_process()` 에서 `look_at(pos - to_cam)` — QuadMesh +Z 앞면이 카메라를 향하게
- 파벌 서클: CylinderMesh 디스크, `position.y=0.06` (타일 위)
- 아군=선택시 파란색, 적=행동시 빨간색

### FBX Animation Loading
- `ANIM_PATHS` Dict 포맷: `{"path": "res://..fbx", "clip": "ClipName"}`
- `_import_anim(path, anim_name, clip_name)` — 정확 매칭 → `|suffix` 폴백
- 텍스처: `_apply_png_textures()` — 키워드 기반 메시 이름 감지

### Camera Event Timing
- 사망 셰이크: `_on_unit_died()` 즉시 발화 → `animation_config.json` ratio 0.45 이벤트로 이동
- 이유: 사망 애니메이션 타격 시점에 카메라 연출 동기화
