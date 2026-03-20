# Flood API Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace hardcoded `FLOOD_ZONES_BY_BASIN` flood zone simulation in `index.html` with real 7-day river discharge forecasts and 30-day historical baselines from the Open-Meteo Flood API.

**Architecture:** Single-file HTML with vanilla JS. Two parallel API fetches per basin (7-day forecast + 30-day historical). Risk ratio derived from forecast ÷ historical mean. Zone list panel replaced with discharge bar chart + risk summary card.

**Tech Stack:** Vanilla JS, Fetch API, AbortController, Open-Meteo Flood API (free, no key, CORS-enabled)

**Spec:** `docs/superpowers/specs/2026-03-19-flood-api-integration-design.md`

---

## File Map

| File | Action | What Changes |
|------|--------|--------------|
| `index.html` | Modify | All changes — CSS removals, HTML replacements, JS additions and updates |

---

## Task 1: Add `BASIN_COORDS`, `setLiveDotColor`, and `floodCache` Globals

**Files:**
- Modify: `index.html` (JS constants block, top of `<script>`)

- [ ] **Step 1: Add `BASIN_COORDS` constant**

At the top of the `<script>` block (after the `GRADE_CRITERIA` constant), add:

```js
const BASIN_COORDS = {
  '한강':   {lat: 37.57, lon: 126.98},
  '낙동강': {lat: 35.18, lon: 129.08},
  '금강':   {lat: 36.35, lon: 127.38},
  '영산강': {lat: 35.16, lon: 126.85},
  '섬진강': {lat: 34.95, lon: 127.49},
};
```

- [ ] **Step 2: Add `floodCache` variable**

After `let currentBasin = null; let rainSim = 12, ageSim = 45;`, add:

```js
const floodCache = {};
```

- [ ] **Step 3: Add `setLiveDotColor` helper**

After the `updateClock` function, add:

```js
function setLiveDotColor(state) {
  const dot = document.querySelector('.live-dot');
  if (!dot) return;
  dot.style.background = state === 'orange' ? 'var(--warn)' : 'var(--ok)';
}
```

- [ ] **Step 4: Verify no syntax errors**

Open `index.html` in browser. DevTools → Console. No errors on page load.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add BASIN_COORDS, floodCache, setLiveDotColor globals"
```

---

## Task 2: Remove Old Zone List CSS, Add New Discharge CSS

**Files:**
- Modify: `index.html` (`<style>` block)

- [ ] **Step 1: Remove zone-list CSS rules**

In the `<style>` block, find and delete ALL of these rules (including `.zone-row:last-child`):

```css
/* DELETE all of these */
.zone-list{...}
.zone-row{...}
.zone-row:last-child{border:none}
.zone-rank{...}
.zone-name{...}
.zone-bar{...}
.zone-bar-fill{...}
.zone-risk{...}
.zone-level{...}
```

- [ ] **Step 2: Add discharge panel CSS**

In the `<style>` block, add after the `.bar-chart` rule:

```css
/* 하천 유량 패널 */
.discharge-chart-wrap{background:var(--panel);border:1px solid var(--border);padding:16px;
  clip-path:polygon(5px 0%,100% 0%,calc(100% - 5px) 100%,0% 100%)}
.discharge-summary{background:var(--panel);border:1px solid var(--border);padding:16px;
  clip-path:polygon(5px 0%,100% 0%,calc(100% - 5px) 100%,0% 100%);
  display:flex;flex-direction:column;gap:10px}
.ds-row{display:flex;justify-content:space-between;align-items:center;font-size:12px}
.ds-key{font-family:'Space Mono',monospace;font-size:9px;color:var(--muted);letter-spacing:.5px}
.ds-val{font-family:'Space Mono',monospace;font-size:14px;font-weight:700}
.ds-badge{font-family:'Space Mono',monospace;font-size:10px;font-weight:700;
  padding:2px 8px;letter-spacing:.5px}
```

- [ ] **Step 3: Verify in browser**

Open `index.html`. DevTools → Console. Zero CSS errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "refactor: remove zone-list CSS, add discharge chart CSS"
```

---

## Task 3: Replace Zone List HTML with New Panel Elements

**Files:**
- Modify: `index.html` (flood-main section)

- [ ] **Step 1: Replace the zone-list div**

Find this block in `<!-- 침수 위험 뷰 -->`:

