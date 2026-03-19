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

> **Note:** Basin coordinates represent a single representative point. Precipitation gradients across the basin are not captured. Acceptable for demo purposes. 섬진강 in particular spans a ~40km corridor (Gurye → Suncheon); Suncheon may underrepresent upstream events.

### API Endpoint

```
https://api.open-meteo.com/v1/forecast
  ?latitude={lat}
  &longitude={lon}
  &hourly=precipitation
  &past_hours=6
  &forecast_days=2
  &timezone=Asia/Seoul
```

- `past_hours=6` — returns 6 completed hours before now
- `forecast_days=2` — ensures full 6-hour lookahead even when called late in the day (e.g., 22:00 KST)
- No API key required. Open-Meteo is free and CORS-enabled.

Returns `hourly.time[]` (ISO 8601, e.g. `"2026-03-19T14:00"`, KST, no timezone suffix) and `hourly.precipitation[]` (mm, sum of preceding hour).

### Current Hour Index Matching

Open-Meteo returns times in local format (`YYYY-MM-DDTHH:00`) matching the requested timezone. Match using local wall-clock time, not UTC:

```js
const now = new Date();
const pad = n => String(n).padStart(2, '0');
const localHourStr = `${now.getFullYear()}-${pad(now.getMonth()+1)}-${pad(now.getDate())}T${pad(now.getHours())}:00`;
const currentIndex = times.findIndex(t => t === localHourStr);
```

> **Important:** Do NOT use `new Date().toISOString()` — that returns UTC and will never match KST-formatted time strings, silently breaking the index lookup.

### Current Index Guard

If `findIndex` returns `-1` (API clock skew, lag, or timezone mismatch), treat the basin as a failed fetch:

```js
if (currentIndex === -1) {
  rainCache[basin] = null; // triggers simulation fallback
  return;
}
```

This prevents a silent bad slice from `values.slice(-7, 5)` which would silently render wrong data.

> **Note:** The local hour string construction uses `getHours()` (browser local time). The dashboard assumes the user's browser is set to KST (Asia/Seoul). In non-KST environments, `currentIndex` will consistently be `-1` and all basins will fall back to simulation — a safe and visible degradation.

### Chart Slice

Take 12 values centered on `currentIndex`:

```js
const startIndex = Math.max(0, currentIndex - 6);
const slice = values.slice(startIndex, startIndex + 12);
const timeSlice = times.slice(startIndex, startIndex + 12);
```

- **Clamping:** If `currentIndex < 6` (first 6 hours of day), start from index 0. Fewer than 6 past bars will be shown — this is correct and honest behavior.
- Labels use the hour from the time string: `t.slice(11, 13) + 'h'` (e.g., `"03h"`)

### Cache Structure

```js
const rainCache = {};
// Populated on load:
rainCache['한강'] = {
  times: [...],        // full hourly time strings from API
  values: [...],       // full hourly precipitation values (mm)
  currentMmHr: 2.1,   // precipitation[currentIndex] — last completed hour
  currentIndex: 14     // index of current hour in arrays
};
```

> **Semantic note:** `currentMmHr` is the rainfall sum for the *last completed hour* (e.g., 13:00–14:00), not a live real-time reading. Open-Meteo precipitation is an hourly aggregate. The UI label will reflect this.

### Fetch Strategy

```js
async function loadAllRainData() {
  setLiveDotColor('orange');
  const entries = await Promise.all(
    Object.entries(BASIN_COORDS).map(([basin, {lat, lon}]) =>
      fetchBasinRain(basin, lat, lon)
    )
  );
  setLiveDotColor('green');
}
```

Each `fetchBasinRain()` uses `AbortController` with a **10-second timeout**:

```js
const controller = new AbortController();
const timer = setTimeout(() => controller.abort(), 10000);
try {
  const res = await fetch(url, { signal: controller.signal });
  // parse and store in rainCache[basin]
} catch (e) {
  console.warn(`Rain fetch failed for ${basin}:`, e.message);
  rainCache[basin] = null; // triggers fallback to simulation
} finally {
  clearTimeout(timer);
}
```

On timeout or any error, `rainCache[basin]` is set to `null` and the live-dot is restored to green after all fetches settle.

### Error Handling

- If `rainCache[basin]` is `null` → `renderRainBarChart()` falls back to existing Math.sin simulation silently
- Failures are logged to console only — no user-visible error message

---

## UI Integration

### Rain Bar Chart (`renderRainBarChart`)

- When `rainCache[currentBasin]` is non-null: use real data slice (12 bars, clamped as above)
- When null: fall back to Math.sin simulation (existing behavior)
- X-axis labels: real hour strings from `timeSlice` (e.g., `"08h"`, `"09h"`)
- Color thresholds unchanged: ≥30mm danger, ≥15mm warn, ≥5mm blue, else muted

### Rainfall Slider Auto-Set

- On SCAN, if cache available: `rainSim = Math.round(rainCache[currentBasin].currentMmHr)`
- Update slider DOM value: `document.getElementById('rain-sim').value = rainSim`
- Update label: `document.getElementById('rain-val').textContent = rainSim + ' mm/h ★'`
- The `★` suffix signals to the user that this value was auto-set from real data
- User can drag slider to any value; removing `★` is not required (it will update naturally via `oninput`)

### Current Rainfall Stat Card (`sc-rain`)

- On SCAN: display `rainCache[currentBasin].currentMmHr.toFixed(1)` mm/hr
- Card sub-label updated to: `mm/hr (전 1시간)` — "last 1 hour" — to accurately reflect the data semantic

### Loading State

- Before `Promise.all()` resolves: live-dot CSS color set to `var(--warn)` (orange)
- After all fetches settle (success or timeout): restored to `var(--ok)` (green)

### API Note in Sidebar

Update the `※` note to:

```
강수량: Open-Meteo 실시간 연동 (전 1시간 집계)
수질·침수지점: 공공 기준값 시뮬레이션
```

This retains the simulation disclaimer for non-live data while advertising the live rainfall connection.

---

## Constraints

- No backend, no API key — Open-Meteo is entirely free and CORS-enabled
- All changes stay within `index.html` (single file)
- Water quality data (`WATER_DATA`) remains hardcoded — out of scope
- Flood zone data (`FLOOD_ZONES_BY_BASIN`) remains hardcoded — out of scope

---

## Files Changed

| File | Change |
|------|--------|
| `index.html` | Add `BASIN_COORDS`, `rainCache`, `loadAllRainData()`, `fetchBasinRain()`, update `runScan()`, `renderRainBarChart()`, loading state, sidebar note, stat card label |
