# CrownFalle Interlude — 세계관 프로젝트 워크플로우 & 프로토콜

> 날짜: 2026-03-19
> 프로젝트명: crown-falle-Interlude
> 위치: C:\Users\iseul\OneDrive\Dev\Story\crown-falle-Interlude\
> 언어: 한국어
> 도구: Claude Desktop (기획/브레인스토밍) + Claude Code CLI (정리/포맷팅)
> 게임 연결: handoff 리포를 통해 간접 연결 (스토리→게임 단방향)

---

## 1. 프로젝트 목적

CrownFalle Proto의 세계관, 스토리, 캐릭터, 지역, 퀘스트를 독립적으로 설계하는 프로젝트.
게임 시스템(전투/경영)과 분리하여, 순수하게 내러티브 콘텐츠에 집중한다.

**콘텐츠 범위:**
1. 세계 설정 — 지리, 역사, 세력, 문화, 종교, 마법 체계
2. 메인 스토리 — 줄거리, 챕터 구조, 핵심 갈등, 엔딩 분기
3. 캐릭터 — 스토리 동료, 주요 NPC, 적 세력 인물
4. 지역/마을 — 월드맵 위치별 설정, 분위기, 고유 이벤트
5. 퀘스트/이벤트 시나리오 — 메인/사이드 퀘스트, 랜덤 이벤트 풀

---

## 2. 폴더 구조

```
C:\Users\iseul\OneDrive\Dev\Story\crown-falle-Interlude\
│
├── README.md                      ← 프로젝트 개요 + 읽는 순서
├── PROTOCOL.md                    ← 이 문서 (워크플로우/프로토콜)
├── INDEX.md                       ← 전체 문서 색인 (상태 추적)
├── CLAUDE.md                      ← 게임 연동 규칙 + 프로젝트 룰
│
├── agents/                        ← 에이전트 정의 (추후 설계)
│   └── README.md                  ← 에이전트 목록 + 역할 설명
│
├── world/                         ← 세계 설정 (확정본)
│   ├── overview.md                ← 세계 개요 (한 페이지 요약)
│   ├── geography.md               ← 지리 (대륙/지역/기후)
│   ├── history.md                 ← 역사 (연대기/주요 사건)
│   ├── factions.md                ← 세력 (국가/조직/용병단)
│   ├── culture.md                 ← 문화 (종교/관습/언어)
│   └── magic.md                   ← 마법 체계 (원소/규칙)
│
├── story/                         ← 메인 스토리
│   ├── premise.md                 ← 전제 (한 단락 요약)
│   ├── structure.md               ← 챕터 구조 + 흐름도
│   ├── themes.md                  ← 핵심 테마/메시지
│   ├── endings.md                 ← 엔딩 분기 구조
│   ├── drafts/                    ← 초안/작업 공간
│   │   └── braindump/             ← 브레인덤프 (주제별 자유 작성)
│   │       ├── README.md          ← 브레인덤프 사용법 + 목록
│   │       └── (주제별 파일들)
│   └── chapters/                  ← 챕터별 상세 (확정본)
│       ├── ch01-prologue.md
│       └── ...
│
├── characters/                    ← 캐릭터
│   ├── INDEX.md                   ← 캐릭터 목록 + 상태
│   ├── protagonist.md             ← 주인공
│   ├── companions/                ← 스토리 동료
│   │   ├── companion-template.md
│   │   └── ...
│   ├── npcs/                      ← 주요 NPC
│   │   ├── npc-template.md
│   │   └── ...
│   └── antagonists/               ← 적대 세력 인물
│       ├── antagonist-template.md
│       └── ...
│
├── locations/                     ← 지역/마을
│   ├── INDEX.md                   ← 지역 목록 + 월드맵 매핑
│   ├── location-template.md
│   └── region-XX-xxx/             ← 지역별 폴더
│       ├── overview.md
│       ├── town-xxx.md
│       └── poi-xxx.md
│
├── quests/                        ← 퀘스트/이벤트
│   ├── INDEX.md                   ← 퀘스트 목록 + 상태
│   ├── quest-template.md
│   ├── event-template.md
│   ├── main/                      ← 메인 퀘스트
│   ├── side/                      ← 사이드 퀘스트
│   └── events/                    ← 랜덤 이벤트
│
├── exports/                       ← 게임 프로젝트로 내보내기
│   └── README.md
│
└── archive/                       ← 폐기/대체된 설정 보관
    └── README.md
```

### 핵심 구분: drafts/ vs 확정본

| 위치 | 역할 | 상태 |
|------|------|------|
| `story/drafts/braindump/` | 자유 형식 아이디어, 메모, 발산 | 비정형 — 상태 관리 안 함 |
| `story/drafts/` | 구조화된 초안 (템플릿 적용 전) | 🔵 초안 |
| `story/`, `world/`, `characters/` 등 | 확정본 (템플릿 적용, 검토 완료) | 🟡~🟢 |

