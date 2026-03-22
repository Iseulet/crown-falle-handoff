# CrownFalle Interlude — 통합 워크플로우 프로토콜
> 버전: 4.1 | 갱신: 2026-03-22
> 대상: 스토리 + 아트 통합 운영

---

## 1. 워크플로우 개요

스토리와 아트는 **엔티티 중심**으로 통합 관리한다.
같은 캐릭터의 서사 설정, 외형 시트, 프롬프트, 이미지가 한 폴더에 있다.

### 4단계 워크플로우

```
1 발상 → 2 정리 → 3 집필 → 4 교정
                |
                v 프로필 확정 시
           아트 워크플로우
           AD → P → C
```

| 단계 | 목적 | 산출물 |
|------|------|--------|
| 1 발상 | 아이디어 탐색, 설정 논의 | 브레인덤프 갱신 |
| 2 정리 | 확정 항목 독립 문서 분리 | profile.md, setting.md 등 |
| 3 집필 | Beat 설계 → 산문 초안 | 챕터 원고 |
| 4 교정 | 독자 반응, 리듬, 문장 교정 | 교정 원고 |

단계는 순차적이지만 유연하다. 1↔2 수시 왕복, 3 중에도 1로 복귀 가능.

### 1→2 전환 트리거

다음 중 하나를 충족하면 정리 세션 진행:
- 브레인덤프에서 한 엔티티(캐릭터/세력/장소)의 확정 항목이 **3개 이상** 누적
- 브레인덤프 총 분량이 **500줄 초과** 시 강제 정리 세션 권고
- 원주가 아트 작업을 시작하려 할 때 (프로필 분리 필요)

---

## 2. 단계별 상세

### 1 발상 (Ideation)

```
원주 아이디어 / 질문
    ↓
Claude와 대화 (확장, 반론, 레퍼런스)
    ↓
결정 → 브레인덤프에 기록
보류 → 브레인덤프 미결 사항에 기록
```

**투입 에이전트:** 포지 + 왈도 + 히스(영감 모드)

- 포지: 세계관·캐릭터·서사 설계의 중심. 엔티티 구조 유지
- 왈도: 아이디어 발산, 복선·반전·대안 루트 제안
- 히스(영감): 역사적 유사 사례, 신화 모티프 제공

**산출물:** 브레인덤프 갱신 (변경 이력 기록)

---

### 2 정리 (Structuring)

```
브레인덤프에서 확정된 덩어리
    ↓
독립 문서로 분리
    ↓
일관성 체크
    ↓
브레인덤프 경량화 (참조 링크 교체)
```

**투입 에이전트:** 포지 + 키퍼 + 히스(고증 모드) + 캐리

- 포지: 독립 문서 분리 실행. 엔티티 구조에 맞게 배치
- 키퍼(정리 모드): 분리된 문서와 기존 문서 간 교차 검증. "profile.md가 다른 캐릭터 설정과 모순 없는가?"
- 히스(고증): 새 설정의 시대적 정합성 검증
- 캐리: 캐릭터 감정 아크가 서사 흐름과 맞는지 확인. **profile.md에 "감정 아크" 섹션을 작성** — @ArtDirector가 표정·인상 방향을 잡는 근거가 됨

**분리 기준:**
- 캐릭터 설정이 3항목 이상 확정 → `characters/main/{id}/profile.md`
- 장소 설정이 위치+성격+서사 기능 확정 → `world/locations/{id}/setting.md`
- 세력 설정이 정체+구조+서사 접점 확정 → `world/factions/{id}.md`

**산출물:** 독립 문서 + 브레인덤프 경량화

**브레인덤프 경량화 형식:**

분리 완료된 섹션은 요약 + 참조 링크로 교체한다:
```markdown
## Tristram — 뿌리 뽑힌 남자
> 분리 완료 → characters/main/tristram/profile.md
> 아래는 요약. 상세는 profile.md 참조.

이름: Tristram / 20대 중반 / 평민 / Armand의 에코르셔 연락책
```

**부분 확정 처리:**

