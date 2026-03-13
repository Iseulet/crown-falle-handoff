# Project Decisions - CrownFalle Proto
> 🇰🇷 프로젝트 확정 결정사항 - CrownFalle Proto

> Append-only. Never delete entries. Add new decisions at the bottom.
> 🇰🇷 추가 전용. 기존 항목 삭제 금지. 새 결정사항은 아래에 추가.
> Format: `## [YYYY-MM-DD] Decision Title`
> 🇰🇷 형식: `## [YYYY-MM-DD] 결정 제목`
>
> **Metadata format (v2 — added 2026-03-13):**
> Each entry includes: `**Category:** | **Status:** | **Superseded By:**`
> Categories: engine, renderer, combat, camera, animation, asset, data, workflow
> Status: ✅ 확정 / 🔄 수정됨 / ⏸️ 보류
> Reading strategy: read ✅ 확정 entries only, skip 🔄 수정됨
> 🇰🇷 각 항목 메타데이터: Category(분류) | Status(상태) | Superseded By(대체 항목)

---

## [2026-03-03] Engine & Language
> 🇰🇷 [2026-03-03] 엔진 및 언어
> **Category:** engine | **Status:** ✅ 확정 | **Superseded By:** —

- **Engine:** Godot 4.6.1
  > 🇰🇷 **엔진:** Godot 4.6.1
- **Language:** GDScript
  > 🇰🇷 **언어:** GDScript
- **Rationale:** Native integration, no build step, optimal for solo/small team dev
  > 🇰🇷 **근거:** 네이티브 통합, 빌드 단계 없음, 솔로/소규모 팀 개발에 최적

---

## [2026-03-03] Visual Style
> 🇰🇷 [2026-03-03] 비주얼 스타일
> **Category:** renderer | **Status:** 🔄 수정됨 | **Superseded By:** [2026-03-09] 3D Perspective Conversion + Renderer: Forward Plus

- **Projection:** 2D Isometric
  > 🇰🇷 **투영 방식:** 2D 아이소메트릭
- **Renderer:** Compatibility (OpenGL ES 3.0)
  > 🇰🇷 **렌더러:** 호환성(Compatibility) — OpenGL ES 3.0
- **Rationale:** Broader device support, stable performance on lower-end hardware
  > 🇰🇷 **근거:** 더 넓은 기기 지원, 저사양 하드웨어에서 안정적 성능

---

## [2026-03-03] Combat System
> 🇰🇷 [2026-03-03] 전투 시스템
> **Category:** combat | **Status:** ✅ 확정 | **Superseded By:** —

- **Type:** Grid-based turn-based combat
  > 🇰🇷 **유형:** 격자 기반 턴제 전투
- **Layout:** Tile grid (exact dimensions TBD)
  > 🇰🇷 **레이아웃:** 타일 격자 (정확한 크기는 추후 결정)
- **Rationale:** Tactical depth with clear action sequencing
  > 🇰🇷 **근거:** 명확한 행동 순서를 통한 전술적 깊이

---

## [2026-03-06] Project Folder Structure
> 🇰🇷 [2026-03-06] 프로젝트 폴더 구조
> **Category:** workflow | **Status:** ✅ 확정 | **Superseded By:** —

```
scenes/
├── combat/    ← Battle scenes
├── world/     ← World map scenes
├── ui/        ← UI scenes
└── camp/      ← Camp scenes

scripts/
├── combat/    ← Combat system scripts
├── world/     ← World map scripts
├── ui/        ← UI logic scripts
└── data/      ← Resource data files

assets/
├── sprites/   ← Sprite images
├── audio/     ← Sound / music
└── fonts/     ← Fonts
```

- **Rationale:** Feature-based organization for clarity and scalability
  > 🇰🇷 **근거:** 기능 기반 분류로 가독성 및 확장성 확보

---

## [2026-03-06] Reference Documents
> 🇰🇷 [2026-03-06] 공식 참조 문서 지정
> **Category:** workflow | **Status:** ✅ 확정 | **Superseded By:** —

The following files are designated as primary reference documents and must be read before any design or implementation task:
> 🇰🇷 아래 두 파일을 공식 참조 문서로 지정. 설계/구현 전 반드시 참조할 것.

- **`handoff/plans/design/2026-03-06-crownfalle-concept.md`** — Game concept and design direction guide
  > 🇰🇷 게임 컨셉 및 설계 방향 기준 문서
- **`handoff/plans/design/2026-03-06-crownfalle-roadmap.md`** — Development sequence and phase tracker
  > 🇰🇷 개발 순서 및 Phase 진행 추적 문서
