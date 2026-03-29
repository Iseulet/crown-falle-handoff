# CrownFalle — Dialogue Prototype 시스템 레퍼런스
> 갱신: 2026-03-29 (2차) | 스토리 내용 미포함

---

## 시스템 아키텍처

### C↔A 하이브리드 모드

| 모드 | 레이아웃 | 히스토리 | 용도 |
|------|---------|---------|------|
| C모드 | 좌측 배경 + 우측 텍스트 패널 | 누적 (스크롤) | 기본. 나레이션+대사+선택지 |
| A모드 | 전체화면 배경 + 스탠딩 + 하단 대화박스 | 교체 | 핵심 씬 전용 |

**전환:** `mode_switch` 노드로 제어. 전환 시 히스토리 클리어.

### JS 모듈 구조

```
engine.js       — 노드 순회, 모드 관리, 배경 관리, 씬 체인, 선택 기록
renderer-c.js   — C모드 렌더러 (히스토리 누적, placeholder)
renderer-a.js   — A모드 렌더러 (스탠딩 배치, 텍스트 교체)
main.js         — 초기화, JSON fetch, 이벤트 바인딩
```

---

## 데이터 스키마

### characters.json — 캐릭터 정의

```json
{
  "tristram": {
    "display_name": "Tristram",
    "color": "#6490B4",
    "role": "protagonist",
    "portrait_position": "center 15%",
    "profiles": {
      "default":  "assets/profiles/tristram/default.png",
      "smile":    "...",
      "angry":    "...",
      "thinking": "...",
      "sad":      "..."
    },
    "standings": {
      "default":     "assets/standings/tristram/default.png",
      "speaking":    "...",
      "threatening": "...",
      "back_turned": "..."
    },
    "illustrations": {
      "campfire": "assets/illustrations/tristram/campfire.png"
    }
  }
}
```

### actions.json — 액션 정의

```json
{
  "profiles":  { "default": {}, "smile": {}, "angry": {}, "thinking": {}, "sad": {} },
  "standings": { "default": {}, "speaking": {}, "threatening": {}, "back_turned": {} }
}
```

