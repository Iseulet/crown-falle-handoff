# CrownFalle Handoff
> 🇰🇷 CrownFalle 공개 핸드오프 리포지토리

Public management documents for **CrownFalle** — a turn-based tactical RPG (Unity 6.3 LTS + C# + TBSF) with integrated worldbuilding and art generation.

This repo enables Claude Desktop to read the latest project context without manual file uploads.
> 🇰🇷 Claude Desktop이 수동 파일 업로드 없이 최신 프로젝트 컨텍스트를 읽기 위한 공개 리포

**Source code lives in private repos.** This repo contains only management documents, organized by project domain.
> 🇰🇷 소스코드는 비공개 리포에 있음. 이 리포는 프로젝트 도메인별 관리 문서만 포함.

---

## Raw URL Index (for Claude Desktop)
> 🇰🇷 Claude Desktop용 Raw URL 목록

Base: `https://raw.githubusercontent.com/Iseulet/crown-falle-handoff/main/`

### Game

| File | URL | When to Read |
|------|-----|-------------|
| project-context.md | `game/context/project-context.md` | Every session start |
| project-reference.md | `game/context/project-reference.md` | Every session start |
| DECISIONS.md | `game/decisions/DECISIONS.md` | Architecture questions |
| SCRATCHPAD.md | `_shared/SCRATCHPAD.md` | Before any implementation advice |
| Design Index | `game/designs/INDEX.md` | Design discussions |

### Contents — Protocol / Agents / Workflow (story content excluded)

| File | URL | When to Read |
|------|-----|-------------|
| Contents Protocol | `contents/PROTOCOL.md` | Session start (story/art work) |
| Story Protocol | `contents/crown-falle-interlude/story/PROTOCOL.md` | Story work |
| Story Index | `contents/crown-falle-interlude/story/INDEX.md` | Story work |
| Story Agent Pipeline | `contents/crown-falle-interlude/story/agents/pipeline.md` | Story work |
| Story Agent Systems | `contents/crown-falle-interlude/story/agents/systems.md` | Story work |
| Art Agents | `contents/crown-falle-interlude/art/AGENTS.md` | Art work |
| Art Config | `contents/crown-falle-interlude/art/prompts/_config.json` | Art work |
| Art Categories | `contents/crown-falle-interlude/art/prompts/_categories.md` | Art work |

> Story agent individual definitions (17 files) also available at `contents/crown-falle-interlude/story/agents/`

---

## Directory Structure
> 🇰🇷 디렉토리 구조

```
crown-falle-handoff/
├── _shared/              ← Cross-project shared resources
│   └── SCRATCHPAD.md
├── game/                 ← Game project (TBSF) handoff
│   ├── context/          ← 🔄 Changes every sync — Claude Desktop primary read
│   │   ├── project-context.md
│   │   └── project-reference.md
│   ├── decisions/        ← 🔄 Changes on architecture decisions
│   │   └── DECISIONS.md
│   ├── agents/           ← 📌 Rarely changes — role definitions
│   │   ├── ARCHITECTURE.md
│   │   ├── PLANNER.md
│   │   ├── IMPLEMENTOR.md
│   │   └── REVIEWER.md
│   ├── skills/           ← 📌 Rarely changes — domain knowledge
│   │   ├── gdscript-conventions.md
│   │   ├── data-driven-design.md
│   │   └── asset-pipeline.md
│   └── designs/          ← 📌 Grows over time — all design documents
│       ├── INDEX.md
│       └── *.md
└── contents/             ← Creative project handoff (protocol/agents/workflow ONLY)
    ├── PROTOCOL.md       ← story<->art integrated workflow
    └── crown-falle-interlude/
        ├── story/
        │   ├── PROTOCOL.md
        │   ├── INDEX.md
        │   └── agents/   ← Story agent definitions (17 files)
        └── art/
            ├── AGENTS.md ← @ArtDirector, @Prompter, @Curator
            └── prompts/
                ├── _config.json
                └── _categories.md
```

---

## Sync Policy
> 🇰🇷 동기화 정책

- **Direction:** One-way only — private repos → `crown-falle-handoff`
- **Trigger:** `/sync` command in Claude Code CLI, or manual sync
- **Never edit this repo directly** — always sync from the source repos
- **Commit format:** `[싱크] YYYY-MM-DD 컨텍스트 갱신`
- **History:** No archive/ folder — use `git log -- <file>` for past versions
