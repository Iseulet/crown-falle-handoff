# CrownFalle Proto — 프로젝트 레퍼런스
> 생성일: 2026-03-13 | 갱신: 2026-03-19 (Unity 엔진 전환 반영)

---

## 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 장르 | 턴제 전술 RPG (레퍼런스: Wartales, Dust and Salt) |
| 엔진 | Unity 6.3 LTS (6000.3.11f1) + C# |
| 프레임워크 | Turn Based Strategy Framework (TBSF) |
| 렌더링 | 2D (Dust and Salt 스타일 — 다크 중세 판타지, 일러스트+텍스트) |
| 세계관 | 중세 다크 판타지 — 용병단 "에코르셔"의 탈주자들이 이끄는 부대 |
| 데이터 | JSON (Newtonsoft Json.NET으로 로드) |

---

## 게임 루프 구조 (4-Layer)

| 레이어 | 역할 |
|--------|------|
| Town/Camp | 마을 순회 + 이동식 캠프. 자원: Gold/Food/Morale/Reputation |
| WorldMap | 포인트 클릭 이동. 식량+피로도 소모. 이벤트 4종 |
| Battle | 헥스 턴제 전투 3~10명. XP+루팅. HP→캠프/부상→마을 |
| Roster Management | 유닛 관리. 무기 기반 정체성. 영구 사망 |

---

## 스탯 체계

### 1차 스탯
STR / DEX / CON / INT / WIL
- 클래스 없음 → 무기 선택 + 스탯 배분 + 스킬 습득으로 정체성 결정

### 2차 스탯 (이식 대상)
| 스탯 | 공식 | 비고 |
|------|------|------|
| HP | 30 + CON × 10 | 전투 간 영속 |
| STA | 20 + CON × 5 | effective = base - weight - crit - milestone |
| HIT | weapon_hit_base + DEX × hit_per_dex + 보정 8단계 | |
| MV | base_mv + DEX × mv_per_dex - overweight - 부상 | 하한 0 |
| CRIT | weapon_crit_base + DEX × crit_per_dex + 보정 | 하한 0 |
| ARM | 방어구 4부위 합산 (스탯 파생 아님) | 전투 후 차감 유지 |

### 물리 공격 3종 (이식 대상)
| 유형 | 대상 | 특징 |
|------|------|------|
| Strike | ARM+STA+HP(전이) | ARM 70%/STA 30% 동적 분배 |
| Thrust | HP | ARM 관통 마일스톤 4구간 |
| Slash | HP | ARM>50% 무효, 전용 마일스톤 |

---

## 핵심 아키텍처 원칙

### 1. 데이터 기반 설계
- 모든 수치는 JSON. 코드에 수치 직접 기재 금지.
- Unity에서 Newtonsoft Json.NET으로 로드
- 기존 JSON 파일 구조 유지 (경로만 Unity 프로젝트에 맞게 조정)

### 2. TBSF 활용
- TBSF 내장 그리드/턴/유닛 시스템 최대한 활용
- 커스텀 로직은 TBSF 위에 레이어로 추가

### 3. 전투 알고리즘 이식
- C#으로 재구현 (GDScript → C# 변환)
- 클래스 구조: 정적 계산기 패턴 유지 (AttackResolver, BonusCalculator 등)

---

## 이식 대상 시스템

### 전투 알고리즘 (이식)
| 시스템 | 역할 |
|--------|------|
| AttackResolver | Strike/Thrust/Slash 3종 데미지 해결 |
| BonusCalculator | HIT/CRIT/데미지 8단계 보정 |
| StatCalculator | base_damage, overweight 계산 |
| EngagementSystem | 교전 쌍 캐싱, 다중 교전 방지, 기회공격 |
| StatusManager | 상태이상 13종 관리 |
| SkillManager → SkillResolver | 사거리, STA 비용, 사용횟수 검증 |
| ElementResolver | 원소 6종, 3레이어 귀속 |
| VPManager | Valor Points 파티 공유 풀 |
| WoundManager | 부상 3등급×3부위 |
| FacingSystem | 전방호 판정 (is_frontal, is_backattack) |

### 이식 JSON 데이터
| 파일 | 내용 |
|------|------|
| `combat_rules.jsonc` | T1~T4 통합 전투 규칙 |
| `weapon_table.jsonc` | 무기 11종 |
| `skill_table.jsonc` | 스킬 17종 |
| `preset_config.jsonc` | 아키타입 6종 |
| `wound_config.jsonc` | 부상 3등급×3부위 |
| `status_effects.json` | 상태이상 13종 통합 스키마 |

