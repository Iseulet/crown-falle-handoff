# CrownFalle — Asset Pipeline Guide
> 🇰🇷 에셋 파이프라인 가이드

Full authoritative policy: `handoff/plans/policy/ASSET_POLICY.md`
> 🇰🇷 전체 원본 정책: `handoff/plans/policy/ASSET_POLICY.md`

Git exclusion policy: `handoff/DECISIONS.md` [2026-03-12] Git–Drive Asset Sync Policy
> 🇰🇷 git 제외 정책: `handoff/DECISIONS.md` [2026-03-12]

---

## Folder Structure
> 🇰🇷 폴더 구조

```
assets/
  _library/                    ← External resources staging area (.gdignore applied)
    characters/<source>/       ← e.g. quaternius, synty, meshy
    environment/<source>/
    raw/                       ← Unclassified post-extraction
  characters/                  ← Promoted (production-ready) assets only
    shared/
    {class}/
  environment/
  animations/
```

---

## Library Rules (L-1 ~ L-6)
> 🇰🇷 라이브러리 규칙

| # | Rule |
|---|------|
| L-1 | Always place external resources in `_library/<source>/` first |
| L-2 | Keep resources from different sources in different folders |
| L-3 | Never modify files in `_library/` — preserve originals |
| L-4 | Never delete arbitrarily |
| L-5 | Keep `.gdignore` at `_library/` root to prevent Godot auto-import |
| L-6 | Create `LICENSE.txt` in each source folder |

> 🇰🇷 L-1: 외부 리소스는 반드시 `_library/<출처명>/`에 먼저 배치
> 🇰🇷 L-2: 출처가 다른 리소스는 다른 폴더에 보관
> 🇰🇷 L-3: `_library` 내 파일 수정 금지 (원본 보존)
> 🇰🇷 L-4: 임의 삭제 금지
> 🇰🇷 L-5: `_library` 루트에 `.gdignore` 유지
> 🇰🇷 L-6: 각 출처 폴더에 `LICENSE.txt` 필수 작성

---

## Promotion Procedure (_library → Production)
> 🇰🇷 승격 절차 (_library → 정식 경로)

```
1. [Select]   Choose target file in _library/
2. [Verify]   Run tools/verify_skeleton.py — check bone names against common skeleton
              If mismatch: fix in Blender, re-export, then proceed
3. [Copy]     Copy to production path (NEVER move — keep original in _library)
4. [Import]   Apply settings in Godot Import tab
5. [Log]      Add entry to ASSET_LOG.md
```

### Promotion Rules (P-1 ~ P-4)
> 🇰🇷 승격 규칙

| # | Rule |
|---|------|
| P-1 | COPY only — never move originals out of `_library/` |
| P-2 | Bone name verification required before promotion |
| P-3 | ASSET_LOG.md entry required for every promotion |
| P-4 | Production filenames must follow `snake_case` convention |

> 🇰🇷 P-1: 이동(move) 금지, 반드시 복사(copy)
> 🇰🇷 P-2: 본 이름 검증 통과 후에만 승격
> 🇰🇷 P-3: 승격 시 반드시 ASSET_LOG.md 기록
> 🇰🇷 P-4: 정식 경로 파일명은 snake_case

---

## ASSET_LOG.md Recording Rules (A-1 ~ A-4)
> 🇰🇷 ASSET_LOG.md 기록 규칙

| # | Rule |
|---|------|
| A-1 | Record every new acquisition |
| A-2 | Record every promotion |
| A-3 | Record every deletion or replacement |
| A-4 | Separate sections by date |

Log format:
```markdown
## YYYY-MM-DD

### Promoted
- `_library/characters/quaternius/Knight.glb`
  → `assets/characters/fighter/knight_base.glb`
  - Source: quaternius.com (CC0)
  - Bone verification: Mixamo-compatible confirmed
  - Used in: test_combat.tscn

### New Acquisition
- Quaternius Medieval Village Pack
  - Path: `_library/environment/quaternius/medieval_village/`
  - License: CC0
```

---

## Current Assets in Production
> 🇰🇷 현재 사용 중인 에셋

| Asset | Source | License | Production Path |
|-------|--------|---------|-----------------|
| warrior_rpg.fbx | Quaternius RPG Classes | CC0 | `assets/characters/fighter/` |
| ranger_rpg.fbx | Quaternius RPG Classes | CC0 | `assets/characters/archer/` |
| rogue_rpg.fbx | Quaternius RPG Classes | CC0 | `assets/characters/rogue/` |
| wizard_rpg.fbx | Quaternius RPG Classes | CC0 | `assets/characters/mage/` |

---

## Git Exclusion Policy
> 🇰🇷 git 제외 정책

- Binary assets (FBX/GLB/PNG) are excluded from git — Google Drive handles backup
- When adding new asset directories: always update `.gitignore`
- Git tracks: `.gd`, `data/` JSON, `project.godot`, `handoff/`, `tools/`

> 🇰🇷 바이너리 에셋은 git 제외 — Google Drive가 백업 담당
> 🇰🇷 에셋 경로 신설 시 반드시 `.gitignore` 갱신

---

## New Source Checklist
> 🇰🇷 신규 출처 추가 체크리스트

- [ ] Create `_library/<source>/` folder
- [ ] Write `LICENSE.txt`
- [ ] Confirm `.gdignore` covers this path (parent folder check)
- [ ] Add acquisition entry to `ASSET_LOG.md`