profile.md 내 항목별로 확정/미확정을 마커로 구분한다:
```markdown
## 기본 정보
- 이름: Tristram [확정]
- 나이: 20대 중반 [확정]
- 역할: Armand의 에코르셔 연락책 [확정]

## 서사 아크
- 초반~중반: 도구→암살→와해 [확정]
- 최종 운명: [미확정]
```
[미확정] 항목이 있어도 분리는 가능하다. 아트 진입에 필요한 최소 항목(이름·나이·성별·성격·역할)만 [확정]이면 된다.

**2 → 아트 연결:** 프로필 확정 시 아트 워크플로우 트리거 (섹션 4 참조)

---

### 3 집필 (Writing)

```
확정된 설정 + 서사 골격
    ↓
Beat 설계
    ↓
초안 집필
    ↓
검증 (논리, 설정, 시점, 감정)
```

**투입 에이전트:** 아크 → 제니 → 레드 + 키퍼 + 퍼스 + 캐리

- 아크: Beat Sheet 설계. 챕터 구조·씬 목적·갈등 설정
- 제니: 확정된 Beat를 산문으로 변환
- 레드: 논리 결함·서사 약점 지적
- 키퍼(집필 모드): 원고와 설정 간 일관성 검증. "이 씬의 행동이 profile.md 성격과 맞는가?"
- 퍼스: 다른 캐릭터 시점 재해석
- 캐리: 캐릭터 말투 일관성, 감정선 검토

**산출물:** Beat Sheet + 초안 원고 + 검증 보고서

> 3의 세부 프로세스는 실제 집필 시작 시 경험하며 정립한다.

---

### 4 교정 (Polish)

```
검증 통과한 원고
    ↓
독자 경험 시뮬레이션
    ↓
리듬·페이싱 분석
    ↓
문장 수준 최종 교정
```

**투입 에이전트:** 리더 → 페이서(매크로+마이크로) → 쿳시

- 리더: 독자 몰입도·이탈 포인트 감지
- 페이서: 거시적 긴장 곡선 + 미시적 씬 내 리듬 분석
- 쿳시: 문장 수준 최종 교정

**산출물:** 교정 원고

---

## 3. 스토리 에이전트 총괄

### 에이전트 목록 (기존 페르소나 유지)

| # | 에이전트 | 투입 단계 | 역할 |
|---|---------|----------|------|
| 1 | 포지 (World Forge) | 1 2 | 세계관·캐릭터 설계. SSOT. 문서 분리. 서사 골격 큰 흐름 |
| 2 | 왈도 (Dreamer) | 1 | 아이디어 발산. 복선·반전·씨앗 |
| 3 | 히스 (Historian) | 1 2 | 영감 모드(1) + 고증 모드(2) |
| 4 | 캐리 (Carry) | 2 3 | 감정 아크 + 말투 일관성. 2에서 profile.md 감정 섹션 작성 |
| 5 | 키퍼 (Keeper) | 2 3 | 정리 모드(2): 문서 간 교차 검증. 집필 모드(3): 원고↔설정 검증 |
| 6 | 아크 (Architect) | 3 | 서사 골격 → 챕터별 Beat 분해 |
| 7 | 제니 (Scribe) | 3 | 산문 집필 |
| 8 | 레드 (Challenger) | 3 | 논리 비판 |
| 9 | 퍼스 (Expander) | 3 | 시점 전환 재해석 |
| 10 | 리더 (Reader) | 4 | 독자 경험 시뮬레이션 |
| 11 | 페이서 (Pacemaker) | 4 | 매크로+마이크로 통합 페이싱 |
| 12 | 쿳시 (Editor) | 4 | 문장 수준 최종 교정 |

**변경사항 (v3 → v4):**
- 시스(Systems) → 프로토콜 규칙으로 흡수. 에이전트 아님
- 페이서-매크로 + 페이서-마이크로 → 페이서 1개로 통합
- 5 Phase → 4단계로 단순화
- 게이트 체크 → 에이전트가 아닌 프로토콜 규칙
- 총 13+1 → 12개

### 현재 단계에서의 운용

브레인덤프 단계에서는 1 2 에이전트만 활성:

