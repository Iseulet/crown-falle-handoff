# TODO 프로세스 — CLI 등록 규칙 (Addendum)

- Date: 2026-03-13
- 적용 대상: `2026-03-13-todo-process-design.md` 섹션 5 확장
- 구현 위치: `CLAUDE.md` (프로젝트 룰) + `handoff/TODO.md`

---

## 1. 인식 패턴

Claude Code CLI는 다음 두 방식 모두 TODO 등록으로 인식한다.

### 방식 A: 슬래시 커맨드

```
/todo 투사체 hit FX가 안 보여
/todo Archer idle 모션이 너무 빠름
/todo 교전 해제 시 파벌 서클 안 사라짐
```

### 방식 B: 자연어

```
TODO: 투사체 hit FX가 안 보여
todo: Archer idle 모션이 너무 빠름
할일: 교전 해제 시 파벌 서클 안 사라짐
해야할것: VP 0일 때 이탈 연출 추가
```

### 인식 트리거 키워드

| 패턴 | 대소문자 | 위치 |
|------|---------|------|
| `/todo` | 무관 | 메시지 시작 |
| `TODO:` | 무관 | 메시지 시작 또는 중간 |
| `할일:` | — | 메시지 시작 또는 중간 |
| `해야할것:` | — | 메시지 시작 또는 중간 |

> 메시지 중간에 있어도 인식한다. 예: "아 그리고 TODO: dodge 에셋도 필요해"

---

## 2. 처리 흐름

```
원주 메시지: "/todo 투사체 hit FX가 안 보여"
  │
  ▼
Claude Code 인식
  │
  ├── 1. TODO.md 🟡 다음 섹션 맨 아래에 항목 추가
  │     - [ ] 투사체 hit FX가 안 보여 `[테스트:03-13]` `[2026-03-13]`
  │
  ├── 2. 확인 응답
  │     "✅ TODO 등록: 투사체 hit FX가 안 보여 → 🟡 다음"
  │
  └── 3. 현재 작업 계속 (흐름 중단 없음)
```

**핵심: 등록 후 즉시 기존 작업으로 복귀.** 별도 확인/옵션 선택 없음.

---

## 3. 출처 태그 자동 결정

Claude Code가 현재 컨텍스트에서 출처를 자동 판단한다:

| 현재 상태 | 자동 태그 |
|----------|----------|
| `@Implementor` 활성 중 | `[구현:MM-DD]` |
| `@Reviewer` 활성 중 | `[리뷰:MM-DD]` |
| `@Planner` 활성 중 | `[설계:MM-DD]` |
| 에이전트 없음 (일반 대화) | `[테스트:MM-DD]` |

> 대부분의 테스트 중 TODO는 에이전트 비활성 상태이므로 `[테스트:]`가 기본값.

---

## 4. 복수 항목 등록

한 메시지에 여러 TODO를 넣을 수 있다:

```
/todo 투사체 hit FX 안 보임
/todo Archer idle 너무 빠름
/todo 사망 시 파벌 서클 잔류
```

또는:

```
TODO 몇 개 추가해줘:
- 투사체 hit FX 안 보임
- Archer idle 너무 빠름
- 사망 시 파벌 서클 잔류
```

응답:

```
✅ TODO 3건 등록 → 🟡 다음:
  • 투사체 hit FX 안 보임
  • Archer idle 너무 빠름
  • 사망 시 파벌 서클 잔류
```

---

## 5. 조회 커맨드

등록만이 아니라 현재 TODO 상태도 빠르게 확인할 수 있다:

| 커맨드 | 동작 |
|--------|------|
| `/todo list` 또는 `할일 목록` | 🔴 + 🟡 항목만 출력 (백로그 제외) |
| `/todo all` | 전체 출력 (백로그 포함) |
| `/todo done {번호 또는 키워드}` | 항목을 ✅ 완료로 이동 |
| `/todo urgent {번호 또는 키워드}` | 🟡 → 🔴 승격 |
| `/todo backlog {번호 또는 키워드}` | 🟡 → 🟢 강등 |

### 예시

```
원주: /todo list

Claude Code:
  📋 TODO (🔴 0 / 🟡 8)

  🟡 다음:
   1. FBX 클립 이름 확인
   2. HP_BAR_Y 높이 조정
   3. 싱크 파일 갱신
   4. T1+T2 전체 검증
   5. 샌드박스 시나리오 8~11 연결
   6. ProjectileManager 구현
   7. 투사체 hit FX 안 보임      ← NEW
   8. Archer idle 너무 빠름       ← NEW
```

```
원주: /todo done 투사체 hit FX

Claude Code:
  ✅ 완료 처리: 투사체 hit FX 안 보임
```

```
원주: /todo urgent HP_BAR

Claude Code:
  🔴 긴급 승격: HP_BAR_Y 높이 조정
```

---

## 6. CLAUDE.md 추가 규칙

```markdown
## TODO 관리

- 메시지에서 `/todo`, `TODO:`, `할일:`, `해야할것:` 패턴 감지 시
  `handoff/TODO.md`의 🟡 다음 섹션에 자동 추가
- 출처 태그는 현재 활성 에이전트 기반으로 자동 결정
- 등록 후 "✅ TODO 등록: {내용} → 🟡 다음" 확인 출력
- `/todo list` — 🔴 + 🟡 출력
- `/todo done {키워드}` — ✅ 완료 이동
- `/todo urgent {키워드}` — 🔴 승격
- `/todo backlog {키워드}` — 🟢 강등
- TODO 등록/변경 후 현재 작업 흐름을 중단하지 않는다
```

---

## 7. Desktop에서의 TODO 등록

Desktop 채팅에서도 동일 패턴을 인식한다.
단, Desktop은 `TODO.md` 파일을 직접 수정할 수 없으므로:

```
원주 (Desktop): TODO: 원소 반응 이펙트가 너무 약해

Claude Desktop:
  📝 TODO 대기: "원소 반응 이펙트가 너무 약해"
  → 다음 CLI 세션에서 TODO.md에 반영됩니다.
```

Desktop에서 등록된 항목은 **메모리에 임시 저장**되고,
다음 CLI 세션 시작 시 Claude Code가 TODO.md에 일괄 기입한다.
