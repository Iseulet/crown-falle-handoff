# CrownFalle — Dialogue Prototype 현황
> 갱신: 2026-03-29 (3차) | 스토리 내용 미포함 (시스템/진행현황만)

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
- [x] C모드 렌더러 (텍스트 히스토리 누적 — 최신 노드 상단 갱신 방식)
- [x] A모드 렌더러 (전체화면 VN, 텍스트 교체)
- [x] A모드 스탠딩 컬러 키 — canvas 기반 배경 제거 (코너 자동감지, tolerance=22)
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
- 노드 파일: 17개 (`data/nodes/*.json`, test 포함)

### 미완료 / 다음 작업
- [ ] 이미지 에셋 배치 — profiles/standings/bg 실제 이미지 연결
- [ ] 노드 파일 내용 충실화 (현재 placeholder 또는 미작성)
- [ ] 씬 간 자동 연결 실행 (next 필드 구현됨, 연결 UI 미구현)
- [ ] 성향 태그(tags) 시스템 연동

---

## ⭐ 신규 작업: 웹 Ink.js C모드 프로토타입
> 결정일: 2026-03-29 | 우선순위: Unity 구현 전 웹 검증 먼저

### 배경
Unity ink C모드 구현을 진행 중 UI 레이아웃과 인터랙션 검증에 시간이 많이 소요됨.
**Ink.js로 웹 프로토타입을 먼저 만들어 UI/내러티브 동작을 검증한 후 Unity로 이식하는 방향으로 전환.**

### 목표
레퍼런스 HTML(`crownfalle_c_inline_drift.html`)의 UI + Ink.js 런타임 연결.
하드코딩된 NODES 배열을 실제 ink 스토리로 교체.

### 레퍼런스 파일
| 파일 | 경로 | 역할 |
|------|------|------|
| UI 레퍼런스 | `Dev/Projects/crown-falle-interlude/Docs/Sample/crownfalle_c_inline_drift.html` | C모드 완성 UI (하드코딩 NODES) |
| Ink 스크립트 | `Dev/Projects/crown-falle-interlude/Assets/Ink/test/test_c_mode.ink` | 실제 테스트 시나리오 |
| Ink 변수 | `Dev/Projects/crown-falle-interlude/Assets/Ink/includes/variables.ink` | 3축 파라미터 정의 |

### 작업 목록
- [ ] `test_c_mode.ink` → `story.json` 컴파일 (Inky 에디터 Export 또는 inklecate)
- [ ] Ink.js 라이브러리 로드 (CDN 또는 로컬)
- [ ] HTML에서 NODES 배열 제거 → `inkjs.Story` 런타임으로 교체
- [ ] 태그 파서 JS 구현 (InkTagParser.cs 로직 포팅)
- [ ] voice_pair 버퍼링 로직 JS 구현
- [ ] 선택지 인터랙션 연결 (`story.ChooseChoiceIndex()`)
- [ ] 분위기 레이어 연결 (`# mood:xxx` 태그)
- [ ] 로컬 서버 실행 확인 (fetch CORS 이슈)

### 주요 이슈 (사전 파악)

| 이슈 | 심각도 | 해결책 |
|------|--------|--------|
| Ink 컴파일 필요 | 🔴 필수 | Inky Export / inklecate CLI |
| INCLUDE 경로 | 🟡 주의 | 컴파일러 실행 위치에서 `../includes/variables.ink` 경로 확인 |
| voice_pair 버퍼링 | 🟡 구현 필요 | `voice_pair_start` / `voice_pair_end` 태그 사이 항목 버퍼링 (~50줄 JS) |
| 로컬 서버 필요 | 🟡 주의 | VSCode Live Server 또는 `python -m http.server` |
| 태그 파싱 | 🟢 단순 | C# InkTagParser 로직 그대로 JS 포팅 |