---

## 폐기된 시스템 (Godot 전용)

| 시스템 | 사유 |
|--------|------|
| [폐기] UnitRenderer3D | Godot 3D 렌더러 — Unity 2D로 대체 |
| [폐기] CameraDirector | Godot 3D 카메라 — Unity 2D 카메라로 대체 |
| [폐기] AnimEventDispatcher | Godot AnimationPlayer 기반 |
| [폐기] CompositionBuilder | Godot AnimationTree 기반 |
| [폐기] GridManager | TBSF 내장 그리드로 대체 |
| [폐기] DebugCombatPanel | Unity에서 재구현 예정 |
| [폐기] EnvironmentBuilder | Godot 3D 환경 — 폐기 |
| [폐기] FBX 파이프라인 | 3D 에셋 전면 폐기 |

---

## 교전 시스템 알고리즘 (이식)

교전 조건: 인접(거리 1) + 상호 전방 대치.

```
_pairs: { unitA: unitB, unitB: unitA }

refresh() 흐름:
  1. 기존 쌍 유효성 검사 (거리>1 or 정면 해제 → 삭제)
  2. 미교전 유닛끼리만 탐색 → 조건 충족 첫 상대와 쌍 등록
  3. 모든 유닛 SetEngaged() 갱신

GetOpportunityAttackers() → 교전 파트너 1명만 반환
```

---

## 코딩 컨벤션 (Unity C#)

| 항목 | 규칙 |
|------|------|
| 클래스 | `PascalCase` |
| 함수 | `PascalCase` (C# 표준) |
| 변수 | `camelCase` (private: `_camelCase`) |
| 상수 | `UPPER_SNAKE_CASE` |
| JSON 키 | `snake_case` (기존 유지) |
| 수치 | JSON에만 — 코드 매직 넘버 금지 |

---

## Claude Code 운영 환경

### 슬래시 커맨드
| 커맨드 | 역할 |
|--------|------|
| `/plan [request]` | @Planner 전환, 설계 시작 |
| `/implement [target]` | @Implementor 전환, 구현 시작 |
| `/review [target]` | @Reviewer 전환, 리뷰 |
| `/commit [notes]` | 체크리스트 → 커밋 메시지 제안 |
| `/sync [scope]` | 싱크 패키지 생성 |

### 에이전트 구조 (유지)
| 에이전트 | 역할 | 도구 변경 |
|---------|------|----------|
| @Planner | 아키텍처 설계, 기능 기획 | Godot → Unity |
| @Implementor | 코드 구현, 버그 수정 | GDScript → C# |
| @Reviewer | 코드 리뷰, 품질 감시 | GDScript → C# |

### 스킬 파일 (변경 예정)
- `skills/gdscript-conventions.md` → 추후 `skills/csharp-conventions.md`로 교체
- `skills/data-driven-design.md` — 유지 (JSON 기반 설계 원칙 동일)

### 공개 핸드오프 리포
- `Iseulet/crown-falle-handoff` — Claude Desktop raw URL 접근
- 동기화: `tools/sync_to_public.ps1` (proto→handoff 단방향)

---

## 확정 결정사항 이력 (주요)

| 날짜 | 결정 | 상태 |
|------|------|------|
| 2026-03-19 | Godot → Unity 6.3 LTS + TBSF 전환 | ✅ 확정 |
| 2026-03-19 | 3D → 2D (Dust and Salt 스타일) | ✅ 확정 |
| 2026-03-19 | 4-Layer 게임 루프 구조 | ✅ 확정 |
| 2026-03-17 | EngagementSystem 쌍 캐싱 방식 | ✅ 확정 (이식) |
| 2026-03-16 | 물리 공격 3종 체계 (strike/thrust/slash) | ✅ 확정 (이식) |
| 2026-03-13 | ARM/RES 게이지형 + CC 게이팅 | ✅ 확정 (이식) |
| 2026-03-13 | VP(사기) 시스템 — 파티 공유 풀 | ✅ 확정 (이식) |
| 2026-03-13 | 원소 6종 3레이어 구조 | ✅ 확정 (이식) |
| 2026-03-09 | 렌더러 Forward Plus | 🔄 수정됨 (Unity로 대체) |
| 2026-03-09 | 3D Perspective 전환 | 🔄 수정됨 (2D로 대체) |
| 2026-03-03 | Godot 4.6.1 + GDScript | 🔄 수정됨 (Unity로 대체) |