- **Rationale:** Ensures all agents and tools share the same game identity and development direction
  > 🇰🇷 **근거:** 모든 에이전트와 도구가 동일한 게임 정체성 및 개발 방향을 공유하도록 보장

---

## [2026-03-08] Wartales Grid System Analysis
> 🇰🇷 [2026-03-08] 워테일즈 그리드 시스템 분석 완료
> **Category:** combat | **Status:** ✅ 확정 | **Superseded By:** —

Wartales grid system analyzed and documented as a design reference.
Full analysis saved to: `handoff/plans/design/2026-03-06-wartales-grid-analysis.md`
> 🇰🇷 워테일즈 그리드 시스템 분석 완료. 별도 문서로 저장.

Implementation priority for Phase 5+:
> 🇰🇷 Phase 5 이후 구현 우선순위:

- **1순위:** 대각선+Radius 이동, 교전(Engagement) 시스템
  > 🇰🇷 전술 핵심 — 가장 먼저 구현
- **2순위:** 표면 레이어 (진흙 이동비용), 효과 레이어 (화염 도트)
  > 🇰🇷 전장 환경 다양화
- **3순위:** 대형 유닛 (2x2), 배치 구역, 시야 차단
  > 🇰🇷 고급 전술 요소

## [2026-03-08] Data-Driven Architecture
> 🇰🇷 [2026-03-08] 데이터 기반 아키텍처 확립
> **Category:** data | **Status:** ✅ 확정 | **Superseded By:** —

All content values (numbers, positions, names) must be defined in JSON data files, not hardcoded in GDScript.
> 🇰🇷 모든 콘텐츠 수치/위치/이름은 JSON 파일에 정의. GDScript 하드코딩 금지.

Migration completed (2026-03-08):
> 🇰🇷 전환 완료 항목 (2026-03-08):

- Enemy spawn / ally spawn → `data/encounters/encounter_01.json`
- Combat rules (max_turns, hit_radius) → `data/combat_config.json`
- Initial roster paths → `data/mercenaries/roster_config.json`
- World node positions (NPC, EncounterZone) → `data/world/world_nodes.json`
- NPC dialogue / quest_id → `data/npcs/npc_01.json`
- Quest rewards (food, gold) → `data/quests/quest_01.json`
- Level-up amounts per class → `data/classes/levelup_config.json`
- Fatigue / move speed → `data/world_config.json`
- Camp recovery values → `data/camp_config.json`
- Class base stats (for future recruit) → `data/classes/class_config.json`

Rule: New content must follow this pattern. Adding content = adding/editing JSON only.
> 🇰🇷 원칙: 신규 콘텐츠는 반드시 이 패턴 준수. 콘텐츠 추가 = JSON 추가/수정만으로 가능해야 함.

## [2026-03-09] Renderer: Forward Plus
> 🇰🇷 [2026-03-09] 렌더러 Forward Plus 전환
> **Category:** renderer | **Status:** ✅ 확정 | **Superseded By:** —

- **변경:** `gl_compatibility` → `forward_plus`
  > 🇰🇷 OpenGL Compatibility에서 Forward Plus(D3D12/Vulkan)로 전환
- **원인:** OpenGL Compatibility + NVIDIA 드라이버 566.36 조합에서 GPU TDR 크래시 발생 (전투 중 컴퓨터 완전 멈춤)
  > 🇰🇷 GL Compatibility 렌더러의 TileMapLayer 반복 clear/set_cell이 GPU 드라이버 타임아웃 유발
- **결과:** Forward Plus 전환 후 크래시 미발생 확인
  > 🇰🇷 RTX 3060 환경에서 Forward Plus 안정 동작 확인
- **모바일:** `gl_compatibility` 유지 (저사양 기기 지원)
  > 🇰🇷 모바일 빌드는 기존 GL Compatibility 유지

## [2026-03-09] 3D Perspective Conversion
> 🇰🇷 [2026-03-09] 3D 원근 시점 전환
> **Category:** renderer | **Status:** ✅ 확정 | **Superseded By:** —

- **변경:** 2D 아이소메트릭 → 3D BG3 스타일 원근 시점
- **렌더러:** Compatibility → Forward Plus (Vulkan/D3D12) — GPU TDR 크래시 해소
- **그리드:** staggered hex-like (홀수 열 z+0.5 오프셋) — TILE_SIZE = 1.0
- **카메라:** CameraDirector.gd — orbit/zoom/pan + WASD + 시네마틱 4종

---