| 활성 | 에이전트 |
|------|---------|
| **상시** | 포지, 키퍼 |
| **요청 시** | 왈도, 히스, 캐리 |
| **휴면** | 아크, 제니, 레드, 퍼스, 리더, 페이서, 쿳시 |

---

## 4. 스토리 → 아트 연결

### 순방향: 프로필 확정 → 아트 작업 시작

```
2 정리 완료
   포지: profile.md 확정
   캐리: 감정 아크·성격 톤 확인
   키퍼: 설정 모순 없음 확인
        |
        v 프로필 확정 — 아트 작업 가능
        |
   @ArtDirector: profile.md 읽고 맥락 분석
        | 대화형 외형 확정 (큰 방향 → 세부 → 시트)
        v
   외형 시트 확정 → characters/{id}/art/sheet.json
        |
        v
   @Prompter: prompts.json 작성
        | 이미지 생성
        v
   @Curator: 시트 대조 → 보강/채택
        | 채택 → characters/{id}/art/_favorites/
        v
   완료
```

**트리거 조건:** `characters/main/{id}/profile.md`가 존재하고, 최소한 이름·나이·성별·성격·역할이 확정된 상태.

### 역방향: 아트 → 프로필 피드백

```
   @Curator: 이미지 평가 중 "이 인상이 더 맞다"
        |
        v
   @ArtDirector → 원주: "시트를 이쪽으로 갱신할까?"
        | 원주 승인
        v
   profile.md 외형 관련 항목만 갱신 (additive)
```

**규칙:**
- 서사 설정(성격, 동기, 관계)은 아트가 변경하지 않는다
- 외형 묘사 항목(체형, 머리, 복장 인상 등)만 갱신 가능
- 갱신 시 원주 승인 필수

### 장소 아트

캐릭터와 동일 구조:

```
world/locations/ironmead/setting.md 확정
        ↓
@ArtDirector: 배경 비주얼 방향
        ↓
@Prompter: world/locations/ironmead/art/ 에 프롬프트·결과물
```

### 장면 CG

장면은 여러 엔티티를 참조하므로 별도 경로:

```
참조 입력:
  characters/main/tristram/art/sheet.json --+
  characters/main/seren/art/sheet.json -----+
  world/locations/highclere/setting.md -----+
  narrative/structure.md (해당 Beat) --------+
        |
        v
@ArtDirector: 장면 구도·분위기 방향
        ↓
@Prompter: narrative/scenes/{scene-id}/scene.json
        ↓
@Curator: 평가 → narrative/scenes/{scene-id}/_favorites/
```

---

## 5. 아트 에이전트 (현행 유지)

| 에이전트 | 역할 |
|---------|------|
| **@ArtDirector** | 비주얼 방향, 대화형 외형 확정, 컨벤션 그룹 관리 |
| **@Prompter** | 시트 → JSON → 프롬프트 변환, 컨벤션 기반 스타일 적용 |
| **@Curator** | 생성 결과 ↔ 시트 대조 평가, 보강/채택 관리 |

상세 정의는 `art-agents.md` 참조.

---

## 6. 폴더 구조

