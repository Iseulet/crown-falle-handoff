# CrownFalle - Game Concept Document
> 🇰🇷 크라운폴 - 게임 컨셉 문서

## Document Info
> 🇰🇷 문서 정보

- Created: 2026-03-06
- Status: Active Reference Document
- Purpose: Game identity and design direction guide for all agents
> 🇰🇷 생성일: 2026-03-06
> 🇰🇷 상태: 활성 참조 문서
> 🇰🇷 목적: 모든 에이전트의 게임 정체성 및 설계 방향 기준

---

## Game Overview
> 🇰🇷 게임 개요

- Title: CrownFalle
- Genre: Turn-based Tactical RPG
- Reference: Wartales (Shiro Games)
- Perspective: 2D Isometric (Quarterview)
- Engine: Godot 4.6.1 / GDScript only (no C#)
- Scale: Small-scale prototype
> 🇰🇷 제목: 크라운폴
> 🇰🇷 장르: 턴제 전술 RPG
> 🇰🇷 레퍼런스: 워테일즈 (Shiro Games)
> 🇰🇷 시점: 2D 아이소메트릭 (쿼터뷰)
> 🇰🇷 엔진: Godot 4.6.1 / GDScript 전용 (C# 사용 금지)
> 🇰🇷 규모: 소규모 프로토타입

---

## World Setting
> 🇰🇷 세계관 설정

### Background
> 🇰🇷 배경

A medieval dark fantasy world where kingdoms wage endless war.
Magic exists but is rare and considered dangerous.
> 🇰🇷 왕국들이 끊임없이 전쟁을 벌이는 중세 다크 판타지 세계
> 🇰🇷 마법은 존재하지만 희귀하고 위험한 것으로 여겨짐

### Atmosphere
> 🇰🇷 분위기

Dark and survival-focused.
The world cannot be trusted — contracts are taken for money,
but betrayal and conspiracy lurk everywhere.
> 🇰🇷 암울하고 생존 중심
> 🇰🇷 신뢰할 수 없는 세계
> 🇰🇷 의뢰는 돈을 위해 받지만 배신과 음모가 도처에 도사림

### Player Identity
> 🇰🇷 플레이어 정체성

Deserters from "Écorcheur" — the most feared mercenary company
on the continent. The player leads a small group who defied
their commander's orders and fled.
Now hunted, they survive by rebuilding a new mercenary band.
> 🇰🇷 대륙 최강 용병단 "에코르셔(Écorcheur)"의 탈주자들
> 🇰🇷 단장의 명령에 반기를 들고 도망친 소수의 용병들
> 🇰🇷 쫓기는 신세지만 생존하며 새 용병단을 재건하려 함
> 🇰🇷 Écorcheur = 중세 프랑스어로 "약탈자/가죽 벗기는 자"

---

## Core Concept
> 🇰🇷 핵심 컨셉

Player leads a small band of deserters exploring a dangerous world,
taking contracts to survive, fighting enemies,
and managing scarce resources — all while being hunted by Écorcheur.
> 🇰🇷 플레이어는 탈주 용병단을 이끌고 위험한 세계를 탐험
> 🇰🇷 생존을 위해 의뢰를 수행하고, 전투하고, 자원을 관리
> 🇰🇷 에코르셔에게 쫓기는 긴장감 속에서 진행

---

## Core Game Loop
> 🇰🇷 핵심 게임 루프

```
World Map Exploration
→ Encounter (Enemy / NPC / Location)
→ Combat (Turn-based tactical)
→ Camp (Rest / Manage resources)
→ Repeat
```

> 🇰🇷 월드맵 탐험
> 🇰🇷 → 조우 (적군 / NPC / 지점)
> 🇰🇷 → 전투 (턴제 전술)
> 🇰🇷 → 캠프 (휴식 / 자원 관리)
> 🇰🇷 → 반복

---

## Classes (Prototype)
> 🇰🇷 직업 (프로토타입)

### Fighter (전사)
- HP: High / STR: High / MV: Low
- Role: Frontline melee, high survivability
- Special: Absorb damage for allies
> 🇰🇷 HP: 높음 / STR: 높음 / MV: 낮음
> 🇰🇷 역할: 근접 전투, 높은 생존력
> 🇰🇷 특징: 아군 방어 / 피해 흡수

### Archer (궁수)
- HP: Medium / STR: Medium / MV: Medium
- Role: Ranged attack (2~3 tiles range)
- Special: Weak in close combat
> 🇰🇷 HP: 중간 / STR: 중간 / MV: 중간
> 🇰🇷 역할: 원거리 공격 (2~3칸)
> 🇰🇷 특징: 근접 전투 약함

### Rogue (도적)
- HP: Low / STR: Medium / MV: High
- Role: Fast flanker
- Special: Backstab — bonus damage when attacking from behind
> 🇰🇷 HP: 낮음 / STR: 중간 / MV: 높음
> 🇰🇷 역할: 빠른 측면 공격
> 🇰🇷 특징: 배후 공격 시 추가 데미지 (백스탭)

### Mage (마법사)
- HP: Low / STR: High / MV: Low
- Role: Ranged area attack
- Special: Costs MP per cast (simplified for prototype)
> 🇰🇷 HP: 낮음 / STR: 높음 / MV: 낮음
> 🇰🇷 역할: 원거리 광역 공격
> 🇰🇷 특징: 시전 시 MP 소모 (프로토타입에서 단순화)

---

## Combat Rules
> 🇰🇷 전투 규칙

### Victory Condition
> 🇰🇷 승리 조건

Eliminate all enemy units within the turn limit.
> 🇰🇷 제한 턴 내에 모든 적군 전멸

### Defeat Conditions
> 🇰🇷 패배 조건

- All player units eliminated, OR
- Turn limit exceeded
> 🇰🇷 아군 전멸, 또는
> 🇰🇷 제한 턴 초과

### Turn Limit
> 🇰🇷 턴 제한

Default: 10 turns per combat (adjustable per encounter)
> 🇰🇷 기본: 전투당 10턴 (조우별 조정 가능)

### Turn Order
> 🇰🇷 턴 순서

Ally units act first → Enemy units act second (alternating)
> 🇰🇷 아군 먼저 행동 → 적군 행동 (번갈아 진행)

---

## Resources (Prototype)
> 🇰🇷 자원 (프로토타입)

### Food (식량)
- Consumed at camp to recover fatigue
- Running out increases fatigue penalty
- Obtained from: quests, looting enemies, towns
> 🇰🇷 캠프에서 소모하여 피로도 회복
> 🇰🇷 식량 부족 시 피로도 페널티 증가
> 🇰🇷 획득처: 퀘스트 보상, 적 전리품, 마을 구매

### Gold (골드)
- Used to hire new mercenaries
- Used to purchase supplies (food) at towns
- Obtained from: quests, looting enemies
> 🇰🇷 새 용병 고용에 사용
> 🇰🇷 마을에서 식량 등 보급품 구매
> 🇰🇷 획득처: 퀘스트 보상, 적 전리품

---

## Technical Direction
> 🇰🇷 기술 방향

### Prototype Strategy
> 🇰🇷 프로토타입 전략

Validate all game logic with placeholder graphics first.
3D conversion possible after prototype is complete.
> 🇰🇷 플레이스홀더 그래픽으로 로직 먼저 검증
> 🇰🇷 프로토타입 완성 후 3D 전환 가능

### Tech Stack
> 🇰🇷 기술 스택

- Engine: Godot 4.6.1
- Language: GDScript only (no C#)
- Grid: TileMap node (isometric diamond, 128x64px)
- Pathfinding: AStar2D
- Character data: Godot Resource (.tres)
- Quest data: JSON
- Version control: Git / GitHub Private
> 🇰🇷 언어: GDScript 전용 (C# 사용 금지)
> 🇰🇷 격자: TileMap 사각형 격자
> 🇰🇷 경로탐색: AStar2D
> 🇰🇷 캐릭터 데이터: Godot Resource (.tres)
> 🇰🇷 퀘스트: JSON

---

## Design Principles
> 🇰🇷 설계 원칙

1. Logic first, visuals later
   Validate all systems with placeholder graphics first.
   > 🇰🇷 로직 우선, 비주얼 나중

2. Small and completable
   Each feature must be independently testable.
   > 🇰🇷 각 기능은 독립적으로 테스트 가능해야 함

3. GDScript readability
   Write clean, readable GDScript for AI collaboration.
   > 🇰🇷 AI 협업을 위해 깔끔하고 읽기 쉬운 GDScript 작성

---

## Out of Scope (Prototype)
> 🇰🇷 프로토타입 범위 외

- 3D graphics
- Online multiplayer
- Full story / narrative
- Advanced enemy AI
- Full sound design
- Advanced magic system (MP simplified for prototype)
> 🇰🇷 3D 그래픽
> 🇰🇷 온라인 멀티플레이어
> 🇰🇷 완전한 스토리/내러티브
> 🇰🇷 고급 적 AI
> 🇰🇷 완전한 사운드 디자인
> 🇰🇷 마법 고급 시스템 (프로토타입에서 단순화)

---

## Reference
> 🇰🇷 참조

- Wartales (Shiro Games) — core gameplay reference
- Roadmap: handoff/plans/design/2026-03-06-crownfalle-roadmap.md
- Grid system analysis: handoff/plans/design/2026-03-06-wartales-grid-analysis.md
> 🇰🇷 워테일즈 (Shiro Games) - 핵심 게임플레이 레퍼런스
> 🇰🇷 로드맵: handoff/plans/design/2026-03-06-crownfalle-roadmap.md
> 🇰🇷 그리드 시스템 분석: handoff/plans/design/2026-03-06-wartales-grid-analysis.md
