# CrownFalle — Learning Scratchpad
> 🇰🇷 학습 스크래치패드
> This file is read by every agent at task start.
> Update immediately when discovering new mistakes or patterns.
> 🇰🇷 이 파일은 모든 에이전트가 작업 시작 시 읽는다.
> 새로운 실수/패턴 발견 시 즉시 업데이트한다.

---

## 🔴 Absolute Rules (Check Every Time)
> 🇰🇷 절대 규칙 (매번 확인)

### Data & Code

- JSON key `int` → GDScript `int_stat` mapping (reserved keyword conflict)
  > 🇰🇷 JSON의 `int` 키 → GDScript에서 `int_stat`으로 매핑 (예약어 충돌)
- JSONC comments: only `//`. Never use `_comment` field (tried and reverted)
  > 🇰🇷 JSONC 코멘트는 `//`만 사용. `_comment` 필드 사용 금지
- `assets/` folder: do NOT modify — Godot editor only
  > 🇰🇷 `assets/` 폴더 직접 수정 금지 — Godot 에디터 전용
- `project.godot`: do NOT modify without explicit user approval
  > 🇰🇷 `project.godot` 수정 금지 — 명시적 승인 필요

### Operational [V1, V2, V5]

- CURRENT.md writes: always verify after writing. If interrupted mid-write,
  next session must reconcile from `handoff/LOG/YYYY-MM-DD.md` as source of truth.
  > 🇰🇷 CURRENT.md 쓰기 후 반드시 검증. 중단 시 LOG가 진실의 원천.
- After ANY file modification (even 1-line fix): run the 5-step completion checklist + 6. TODO.md 갱신.
  If unsure whether it's a task or conversation: it's a task. Log it.
  > 🇰🇷 파일 수정 후 (1줄이라도) 반드시 5+1단계 체크리스트 실행. 6. TODO.md 갱신 확인. 애매하면 작업으로 간주.
- Large implementations (3+ files): always propose checkpoint before starting.
  Never modify 3+ files without a safety net (git stash or WIP commit).
  > 🇰🇷 대규모 구현 (3+ 파일): 시작 전 항상 체크포인트 제안. 안전망 없이 진행 금지.

---

## ⚠️ Frequent Mistakes
> 🇰🇷 자주 발생하는 실수

- Passive skill `effects[]`: missing `duration` and `chance` fields
  > 🇰🇷 패시브 스킬 `effects[]`에 `duration`과 `chance` 필드 누락
- `on_engage` trigger: needs `engage_just_started` condition to prevent stacking
  > 🇰🇷 `on_engage` 트리거에서 `engage_just_started` 조건 없이 스택 방지 안 됨
- FBX clip names: `CharacterArmature|Idle` format — need `|suffix` fallback
  > 🇰🇷 FBX 클립 이름: `CharacterArmature|Idle` 형식 — `|suffix` 폴백 필요
- QuadMesh HP bar `look_at()` direction: `pos - to_cam` (faces camera, not away)
  > 🇰🇷 QuadMesh HP바의 `look_at()` 방향: `pos - to_cam`

---

## 💡 Useful Patterns
> 🇰🇷 유용한 패턴

### Game Systems

- Class overrides: `class_overrides` for mv_base, crit_bonus
  > 🇰🇷 직업별 오버라이드: `class_overrides`로 mv_base, crit_bonus 처리
- Enemy level growth: automatic primary stat allocation per class
  > 🇰🇷 적 레벨 성장: 자동 1차 스탯 할당 per class
- Camera event timing: sync via animation_config.json ratio (never fire immediately)
  > 🇰🇷 카메라 이벤트 타이밍: ratio로 동기화 (즉시 발화 X)
- Faction disc: default `visible = false`, conditional show (ally=selected, enemy=acting)
  > 🇰🇷 파벌 서클: 기본 `visible = false`, 조건부 표시

### Operational [V4, V9, V10]

- Agent switches: declare what was re-read from disk. "📖 Read: [file list]"
  If you feel like skipping because "I already know it" — that's exactly when you must read.
  > 🇰🇷 에이전트 전환: 읽은 파일을 선언. "이미 알고 있어서" 스킵하려는 순간이
  > 반드시 읽어야 하는 순간.
- LOG entry format:
  ```
  ## YYYY-MM-DD — [Agent] [Task Summary]
  ### Changes
  - [file]: [what changed]
  ### Decisions (if any)
  - [decision]: [rationale]
  ### Notes
  - [learnings, issues encountered]
  ```
  > 🇰🇷 LOG 엔트리: 에이전트명, 변경파일, 결정사항, 메모 포함
- When design docs in `handoff/plans/design/` exceed 10: create `INDEX.md` with status markers
  > 🇰🇷 설계 문서 10개 초과 시 INDEX.md 생성하여 상태(active/done) 표시

---

## 📝 Session Log
> 🇰🇷 세션 기록
<!-- New entries at top -->

### 2026-03-12

- FBX texture path broken → `_apply_png_textures()` keyword-based mesh detection
  > 🇰🇷 FBX 텍스처 경로 깨짐 → 키워드 기반 감지로 해결
- Death camera shake: immediate fire → moved to ratio 0.45 event (more natural)
  > 🇰🇷 사망 카메라 셰이크: 즉시 발화 → ratio 0.45 이벤트로 이동
- HP_BAR_Y=1.8 — needs final adjustment after model size check
  > 🇰🇷 HP_BAR_Y=1.8 — 모델 크기 확인 후 최종 조정 필요