```
crown-falle-interlude/
|
+-- _protocol/                        <- 워크플로우·에이전트 정의
|   +-- PROTOCOL.md                   <- 이 문서
|   +-- story-agents/                 <- 스토리 에이전트 페르소나 (12개)
|   |   +-- forge.md
|   |   +-- dreamer.md
|   |   +-- historian.md
|   |   +-- carry.md
|   |   +-- keeper.md
|   |   +-- architect.md
|   |   +-- scribe.md
|   |   +-- challenger.md
|   |   +-- expander.md
|   |   +-- reader.md
|   |   +-- pacemaker.md              <- 매크로+마이크로 통합
|   |   +-- editor.md
|   +-- art-agents.md                 <- @ArtDirector, @Prompter, @Curator
|   +-- art-config.json               <- 컨벤션 그룹, 스타일 프리픽스
|   +-- art-character-template.json
|   +-- art-scene-template.json
|
+-- _braindump/                       <- 브레인덤프 (미확정, 성장 중)
|   +-- crownfalle-interlude_braindump.md
|   +-- STATUS.md
|   +-- desktop-context/              <- Desktop 프로젝트 지식 갱신 출력
|       +-- canon-summary.md
|       +-- crownfalle-interlude_braindump.md (복사본)
|
+-- _tools/                           <- 도구 (이미지 생성 스크립트 등)
|   +-- art-gen/
|       +-- generate.py
|       +-- utils.py
|       +-- .env
|       +-- requirements.txt
|       +-- .venv/
|
+-- world/                            <- 세계 설정 (확정 시 분리)
|   +-- overview.md                   <- 시대, 마법 체계, 핵심 갈등
|   +-- factions/                     <- 세력별
|   |   +-- ecorcheurs.md
|   |   +-- church.md
|   |   +-- barbarian-tribes.md
|   +-- locations/                    <- 장소별 (설정+아트 통합)
|   |   +-- ironmead/
|   |   |   +-- setting.md
|   |   |   +-- art/
|   |   +-- cendrefort/
|   |   +-- highclere/
|   |   +-- royal-capital/
|   +-- rules/                        <- 마법·제도·지형·전쟁
|       +-- magic.md                  <- 로고스 + 그림메-에코 + 영혼 이주 통합
|       +-- succession.md
|       +-- geography.md
|       +-- scepter-war.md
|
+-- characters/                       <- 캐릭터 (서사+아트 통합)
|   +-- main/                         <- 주요 캐릭터 (풀 프로필 + 아트)
|   |   +-- tristram/
|   |   |   +-- profile.md
|   |   |   +-- art/
|   |   |       +-- sheet.json
|   |   |       +-- prompts.json
|   |   |       +-- _favorites/
|   |   +-- seren/
|   |   +-- colt/
|   |   +-- wren/
|   |   +-- flay/
|   |   +-- the-scald/
|   |
|   +-- minor/                        <- 엑스트라 (간략 설정, 아트 선택적)
|       +-- armand.md
|       +-- gaspard.md
|       +-- edmund-castleigh.md
|       +-- malpas.md
|
+-- narrative/                        <- 서사 구조 (3 시작 후 활성)
|   +-- structure.md
|   +-- chapters/
|   +-- scenes/                       <- 장면 CG (여러 엔티티 교차)
|       +-- gaspard-assassination/
|       |   +-- scene.json
|       |   +-- _favorites/
|       +-- campfire-isolation/
|
+-- maps/                             <- 지도 (세계 지도, 지역 지도)
|   +-- prompts/
|   +-- _favorites/
|
+-- meta/                             <- 3작품 관통 구조 등 메타 설정
|   +-- trilogy-structure.md
|
+-- _archive/                         <- 폐기·백업
    +-- agents-v3/
    +-- agents-v2/
```

### 폴더 구조 핵심 원칙

1. **엔티티 = 폴더.** Tristram의 모든 것이 `characters/main/tristram/`에
2. **main vs minor.** 서사 비중 + 아트 필요 여부로 분리. 격상 시 이동
3. **art/는 엔티티 하위.** 캐릭터든 장소든 해당 엔티티 폴더 안에 `art/`
4. **장면만 별도.** 여러 엔티티가 교차하므로 `narrative/scenes/`에
5. **브레인덤프는 독립.** 미확정 혼합 문서이므로 `_braindump/`에 유지
6. **확정 → 분리.** 브레인덤프에서 확정되면 해당 폴더로 이동, 브레인덤프는 참조 링크로 교체

---

## 7. 프로토콜 규칙 (시스 대체)

기존 시스(Systems) 에이전트의 게이트키퍼 역할을 규칙으로 대체한다.

### 엔티티 상태 추적

`_braindump/STATUS.md`에 간략한 상태표를 유지한다:

```markdown
# 엔티티 상태표

## 캐릭터
| 엔티티 | 상태 | 경로 | 아트 |
|--------|------|------|------|
| tristram | 분리 완료 | characters/main/tristram/ | 미착수 |
| seren | 브레인덤프 | _braindump/ Seren | - |
| armand | 브레인덤프 | _braindump/ Armand | - |

## 장소
| 엔티티 | 상태 | 경로 | 아트 |
|--------|------|------|------|
| ironmead | 브레인덤프 | _braindump/ Ironmead | - |
```

