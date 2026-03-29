# CrownFalle — Dialogue Prototype 현황
> 갱신: 2026-03-29 (4차) | 스토리 내용 미포함 (시스템/진행현황만)

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
| 실행 주소 | `http://localhost:8090` | `http://localhost:8090/c-mode/` |
| 데이터 | `data/nodes/*.json` | `ink/stories/*.ink.json` |
| 상태 | 892노드 / 15씬 완성 | **시스템 구축 완료, 테스트씬 동작** |

```bash
cd "C:\Users\iseul\OneDrive\Dev\Projects\crown-falle-dialogue-proto"
python -m http.server 8090
```

---

## ink.js C모드 — 구현 현황 (2026-03-29 완료)

### 완료
- [x] 태그 파싱 시스템 (`tag-parser.js`) — 전체 태그 지원
- [x] ink 엔진 래퍼 (`ink-engine.js`) — 큐 기반 step(), 이벤트 분류
- [x] C모드 렌더러 (`c-mode-renderer.js`) — 전체 요소 렌더링
- [x] 메인 루프 (`main.js`) — advance/step, 디버그 패널(D키), nextScene 로깅
- [x] 스타일 (`c-mode.css`) — 전 캐릭터 색상, mode-divider
- [x] `data/characters_traits.json` — 5캐릭터 axis_weights
- [x] `ink/includes/variables.ink` — met_armand/met_employer_woman 추가
- [x] `ink/stories/test_c_mode.ink` — 동작 확인
- [x] ink 씬 작성 컨벤션 문서 작성

### 다음 작업
- [ ] `ch1_ironmead_square.ink` — 1-1 이식 (콘텐츠 작업)
- [ ] `relation-engine.js` — axis_weights → rel_* 계산 (2번 작업, Desktop 설계 후)

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
| `2026-03-29-ink-scripting-conventions.md` | ink 씬 작성 컨벤션 |
| `2026-03-29-game-direction-v2.md` | 3축/관계 시스템 전체 설계 |
