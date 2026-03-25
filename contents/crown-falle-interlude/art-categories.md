# Image Categories
> 🇰🇷 이미지 카테고리 전체 목록

Game 4-Layer loop (Town/Camp -> WorldMap -> Battle -> Roster) and 2D visual categories.
Each category belongs to a **convention group** with independent style/resolution/tool settings.

| Category | Description | Convention Group | Style Direction | Status |
|----------|-------------|-----------------|----------------|--------|
| **characters** | Portraits (`portrait`), standing composites (`standing_composite`), fullbody/scene illustrations | `narrative` | Dark Oil Renaissance, Caravaggio chiaroscuro | Active |
| **scenes** | Story event CGs | `narrative` | Dark Oil Renaissance, Caravaggio chiaroscuro | Active |
| **backgrounds** | Environment landscapes (no characters) | `narrative` | Dark Oil Renaissance (same tone) | Future |
| **creatures** | Creatures/monsters | `narrative` | Dark Oil Renaissance (same tone) | Future |
| **maps** | World map, region maps | `worldmap` | Point-click, parchment/map style | Future |
| **ingame** | In-game character sprites | `ingame` | Pixel/sprite/SD (TBD) | Future |
| **items** | Items/equipment standalone | `ingame` (tentative) | TBD | Future |
| **ui** | UI assets — icons, frames | `ui` | Vector/flat/medieval ornament (TBD) | Future |
| **promo** | Key visuals, promotional | `narrative` | Dark Oil Renaissance (high-res) | Future |

> 🇰🇷 새 카테고리 추가 시 _config.json의 category_convention_map에도 등록 필요.

---

## 파일 네이밍 컨벤션

> 🇰🇷 `_favorites/` 저장 파일명 규칙

```
YYYY-MM-DD_{type}_{action}_{seq}.png
```

### `{type}` 목록

| type | 비율 | 용도 | proto 경로 |
|------|------|------|-----------|
| `portrait` | 1:1 | 프로필 이미지 | `assets/profiles/{charId}/{action}.png` |
| `standing_composite` | 9:16 | 다이얼로그 스탠딩 (grey bg) | `assets/standings/{charId}/{action}.png` |
| `fullbody` | 9:16 | 일러스트 전신 (배경 있음) | `assets/illustrations/{charId}/{variant}.png` |
| `scene` | 16:9 | 일러스트 장면 | `assets/illustrations/{charId}/{variant}.png` |
| `bg` | 16:9 | 배경 | `assets/bg/{locationId}_{variant}.png` |

### `{action}` 규칙

- `portrait`, `standing_composite` 타입에만 적용
- `art-config.json` → `actions.registry` 키만 사용
- `fullbody`, `scene`, `bg` 타입은 용도별 자유 서술 variant 사용 (`field`, `campfire` 등)

### `{seq}`

- `001`, `002`, ... — 동일 type+action의 복수 버전
- proto에는 최신 버전 1개만 사용
