# Open-Meteo Flood API Integration Design

**Date:** 2026-03-19
**Project:** Water Watch — 수질·침수 통합 모니터링
**Scope:** Replace hardcoded `FLOOD_ZONES_BY_BASIN` simulation with real river discharge data from Open-Meteo Flood API

---

## Goal

Replace the 6-zone hardcoded flood risk list per basin with a real 7-day river discharge forecast from the Open-Meteo Flood API (GloFAS-based, free, no API key required). The zone list panel is replaced with a discharge bar chart and a risk summary card.

---

## Data Layer

### API Calls Per Basin (2 fetches, run in parallel)

#### Fetch 1 — 7-day Forecast

```
https://flood-api.open-meteo.com/v1/flood
  ?latitude={lat}&longitude={lon}
  &daily=river_discharge
  &forecast_days=7
```

Returns `daily.time[]` (7 YYYY-MM-DD strings) and `daily.river_discharge[]` (7 m³/s values).

> **Note:** `river_discharge_mean` from this endpoint is the mean across 50 GloFAS ensemble members for the same forecast window — NOT a historical climatological baseline. It is not used in this design.

#### Fetch 2 — 30-day Historical Baseline

```
https://flood-api.open-meteo.com/v1/flood
  ?latitude={lat}&longitude={lon}
  &daily=river_discharge
  &start_date={30 days ago}
  &end_date={yesterday}
```

Date construction (run at fetch time):
```js
const today = new Date();
const yesterday = new Date(today); yesterday.setDate(today.getDate() - 1);
const thirtyDaysAgo = new Date(today); thirtyDaysAgo.setDate(today.getDate() - 30);
const fmt = d => d.toISOString().slice(0, 10); // YYYY-MM-DD
// Use fmt(thirtyDaysAgo) and fmt(yesterday) in the URL
```

Returns 30 daily discharge values. Compute `historicalMean = average(values)`.

- Both fetches use `AbortController` with a **10-second timeout** per fetch.
- Basin coordinates reuse `BASIN_COORDS` already defined for rain fetch.

### Risk Mapping

Flood risk is derived from the ratio: `today's discharge ÷ 30-day historical mean`.

| Ratio | Label | Colour | Interpretation |
|-------|-------|--------|----------------|
| < 1.0× | 안전 | `#00d97e` | Below recent normal |
| 1.0–1.5× | 관찰 | `#f5c400` | Above recent normal |
| 1.5–2.5× | 주의 | `#ff8c42` | Significantly elevated |
| > 2.5× | 경보 | `#ff3b5c` | Major flood conditions |

Today's discharge = `forecastDischarge[0]`.
`historicalMean` = arithmetic mean of the 30 historical values.

**Guard against zero/null mean:**
```js
if (!historicalMean || historicalMean === 0) {
  floodCache[basin] = null; // trigger fallback
  return;
}
```

### Cache Structure

```js
const floodCache = {};
floodCache['한강'] = {
  dates: ['2026-03-19', ...],          // 7 forecast date strings
  discharge: [142, 138, 151, ...],     // 7 daily forecast values (m³/s)
  historicalMean: 79,                  // arithmetic mean of past 30 days (m³/s)
  ratio: 1.80,                         // discharge[0] / historicalMean
  label: '주의',
  color: '#ff8c42'
};
```

### Per-Day Bar Color

Each of the 7 bars is colored by its own day's ratio:
```js
const dayRatio = (discharge[i] ?? 0) / historicalMean;
```
- `discharge[i] ?? 0` guards against partial API responses (fewer than 7 days returned)
- Same risk mapping table applied per bar

### Fetch Strategy

All flood fetches run silently alongside rain fetches in the same `loadAllData()` call on page load. The live-dot color lifecycle is managed solely by the rain fetch (`loadAllRainData`); flood fetches do not modify dot state.

```js
async function loadAllData() {
  setLiveDotColor('orange');
  await Promise.all([
    loadAllRainData(),   // manages live-dot internally
    loadAllFloodData()   // silent — no dot state change
  ]);
}
```

### Error Handling

- If either fetch (forecast or historical) fails for a basin → `floodCache[basin] = null`
- Null cache: discharge chart shows `데이터 없음` message; sidebar risk card omits 하천수위 factor row; fallback `discharge_factor = 0.5` used in risk formula
- All errors logged to console only — no user-visible error message