### 지도 워크플로우

지도는 아트 3에이전트 파이프라인 밖에서 운영한다. 원주가 Gemini 이미지 생성으로 직접 프롬프트를 관리한다.

- `maps/prompts/`: 프롬프트 버전 보관 (gemini-map-default-v10.md 등)
- `maps/_favorites/`: 채택 지도
- 세계관 정합성(지형↔설정 일치)은 포지/키퍼가 확인

### 문서 분리 체크리스트

브레인덤프에서 독립 문서로 분리할 때:

- [ ] 해당 항목이 브레인덤프에서 확정 표시인가?
- [ ] 분리된 문서가 다른 확정 문서와 모순 없는가? (키퍼 확인)
- [ ] 브레인덤프의 해당 섹션이 참조 링크로 교체되었는가?
- [ ] 캐릭터인 경우 main/minor 분류가 맞는가?

### 집필 진입 체크리스트

3 집필을 시작하기 전:

- [ ] 해당 챕터에 등장하는 캐릭터의 profile.md가 존재하는가?
- [ ] 해당 챕터의 장소 setting.md가 존재하는가?
- [ ] 서사 골격(narrative/structure.md)에 해당 챕터 위치가 명시되어 있는가?
- [ ] 브레인덤프에 미결 사항 중 해당 챕터를 블로킹하는 항목이 없는가?

### 아트 진입 체크리스트

아트 워크플로우를 시작하기 전:

- [ ] profile.md 또는 setting.md가 확정 상태인가?
- [ ] 최소 필수 항목이 기재되어 있는가?
  - 캐릭터: 이름, 나이, 성별, 성격, 역할
  - 장소: 위치, 성격, 서사 기능

---

## 8. 도구별 역할

| 도구 | 역할 | 대상 |
|------|------|------|
| **Claude Desktop** | 발상, 정리, 에이전트 호출, 아트 디렉션 | 1 2 3 4 + 아트 |
| **Claude Code CLI** | 파일 생성·이동, 폴더 구조, 핸드오프 동기화 | 폴더 작업 + sync |

---

## 9. 핸드오프 리포 동기화

### 동기화 대상 (public)

로컬(`crown-falle-interlude/`)에서 핸드오프 리포(`crown-falle-handoff/contents/crown-falle-interlude/`)로 동기화:

| 로컬 경로 | 핸드오프 경로 |
|----------|-------------|
| `_protocol/PROTOCOL.md` | `contents/crown-falle-interlude/PROTOCOL.md` |
| `_protocol/story-agents/*.md` | `contents/crown-falle-interlude/story-agents/` |
| `_protocol/art-agents.md` | `contents/crown-falle-interlude/art-agents.md` |
| `_protocol/art-config.json` | `contents/crown-falle-interlude/art-config.json` |

### 동기화 제외 (private)

| 콘텐츠 | 이유 |
|--------|------|
| _braindump/ | 스토리 내용 |
| characters/ (profile.md, art/) | 캐릭터 설정 + 이미지 |
| world/ | 세계관 설정 |
| narrative/ | 서사 구조 + 원고 |
| maps/ | 지도 |

---

## 10. Desktop Context Refresh

### 개요

Desktop 프로젝트 지식에 올리는 파일 3종을 최신 상태로 유지하는 규칙.
갱신은 CLI의 `/desktop-refresh` 명령어로 실행한다.

### Desktop 프로젝트 지식 구성 (3파일)

| # | 파일 | 역할 | 갱신 빈도 |
|---|------|------|----------|
| 1 | `desktop-project-instructions.md` | 세션 프로토콜, 역할 분담, 프로젝트 경로 | 드물게 (프로토콜 변경 시) |
| 2 | `canon-summary.md` | 전체 캐논 요약 (캐릭터/장소/세력/규칙/서사 축약) | 캐논 변경 시 |
| 3 | `crownfalle-interlude_braindump.md` | 경량화 브레인덤프 (서사 흐름 + 장면 상세 + 미결 사항) | 브레인덤프 변경 시 |

### 갱신 트리거

