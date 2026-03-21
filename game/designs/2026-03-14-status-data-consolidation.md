# CrownFalle — 상태이상 데이터 통합 설계서

> 작성일: 2026-03-14
> Agent: @Planner
> Status: **확정** — 구현 착수 가능
> 의존성: T2-1 (Tier 2 상태이상 시스템), passive-skill-revision (패시브 개정안)
> 참조: `2026-03-13-tier2-status-effects.md`, `2026-03-13-passive-skill-revision.md`

---

## 1. 통합 목적

현재 상태이상 데이터가 **두 파일**에 중복 정의되어 불일치 발생:

| 파일 | 효과 수 | ID 포맷 | 구조 | 사용처 |
|------|---------|---------|------|--------|
| `data/effects/status_effects.json` | 11개 | 소문자 (stun, burning) | 키-오브젝트 맵, 고정 `dot_dmg` | StatusManager 런타임 전체 |
| `data/skills/status_effects.json` | 7개 | 대문자 (STUNNED, BURNING) | 배열, 스탯연동 `dot_damage_coefficient` | `is_action_disabled()`에서만 부분 참조 |

**문제:**
1. 같은 개념의 효과가 다른 ID/스키마로 존재 (stun vs STUNNED, burning vs BURNING)
2. `is_action_disabled()`가 매 호출마다 두 번째 파일을 로드 (캐싱 없음, 성능 문제)
3. 패시브 개정안이 대문자 ID 사용을 전제하나, 런타임 코드 전체가 소문자
4. skills 파일의 visual/motion_override/stacks/refresh_on_reapply 필드는 사용처 없음 (데드 필드)

**결정:**
- 단일 파일 통합: `data/effects/status_effects.json`으로 일원화
- ID 포맷: 소문자 유지 (런타임 표준)
- `data/skills/status_effects.json` 폐기

---

## 2. 통합 스키마 정의

두 파일의 필드를 소문자 ID 기반 단일 스키마로 병합한다.

### CC 예시 (stun)

```json
{
  "stun": {
    "name": "기절",
    "type": "cc",
    "category": "physical",
    "gate": "arm",
    "duration": 1,
    "effect": "skip_turn",
    "disable_actions": ["move", "attack", "skill"],
    "stacks": false,
    "refresh_on_reapply": false,
    "visual": "fx_stun_sparks",
    "motion_override": "stun_loop"
  }
}
```

### DoT 예시 (poisoned)

```json
{
  "poisoned": {
    "name": "중독",
    "type": "dot",
    "category": "physical",
    "element": "poison",
    "gate": null,
    "duration": 3,
    "dot_dmg": 3,
    "dot_damage_stat": null,
    "dot_damage_coefficient": null,
    "stacks": false,
    "refresh_on_reapply": true,
    "visual": "fx_bleed_drip",
    "motion_override": null
  }
}
```

### 핵심 설계 결정

**`dot_dmg` (고정값) vs `dot_damage_stat` + `coefficient` (스탯연동):**
- **고정값 유지.** 스탯연동 DoT는 Tier 3 확장 시 `dot_damage_stat`/`dot_damage_coefficient` 필드를 nullable로 예비.
- 두 필드가 null이면 `dot_dmg` 고정값 사용, non-null이면 스탯연동 공식 적용.

**`disable_actions` 필드 추가:**
- `effect: "skip_turn"`과 중복되지만, cc 외 세밀한 행동 제어(fear의 skill만 차단 등)를 위해 명시적 배열로 병기.
- `disable_actions.size() > 0`이면 해당 행동 차단 → `is_action_disabled()` 구현 단순화.

**`prevents_action` 필드 폐기:**
- skills 파일에 존재했던 불리언 필드. `disable_actions.size() > 0`으로 대체 가능하여 통합 스키마에서 제거.

### 전체 필드 명세