---

## UI Changes

### Zone List Panel — Replaced

The `<div class="zone-list">` element and its title div are replaced with two stacked components inside `.flood-main`:

#### 1. 7-day Discharge Bar Chart

```html
<div class="discharge-chart-wrap">
  <div class="chart-title">// 7일 하천 유량 예보 (Open-Meteo GloFAS)</div>
  <div class="bar-chart" id="discharge-bar-chart"></div>
</div>
```

- 7 bars, one per day
- Bar height proportional to discharge value (max of 7 values = 100% height)
- Bar colour: risk colour for that day's ratio vs `historicalMean`
- Today's bar (index 0): `border: 1px solid var(--c1)` to distinguish it
- X-axis labels: `오늘`, `내일`, `+2일`, `+3일`, `+4일`, `+5일`, `+6일`
- Value label above each bar: `${Math.round(discharge[i])} m³/s` (omit if `discharge[i]` is null/undefined)
- If `floodCache[basin]` is null: render a single centred message div `데이터 없음`

#### 2. Risk Summary Card

```html
<div class="discharge-summary" id="discharge-summary"></div>
```

Rendered content:
```
현재 유량   142 m³/s
평년 대비   1.8×  ── 주의  [orange badge]
최근 30일 평균   79 m³/s
```

- Three rows, monospace font, consistent with `.metric` panel style
- Badge: `<span style="background:${color};color:#070b12;padding:2px 8px">${label}</span>`

### Stat Cards Updated

Third stat card (`sc-zone`): replace "침수 위험 지점 개소" content:
- **Key:** `하천 위험 등급`
- **Value (`sc-zone`):** risk label (안전 / 관찰 / 주의 / 경보) in risk colour
- **Sub-label:** `GloFAS 예보 기준`

On SCAN, set via:
```js
document.getElementById('sc-zone').textContent = floodCache[basin]?.label ?? '--';
document.getElementById('sc-zone').style.color = floodCache[basin]?.color ?? 'var(--c1)';
```

### Sidebar Risk Card Updated

Replace `관로부하` factor row with `하천수위`. Unify bar scale with formula:

```js
const discharge_factor = floodCache[currentBasin]
  ? Math.min(floodCache[currentBasin].ratio / 3, 1)
  : 0.5; // neutral fallback

const factors = [
  {k:'강수량',  v: Math.round(rain_factor * 100),       color:'#3b9eff'},
  {k:'관로노후', v: ageSim,                              color:'#ff8c42'},
  {k:'하천수위', v: Math.round(discharge_factor * 100), color:'#00e5c0'},
];
```

**Bar display and formula now use the same scale** (`discharge_factor * 100`).

Overall risk formula updated:
```js
const risk_pct = Math.min(
  Math.round((discharge_factor * 0.5 + rain_factor * 0.35 + age_factor * 0.15) * 100),
  99
);
```

### Functions Updated (in addition to new functions)

| Function | Required Change |
|----------|----------------|
| `renderAlerts()` | Replace `FLOOD_ZONES_BY_BASIN[currentBasin]` reference with `floodCache[currentBasin]?.ratio`; derive high-risk flag from ratio ≥ 1.5 |
| `updateAge()` | Remove `renderZoneList()` call; replace with `renderDischargeChart()` and `renderRiskSummaryCard()` |
| `renderRiskCard()` | Replace `capacity_ratio`-based `base_risk` with `discharge_factor` from `floodCache` |
| `runScan()` | Call `renderDischargeChart()` and `renderRiskSummaryCard()` instead of `renderZoneList()` |

### Sidebar API Note Updated

Add GloFAS to the `// 연동 API` block:
```
■ Open-Meteo Flood API (GloFAS)
    flood-api.open-meteo.com
```

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
- Rainfall fetch (`loadAllRainData`) unchanged in logic; wrapped in shared `loadAllData()`
- `BASIN_COORDS` object reused — no duplication

---

## Files Changed

| File | Change |
|------|--------|
| `index.html` | Add `floodCache`, `fetchBasinFlood()`, `loadAllFloodData()`, `loadAllData()`, `renderDischargeChart()`, `renderRiskSummaryCard()`, update `renderRiskCard()`, `renderAlerts()`, `updateAge()`, `runScan()`, remove zone list code and CSS, update sidebar API note |