| 트리거 | 갱신 대상 |
|--------|----------|
| 브레인덤프에 새 확정 사항 추가 (1 발상 세션 후) | braindump |
| 캐논 분리 발생 (2 정리 세션 후) | canon-summary + braindump |
| 캐논 파일의 [미확정] → [확정] 전환 | canon-summary |
| 서사 흐름에 구조적 변경 | canon-summary + braindump |
| 새 엔티티 캐논 추가 | canon-summary |
| 프로토콜/역할/경로 변경 | desktop-project-instructions |

### 갱신 규칙

1. **canon-summary.md는 캐논 파일에서 추출.** 브레인덤프가 아니라 분리된 캐논 파일이 소스.
2. **braindump는 원본 복사.** `_braindump/crownfalle-interlude_braindump.md`를 그대로 출력.
3. **canon-summary 분량 제한: 200줄 이내.** Desktop 컨텍스트 효율을 위해.
4. **갱신 시 이전 버전을 덮어쓴다.** 버전 관리는 git이 담당.
5. **갱신 후 원주에게 알림.** "Desktop 프로젝트 지식 갱신 완료. 파일을 교체해주세요."

### 출력 경로

```
_braindump/desktop-context/
├── canon-summary.md
├── crownfalle-interlude_braindump.md   (원본 복사)
└── desktop-project-instructions.md     (변경 시에만)
```

원주가 이 폴더에서 파일을 가져다 Desktop 프로젝트 지식에 올린다.

### 명령어 형식

```
/desktop-refresh              # 전체 갱신 (canon-summary + braindump 복사)
/desktop-refresh --summary    # canon-summary만 갱신
/desktop-refresh --all        # desktop-project-instructions 포함 전체
```

---

## 11. 후속 과제 (해당 단계 진입 시 정립)

| # | 항목 | 트리거 |
|---|------|--------|
| F-1 | 3→4 진입 조건 상세 정의 | 3 집필 시작 시 |
| F-2 | 게임 프로젝트(TBSF) ↔ 세계관 정합성 연결 지점 명시 | TBSF Phase 0 완료 후 |
| F-3 | 3 집필 세부 프로세스 (Beat 설계 상세, 검증 루프) | 3 집필 시작 시 |
| F-4 | generate.py 경로 로직 수정 (엔티티 분산 구조 대응) | 아트 생성 재개 시 |

---

## 변경 이력

| 버전 | 날짜 | 변경 |
|------|------|------|
| v3.0 | 2026-03-20 | 스토리 에이전트 13+1개, 5 Phase 파이프라인 |
| v4.0 | 2026-03-22 | 4단계 워크플로우 재편. 시스 → 프로토콜 규칙. 페이서 통합. 폴더 구조 엔티티 중심 재편. 스토리↔아트 연결 흐름 명시. 캐릭터 main/minor 분리 |
| v4.1 | 2026-03-22 | 1→2 전환 트리거 추가. 브레인덤프 경량화 형식 정의. 부분 확정 마커. 포지/아크 역할 경계. 키퍼 이중 모드. 캐리 2 산출물(감정 아크 섹션). 지도 워크플로우 명시. 핸드오프 매핑 갱신. _tools/ 추가. generate.py 경로 수정을 후속 과제(F-4)로 등록 |
| v4.2 | 2026-03-22 | 캐논 분리 17건 반영. world/rules/ 파일명 변경(logos+grimme-echo+soul-transfer→magic.md 통합, succession/geography/scepter-war 신규). meta/ 폴더 추가. STORY_INDEX.json 복원(STATUS.md=사람용 상태표, STORY_INDEX.json=기계적 경로 추적으로 역할 분리). Armand main 유지 확정 |
| v4.3 | 2026-03-22 | Desktop Context Refresh 섹션 추가 (섹션 10). /desktop-refresh 명령어 정의. desktop-context/ 폴더 추가. 후속 과제 → 섹션 11로 이동 |
| v4.4 | 2026-03-23 | 프롬프트 파일 통합: portrait.json+fullbody.json → prompts.json 단일 파일. 프롬프트 type은 파일 내 type 필드로 구분. generate_v2.py, build.py 경로 갱신. art-agents.md 산출물/워크플로우 갱신 |
