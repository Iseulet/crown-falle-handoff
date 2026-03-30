# CrownFalle — Dialogue Prototype 현황
> 갱신: 2026-03-30 (5차) | 스토리 내용 미포함 (시스템/진행현황만)

---

## 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 목적 | C↔A 하이브리드 대화 씬 + ink.js 파라미터 시스템 검증 |
| 범위 | 대화/스토리 전개 (전투/자원관리 제외) |
| 구현 | 웹 (HTML/CSS/JS) — JSON 기반 + ink.js C모드 병존 |
| 경로 | `C:\Users\iseul\OneDrive\Dev\Projects\crown-falle-dialogue-proto\` |

---

## 시스템 구조 (2개 병존)

| | 기존 JSON 시스템 | 신규 ink.js C모드 |
|---|---|---|
| 진입점 | `index.html` | `c-mode/index.html` |
| 실행 주소 | `http://localhost:8080` | `http://localhost:8080/c-mode/` |
| 데이터 | `data/nodes/*.json` | `ink/stories/*.ink.json` |
| 상태 | 892노드 / 15씬 완성 | **1-1 씬 완성 + 배경 이미지 연결** |

```bash
cd "C:\Users\iseul\OneDrive\Dev\Projects\crown-falle-dialogue-proto"
python -m http.server 8080
```

---

## ink.js C모드 — 구현 현황 (2026-03-30 완료)

### 완료
- [x] 태그 파싱 시스템, ink 엔진 래퍼, C모드 렌더러, 메인 루프, 스타일
- [x] `ch1_ironmead_square.ink` — 9비트 전체 작성 + 에이전트 피드백 반영
- [x] 선택지 컨벤션 확립 (같은 축 페어링 / 3칸 레이아웃 / neutral 가운데)
- [x] voice_pair + choices 병합 렌더링 (엔진 레벨 — voices 파라미터 전달)
- [x] 씬별 배경 이미지 연결 (scene-1a~1d, fade 전환 0.6s)
- [x] `handoff/ink-scripting-conventions.md` 작성
- [x] art-direction-viewer: 이미지 영속성 수정 (blob→서버 파일), ComfyUI 탭 추가

### 다음 작업
- [ ] `relation-engine.js` — axis_weights → rel_* 계산 (Desktop 설계 후)
- [ ] 추가 씬 ink 작성 (ch1_ironmead_warehouse 등)
- [ ] scene-1d 배경 이미지 생성 (현재 미완)

---

## 기존 JSON 프로토 — 데이터 현황

- 캐릭터: **9명** (tristram, seren, wren, armand, colt, flay, the-scald, dealer, employer)
- 씬: **15개** (ch1 × 4, ch2 × 6, ch3 × 5) — 유지
- 노드: **892개** (2026-03-29 기준)

---

## 계획 문서 목록

| 문서 | 내용 |
|---|---|
| `2026-03-29-web-inkjs-cmode-plan-v1.md` | ink.js C모드 구조 |
| `2026-03-29-ch1-1-ink-migration-plan.md` | 1-1 이식 계획 (v2) |
| `2026-03-29-choice-system-design.md` | 선택지/캐릭터 반응 시스템 |
| `handoff/ink-scripting-conventions.md` | ink 씬 작성 컨벤션 (신규) |