```html
<div class="zone-list">
  <div class="chart-title">// 위험 지점 순위 (관로 부하율 기준)</div>
  <div id="zone-list"></div>
</div>
```

Replace with:

```html
<div class="discharge-chart-wrap">
  <div class="chart-title">// 7일 하천 유량 예보 (Open-Meteo GloFAS)</div>
  <div class="bar-chart" id="discharge-bar-chart"></div>
</div>
<div class="discharge-summary" id="discharge-summary">
  <div style="font-family:'Space Mono',monospace;font-size:11px;color:var(--muted);text-align:center;padding:12px 0">
    수계를 선택하고 SCAN을 누르세요
  </div>
</div>
```

- [ ] **Step 2: Update the third stat card**

Find the third `.stat-card` in `.flood-header-cards`:

```html
<div class="stat-card">
  <div class="sc-k">침수 위험 지점</div>
  <div class="sc-v" id="sc-zone">--</div>
  <div class="sc-u">개소 (주의 이상)</div>
</div>
```

Replace with:

```html
<div class="stat-card">
  <div class="sc-k">하천 위험 등급</div>
  <div class="sc-v" id="sc-zone">--</div>
  <div class="sc-u">GloFAS 예보 기준</div>
</div>
```

- [ ] **Step 3: Update the second stat card label**

The second stat card currently shows "관로 부하율". After integration it shows a composite flood index. Update it:

```html
<!-- Find: -->
<div class="sc-k">관로 부하율</div>
...
<div class="sc-u">% (설계 용량 대비)</div>

<!-- Replace sub-label only: -->
<div class="sc-k">복합 부하 지수</div>
...
<div class="sc-u">% (하천·강수 기반)</div>
```

- [ ] **Step 4: Update the sidebar API note**

In the `.api-note` block, after the Open-Meteo rain paragraph, add:

```html
<p>■ Open-Meteo Flood API (GloFAS)<br>&nbsp;&nbsp;flood-api.open-meteo.com</p>
```

- [ ] **Step 5: Verify layout in browser**

Open `index.html`. Flood tab shows two placeholder panels. Third stat card shows "하천 위험 등급". No JS errors.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "refactor: replace zone-list HTML with discharge chart and summary panels"
```

---

## Task 4: Remove Old Zone List JS (`FLOOD_ZONES_BY_BASIN`, `renderZoneList`, `renderFloodDash`)

**Files:**
- Modify: `index.html` (JS block)

> Note: After this task, clicking SCAN will temporarily error until Task 7 updates the callers. Do not verify SCAN functionality until Task 7 is complete.

- [ ] **Step 1: Delete `FLOOD_ZONES_BY_BASIN` constant**

Find and delete the entire `const FLOOD_ZONES_BY_BASIN = { ... };` object.

- [ ] **Step 2: Delete `renderZoneList()` function**

Find and delete the entire `function renderZoneList() { ... }` function.

- [ ] **Step 3: Delete `renderFloodDash()` function**

Find and delete: `function renderFloodDash(){ renderRainBarChart(); renderZoneList(); }`

- [ ] **Step 4: Verify page loads without errors**

Open `index.html`. Page loads. Do NOT click SCAN yet. Console shows zero errors on load.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "refactor: remove FLOOD_ZONES_BY_BASIN, renderZoneList, renderFloodDash"
```

---

## Task 5: Add `getFloodRisk()`, `fetchBasinFlood()`, `loadAllFloodData()`

**Files:**
- Modify: `index.html` (JS functions)

- [ ] **Step 1: Add `getFloodRisk()` helper**

After the `getGrade()` function, add:

```js
function getFloodRisk(ratio) {
  if (ratio < 1.0) return {label:'안전', color:'#00d97e'};
  if (ratio < 1.5) return {label:'관찰', color:'#f5c400'};
  if (ratio < 2.5) return {label:'주의', color:'#ff8c42'};
  return {label:'경보', color:'#ff3b5c'};
}
```

- [ ] **Step 2: Add `fetchBasinFlood()` function**

Add after `getFloodRisk()`:

```js
async function fetchBasinFlood(basin, lat, lon) {
  const today = new Date();
  const yesterday = new Date(today); yesterday.setDate(today.getDate() - 1);
  const thirtyDaysAgo = new Date(today); thirtyDaysAgo.setDate(today.getDate() - 30);
  const fmt = d => d.toISOString().slice(0, 10);

  const forecastUrl = `https://flood-api.open-meteo.com/v1/flood?latitude=${lat}&longitude=${lon}&daily=river_discharge&forecast_days=7`;
  const histUrl = `https://flood-api.open-meteo.com/v1/flood?latitude=${lat}&longitude=${lon}&daily=river_discharge&start_date=${fmt(thirtyDaysAgo)}&end_date=${fmt(yesterday)}`;

  try {
    const ctrl = new AbortController();
    const timer = setTimeout(() => ctrl.abort(), 10000);
    const [fRes, hRes] = await Promise.all([
      fetch(forecastUrl, {signal: ctrl.signal}),
      fetch(histUrl, {signal: ctrl.signal})
    ]);
    clearTimeout(timer);

    const fData = await fRes.json();
    const hData = await hRes.json();

    const discharge = fData.daily.river_discharge;
    const dates = fData.daily.time;
    const histValues = hData.daily.river_discharge.filter(v => v !== null);
    const historicalMean = histValues.length
      ? histValues.reduce((a, b) => a + b, 0) / histValues.length
      : 0;

    if (!historicalMean || historicalMean === 0) {
      floodCache[basin] = null;
      return;
    }

    const ratio = (discharge[0] ?? 0) / historicalMean;
    const risk = getFloodRisk(ratio);

    floodCache[basin] = {
      dates,
      discharge,
      historicalMean: Math.round(historicalMean),
      ratio: Math.round(ratio * 100) / 100,
      label: risk.label,
      color: risk.color
    };
  } catch (e) {
    console.warn(`Flood fetch failed for ${basin}:`, e.message);
    floodCache[basin] = null;
  }
}
```

- [ ] **Step 3: Add `loadAllFloodData()` function**

```js
async function loadAllFloodData() {
  await Promise.all(
    Object.entries(BASIN_COORDS).map(([basin, {lat, lon}]) =>
      fetchBasinFlood(basin, lat, lon)
    )
  );
}
```

- [ ] **Step 4: Verify no syntax errors**

Open `index.html`. DevTools Console: zero errors on load.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add getFloodRisk, fetchBasinFlood, loadAllFloodData"
```

---

## Task 6: Wire `loadAllData()` on Page Load

**Files:**
- Modify: `index.html` (JS — bottom of script)

- [ ] **Step 1: Add `loadAllData()` function**

At the bottom of the script (just before or after `initWQGrid` IIFE), add:

```js
async function loadAllData() {
  setLiveDotColor('orange');
  await Promise.all([
    loadAllFloodData()
  ]);
  setLiveDotColor('green');
}
```

> Note: Rain fetching will be added in a future task (it is not yet implemented). For now, only `loadAllFloodData()` runs here.

- [ ] **Step 2: Replace the existing page-load call**

Find the current page-load call at the bottom of the script. It may be just the `initWQGrid` IIFE or nothing. Add after it:

```js
loadAllData();
```

- [ ] **Step 3: Verify in browser**

