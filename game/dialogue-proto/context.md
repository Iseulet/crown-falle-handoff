# CrownFalle — Dialogue Prototype 현황
> 갱신: 2026-03-28 | 스토리 내용 미포함 (시스템/진행현황만)

---

## 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 목적 | C↔A 하이브리드 대화 씬의 체감과 리듬 검증 |
| 범위 | 대화/스토리 전개 (전투/자원관리 제외) |
| 구현 | 웹 (HTML/CSS/JS + JSON 데이터) |
| 경로 | `C:\Users\iseul\OneDrive\Dev\Projects\crown-falle-dialogue-proto\` |
| 계획서 | `Dev/handoff/plans/design/2026-03-24-dialogue-proto-plan-v1.1.md` |

---

## 실행 방법 / 접속 주소

```bash
cd "C:\Users\iseul\OneDrive\Dev\Projects\crown-falle-dialogue-proto"
python -m http.server 8080
```

**접속:** `http://localhost:8080`

> 반드시 로컬 서버로 실행해야 함. `file://` 프로토콜은 CORS 오류 발생.

---

## 구현 현황

### 완료
- [x] C모드 렌더러 (텍스트 히스토리 누적, Citizen Sleeper 방식)
- [x] A모드 렌더러 (전체화면 VN, 텍스트 교체)
- [x] C↔A 모드 전환 (fade/cut)
- [x] 선택지 분기 + 합류
- [x] 씬 종료 화면 (fade to black + 다시 시작)
- [x] 이미지 placeholder fallback (파일 없을 때 자동 생성)
- [x] 키보드 단축키 (스페이스/Enter/숫자키)
- [x] 배경 전환 (bg_change 노드)
- [x] 씬 체인 (next 필드로 씬 간 연결)

### 데이터 현황
- 캐릭터: **9명** (tristram, seren, wren, armand, colt, flay, the-scald, dealer, employer)
- 씬: **15개** (ch1 × 4, ch2 × 6, ch3 × 5)
- 노드 파일: 15개 (`data/nodes/*.json`)

### 미완료 / 다음 작업
- [ ] 이미지 에셋 배치 — profiles/standings/bg 실제 이미지 연결
- [ ] 노드 파일 내용 충실화 (현재 placeholder 또는 미작성)
- [ ] 씬 간 자동 연결 실행 (next 필드 구현됨, 연결 UI 미구현)
- [ ] 성향 태그(tags) 시스템 연동

---

## 관련 툴 접속 주소

| 툴 | 주소 | 실행 명령 |
|---|---|---|
| dialogue-proto | `http://localhost:8080` | `python -m http.server 8080` (프로젝트 루트) |
| doc-viewer (프론트) | `http://localhost:5400` | `npm run dev` (`_tools/doc-viewer/`) |
| doc-viewer (API 서버) | `http://localhost:8000` | `python server.py` (`_tools/doc-viewer/`) |
