# Contents Protocol — story <-> art Integrated Workflow
> 🇰🇷 story와 art 간 통합 워크플로우 지침

---

## 1. Story Workflow
> 🇰🇷 스토리 작업 워크플로우

| Step | Actor | Output | Path |
|------|-------|--------|------|
| Braindump | User + Desktop | Markdown draft | `story/story/drafts/braindump/` |
| Canon promotion | User + CLI/Cowork | Formal setting doc | `story/story/canon/{category}/` (future) |
| STORY_INDEX update | Story worker | STORY_INDEX.json update | Project root |
| Art impact notice | Story worker -> @ArtDirector | Change summary + affected entities | Handoff or chat |

> 🇰🇷 브레인덤프 -> 캐논 확정 -> STORY_INDEX 갱신 -> art 영향 알림

---

## 2. Story Change -> Art Notification Protocol
> 🇰🇷 스토리 변경 시 art 알림 프로토콜

1. **STORY_INDEX.json updated** — record changed entities (version field updated)
> 🇰🇷 STORY_INDEX.json 갱신 시 변경 엔티티 기록

2. **@ArtDirector session start** — check if STORY_INDEX.json version differs from last check
> 🇰🇷 @ArtDirector 세션 시작 시 version 차이 감지

3. **Impact analysis** — find sheet.json/scene.json with story_ref pointing to changed entities
> 🇰🇷 영향 분석: 변경 엔티티에 story_ref가 걸린 시트/장면 목록

4. **If needed** — re-enter character design process or update scene prompts
> 🇰🇷 필요 시 외형 시트 수정 또는 장면 프롬프트 갱신

---

## 3. Art -> Game Asset Delivery (TBD)
> 🇰🇷 art에서 게임으로 에셋 전달 경로 (미확정)

```
contents/.../art/output/_favorites/ -> Projects/crown-falle-TBSF/Assets/Art/
```

Delivery method (manual copy / symlink / script) to be decided after TBSF demo.
Current status: path confirmed, method TBD.

> 🇰🇷 전달 방식은 TBSF 데모 후 확정. 현재는 경로만 확인.

---

## 4. Core Principles
> 🇰🇷 핵심 원칙

- art -> story is **read-only** — art never modifies story files
> 🇰🇷 art에서 story는 읽기 전용

- STORY_INDEX.json is **managed by story project** — art does not modify it
> 🇰🇷 STORY_INDEX.json은 story 프로젝트가 관리

- story_ref in art JSON is for **tracking/reference** — not auto-inserted into prompts
> 🇰🇷 story_ref는 추적/참고용, 프롬프트 자동 삽입 아님

- story_ref absence does not block generation — only limits @ArtDirector context analysis
> 🇰🇷 story_ref 없어도 생성 가능, @ArtDirector 맥락 분석 근거만 없어짐

---

## 5. Claude Desktop Sync Protocol
> 🇰🇷 Claude Desktop 컨텍스트 동기화 프로토콜

### 5-1. Background
> 🇰🇷 배경

Desktop reads context via **raw URLs** from the public handoff repo (`Projects/crown-falle-handoff/`).
Story/art **protocol, agents, workflow** are synced to the handoff repo.
Story/art **content** (braindump, prompts, images) stays private — never synced.
For content that is not in the handoff repo, use **Cowork** (CLI file reads) on demand.
> 🇰🇷 Desktop은 핸드오프 리포(public)의 raw URL로 컨텍스트를 읽음. 프로토콜/에이전트/워크플로우만 동기화. 스토리/아트 내용은 비공개 — 필요 시 코워크로 CLI에게 읽기 요청.

### 5-2. Handoff Repo Sync — Protocol / Agents / Workflow
> 🇰🇷 핸드오프 리포 동기화 — 프로토콜/에이전트/워크플로우

These files are synced from `contents/` to `Projects/crown-falle-handoff/contents/`.
Desktop reads them via GitHub raw URLs. CLI must re-sync after changes.
> 🇰🇷 아래 파일은 contents/ → 핸드오프 리포 contents/로 동기화. 변경 시 CLI가 재동기화.

**Sync source:** `Dev/contents/` → **Sync dest:** `Projects/crown-falle-handoff/contents/`

**Workflow / Protocol:**

| File | Handoff Path | Purpose |
|------|-------------|---------|
| Contents Protocol | `contents/PROTOCOL.md` | story<->art workflow + Desktop sync rules |
| Story Protocol | `contents/crown-falle-interlude/story/PROTOCOL.md` | Story naming, status, folder rules |
| Story Index | `contents/crown-falle-interlude/story/INDEX.md` | Document status tracker |

