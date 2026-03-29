# CrownFalle — Dialogue Prototype 시스템 레퍼런스
> 갱신: 2026-03-29 (3차) | 스토리 내용 미포함

---

## 1. 시스템 아키텍처

### 전체 구조

```
crown-falle-dialogue-proto/
├── index.html                  ← 기존 JSON 기반 C/A 프로토
├── c-mode/                     ← 신규 ink.js C모드
│   ├── index.html
│   ├── css/c-mode.css
│   └── js/
│       ├── tag-parser.js       ← ink 태그 → 구조체 파싱
│       ├── ink-engine.js       ← inkjs Story 래핑 + 큐 기반 step()
│       ├── c-mode-renderer.js  ← DOM 렌더링 전체
│       └── main.js             ← 초기화, advance 루프, 디버그
├── ink/
│   ├── stories/                ← 씬별 .ink + .ink.json
│   └── includes/
│       ├── variables.ink       ← 3축 파라미터 + 관계 변수 + 플래그
│       └── functions.ink
└── data/
    ├── characters.json         ← UI/에셋용 캐릭터 정의
    ├── characters_traits.json  ← axis_weights (relation-engine용)
    ├── scenes.json
    └── nodes/*.json            ← 기존 JSON 노드 데이터
```

---

## 2. ink.js C모드 — 이벤트 시스템

### 이벤트 분류

| 분류 | 이벤트 종류 | 처리 방식 |
|---|---|---|
| **Side-effect** | mood, bg, location | advance 없이 즉시 처리 |
| **Content** | text, voice_pair, mode_switch | advance 1회 = 1항목 출력 |
| **Special** | choices, end | choices: 카드 클릭 대기 / end: 씬 종료 |

### 이벤트 흐름

```
story.Continue() → tag-parser.js 파싱
    ↓
Side-effect 태그 → 즉시 큐 삽입 (mood/bg/location)
mode_switch → Content 이벤트로 큐 삽입
next_scene → _lastNextScene 저장
scene_chars → _sceneChars 저장
voice_pair → 버퍼링 후 voice_pair 이벤트
    ↓
advance() → step() → 사이드이펙트 즉시 소비 → Content 1개 렌더링
```

---

## 3. ink 태그 전체 목록

| 태그 | 분류 | UI 결과 |
|---|---|---|
| (태그 없음) | Content | 나레이션 (이탤릭 흐린) |
| `# speaker:id` | Content | 이름(색상) + 대사 |
| `# voice:faith/cautious` | Content | 좌측 voice |
| `# voice:cynical/bold` | Content | 우측 voice |
| `# voice_pair_start/end` | 제어 | 2개 voice → 좌우 한 줄 |
| `# mood:presence/tension/cold` | Side-effect | 배경 레이어 활성화 |
| `# bg:id` | Side-effect | 배경 변경 (예정) |
| `# location:텍스트` | Side-effect | location badge 갱신 |
| `# scene_chars:a,b,c` | 메타 | _sceneChars 저장 |
| `# mode_switch:c_to_a/a_to_c` | Content | `— — —` 구분선 |
| `# next_scene:씬id` | 메타 | end 이벤트에 포함 |
| `# axis:faith/cautious` | choice | 좌카드 |
| `# axis:cynical/bold` | choice | 우카드 |
| `# whisper:텍스트` | choice | 카드 상단 속삭임 |
| `# consequence:텍스트` | choice | 카드 하단 결과 암시 |
| `# neutral` | choice | 중립 선택지 (가운데) |
| `> aside: 텍스트` | 텍스트 내 | 대사 아래 흐린 이탤릭 |

---

## 4. 씬 파일 구조 컨벤션

```ink
INCLUDE ../includes/variables.ink

-> scene_id

=== scene_id ===
# bg:배경id
# location:위치명
# scene_chars:char1,char2

[본문]

~ met_xxx = true
# next_scene:다음씬id
-> END
```