**작업 흐름:**
```
braindump/ (자유 발산)
    ↓ 정리
drafts/ (구조화된 초안)
    ↓ 포맷팅 + 검토
확정본 위치 (world/, story/, characters/ 등)
```

---

## 3. 워크플로우

### 3-1. 이중 도구 역할 분담

| 도구 | 역할 | 산출물 |
|------|------|--------|
| **Claude Desktop** | 브레인스토밍, 설정 토론, 초안 작성, 선택지 결정 | braindump 파일 + 초안 |
| **Claude Code CLI** | 파일 정리, 포맷팅, 일관성 검증, 템플릿 적용, 색인 갱신 | 확정본 마크다운 파일 |

### 3-2. 콘텐츠 생성 흐름

```
1. 브레인덤프 (Desktop)
   └── 원주 + Claude가 대화하며 아이디어 자유 발산
   └── 결과를 braindump/ 폴더에 주제별 파일로 저장
   └── 형식 자유 — 키워드, 문장, 질문, 스케치 모두 OK

2. 초안 작성 (Desktop)
   └── braindump에서 방향이 잡힌 내용을 drafts/에 구조화
   └── 핵심 결정사항을 대화 내에서 확정

3. 정리/포맷팅 (CLI)
   └── drafts/ → 확정본 위치로 이동
   └── 템플릿에 맞춰 포맷 정리
   └── INDEX.md 갱신
   └── 다른 문서와의 상호 참조 확인

4. 검토/수정 (Desktop)
   └── 정리된 문서를 읽고 수정 지시
   └── 반복 (2-3 사이클)

5. 확정 (CLI)
   └── 문서 상태를 "확정"으로 변경
   └── 필요 시 exports/ 폴더에 게임용 요약 생성
```

### 3-3. 게임 프로젝트 연결 흐름

```
crown-falle-Interlude/          crown-falle-handoff/
(스토리 프로젝트)                (핸드오프 리포)

exports/world-summary.md ──→    worldbuilding/world-summary.md
exports/companion-list.md──→    worldbuilding/companion-list.md
exports/location-map.md  ──→    worldbuilding/location-map.md
exports/quest-outline.md ──→    worldbuilding/main-quest-outline.md

방향: 단방향 (스토리 → 게임)
시점: 스토리 설정이 충분히 확정된 후 수동 내보내기
위치: exports/ 폴더에 게임용 요약본 생성 → handoff 리포로 복사
```

---

## 4. 프로토콜

### 4-1. 문서 네이밍 규칙

| 카테고리 | 패턴 | 예시 |
|----------|------|------|
| 세계 설정 | `{주제}.md` | `geography.md`, `factions.md` |
| 브레인덤프 | `bd-{주제}.md` | `bd-world-tone.md`, `bd-magic-ideas.md` |
| 챕터 | `ch{번호}-{짧은제목}.md` | `ch01-prologue.md` |
| 동료 | `comp-{번호}-{이름}.md` | `comp-001-aldric.md` |
| NPC | `npc-{번호}-{이름}.md` | `npc-001-merchant-karl.md` |
| 적대자 | `ant-{번호}-{이름}.md` | `ant-001-lord-vex.md` |
| 지역 | `region-{번호}-{이름}/` | `region-01-ashvale/` |
| 메인 퀘스트 | `mq-{번호}-{제목}.md` | `mq-001-fathers-oath.md` |
| 사이드 퀘스트 | `sq-{번호}-{제목}.md` | `sq-001-missing-caravan.md` |
| 랜덤 이벤트 | `evt-{번호}-{제목}.md` | `evt-001-campfire-tale.md` |

### 4-2. 문서 상태 관리

확정본 문서는 상단에 상태를 표기한다 (braindump는 상태 관리 안 함):

```markdown
> 상태: 🔵 초안 | 🟡 검토 중 | 🟢 확정 | 🔴 수정 필요 | ⚫ 폐기
```

| 상태 | 의미 | 다음 행동 |
|------|------|----------|
| 🔵 초안 | Desktop에서 작성된 1차 버전 | CLI에서 포맷팅 |
| 🟡 검토 중 | 포맷팅 완료, 내용 검토 필요 | Desktop에서 검토 |
| 🟢 확정 | 최종 승인, 변경 시 버전 관리 | exports/ 대상 |
| 🔴 수정 필요 | 다른 문서와 충돌 또는 수정 지시 | Desktop에서 수정 |
| ⚫ 폐기 | 설정 변경으로 무효화 | archive/로 이동 |

### 4-3. 상호 참조 규칙

문서 간 참조 시 상대 경로를 사용한다:

