# 에이전트 파이프라인 v3 — 워크플로우 다이어그램

> 옵시디언에서 열면 Mermaid 다이어그램이 자동 렌더링됩니다.

---

## 전체 파이프라인

```mermaid
flowchart TD
    subgraph P0["Phase 0 — 발상 (Ideation)"]
        direction LR
        WALDO["왈도 (Dreamer)<br/>아이디어 발산"]
        HIST_I["히스 (Historian)<br/>영감 모드"]
        WALDO -. "병렬" .- HIST_I
    end

    P0 -->|"작가 승인"| P1

    subgraph P1["Phase I — 설계 (Design)"]
        direction LR
        FORGE["포지 (Forge)<br/>SSOT · 엔티티"]
        ARCH["아크 (Architect)<br/>Beat Sheet"]
        PACE_MA["페이서-매크로<br/>긴장 곡선 점검"]
        FORGE --> ARCH
        ARCH -. "병렬" .- PACE_MA
        HIST_V["히스 (고증 모드)"]
        FORGE -. "온콜" .-> HIST_V
    end

    P1 --> GATE1

    GATE1{"시스 gate check<br/>pending · 참조 · INDEX"}

    GATE1 -->|"통과"| P2

    subgraph P2["Phase II — 집필 (Writing)"]
        SCRIBE["제니 (Scribe)<br/>초안 집필"]
    end

    P2 --> GATE2

    GATE2{"시스 gate check<br/>초안 완성 · summary"}

    GATE2 -->|"통과"| P3

    subgraph P3["Phase III — 검증 (Verification)"]
        direction LR
        RED["레드 (Challenger)<br/>논리 비판 + 프로포잘"]
        KEEPER["키퍼 (Keeper)<br/>설정 정합성 + 프로포잘"]
        EXPAND["퍼스 (Expander)<br/>시점 확장"]
        RED -. "병렬" .- KEEPER
        KEEPER -. "병렬" .- EXPAND
    end

    P3 --> GATE3

    GATE3{"시스 gate check<br/>🔴 이슈 해소"}

    GATE3 -->|"통과"| P4

    subgraph P4["Phase IV — 감성 + 마무리 (Emotion + Polish)"]
        direction TB
        subgraph P4_PAR["병렬"]
            direction LR
            CARRY["캐리 (Carry)<br/>감정 + 말투 검증"]
            READER["리더 (Reader)<br/>독자 시뮬레이션"]
            CARRY -. "병렬" .- READER
        end
        subgraph P4_SEQ["순차"]
            direction LR
            PACE_MI["페이서-마이크로<br/>씬 내 리듬"] --> EDITOR["쿳시 (Editor)<br/>최종 교정"]
        end
        P4_PAR --- P4_SEQ
    end

    style P0 fill:#FBEAF0,stroke:#993556,stroke-width:1px
    style P1 fill:#E1F5EE,stroke:#0F6E56,stroke-width:1px
    style P2 fill:#EEEDFE,stroke:#534AB7,stroke-width:1px
    style P3 fill:#FAECE7,stroke:#993C1D,stroke-width:1px
    style P4 fill:#E6F1FB,stroke:#185FA5,stroke-width:1px
    style GATE1 fill:#F1EFE8,stroke:#5F5E5A,stroke-width:1px
    style GATE2 fill:#F1EFE8,stroke:#5F5E5A,stroke-width:1px
    style GATE3 fill:#F1EFE8,stroke:#5F5E5A,stroke-width:1px
```

---

## 히스 이중 모드

```mermaid
flowchart LR
    subgraph INSP["영감 모드"]
        direction TB
        I1["Phase 0"]
        I2["왈도와 병렬"]
        I3["역사적 유사 사례<br/>신화 모티프 제공"]
        I1 --> I2 --> I3
    end

    HIST["히스<br/>(Historian)"]

    subgraph VERI["고증 모드"]
        direction TB
        V1["Phase I"]
        V2["포지가 온콜 호출"]
        V3["설정 정합성 검증<br/>시대 고증"]
        V1 --> V2 --> V3
    end

    HIST -->|"MODE: INSPIRATION"| INSP
    HIST -->|"MODE: VERIFICATION"| VERI

    style INSP fill:#FAEEDA,stroke:#854F0B,stroke-width:1px
    style VERI fill:#FAEEDA,stroke:#854F0B,stroke-width:1px
    style HIST fill:#EF9F27,stroke:#854F0B,stroke-width:2px,color:#412402
```

---

## 축약 워크플로우

```mermaid
flowchart LR
    subgraph BD["브레인덤프 정리"]
        direction LR
        b1[시스] --> b2[포지] --> b3[키퍼]
    end

    subgraph WS["세계관 설정 추가"]
        direction LR
        w1[포지] --> w2["히스(고증)"] --> w3[키퍼]
    end

    subgraph NC["새 챕터 집필"]
        direction LR
        n1["왈도·히스(영감)"] --> n2["아크·페이서매크로"] --> n3[제니] --> n4["레드·키퍼·퍼스"] --> n5["캐리·리더"] --> n6["페이서마이크로·쿳시"]
    end

    subgraph EC["기존 챕터 수정"]
        direction LR
        e1[제니] --> e2[키퍼] --> e3[쿳시]
    end

    subgraph ID["아이디어 발산"]
        direction LR
        i1["왈도·히스(영감)"] --> i2[포지]
    end
```

---

## 에이전트 목록

| # | 에이전트 | 파일 | 그룹 | Phase |
|---|---------|------|------|-------|
| 1 | 포지 (World Forge) | `world_forge.md` | 설계 | I |
| 2 | 아크 (Architect) | `architect.md` | 설계 | I |
| 3 | 히스 (Historian) | `historian.md` | 설계 | 0 + I(온콜) |
| 4 | 제니 (Scribe) | `scribe.md` | 집필 | II |
| 5 | 페이서-매크로 | `pacemaker_macro.md` | 집필 | I |
| 6 | 페이서-마이크로 | `pacemaker_micro.md` | 집필 | IV |
| 7 | 쿳시 (Editor) | `editor.md` | 집필 | IV |
| 8 | 레드 (Challenger) | `challenger.md` | 검증 | III |
| 9 | 키퍼 (Keeper) | `keeper.md` | 검증 | III |
| 10 | 퍼스 (Expander) | `expander.md` | 검증 | III |
| 11 | 왈도 (Dreamer) | `dreamer.md` | 확장 | 0 |
| 12 | 캐리 (Carry) | `carry.md` | 감성 | IV |
| 13 | 리더 (Reader) | `reader.md` | 감성 | IV |
| — | 시스 (Systems) | `systems.md` | 인프라 | 전 Phase 게이트 |