**선택지 패턴:**
```ink
+ [선택지] # axis:cautious # whisper:속삭임
    ~ axis_method_bold -= 1
    [분기 응답]
+ [선택지] # axis:bold # whisper:속삭임 # consequence:결과
    ~ axis_method_bold += 1
    [분기 응답]
+ [(중립)] # neutral
    [분기 응답]
-
[공통 진행]
```

---

## 5. 3축 파라미터 + 관계 시스템

### ink 변수

```ink
VAR axis_method_bold = 0    // 과감(+) ↔ 신중(-)
VAR axis_stance_faith = 0   // 신념(+) ↔ 냉소(-)
VAR axis_cost_release = 0   // 해방(+) ↔ 억제(-)

VAR rel_seren = 30
VAR rel_wren = 20
VAR rel_armand = 50
VAR rel_colt = 10
VAR rel_flay = -10
```

### characters_traits.json 구조 (relation-engine용, 2번 작업)

```json
{
  "seren": {
    "display_name": "Seren",
    "base_relation": 30,
    "axis_weights": {
      "method":  { "bold": -1, "cautious": 2 },
      "stance":  { "cynical": 1,  "faith": 2 },
      "cost":    { "restrain": 1, "release": -2 }
    }
  }
}
```

**방식 C (확정):**
1. 플레이어 선택 → JS가 axis_weights 조회
2. `rel_xxx += weight × delta` 계산
3. `story.variablesState["rel_xxx"]` 직접 쓰기
4. ink 내부에서 rel 값으로 대사 분기

---

## 6. 캐릭터 색상

| 캐릭터 | 색상 |
|---|---|
| Tristram | `#6490B4` |
| Seren | `#7EB8A0` |
| Armand | `#C4956A` |
| Wren | `#B8A060` |
| Employer | `#8A8A7A` |
| Colt | `#8A7A5A` |
| Flay | `#6A4A6A` |
| The Scald | `#9A5A3A` |

| voice | 색상 | 배치 |
|---|---|---|
| faith (신념) | `#6A9AB8` | 좌 |
| cautious (신중) | `#7A9A6A` | 좌 |
| cynical (냉소) | `#C09060` | 우 |
| bold (과감) | `#B85A48` | 우 |

---

## 7. 기존 JSON 프로토 — 스키마 (유지)

### characters.json
```json
{
  "id": {
    "display_name": "...", "color": "#...", "role": "protagonist|npc",
    "profiles": { "default": "path", ... },
    "standings": { "default": "path", ... }
  }
}
```

### scenes.json
```json
{
  "scene_id": {
    "chapter": 1, "title": "...", "location": "...", "background": "bg_id",
    "characters": ["id"], "initial_mode": "c", "first_node": "n001",
    "next": "next_scene_id",
    "standings": { "a_mode": { "char_id": { "slot": "left", "action": "default" } } }
  }
}
```

### nodes/{scene_id}.json 타입
`narration` / `dialogue` / `choice` / `mode_switch` / `bg_change`

---

## 8. 에셋 경로 컨벤션

```
assets/
├── profiles/    {char}/default.png  — 대화창 아이콘 (192×192)
├── standings/   {char}/{action}.png — A모드 전신 (투명bg PNG)
├── bg/          {bg_id}.png         — 배경 (1920×1080)
└── illustrations/ {char}/{name}.png
```

---

## 9. Unity ink C모드 아키텍처 (참고)

> 웹 프로토 검증 완료 후 이식 예정. 현재 대기 중.

```
InkDialogueManager
  Story.Continue() → InkTagParser → DialogueEvent 발행
       ↓
CModePanelController
  ├─ TextEntryFactory  → NarrationEntry / DialogueEntry
  ├─ VoicePairFactory  → VoiceSingle / VoicePairRow
  ├─ ChoiceCardFactory → ChoiceCardRow / NeutralChoice
  └─ MoodLayerController → Image 레이어 페이드
```
