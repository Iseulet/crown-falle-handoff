# CrownFalle — Data-Driven Design Guide
> 🇰🇷 데이터 기반 설계 가이드

All confirmed decisions documented in `handoff/DECISIONS.md` [2026-03-08].
> 🇰🇷 모든 확정 결정사항은 `handoff/DECISIONS.md` [2026-03-08]에 기록됨.

---

## Core Principles
> 🇰🇷 핵심 원칙

1. All content values (numbers, positions, names) must be in JSON data files — not hardcoded
   > 🇰🇷 모든 콘텐츠 수치/위치/이름은 JSON 파일에 정의. GDScript 하드코딩 금지.
2. Adding content = adding/editing JSON only (no GDScript changes for content)
   > 🇰🇷 콘텐츠 추가 = JSON 추가/수정만으로 가능해야 함
3. Composition over creation — check existing motions before creating new ones
   > 🇰🇷 새 모션 생성 전 기존 모션 조합(Sequence/Blend/Layer) 가능한지 먼저 검토
4. Projectiles are async — attacker returns to idle immediately on projectile launch
   > 🇰🇷 투사체는 비동기 — 발사 즉시 공격자 idle 복귀

---

## Data File Map
> 🇰🇷 데이터 파일 맵

| Domain | File Path | Contents |
|--------|-----------|----------|
| Encounters | `data/encounters/encounter_*.json` | Enemy/ally spawn, positions |
| Combat rules | `data/combat_config.json` | max_turns, hit_radius |
| Roster | `data/mercenaries/roster_config.json` | Initial roster paths |
| World nodes | `data/world/world_nodes.json` | NPC/EncounterZone positions |
| NPC | `data/npcs/npc_*.json` | Dialogue, quest_id |
| Quests | `data/quests/quest_*.json` | Rewards (food, gold) |
| Level-up | `data/classes/levelup_config.json` | Stat allocation per class |
| World | `data/world_config.json` | Fatigue, move speed |
| Camp | `data/camp_config.json` | Camp recovery values |
| Class config | `data/classes/class_config.json` | Base stats per class |
| Class detail | `data/classes/{class}.json` | fighter, archer, mage, rogue |
| Animations | `data/animations/animation_config.json` | Loop, speed, events per motion |
| Bone masks | `data/animations/bone_masks.json` | Bone mask definitions |
| Compositions | `data/animations/compositions/` | Sequence/blend/layer JSONs |
| Camera presets | `data/cameras/camera_config.json` | Camera effect presets |
| Combat camera | `data/cameras/combat_rules.json` | Combat camera rules |
| Projectiles | `data/projectiles/{type}.json` | arrow, magic_bolt, throw_stone |

---

## New Value Addition Procedure
> 🇰🇷 신규 수치 추가 절차

1. Identify which JSON file owns this type of data (see Data File Map above)
   > 🇰🇷 해당 데이터를 소유하는 JSON 파일 식별
2. Add the value to the JSON file
   > 🇰🇷 JSON 파일에 수치 추가
3. Update the corresponding loader/singleton to expose the new value
   > 🇰🇷 해당 로더/싱글톤을 업데이트하여 새 값 노출
4. Reference the value in GDScript via the loader — never inline it
   > 🇰🇷 GDScript에서 로더를 통해 참조 — 직접 기재 절대 금지

---

## Key Singletons and Loaders
> 🇰🇷 핵심 싱글톤 및 로더

| Singleton | Source File | Data |
|-----------|-------------|------|
| `AnimationConfig` | `scripts/singletons/AnimationConfig.gd` | `animation_config.json` |
| `CameraConfig` | `scripts/singletons/CameraConfig.gd` | `camera_config.json` |

---

## Animation System Data Flow
> 🇰🇷 애니메이션 시스템 데이터 흐름

```
Motion name
  → AnimationConfig.get_motion(name)
    → AnimationPlayer.play(clip)
    → event ratio triggers
      → AnimEventDispatcher.event_triggered signal
```

- Never fire animation events immediately — always use ratio-based timing
  > 🇰🇷 애니메이션 이벤트는 즉시 발화 금지 — 반드시 ratio 기반 타이밍 사용

---

## FBX Animation Loading Pattern
> 🇰🇷 FBX 애니메이션 로딩 패턴

Quaternius RPG Classes FBX — all clips in one file:
> 🇰🇷 Quaternius RPG Classes FBX는 한 파일에 모든 클립 내장:

- `ANIM_PATHS` entry format: `{"path": "res://..fbx", "clip": "ClipName"}` Dict
  > 🇰🇷 `ANIM_PATHS` 항목 형식: `{"path": "res://..fbx", "clip": "ClipName"}` Dict
- `_import_anim(path, anim_name, clip_name)` — exact match then `|suffix` fallback
  > 🇰🇷 정확 매칭 → `|suffix` 폴백 순서
- Textures: FBX external refs broken → use `_apply_png_textures()` with keyword-based mesh detection
  > 🇰🇷 FBX 텍스처 외부 참조 깨짐 → 키워드 기반 감지로 PNG 직접 적용
