# Open-Meteo API Integration Design

**Date:** 2026-03-19
**Project:** Water Watch — 수질·침수 통합 모니터링
**Scope:** Replace simulated rainfall data in `index.html` with live Open-Meteo precipitation data

---

## Goal

Connect the existing Water Watch dashboard to the Open-Meteo free weather API so that rainfall data shown in the flood risk tab reflects real precipitation rather than a Math.sin simulation.

---

## Data Layer

### Basin Coordinates

Each of the 5 Korean river basins maps to a representative city lat/lon:

| Basin | City | Latitude | Longitude |
|-------|------|----------|-----------|
| 한강 | Seoul | 37.57 | 126.98 |
| 낙동강 | Busan | 35.18 | 129.08 |
| 금강 | Daejeon | 36.35 | 127.38 |
| 영산강 | Gwangju | 35.16 | 126.85 |
| 섬진강 | Suncheon | 34.95 | 127.49 |

### API Endpoint

```
https://api.open-meteo.com/v1/forecast
  ?latitude={lat}
  &longitude={lon}
  &hourly=precipitation
  &past_hours=6
  &forecast_days=1
  &timezone=Asia/Seoul
```

No API key required. Returns `hourly.time[]` and `hourly.precipitation[]`.

### Fetch Strategy

- **Timing:** On page load, fire 5 parallel `fetch()` calls using `Promise.all()`
- **Cache:** Store results in a module-level `rainCache` object:
  ```js
  rainCache = {
    '한강': { times: [...], values: [...], currentMmHr: 2.1 },
    ...
  }
  ```
- **Current hour:** Find the index in `hourly.time[]` matching the current hour; extract `precipitation[index]` as `currentMmHr`
- **Chart slice:** Take indices `[currentIndex - 6 ... currentIndex + 6]` (12 values total)
- **Error handling:** If any basin fetch fails, log to console and leave that basin's cache entry null; `renderRainBarChart()` falls back to the existing Math.sin simulation for null entries

---

## UI Integration

### Rain Bar Chart (`renderRainBarChart`)

- Reads from `rainCache[currentBasin]` when available
- Uses real `times[]` labels (e.g., `03h`, `09h`) for the x-axis
- Falls back to Math.sin simulation if cache entry is null

### Rainfall Slider Auto-Set

- On SCAN, set `rainSim = rainCache[currentBasin].currentMmHr` (rounded to integer)
- Update slider DOM value and label display
- User can still drag the slider to override for scenario simulation

### Current Rainfall Stat Card (`sc-rain`)

- Display `rainCache[currentBasin].currentMmHr` on SCAN instead of the slider default

### Loading State

- While `Promise.all()` is pending, set the live dot (`live-dot`) color to orange
- On completion (success or failure), restore to green

### API Note in Sidebar

- Update the ※ note from "데모: 공공 기준값 기반 시뮬레이션" to "Open-Meteo 실시간 강수량 연동 · 슬라이더로 시나리오 조정 가능"

---

## Constraints

- No backend, no API key — Open-Meteo is entirely free and CORS-enabled
- All changes stay within `index.html` (single file)
- Water quality data (`WATER_DATA`) remains hardcoded — out of scope for this change
- Flood zone data (`FLOOD_ZONES_BY_BASIN`) remains hardcoded — out of scope

---

## Files Changed

| File | Change |
|------|--------|
| `index.html` | Add `BASIN_COORDS`, `rainCache`, `loadAllRainData()`, update `runScan()`, `renderRainBarChart()`, loading state |