`profiles` = 대사창 표정 아이콘 (tight bust, 192×192)
`standings` = A모드 전신 컴포지팅 레이어 (grey bg #888888)

### scenes.json — 씬 메타

```json
{
  "scene_id": {
    "chapter": 1,
    "title": "씬 제목",
    "location": "장소명",
    "background": "backgrounds.json 키",
    "characters": ["character_id"],
    "initial_mode": "c",
    "first_node": "n001",
    "next": "다음_씬_id",
    "standings": {
      "a_mode": {
        "character_id": { "slot": "left", "action": "default" }
      }
    }
  }
}
```

### nodes/{scene_id}.json — 노드 파일

5가지 노드 타입:

| 타입 | 필드 | 설명 |
|------|------|------|
| `narration` | text, next | 나레이션 텍스트 |
| `dialogue` | speaker, text, aside, next | 캐릭터 대사 |
| `choice` | options[] | 선택지 분기 |
| `mode_switch` | from, to, transition, next | C↔A 전환 |
| `bg_change` | background, transition, next | 씬 내 배경 교체 |

```json
{
  "n001": { "type": "narration", "text": "...", "next": "n002" },
  "n002": { "type": "dialogue", "speaker": "seren", "text": "...", "next": "n003" },
  "n003": {
    "type": "choice",
    "options": [
      { "text": "선택지A", "next": "n004a", "tags": ["curious"] },
      { "text": "선택지B", "next": "n004b", "tags": ["cold"] }
    ]
  }
}
```

### backgrounds.json — 배경 + 슬롯

```json
{
  "bg_id": {
    "image": "assets/bg/filename.png",
    "label": "UI 표시용 장소명",
    "standing_slots": [
      { "id": "left",   "x_percent": 25, "y_percent": 40 },
      { "id": "right",  "x_percent": 75, "y_percent": 40 },
      { "id": "center", "x_percent": 50, "y_percent": 40 }
    ]
  }
}
```

---

## 에셋 경로 컨벤션

```
assets/
├── profiles/    {char}/default.png  — 대화창 표정 아이콘 (192×192)
├── standings/   {char}/{action}.png — A모드 전신 (투명bg PNG)
├── bg/          {bg_id}.png         — 배경 이미지 (1920×1080)
└── illustrations/ {char}/{name}.png — 삽화 (16:9 또는 자유)
```

**에셋 소스:** `Dev/contents/crown-falle-interlude/characters/main/{char}/art/_favorites/`

standing composite 이미지 (grey bg ~#7f7f7f) → renderer-a.js `_applyColorKey()` 로 배경 제거 (canvas, 코너 자동감지)
또는 `rembg`로 전처리 후 투명 PNG 배치도 지원.

---

## 씬 체인 구조 (챕터 개요)

```
Ch1 (Ironmead, 4씬) → Ch2 (南路, 6씬) → Ch3 (야영지, 5씬) → null
```

각 씬의 `next` 필드로 순차 연결. 마지막 씬은 `next: null`.

---

## 입력 바인딩

| 입력 | 동작 |
|------|------|
| 클릭 / 스페이스 / Enter | 다음 노드 진행 |
| 1 / 2 / 3 숫자키 | 선택지 단축키 |

---

## Unity ink C모드 아키텍처

> 웹 프로토타입에서 검증한 C모드를 Unity 6.3 LTS + ink로 구현.
> 범위: C모드 단독. A모드 및 C↔A 전환은 후속 작업.

### C# 클래스 구조

```
InkDialogueManager
  Story.Continue() → 태그 파싱(InkTagParser) → DialogueEvent 발행
       ↓
CModePanelController
  ├─ Narration/Dialogue → TextEntryFactory  → NarrationEntry/DialogueEntry 프리팹
  ├─ VoiceSingle        → VoicePairFactory  → VoiceSingle 프리팹
  ├─ VoicePair          → VoicePairFactory  → VoicePairRow 프리팹
  ├─ Choice             → ChoiceCardFactory → ChoiceCardRow/NeutralChoice 프리팹
  └─ MoodChange         → MoodLayerController → Image 레이어 페이드

선택 클릭 → ink ChooseChoiceIndex() → Continue()
```

### ink 태그 컨벤션

| 태그 | 효과 |
|------|------|
| `# speaker:seren` | 대사 렌더링, 캐릭터 이름 색상 |
| `# voice:faith` | 단독 내면 목소리 (좌측) |
| `# voice:cynical` | 단독 내면 목소리 (우측) |
| `# voice_pair_start` / `# voice_pair_end` | 대립 목소리 쌍 — 한 줄 배치 |
| `# axis:cautious` | 선택지 카드 축 (신중=Left) |
| `# axis:bold` | 선택지 카드 축 (과감=Right) |
| `# axis:faith` | 선택지 카드 축 (신념=Left) |
| `# axis:cynical` | 선택지 카드 축 (냉소=Right) |
| `# whisper:텍스트` | 카드 상단 속삭임 |
| `# consequence:텍스트` | 카드 하단 결과 암시 (빨간 이탤릭) |
| `# neutral` | 중립 선택지 (카드 아래 가운데) |
| `# mood:presence/tension/cold` | 분위기 레이어 전환 |
| `# bg:bg_id` | 배경 교체 (후속 구현) |
| `> aside: 텍스트` | 텍스트 내 부연 (대사/나레이션 내 인라인) |

### 3축 파라미터 (ink 변수)

| 변수 | 방향 | 선택 유형 |
|------|------|-----------|
| `axis_stance_faith` | 신념(+) ↔ 냉소(-) | 태도 선택 |
| `axis_method_bold` | 과감(+) ↔ 신중(-) | 방법 선택 |
| `axis_cost_release` | 해방(+) ↔ 억제(-) | 대가 선택 |

### 색상 기준

| 용도 | 색상 |
|------|------|
| 배경 | `#060504` |
| 오버레이 패널 | `rgba(6,5,4,0.84)` |
| 본문 텍스트 | `#D4CBBE` |
| 나레이션 | `#A09484` |
| 흐림 | `#6A6054` |
| 결과 암시 | `#8A5040` |
| Seren | `#7EB8A0` |
| Tristram | `#6490B4` |
| 신념/Left | `#6A9AB8` |
| 냉소/Right | `#C09060` |
| 신중/Left | `#7A9A6A` |
| 과감/Right | `#B85A48` |

### ScriptableObject 타입

- `CharacterData` — `characterId`, `displayName`, `nameColor`
- `AxisData` — `axisId`, `displayLabel`, `color`, `side(Left/Right)`

### 프리팹 목록

| 프리팹 | 필수 자식 오브젝트 이름 |
|--------|------------------------|
| NarrationEntry | TMP (이탤릭) |
| DialogueEntry | `SpeakerName`, `DialogueText`, `AsideText` TMP |
| VoicePairRow | 좌VoiceCell + 우VoiceCell (각 `labelText`, `bodyText`, `borderImage`) |
| VoiceSingle | 단일 VoiceCell |
| ChoiceCardRow | HorizontalLayoutGroup 컨테이너 |
| ChoiceCard | `AxisLabel`, `WhisperText`, `ActionText`, `ConsequenceText` TMP + `Border` Image |
| NeutralChoice | Button + TMP |
