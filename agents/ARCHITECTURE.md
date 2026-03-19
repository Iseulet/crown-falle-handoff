# Agent Architecture
> 🇰🇷 에이전트 아키텍처

This document defines the roles and interactions between the specialized agents in this project.
> 🇰🇷 이 문서는 프로젝트 내 전문 에이전트의 역할과 상호작용을 정의합니다.

## Roles
- **Planner:** Architectural design and feature planning.
  > 🇰🇷 **Planner:** 아키텍처 설계 및 기능 기획
- **Implementor:** Code implementation and bug fixes.
  > 🇰🇷 **Implementor:** 코드 구현 및 버그 수정
- **Reviewer:** Quality assurance and performance audit.
  > 🇰🇷 **Reviewer:** 품질 보증 및 성능 감시

## Agent Conflict Resolution Rule
> 🇰🇷 에이전트 충돌 해결 규칙

When two agents produce conflicting outputs:
> 🇰🇷 두 에이전트 산출물이 충돌할 때

Priority order:
1. DECISIONS.md confirmed decisions always win
2. @Reviewer overrides @Implementor on code quality issues
3. @Planner overrides @Implementor on architecture issues
4. If unresolvable, notify user immediately:
   "⚠️ 에이전트 충돌 감지
   Planner 의견: [내용]
   Reviewer 의견: [내용]
   어떻게 처리할까요?"
> 🇰🇷 우선순위:
> 🇰🇷 1. DECISIONS.md 확정 사항 최우선
> 🇰🇷 2. 코드 품질 이슈는 Reviewer 우선
> 🇰🇷 3. 아키텍처 이슈는 Planner 우선
> 🇰🇷 4. 해결 불가 시 사용자에게 즉시 보고

---

## Test Protocol
> 🇰🇷 테스트 프로토콜

Implementor completion checklist — after implementation, always ask:
"✅ 구현 완료. 아래 항목 테스트 부탁드립니다:
1. [ ] 기본 동작 확인 (Godot에서 실행)
2. [ ] 예외 상황 확인 (엣지 케이스)
3. [ ] 기존 기능 영향 없음 확인
테스트 완료 후 결과 알려주세요."
> 🇰🇷 구현 완료 후 위 체크리스트로 테스트 요청

Reviewer checklist — when @Reviewer is called, check:
1. GDScript 컨벤션 준수 여부
2. 네이밍 컨벤션 준수 여부
3. 설계 문서와 구현 일치 여부
4. 버그/엣지 케이스 여부
5. 성능 이슈 여부 (불필요한 루프, 메모리 누수)

Output format:
"📋 코드 리뷰 결과
✅ 통과 항목: [목록]
⚠️ 이슈 항목: [목록]
🔧 수정 권고: [내용]
👉 전달 대상: @Implementor / @Planner / 커밋 승인"
> 🇰🇷 리뷰 완료 후 위 형식으로 결과 보고
> 🇰🇷 직접 코드 수정 금지, 리포트만 작성

---

## Context Overload Handling
> 🇰🇷 컨텍스트 과부하 대응

When context approaches limit (80%+):
1. Notify user:
   "⚠️ 컨텍스트 한계 근접.
   현재 작업 상태를 handoff/CURRENT.md에 저장합니다.
   새 세션 시작을 권장합니다."
2. Save current state to handoff/CURRENT.md immediately
3. Save work log to handoff/LOG/날짜.md
4. List pending tasks clearly
> 🇰🇷 컨텍스트 80% 도달 시 즉시 상태 저장
> 🇰🇷 새 세션 시작 권장
> 🇰🇷 미완료 작업 목록 명시

New session recovery after context reset:
1. Read handoff/CURRENT.md
2. Read handoff/LOG/최근날짜.md
3. Notify user:
   "🔄 이전 세션 복구 완료.
   마지막 작업: [내용]
   이어서 진행할까요?"
> 🇰🇷 새 세션 시작 시 이전 상태 자동 복구

---

## Agent Transition Rule
> 🇰🇷 에이전트 전환 규칙

When switching agents, always:
1. Save current state to handoff/CURRENT.md
2. Record active agent: "Active Agent: @에이전트명"
3. Explicitly reset role context: "Switching to @에이전트명. Previous agent context cleared."
4. New agent reads handoff/CURRENT.md before starting
> 🇰🇷 에이전트 전환 시 반드시:
> 🇰🇷 1. CURRENT.md에 현재 상태 저장
> 🇰🇷 2. 활성 에이전트 명시
> 🇰🇷 3. 이전 에이전트 컨텍스트 명시적 초기화
> 🇰🇷 4. 새 에이전트는 CURRENT.md 읽고 시작

---

## Concurrent Execution Prevention Rule
> 🇰🇷 동시 실행 방지 규칙

Before starting any agent task:
1. Check handoff/CURRENT.md for active agent
2. If another agent is already active, notify user:
   "⚠️ 현재 @에이전트명 실행 중입니다.
   완료 후 실행하거나 중단할까요?"
3. Only one agent can be active at a time
> 🇰🇷 작업 시작 전 CURRENT.md에서 활성 에이전트 확인
> 🇰🇷 동시에 하나의 에이전트만 실행 가능

---

## Agent Name Recognition Rule
> 🇰🇷 에이전트 호출 인식 규칙

Accept all case variations:
- @Planner = @planner = @PLANNER
- @Implementor = @implementor = @IMPLEMENTOR
- @Reviewer = @reviewer = @REVIEWER
> 🇰🇷 대소문자 무관하게 에이전트 호출 인식

---

## Agent Call Scope Rule
> 🇰🇷 에이전트 호출 범위 규칙

Agent calls are only valid inside project folders containing an `agents/` directory.
When called from `C:\Users\iseul\OneDrive\Dev\` (global):
"⚠️ 에이전트는 프로젝트 폴더에서만 호출 가능합니다.
crown-falle-proto 폴더로 이동 후 실행해주세요."
> 🇰🇷 에이전트 호출은 agents/ 폴더가 있는 프로젝트에서만 유효
> 🇰🇷 글로벌에서 호출 시 프로젝트 폴더로 안내

---

## Pending Improvements
> 🇰🇷 향후 개선 사항

- [x] Agent handoff format definition
  > 🇰🇷 에이전트 핸드오프 형식 정의
- [x] Agent permission boundaries
  > 🇰🇷 에이전트 권한 경계 설정
- [x] Agent conflict resolution rule
  > 🇰🇷 에이전트 충돌 해결 규칙
- [x] Test protocol definition
  > 🇰🇷 테스트 프로토콜 정의
- [x] Context overload handling
  > 🇰🇷 컨텍스트 과부하 대응
- [x] Agent transition rule
  > 🇰🇷 에이전트 전환 규칙
- [x] Concurrent execution prevention
  > 🇰🇷 동시 실행 방지 규칙
- [x] Agent name recognition (case-insensitive)
  > 🇰🇷 에이전트 호출 대소문자 무관 인식
- [x] Agent call scope rule
  > 🇰🇷 에이전트 호출 범위 규칙
- [ ] Automated regression testing pipeline
  > 🇰🇷 자동화된 회귀 테스트 파이프라인