## [2026-03-11] Animation System Architecture
> 🇰🇷 [2026-03-11] 애니메이션 시스템 아키텍처
> **Category:** animation | **Status:** ✅ 확정 | **Superseded By:** —

데이터 기반 애니메이션 파이프라인 (Doc 1–6 설계 완료):
- **AnimationConfig** (static class) — `animation_config.json` 파싱, loop/speed/event 쿼리
- **AnimEventDispatcher** — ratio 기반 이벤트 발화 (hit_frame / sfx / fx / camera_effect)
- **UnitRenderer3D** — 모든 3D 비주얼 담당 (모델 로드, 애니, HP바, 발판)
- **CameraDirector** — execute(preset, ctx) 단일 진입점, 4종 시네마틱
- **CompositionBuilder** — AnimationTree sequence/blend/layer 빌더
- 모션 이름 → JSON → AnimationConfig 조회 → AnimationPlayer 재생

---

## [2026-03-11] Asset Library System
> 🇰🇷 [2026-03-11] 에셋 라이브러리 시스템
> **Category:** asset | **Status:** ✅ 확정 | **Superseded By:** —

- `assets/_library/` — 원본 에셋 보관소 (`.gdignore`로 Godot 임포트 차단)
- `assets/characters/{class}/` — 승격된 실사용 에셋 위치
- 승격 이력: `ASSET_LOG.md` 기록 (출처 / 라이선스 / 클립명 포함)
- 정책 문서: `handoff/plans/policy/ASSET_POLICY.md`
- **현재 사용 모델:** Quaternius RPG Classes FBX (CC0) — warrior/ranger/rogue/wizard

---

## [2026-03-11] FBX Animation Loading Pattern
> 🇰🇷 [2026-03-11] FBX 애니메이션 로딩 패턴
> **Category:** asset | **Status:** ✅ 확정 | **Superseded By:** —

Quaternius RPG Classes FBX는 한 파일에 모든 클립 내장:
- `ANIM_PATHS` 엔트리: `{"path": "res://..fbx", "clip": "ClipName"}` Dict 형식
- `_import_anim(path, anim_name, clip_name)` — 정확 매칭 → `|suffix` 폴백
- 텍스처: FBX 외부 참조 경로 깨짐 → `_apply_png_textures()` 로 PNG 직접 적용

---

## [2026-03-12] Unit Visual System
> 🇰🇷 [2026-03-12] 유닛 비주얼 시스템
> **Category:** renderer | **Status:** ✅ 확정 | **Superseded By:** —

**HP 게이지:**
- UnitRenderer3D 내부에서 QuadMesh 2개 (배경 + 채움)
- `_process()` 에서 `look_at(pos - to_cam)` — QuadMesh +Z 앞면 카메라 향함
- `HP_BAR_Y = 1.8` — 모델 크기에 따라 조정

**파벌 서클 (Faction Disc):**
- CylinderMesh 평면 디스크 — 아군 파란, 적 빨간
- `position.y = 0.06` — 타일 상단(0.05) 위에 위치
- 표시 조건: 아군=선택시, 적=행동시

**초기 방향:**
- 스폰 직후 `_face_units_toward_opponents()` — 가장 가까운 적 방향

## [2026-03-12] Git–Drive Asset Sync Policy
> 🇰🇷 [2026-03-12] Git–Drive 에셋 동기화 정책
> **Category:** workflow | **Status:** ✅ 확정 | **Superseded By:** —

Dev 폴더 전체가 Google Drive로 동기화되므로, 바이너리 에셋은 git에서 제외한다.
> 🇰🇷 Google Drive가 전체 백업을 담당하므로 git은 코드/데이터만 관리.

- **git 관리 대상:** GDScript (`.gd`), JSON (`data/`), 설정 (`project.godot`), 문서 (`handoff/`), 도구 (`tools/`)
  > 🇰🇷 텍스트 기반 파일만 git 추적
- **git 제외 (`.gitignore`):** `assets/_library/`, `assets/characters/`, `assets/models/units/`
  > 🇰🇷 FBX/GLB/PNG 등 바이너리 에셋은 Drive 동기화에 위임
- **Rationale:** git은 바이너리 diff 불가 → 리포 비대화만 초래. Drive가 이미 백업 역할 수행.
  > 🇰🇷 근거: git 바이너리 버전관리 비효율. Drive 동기화로 충분.
- **Rule:** 새 에셋 디렉토리 추가 시 `.gitignore`에 함께 등록할 것.
  > 🇰🇷 원칙: 에셋 경로 신설 시 반드시 .gitignore 갱신.

<!-- Add new decisions below this line -->
