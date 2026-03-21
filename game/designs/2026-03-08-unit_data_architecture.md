# CrownFalle Unit Data Architecture
> 유닛 데이터 구조 설계 문서

- Created: 2026-03-08
- Status: ✅ 확정
- References:
  - `handoff/plans/design/2026-03-08-stat_system.md`
  - `handoff/plans/design/2026-03-08-stamina_system.md`
  - `handoff/plans/design/2026-03-08-passive_skill_system.md`

---

## 1. 개요

플레이어 캐릭터(용병)와 적 모두 동일한 클래스 테이블을 공유.
클래스가 스탯 초기값 + 패시브 스킬을 결정하며,
UnitData 공통 베이스에서 MercenaryData / EnemyData가 상속.

---

## 2. 클래스 테이블 공유 구조

```
data/classes/class_config.json
  └─ Fighter, Archer, Rogue, Mage  (플레이어/적 공통)

data/mercenaries/roster_config.json
  └─ 각 용병: class_id 참조 → 스탯/패시브 자동 적용

data/encounters/encounter_01.json
  └─ 각 적: class_id 참조 → 스탯/패시브 자동 적용
```

> 적 Fighter는 플레이어 Fighter와 동일한 기본 스탯/패시브를 가짐
> 개별 조정이 필요한 경우 stat_overrides / class_overrides 로 덮어쓰기

---

## 3. UnitData 베이스 구조 (GDScript Resource)

```gdscript
# scripts/data/UnitData.gd
class_name UnitData
extends Resource

# 기본 정보
@export var unit_id: String
@export var unit_name: String
@export var class_id: String       # class_config.json 참조 키
@export var level: int = 1

# 1차 스탯 (class_config base_stats + stat_overrides 적용 후 저장)
@export var str: int
@export var dex: int
@export var con: int
@export var int_stat: int          # GDScript 예약어 충돌 방지 — JSON 키는 "int"
@export var wil: int

# 추가 습득 패시브 (직업 외 스킬 — 아이템/이벤트 등으로 획득)
@export var extra_passives: Array[String] = []

# 런타임 상태 (저장 안 함 — 전투 중 CombatUnit에서 계산)
# hp, mp, sta, hit, mv, crit, arm, res
```

### int_ 키 매핑 주의사항
```
JSON 파일:    "int": 9
GDScript:     int_stat: int

DataLoader.gd에서 명시적 매핑 필요:
  unit_data.int_stat = json_data["int"]
```

---

## 4. MercenaryData (UnitData 상속)

```gdscript
# scripts/data/MercenaryData.gd
class_name MercenaryData
extends UnitData

# 용병 전용 필드
@export var nickname: String = ""
@export var experience: int = 0
@export var is_available: bool = true   # 부상 등으로 출전 불가 여부
```

### roster_config.json 예시
```json
{
  "mercenaries": [
    {
      "unit_id": "MERC_00001",
      "unit_name": "Aldric",
      "class_id": "Fighter",
      "level": 1,
      "stat_overrides": {},
      "class_overrides": {},
      "extra_passives": []
    },
    {
      "unit_id": "MERC_00002",
      "unit_name": "Sera",
      "class_id": "Archer",
      "level": 1,
      "stat_overrides": {},
      "class_overrides": {},
      "extra_passives": []
    }
  ]
}
```

---

## 5. EnemyData (UnitData 상속)

```gdscript
# scripts/data/EnemyData.gd
class_name EnemyData
extends UnitData

# 적 전용 필드
@export var ai_behavior: String = "aggressive"
@export var loot_table: String = ""
@export var exp_reward: int = 0
```

### encounter_01.json 예시
```json
{
  "encounter_id": "ENC_00001",
  "enemies": [
    {
      "unit_id": "ENEMY_00001",
      "unit_name": "용병단 낙오자",
      "class_id": "Fighter",
      "level": 1,
      "ai_behavior": "aggressive",
      "exp_reward": 30,
      "stat_overrides": {},
      "class_overrides": {},
      "extra_passives": []
    },
    {
      "unit_id": "ENEMY_00002",
      "unit_name": "숲속 궁수",
      "class_id": "Archer",
      "level": 1,
      "ai_behavior": "ranged",
      "exp_reward": 25,
      "stat_overrides": {},
      "class_overrides": {},
      "extra_passives": []
    }
  ]
}
```