**Art System:**

| File | Handoff Path | Purpose |
|------|-------------|---------|
| Art Agents | `contents/crown-falle-interlude/art/AGENTS.md` | @ArtDirector, @Prompter, @Curator definitions |
| Art Config | `contents/crown-falle-interlude/art/prompts/_config.json` | Convention groups, style prefixes |
| Art Categories | `contents/crown-falle-interlude/art/prompts/_categories.md` | Image category list |

**Story Agents (all `contents/crown-falle-interlude/story/agents/*.md`):**

| File | Purpose |
|------|---------|
| `README.md` | Agent overview + usage rules |
| `pipeline.md` | Agent pipeline definition (macro/micro) |
| `pipeline-diagram.md` | Pipeline visual diagram |
| `systems.md` | System descriptions |
| `architect.md` | Architect agent definition |
| `carry.md` | Carry agent definition |
| `challenger.md` | Challenger agent definition |
| `dreamer.md` | Dreamer agent definition |
| `editor.md` | Editor agent definition |
| `expander.md` | Expander agent definition |
| `historian.md` | Historian agent definition |
| `keeper.md` | Keeper agent definition |
| `pacemaker_macro.md` | Pacemaker (macro) agent definition |
| `pacemaker_micro.md` | Pacemaker (micro) agent definition |
| `reader.md` | Reader agent definition |
| `scribe.md` | Scribe agent definition |
| `world_forge.md` | World Forge agent definition |

### 5-3. Cowork — Dynamic Files (Read On Demand)
> 🇰🇷 코워크 — 동적 파일 (필요할 때 읽기)

Do NOT attach these to Project Knowledge. Read via Cowork when needed.
> 🇰🇷 프로젝트 지식에 첨부하지 않음. 필요할 때 코워크로 CLI에게 읽기 요청.

| File | When to Read | Cowork Command Example |
|------|-------------|----------------------|
| Braindump | Character design, scene planning | "braindump에서 Tristram 설정 읽어줘" |
| STORY_INDEX.json | Checking entity locations | "STORY_INDEX.json 현재 상태 확인해줘" |
| sheet.json | Before character discussion | "Tristram sheet.json 읽어줘" |
| scene.json | Before scene discussion | "campfire_isolation scene.json 읽어줘" |
| Character sheets (all) | Comparing characters | "현재 확정된 캐릭터 시트 전부 읽어줘" |

### 5-4. Desktop Session Start Checklist
> 🇰🇷 Desktop 세션 시작 체크리스트

```
1. Project Knowledge attached? (AGENTS.md, _config.json at minimum)
2. Which domain?
   - Story work  -> Cowork: "braindump 최신 내용 읽어줘"
   - Art work    -> Cowork: "STORY_INDEX.json + 작업 대상 sheet.json 읽어줘"
   - Game work   -> Handoff repo raw URLs (public)
3. Previous session state?
   - Cowork: "handoff/CURRENT.md 읽어줘"
```

> 🇰🇷 1. 프로젝트 지식 첨부 확인 → 2. 도메인별 코워크 읽기 → 3. 이전 세션 상태 확인

### 5-5. After Changes — Sync to Handoff Repo
> 🇰🇷 작업 후 — 핸드오프 리포 동기화

| What Changed | Action |
|-------------|--------|
| Protocol/agent/workflow files | Re-sync changed files to `Projects/crown-falle-handoff/contents/` |
| Convention confirmed/changed | Re-sync _config.json to handoff repo |
| New agent rules | Re-sync AGENTS.md or story agent files to handoff repo |
| Character sheet confirmed | No handoff sync needed (private) |
| Story braindump updated | No handoff sync needed (private) — Cowork reads on demand |
| New category added | Re-sync _categories.md to handoff repo |

> 🇰🇷 프로토콜/에이전트/워크플로우 변경 시 핸드오프 리포 재동기화. 스토리/아트 내용은 동기화 불필요.

### 5-6. Privacy Boundary
> 🇰🇷 비공개 경계

| Content | Public (Handoff Repo) | Private (Local Only) |
|---------|----------------------|---------------------|
| Game system designs | O | - |
| Game decisions | O | - |
| Story braindump | X | O |
| Character sheets (art) | X | O |
| Scene prompts (art) | X | O |
| Art agent definitions | X | O (Project Knowledge) |
| Generated images | X | O |

> 🇰🇷 스토리/아트 관련 내용은 절대 퍼블릭 리포에 올리지 않음. Desktop에는 프로젝트 지식 또는 코워크로만 전달.
