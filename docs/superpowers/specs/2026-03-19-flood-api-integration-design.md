# Open-Meteo Flood API Integration Design

**Date:** 2026-03-19
**Project:** Water Watch — 수질·침수 통합 모니터링
**Scope:** Replace hardcoded `FLOOD_ZONES_BY_BASIN` simulation with real river discharge data from Open-Meteo Flood API

---

## Goal

Replace the 6-zone hardcoded flood risk list per basin with a real 7-day river discharge forecast from the Open-Meteo Flood API (GloFAS-based, free, no API key required). The zone list panel is replaced with a discharge bar chart and a risk summary card.

---

## Data Layer

### API Endpoint

```
https://flood-api.open-meteo.com/v1/flood
  ?latitude={lat}
  &longitude={lon}
  &daily=river_discharge,river_discharge_mean
  &forecast_days=7
```

- No API key required
- Returns `daily.time[]` (YYYY-MM-DD strings) and `daily.river_discharge[]` (m³/s, 7 values) and `daily.river_discharge_mean[]` (30-year historical mean, same length)
- Basin coordinates reuse `BASIN_COORDS` already defined for rain fetch

### Risk Mapping

Flood risk is derived from the ratio of today's discharge to the 30-year historical mean:

| Ratio (today ÷ mean) | Label | Colour | Approx. Return Period |
|----------------------|-------|--------|-----------------------|
| < 1.0× | 안전 | `#00d97e` (green) | Normal conditions |
| 1.0–1.5× | 관찰 | `#f5c400` (yellow) | ~2-year flood |
| 1.5–2.5× | 주의 | `#ff8c42` (orange) | ~5-year flood |
| > 2.5× | 경보 | `#ff3b5c` (red) | ~20-year flood |

Today's discharge = `river_discharge[0]` (first element of the 7-day forecast).
Mean = `river_discharge_mean[0]` (representative value; use index 0).

### Cache Structure

```js
const floodCache = {};
// Populated on load:
floodCache['한강'] = {
  dates: ['2026-03-19', ...],      // 7 date strings
  discharge: [142, 138, 151, ...], // 7 daily discharge values (m³/s)
  mean: 79,                        // 30-year historical mean (m³/s)
  ratio: 1.80,                     // discharge[0] / mean
  label: '주의',
  color: '#ff8c42'
};
```

If `mean === 0` or missing, set `ratio = 1.0` (treat as normal) to avoid division by zero.

### Fetch Strategy

Flood data is fetched in parallel with rain data on page load using the same `Promise.all()` pattern. Each basin fetch uses `AbortController` with a **10-second timeout**. On failure, `floodCache[basin] = null`.

```js
async function loadAllFloodData() {
  await Promise.all(
    Object.entries(BASIN_COORDS).map(([basin, {lat, lon}]) =>
      fetchBasinFlood(basin, lat, lon)
    )
  );
}
```

### Error Handling

- If `floodCache[basin]` is `null` → sidebar risk card omits the 하천수위 factor row; discharge chart shows a "데이터 없음" message
- No user-visible error — console warning only

---

## UI Changes

### Zone List Panel — Replaced

The existing zone list panel (`<div class="zone-list">`) and its title are replaced with two new components stacked vertically:

#### 1. 7-day Discharge Bar Chart

- Title: `// 7일 하천 유량 예보 (Open-Meteo GloFAS)`
- 7 bars, one per day
- Bar height proportional to discharge value (max of 7 values = 100% height)
- Bar colour: risk colour for that day's ratio vs mean
- Today's bar: slightly brighter / outlined to distinguish it
- X-axis labels: `오늘`, `내일`, `+2일`, `+3일`, `+4일`, `+5일`, `+6일`
- Y-axis: no axis line needed; show discharge value (rounded integer m³/s) above each bar

#### 2. Risk Summary Card

Displayed below the chart:

```
현재 유량   142 m³/s
평년 대비   1.8× ── 주의  [orange badge]
평년 유량   79 m³/s  (30년 평균)
```

- Three rows, monospace font, consistent with existing panel style
- Badge colour matches risk label colour

### Stat Cards Updated

The third stat card currently shows "침수 위험 지점 개소". Replace with:

- **Key:** `하천 위험 등급`
- **Value:** risk label text (안전 / 관찰 / 주의 / 경보) in risk colour
- **Sub-label:** `GloFAS 예보 기준`

### Sidebar Risk Card Updated

The sidebar risk card currently has three factor rows: 강수량, 관로노후, 관로부하.

Replace `관로부하` row with `하천수위`:

```js
{k:'하천수위', v: Math.min(Math.round(ratio * 40), 100), color:'#00e5c0'}
```

The overall risk percentage formula is updated:
```js
const discharge_factor = Math.min(ratio / 3, 1); // normalise: ratio=3 → 100%
const risk_pct = Math.min(
  Math.round((discharge_factor * 0.5 + rain_factor * 0.35 + age_factor * 0.15) * 100),
  99
);
```

- `discharge_factor` replaces `base_risk` (which was derived from hardcoded capacity_ratio)
- If `floodCache[basin]` is null, fall back to `discharge_factor = 0.5` (neutral)

### Removed Entirely

| Item | Location |
|------|----------|
| `FLOOD_ZONES_BY_BASIN` object | JS constants block |
| `renderZoneList()` function | JS functions |
| `<div class="zone-list">` HTML | flood-main section |
| `.zone-row`, `.zone-rank`, `.zone-name`, `.zone-bar`, `.zone-risk`, `.zone-level` CSS | `<style>` block |

---

## Constraints

- No API key required — Open-Meteo Flood API is free and CORS-enabled
- All changes stay within `index.html` (single file)
- Water quality data (`WATER_DATA`) remains hardcoded — out of scope
- Rainfall fetch (`loadAllRainData`) unchanged — flood fetch runs alongside it
- `BASIN_COORDS` object reused from rain integration — no duplication

---

## Files Changed

| File | Change |
|------|--------|
| `index.html` | Add `floodCache`, `fetchBasinFlood()`, `loadAllFloodData()`, `renderDischargeChart()`, `renderRiskSummaryCard()`, update `renderRiskCard()`, update `runScan()`, remove zone list code and CSS |