Open `index.html`. Live-dot pulses orange briefly, then returns to green. Network tab shows 10 API requests to `flood-api.open-meteo.com` (5 forecast + 5 historical). Console: zero errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: wire loadAllData on page load, fetch all basin flood data"
```

---

## Task 7: Add `renderDischargeChart()` and `renderRiskSummaryCard()`

**Files:**
- Modify: `index.html` (JS functions)

- [ ] **Step 1: Add `renderDischargeChart()` function**

Add after `renderRainBarChart()`:

```js
function renderDischargeChart() {
  const el = document.getElementById('discharge-bar-chart');
  if (!el) return;
  const data = floodCache[currentBasin];
  if (!data) {
    el.innerHTML = '<div style="font-family:\'Space Mono\',monospace;font-size:11px;color:var(--muted);text-align:center;width:100%;padding:24px">데이터 없음</div>';
    return;
  }
  const {discharge, historicalMean} = data;
  const dayLabels = ['오늘','내일','+2일','+3일','+4일','+5일','+6일'];
  const maxV = Math.max(...discharge.filter(v => v != null), 1);

  el.innerHTML = discharge.map((v, i) => {
    const val = v ?? 0;
    const pct = Math.round(val / maxV * 100);
    const dayRatio = val / historicalMean;
    const {color} = getFloodRisk(dayRatio);
    const todayStyle = i === 0 ? 'outline:1px solid var(--c1);' : '';
    return `<div class="bar-col">
      <div class="bar-fill-wrap">
        <div class="bar-fill" style="height:${pct}%;background:${color};width:100%;${todayStyle}"></div>
      </div>
      <div class="bar-lbl">${dayLabels[i] ?? '+'+i+'일'}</div>
      <div class="bar-val" style="color:${color}">${v != null ? Math.round(v) : '-'}</div>
    </div>`;
  }).join('');
}
```

- [ ] **Step 2: Add `renderRiskSummaryCard()` function**

Add after `renderDischargeChart()`:

```js
function renderRiskSummaryCard() {
  const el = document.getElementById('discharge-summary');
  if (!el) return;
  const data = floodCache[currentBasin];
  if (!data) {
    el.innerHTML = '<div style="font-family:\'Space Mono\',monospace;font-size:11px;color:var(--muted);text-align:center;padding:12px 0">데이터 없음</div>';
    return;
  }
  const {discharge, historicalMean, ratio, label, color} = data;
  el.innerHTML = `
    <div class="ds-row">
      <span class="ds-key">현재 유량</span>
      <span class="ds-val" style="color:${color}">${Math.round(discharge[0] ?? 0)} m³/s</span>
    </div>
    <div class="ds-row">
      <span class="ds-key">평년 대비</span>
      <span class="ds-val" style="color:${color}">${ratio.toFixed(1)}×
        <span class="ds-badge" style="background:${color};color:#070b12;margin-left:8px">${label}</span>
      </span>
    </div>
    <div class="ds-row">
      <span class="ds-key">최근 30일 평균</span>
      <span class="ds-val" style="color:var(--muted)">${historicalMean} m³/s</span>
    </div>`;
}
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add renderDischargeChart and renderRiskSummaryCard"
```

---

## Task 8: Update `renderRiskCard()`, `renderAlerts()`, `updateFlood()`, `updateAge()`, `runScan()`

**Files:**
- Modify: `index.html` (JS functions)

- [ ] **Step 1: Update `renderRiskCard()`**

Find `function renderRiskCard()`. Replace the `base_risk`/`zones`/`factors` block:

**Remove:**
```js
const zones = FLOOD_ZONES_BY_BASIN[currentBasin];
const rain_factor = Math.min(rainSim / 50, 1);
const age_factor  = ageSim / 100;
const base_risk = zones.reduce((s,z)=>s+z.capacity_ratio,0)/zones.length/100;
const risk_pct = Math.min(Math.round((base_risk * 0.5 + rain_factor * 0.35 + age_factor * 0.15) * 100), 99);
```

**Add:**
```js
const rain_factor = Math.min(rainSim / 50, 1);
const age_factor  = ageSim / 100;
const discharge_factor = floodCache[currentBasin]
  ? Math.min(floodCache[currentBasin].ratio / 3, 1)
  : 0.5;
const risk_pct = Math.min(
  Math.round((discharge_factor * 0.5 + rain_factor * 0.35 + age_factor * 0.15) * 100),
  99
);
```

**Replace `factors` array:**
```js
// Remove:
const factors = [
  {k:'강수량',  v:Math.round(rain_factor*100), color:'#3b9eff'},
  {k:'관로노후',v:ageSim,                     color:'#ff8c42'},
  {k:'관로부하',v:Math.round(base_risk*100),  color:'#cc44ff'},
];