### Ink.js 태그 파싱 참고
현재 `InkTagParser.cs` 파싱 로직:
- `"speaker:seren"` → `{speaker: "seren"}`
- `"voice:faith"` → `{voice: "faith"}`
- `"axis:cautious"` → `{axis: "cautious"}`
- `"whisper:텍스트"` → `{whisper: "텍스트"}`
- `"consequence:텍스트"` → `{consequence: "텍스트"}`
- `"mood:presence"` → `{mood: "presence"}`
- `"neutral"` → `{neutral: true}`
- `"voice_pair_start"` / `"voice_pair_end"` → 버퍼링 시작/종료
- `"> aside: 텍스트"` → 텍스트 내 인라인 부연 (텍스트에서 분리)

### voice_pair 버퍼링 로직 (JS 구현 참고)
```
C# InkDialogueManager 방식:
1. voice_pair_start 태그 감지 → _inVoicePair = true, buffer 초기화
2. 버퍼 중 voice 태그 → buffer에 추가
3. voice_pair_end 태그 감지 → VoicePair 이벤트 발행 (buffer 사용)

JS 포팅 방향:
- story.Continue() 루프에서 동일 로직 적용
- 버퍼 채우는 동안 continue 계속 호출
- voice_pair_end 시점에 렌더링
```

---

## Unity ink C모드 구현 현황
> 경로: `C:\Users\iseul\OneDrive\Dev\Projects\crown-falle-interlude\`
> 계획서: `Dev/handoff/plans/design/2026-03-29-unity-ink-cmode-plan-v1.md`
> 구현 스펙: `Dev/Projects/crown-falle-interlude/Docs/Implementation/cmode-ui-spec.md`

### 완료 (2026-03-29 기준)
- [x] ink-unity-integration 패키지 설치
- [x] 폴더 구조 + C# 스크립트 12개
- [x] `variables.ink`, `test_c_mode.ink` 작성
- [x] 프리팹 8종 자동 생성 (CreateCrownFallePrefabs.cs)
- [x] CModeTest 씬 구성 (Canvas, RightPanel 36%, ScrollRect, MoodLayer 3종)
- [x] 한글 폰트 — 맑은고딕 SDF (Resources 폴더 로드 방식)
- [x] FontApplier.cs — 런타임 전체 TMP 폰트 강제 적용
- [x] RectMask2D — ScrollRect 콘텐츠 클리핑
- [x] Content top-anchored 전환 (텍스트 상단부터 쌓임)
- [x] ChoiceCardRow HorizontalFit 수정 (좌우 배치)
- [x] VoiceCell 라벨 정렬 (side 기반 좌/우)
- [x] ChoiceCardFactory drift 정렬 + gold border + 0.15 fade
- [x] ContinueHint UI (하단 pulse 텍스트)
- [x] FixSceneLayout.cs — 씬 자동 수정 에디터 스크립트

### 미완료 (웹 프로토 검증 후 재개)
- [ ] 선택지 좌우 텍스트 정렬 시각 확인 (패널 경계 및 overflow 추가 점검)
- [ ] ContinueHint Inspector 연결 (DialogueManager → Continue Hint 필드)
- [ ] 중립 선택지 위치 (카드 아래 단독 배치)
- [ ] 배경 일러스트 (아트 에셋 없음)
- [ ] DOTween 애니메이션 (엔트리 fade-in)
- [ ] A모드 / C↔A 전환 (후속)

### 한글 폰트 관련 메모
Unity GUID 기반 에셋 참조가 런타임에 동작하지 않는 문제 발생.
→ **해결책:** `Assets/Resources/MalgunGothic SDF.asset` 배치 + `FontApplier.cs`로 `Resources.Load<TMP_FontAsset>()` 런타임 로드.
→ GUID 참조 방식은 포기. Resources 방식 사용.

---

## 관련 툴 접속 주소

| 툴 | 주소 | 실행 명령 |
|---|---|---|
| dialogue-proto | `http://localhost:8080` | `python -m http.server 8080` (프로젝트 루트) |
| doc-viewer (프론트) | `http://localhost:5400` | `npm run dev` (`_tools/doc-viewer/`) |
| doc-viewer (API 서버) | `http://localhost:8000` | `python server.py` (`_tools/doc-viewer/`) |