| 필드 | 타입 | CC | DoT | Debuff | 설명 |
|------|------|:--:|:---:|:------:|------|
| `name` | String | ✅ | ✅ | ✅ | 한국어 표시명 |
| `type` | String | ✅ | ✅ | ✅ | `"cc"` / `"dot"` / `"debuff"` |
| `category` | String | ✅ | ✅ | ✅ | `"physical"` / `"magical"` |
| `element` | String? | — | ✅ | ✅ | 원소 속성 (cc는 없음) |
| `gate` | String? | ✅ | — | — | `"arm"` / `"res"` / `null` |
| `duration` | int | ✅ | ✅ | ✅ | 기본 지속 턴 |
| `effect` | String? | ✅ | — | — | `"skip_turn"` / `"attack_ally"` / `"random_move"` |
| `disable_actions` | String[] | ✅ | — | — | `["move", "attack", "skill"]` 의 부분집합 |
| `dot_dmg` | int? | — | ✅ | — | 고정 DoT 데미지/턴 |
| `dot_damage_stat` | String? | — | ✅ | — | 스탯연동 시 참조 스탯 (Tier 3 예비, null) |
| `dot_damage_coefficient` | float? | — | ✅ | — | 스탯 계수 (Tier 3 예비, null) |
| `mv_penalty` | int? | — | — | ✅ | 이동력 감소 (shocked/chilled) |
| `physical_bonus_dmg_pct` | int? | ✅ | — | — | 물리 피격 추가 데미지% (frozen) |
| `stacks` | bool | ✅ | ✅ | ✅ | 중첩 가능 여부 |
| `refresh_on_reapply` | bool | ✅ | ✅ | ✅ | 재적용 시 duration 리셋 여부 |
| `visual` | String? | ✅ | ✅ | ✅ | 비주얼 FX 이름 |
| `motion_override` | String? | ✅ | ✅ | ✅ | idle 대체 루프 모션 |

---

## 3. 효과 매핑 테이블

기존 두 파일의 ID 대응 + 통합 후 ID:

| effects/ (현행) | skills/ (폐기) | 통합 ID | 타입 | 비고 |
|-----------------|----------------|---------|------|------|
| stun | STUNNED | `stun` | cc | |
| knockdown | KNOCKDOWN | `knockdown` | cc | |
| frozen | — | `frozen` | cc | skills 파일에 없었음 |
| charm | — | `charm` | cc | skills 파일에 없었음 |
| fear | FRIGHTENED | `fear` | cc | |
| poisoned | — | `poisoned` | dot | BLEED과 별개 |
| burning | BURNING | `burning` | dot | |
| bleeding | BLEED | `bleeding` | dot | |
| shocked | — | `shocked` | debuff | skills 파일에 없었음 |
| wet | — | `wet` | debuff | |
| chilled | — | `chilled` | debuff | |
| — | CRIPPLED | `crippled` | debuff | **신규 추가** |
| — | WEAKENED | `weakened` | debuff | **신규 추가** |

→ 통합 후 **13개** 효과

---

## 4. 코드 변경 명세 (향후 구현용)

| 파일 | 변경 | 설명 |
|------|------|------|
| `data/effects/status_effects.json` | 수정 | 통합 스키마로 13개 효과 재작성 |
| `data/skills/status_effects.json` | **삭제** | 역할 종료 |
| `StatusManager.gd` | 수정 | `is_action_disabled()`: 스킬 파일 로드 제거, `_status_data`의 `disable_actions` 사용 |
| `StatusManager.gd` | 수정 | `setup()`에서 모든 필드 캐싱 (매 호출 로드 제거) |
| `PassiveManager.gd` | 수정 | `apply_status` 시 대문자→소문자 변환 또는 패시브 데이터의 status_id를 소문자로 갱신 |
| `passive-skill-revision.md` | 수정 | 대문자 ID 예시를 소문자로 변경 (본 설계서와 동시 작업) |

---

## 5. 영향받지 않는 파일 (명시)

| 파일 | 이유 |
|------|------|
| `data/effects/element_table.json` | 이미 소문자 ID 사용, 변경 없음 |
| `data/skills/active_skills.json` | cc_type/dot_type이 이미 소문자 |
| `DebugCombatPanel.gd` | effects/ 파일만 참조, 변경 없음 |

---

## 6. 검증 방법

구현 완료 후 아래 항목을 확인한다:

1. **StatusManager 단독 동작:** StatusManager의 모든 public API가 `data/effects/status_effects.json` 단독으로 동작
2. **행동 차단 정확성:** `is_action_disabled("move"/"attack"/"skill")`가 `disable_actions` 기반으로 정확히 반환
3. **디버그 패널:** DebugCombatPanel에서 13개 효과 모두 드롭다운 표시
4. **원소 반응:** ElementResolver 원소 반응 (wet+lightning→shocked 등) 정상 동작
5. **패시브 연동:** PassiveManager의 `apply_status` 효과가 소문자 ID로 정상 동작