```markdown
<!-- 같은 폴더 -->
[세력 참조](factions.md)

<!-- 다른 폴더 -->
[알드릭 상세](../characters/companions/comp-001-aldric.md)

<!-- 앵커 참조 -->
이 지역은 **철왕국**(→ [세력](../../world/factions.md#철왕국))의 영토이다.
```

### 4-4. 설정 충돌 방지 규칙

1. **세계 설정이 최상위 권위** — 개별 캐릭터/퀘스트 문서가 world/ 문서와 충돌하면 world/가 우선
2. **변경 시 영향 스캔** — 세계 설정 변경 시 CLI가 관련 문서를 스캔하여 영향 목록 출력
3. **폐기는 삭제가 아닌 이동** — archive/ 폴더로 이동, 사유 기록
4. **확정 문서 수정 시 버전 표기** — `> 버전: v1.0 → v1.1 (변경 사유)`

### 4-5. 작업 우선순위 원칙

```
1단계: 브레인덤프 (braindump/)
  └── 세계 톤, 핵심 갈등, 주인공 동기, 분위기 키워드 등 자유 발산

2단계: 세계 개요 + 전제 (overview.md + premise.md)
  └── "이 세계는 무엇이고, 무슨 이야기인가"를 한 페이지로

3단계: 지리 + 세력 (geography.md + factions.md)
  └── 월드맵의 뼈대 + 주요 갈등 구조

4단계: 주인공 + 핵심 동료 2~3명
  └── 플레이어가 처음 만나는 인물들

5단계: 챕터 1 상세 + 시작 지역
  └── 프로토타입에 필요한 최소 콘텐츠

6단계: 나머지 확장 (추가 동료, 지역, 퀘스트...)
```

---

## 5. 브레인덤프 가이드

### braindump/ 사용 규칙

- **형식 완전 자유** — 키워드, 문장 조각, 질문, 참고 이미지 링크, 비교("X 같은 느낌") 모두 OK
- **상태 관리 안 함** — INDEX.md에 등록하지 않음
- **삭제 가능** — 확정본으로 승격된 후에는 삭제해도 됨
- **네이밍**: `bd-{주제}.md` (예: `bd-world-tone.md`, `bd-protagonist-motivation.md`)
- **Desktop 세션에서 생성** → CLI가 필요 시 정리하지만, braindump 자체는 손대지 않음

### braindump/ README.md 내용

```markdown
# 브레인덤프

이 폴더는 자유 형식 아이디어 저장소이다.
형식, 완성도, 일관성에 대한 제약 없음.

## 파일 목록
(Desktop 세션마다 갱신)

| 파일 | 주제 | 날짜 |
|------|------|------|
| bd-world-tone.md | 세계 분위기/톤 | 2026-03-XX |
| ... | ... | ... |

## 승격 이력
(braindump → 확정본으로 옮겨진 기록)

| 원본 | 승격 대상 | 날짜 |
|------|----------|------|
| ... | ... | ... |
```

---

## 6. 게임 프로젝트 레퍼런스

이 스토리 프로젝트는 아래 게임 설계와 정합성을 유지해야 한다:

| 참조 문서 | 위치 | 관련 사항 |
|----------|------|----------|
| 게임 루프 설계 | `handoff/.../2026-03-19-crownfalle-game-loop-design.md` | 4-Layer 구조, 4대 자원 |
| 레이어 레퍼런스 | `handoff/.../2026-03-19-crownfalle-layer-reference-research.md` | 6게임 비교, 37개 결정사항 |
| 전투 시스템 | `handoff/plans/design/` 내 T1~T2 설계서 | 교전, 스킬, 상태이상, VP |
| 에셋 요구사항 | `handoff/.../2026-03-19-crownfalle-asset-requirements.md` | 비주얼 스타일, 2D |

**정합성 체크 포인트:**
- 마법 체계(magic.md) ↔ 전투 원소 시스템(ElementResolver, 4원소)
- 세력(factions.md) ↔ 월드맵 지역 구조 + 평판(Reputation) 시스템
- 동료 전투 역할 ↔ 무기 기반 정체성 + 5스탯 시스템
- 퀘스트 보상 ↔ 4대 자원(Gold/Food/Morale/Reputation)
- 지역 시설 ↔ Town/Camp 시설 목록 (기획 세션에서 확정 예정)

---

## 7. Desktop 세션 프로토콜

### 세션 시작 시
1. "Interlude 세션 시작" 선언
2. 현재 작업 우선순위 확인 (4-5 참조)
3. braindump/ 또는 작업 대상 결정 후 진행

### 세션 종료 시
1. 오늘 작성/수정한 파일 목록 정리
2. 다운로드 가능한 파일로 생성
3. "CLI에서 포맷팅 필요" 항목 명시
4. 다음 세션 계획 간단 메모
