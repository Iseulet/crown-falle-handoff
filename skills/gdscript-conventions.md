# CrownFalle — GDScript Conventions
> 🇰🇷 GDScript 코딩 컨벤션

Authoritative reference for all naming and coding standards.
Read this file when writing, reviewing, or evaluating any GDScript code.
> 🇰🇷 모든 네이밍 및 코딩 표준의 권위 있는 참조 문서.
> GDScript 코드 작성, 리뷰, 평가 시 반드시 읽는다.

---

## Naming Rules
> 🇰🇷 네이밍 규칙

| Target | Convention | Example |
|--------|-----------|---------|
| Scene files | `snake_case` | `combat_map.tscn` |
| Script files | `PascalCase` | `CombatManager.gd` |
| `class_name` | `PascalCase` | `class_name CombatManager` |
| Functions | `snake_case` | `func get_unit_hp()` |
| Variables | `snake_case` | `var unit_hp = 0` |
| Constants | `UPPER_SNAKE_CASE` | `const MAX_UNITS = 8` |
| Private members | `_underscore_prefix` | `var _internal_state` |
| Signals | `snake_case` past tense | `signal unit_moved`, `signal turn_ended` |
| JSON keys | `snake_case` | `"move_speed"`, `"max_hp"` |

---

## JSON ↔ GDScript Mapping
> 🇰🇷 JSON ↔ GDScript 매핑

### Reserved Keyword Conflicts
> 🇰🇷 예약어 충돌

| JSON key | GDScript variable | Reason |
|----------|-------------------|--------|
| `"int"` | `int_stat` | `int` is a GDScript reserved keyword |

⚠️ Always check for reserved keyword conflicts when adding new JSON keys.
> 🇰🇷 새 JSON 키 추가 시 예약어 충돌 반드시 확인.

---

## Data Externalization Rules
> 🇰🇷 데이터 외부화 규칙

- ALL numeric values (stats, positions, timings) must be defined in JSON files
  > 🇰🇷 모든 수치 (스탯, 위치, 타이밍)는 JSON 파일에 정의
- NO magic numbers in GDScript — reference JSON values via singletons/loaders
  > 🇰🇷 GDScript에 매직 넘버 기재 금지 — 싱글톤/로더를 통해 JSON 값 참조
- Camera effects: reference by preset name from `data/cameras/camera_config.json`
  > 🇰🇷 카메라 효과: `data/cameras/camera_config.json` 프리셋 이름으로만 참조
- Animation events: timing via ratio from `data/animations/animation_config.json`
  > 🇰🇷 애니메이션 이벤트: `data/animations/animation_config.json`의 ratio 기반

---

## Single Entry Point Principle
> 🇰🇷 단일 진입점 원칙

| Domain | Entry Point | Rule |
|--------|-------------|------|
| Camera effects | `CameraDirector.execute(preset_name, context)` | The ONLY way to trigger camera effects |
| Animation events | `AnimEventDispatcher.event_triggered` signal | The ONLY animation event signal |

Never call camera or animation systems directly.
> 🇰🇷 카메라 및 애니메이션 시스템을 직접 호출 금지 — 반드시 지정된 진입점을 통해서.

---

## Modification-Forbidden Files
> 🇰🇷 수정 금지 파일

| File/Folder | Rule | Reason |
|-------------|------|--------|
| `assets/` | Never modify directly | Godot editor manages this |
| `project.godot` | Only with explicit user approval | Engine config |

---

## JSONC Comments
> 🇰🇷 JSONC 코멘트

- Use `//` for inline comments in JSONC files
  > 🇰🇷 JSONC 인라인 코멘트는 `//`만 사용
- NEVER use `_comment` fields — causes parsing issues (tried and reverted)
  > 🇰🇷 `_comment` 필드 사용 절대 금지 — 파싱 오류 발생 (시도 후 되돌림)

---

## Composition Before Creation
> 🇰🇷 생성 전 컴포지션 우선

Before creating a new motion, check if existing motions can be composed:
- Sequence: play A then B
- Blend: lerp between A and B
- Layer: overlay B on top of A's lower body

Reference: `data/animations/compositions/` for existing composition JSONs
> 🇰🇷 새 모션 생성 전 기존 모션 조합(Sequence/Blend/Layer) 가능한지 먼저 검토.
> 참조: `data/animations/compositions/`