---

## 6. 오버라이드 규칙

### stat_overrides — 1차 스탯 덮어쓰기
클래스 base_stats의 1차 스탯(str/dex/con/int/wil)을 개별 유닛 단위로 조정.

```json
"stat_overrides": {
  "str": 12,
  "con": 10
}
```

### class_overrides — 클래스 속성 덮어쓰기
mv_base / crit_bonus 등 클래스 레벨 속성을 개별 유닛 단위로 조정.

```json
"class_overrides": {
  "mv_base": 6,
  "crit_bonus": 15
}
```

> 보스, 특수 적, 고유 용병 등에 활용
> 오버라이드 필드만 덮어쓰고 나머지는 클래스 기본값 유지

---

## 7. 클래스 로딩 흐름

```
유닛 생성 시:
  1. class_id → class_config.json 조회
  2. base_stats 로드
  3. stat_overrides 적용 (있는 필드만 덮어쓰기)
  4. class_overrides 적용 (mv_base, crit_bonus 등)
  5. passives[] 로드 + extra_passives[] 합산
  6. passive_skills.json에서 스킬 데이터 조회 → unit.active_passives[] 캐싱
  7. CombatUnit에 적용 (2차 스탯 런타임 계산)

패시브 적용:
  플레이어/적 구분 없이 PassiveManager 동일 처리
```

---

## 8. 레벨 및 스탯 성장

### 플레이어 캐릭터
레벨업 시 stat_points_per_level(1) 포인트를 플레이어가 1차 스탯에 직접 배분.
자동 성장 없음 — 배분하지 않으면 스탯 증가 없음.

### 적 자동 성장 ✅ 확정
직업 주 스탯에 레벨당 1포인트 자동 배분.
주 스탯이 2개인 직업은 레벨마다 교대 배분.

| 직업 | 주 스탯 | 성장 방식 |
|------|--------|---------|
| Fighter | STR, CON | 홀수 레벨 → STR +1 / 짝수 레벨 → CON +1 |
| Archer  | DEX     | 매 레벨 → DEX +1 |
| Rogue   | DEX     | 매 레벨 → DEX +1 |
| Mage    | INT, WIL | 홀수 레벨 → INT +1 / 짝수 레벨 → WIL +1 |

```
예: Lv3 Fighter = 기본 STR 8 / CON 8
               + Lv2(CON+1) + Lv3(STR+1)
               = STR 9 / CON 9
```

---

## 9. 데이터 파일 구조

```
data/
  classes/
    class_config.json          ← 클래스 정의 (스탯 + 패시브 목록)
  skills/
    passive_skills.json        ← 패시브 스킬 통합 테이블
  mercenaries/
    roster_config.json         ← 용병 목록
  encounters/
    encounter_01.json          ← 조우 적 목록
```

---

## 10. Implementation Notes

```
변경 파일:
  scripts/data/UnitData.gd       신규 — 공통 베이스
  scripts/data/MercenaryData.gd  수정 — UnitData 상속으로 리팩토링
  scripts/data/EnemyData.gd      신규 — UnitData 상속
  scripts/combat/CombatUnit.gd   수정 — UnitData 기반 스탯 계산
  scripts/data/DataLoader.gd     수정 — 아래 항목 처리

DataLoader.gd 수정 항목:
  1. class_config 로딩 + stat_overrides / class_overrides 적용
  2. int → int_stat 명시적 키 매핑
  3. extra_passives[] 배열 로딩

유의사항:
  기존 MercenaryData 필드 중 UnitData로 이동하는 항목 확인 필요
  CombatUnit이 UnitData 타입을 직접 받도록 수정
  (MercenaryData / EnemyData 모두 UnitData 상속이므로 호환)
```