// Add:
const factors = [
  {k:'강수량',  v: Math.round(rain_factor * 100),       color:'#3b9eff'},
  {k:'관로노후', v: ageSim,                              color:'#ff8c42'},
  {k:'하천수위', v: Math.round(discharge_factor * 100), color:'#00e5c0'},
];
```

**Update `sc-load` line** at bottom of `renderRiskCard()`:
```js
document.getElementById('sc-load').textContent = Math.round(discharge_factor * 100 + rain_factor * 15);
```

- [ ] **Step 2: Update `renderAlerts()`**

Find `function renderAlerts()`. Replace the zones/high block:

**Remove:**
```js
const zones = FLOOD_ZONES_BY_BASIN[currentBasin];
const high = zones.filter(z=>z.capacity_ratio>=85);
```

**Add:**
```js
const floodRisk = floodCache[currentBasin];
const highFlood = floodRisk && floodRisk.ratio >= 1.5;
```

If there is any `if(high.length>0)` flood alert block, replace it with `if(highFlood)`.

- [ ] **Step 3: Update `updateFlood()`**

Find `function updateFlood(val)`. Replace `renderRiskCard(); renderFloodDash(); renderAlerts();` with:

```js
function updateFlood(val) {
  rainSim = parseInt(val);
  document.getElementById('rain-val').textContent = val + ' mm/h';
  if (currentBasin) {
    renderRiskCard();
    renderRainBarChart();
    renderDischargeChart();
    renderRiskSummaryCard();
    renderAlerts();
  }
}
```

- [ ] **Step 4: Update `updateAge()`**

Find `function updateAge(val)`. Replace `renderRiskCard(); renderZoneList(); renderAlerts();` with:

```js
function updateAge(val) {
  ageSim = parseInt(val);
  document.getElementById('age-val').textContent = val + '%';
  if (currentBasin) {
    renderRiskCard();
    renderDischargeChart();
    renderRiskSummaryCard();
    renderAlerts();
  }
}
```

- [ ] **Step 5: Update `runScan()`**

Replace the entire `function runScan()`:

```js
function runScan() {
  if (!currentBasin) { alert('수계를 먼저 선택하세요'); return; }
  renderGradeCard();
  renderRiskCard();
  renderWQGrid();
  renderRainBarChart();
  renderDischargeChart();
  renderRiskSummaryCard();
  renderAlerts();

  // Update sc-zone stat card
  const fd = floodCache[currentBasin];
  const scZone = document.getElementById('sc-zone');
  if (scZone) {
    scZone.textContent = fd?.label ?? '--';
    scZone.style.color = fd?.color ?? 'var(--c1)';
  }
}
```

- [ ] **Step 6: Full browser test**

1. Open `index.html`
2. Select 한강 → click SCAN
3. Confirm: sidebar shows 하천수위 factor, discharge bar chart renders 7 bars with today's bar outlined, risk summary shows 3 rows, third stat card shows 안전/관찰/주의/경보
4. Drag rain slider → risk card updates
5. Drag age slider → discharge chart and risk card update
6. DevTools Console: zero errors

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: wire flood data into renderRiskCard, renderAlerts, updateFlood, updateAge, runScan"
```

---

## Task 9: Final Cleanup and Push

**Files:**
- Modify: `index.html`, `README.md`

- [ ] **Step 1: Search for stale references**

Search `index.html` for each term — all should return zero results:
- `FLOOD_ZONES_BY_BASIN`
- `renderZoneList`
- `renderFloodDash`
- `zone-list`
- `관로부하`

- [ ] **Step 2: Run IDE diagnostics**

Check VS Code Problems panel. Zero errors expected.

- [ ] **Step 3: Test all 5 basins**

For each of 한강, 낙동강, 금강, 영산강, 섬진강:
- Select → SCAN → confirm discharge chart and summary card render with real values

- [ ] **Step 4: Update README release history**

In `README.md`, add a new row to the release table:

```markdown
| v1.2.0 | 2026-03 | Open-Meteo Flood API (GloFAS) 실시간 하천 유량 연동 · 7일 예보 차트 · 위험 등급 실데이터화 |
```

- [ ] **Step 5: Final commit and push**

```bash
git add index.html README.md
git commit -m "feat: complete Flood API integration — real river discharge replaces zone simulation"
git push origin main
```

---

## Verification Checklist

- [ ] Network tab shows 10 flood API requests on page load (5 forecast + 5 historical)
- [ ] Live-dot turns orange on load, returns green when done
- [ ] SCAN renders 7-bar discharge chart with per-day risk colours
- [ ] Today's bar has teal outline
- [ ] Risk summary card shows current flow, ratio, 30-day mean
- [ ] Third stat card shows 안전/관찰/주의/경보 in correct colour
- [ ] Sidebar risk card shows 하천수위 factor (not 관로부하)
- [ ] Rain slider updates risk card and rain bar chart
- [ ] Age slider updates risk card and discharge chart
- [ ] Zero `FLOOD_ZONES_BY_BASIN` / `renderZoneList` / `renderFloodDash` references remain
- [ ] Zero console errors on load and on SCAN
- [ ] README updated to v1.2.0
