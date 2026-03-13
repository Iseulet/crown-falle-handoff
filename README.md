# CrownFalle Handoff
> 🇰🇷 CrownFalle Proto 공개 핸드오프 리포지토리

Public management documents for **CrownFalle Proto** — a turn-based tactical RPG built with Godot 4.6.1 + GDScript.

This repo enables Claude Desktop to read the latest project context without manual file uploads.
> 🇰🇷 Claude Desktop이 수동 파일 업로드 없이 최신 프로젝트 컨텍스트를 읽기 위한 공개 리포

**Source code lives in the private repo** `crown-falle-proto`. This repo contains only management documents.
> 🇰🇷 소스코드는 비공개 리포 `crown-falle-proto`에 있음. 이 리포는 관리 문서만 포함.

---

## Raw URL Index (for Claude Desktop)
> 🇰🇷 Claude Desktop용 Raw URL 목록

Base: `https://raw.githubusercontent.com/Iseulet/crown-falle-handoff/main/`

| File | URL | When to Read |
|------|-----|-------------|
| project-context.md | `context/project-context.md` | Every session start |
| project-reference.md | `context/project-reference.md` | Every session start |
| DECISIONS.md | `decisions/DECISIONS.md` | Architecture questions |
| SCRATCHPAD.md | `memory/SCRATCHPAD.md` | Before any implementation advice |
| Design Index | `designs/INDEX.md` | Design discussions |

---

## Directory Structure
> 🇰🇷 디렉토리 구조

```
crown-falle-handoff/
├── context/          ← 🔄 Changes every sync — Claude Desktop primary read
│   ├── project-context.md
│   └── project-reference.md
├── decisions/        ← 🔄 Changes on architecture decisions
│   └── DECISIONS.md
├── agents/           ← 📌 Rarely changes — role definitions
│   ├── ARCHITECTURE.md
│   ├── PLANNER.md
│   ├── IMPLEMENTOR.md
│   └── REVIEWER.md
├── memory/           ← 🔄 Changes as learnings accumulate
│   └── SCRATCHPAD.md
├── skills/           ← 📌 Rarely changes — domain knowledge
│   ├── gdscript-conventions.md
│   ├── data-driven-design.md
│   └── asset-pipeline.md
└── designs/          ← 📌 Grows over time — all design documents
    ├── INDEX.md
    └── *.md
```

---

## Sync Policy
> 🇰🇷 동기화 정책

- **Direction:** One-way only — `crown-falle-proto` → `crown-falle-handoff`
- **Trigger:** `/sync` command in Claude Code CLI, or `tools/sync_to_public.ps1`
- **Never edit this repo directly** — always sync from the private repo
- **Commit format:** `[싱크] YYYY-MM-DD 컨텍스트 갱신`
- **History:** No archive/ folder — use `git log -- <file>` for past versions
