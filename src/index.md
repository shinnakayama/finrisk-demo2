---
toc: false
theme: "light"
---

```js
import * as aq from 'npm:arquero';
import { op, table } from 'npm:arquero';

```


```js
/***************
 * CONTROLS
 ***************/

// CMIP6 scenario radios
const this_experiment = (() => {
  const wrapper = html`<div class="radio-inline"></div>`;
  const labelEl = html`<span class="radio-label">CMIP6 scenario:</span>`;
  const form = html`<form role="radiogroup" aria-label="CMIP6 scenario:">
    <label><input type="radio" name="exp" value="ssp1_2_6" checked> SSP1-2.6 (sustainability)</label>
    <label><input type="radio" name="exp" value="ssp2_4_5"> SSP2-4.5 (middle of the road)</label>
    <label><input type="radio" name="exp" value="ssp3_7_0"> SSP3-7.0 (regional rivalry)</label>
    <label><input type="radio" name="exp" value="ssp5_8_5"> SSP5-8.5 (fossil-fueled development)</label>
  </form>`;
  wrapper.append(labelEl, form);

  const setValue = () => {
    wrapper.value = form.querySelector('input[name="exp"]:checked')?.value;
    wrapper.dispatchEvent(new InputEvent('input', { bubbles: true }));
  };
  form.addEventListener('change', setValue);
  setValue();
  return wrapper;
})();

// Year slider
const this_year = (() => {
  const input = html`<input type="range" min="2015" max="2080" step="1" value="2025">`;
  const out = html`<span class="year-value">${input.value}</span>`;
  const box = html`<label class="year-inline">Year: ${input} ${out}</label>`;
  input.addEventListener("input", () => {
    out.textContent = input.value;
    box.dispatchEvent(new InputEvent("input", { bubbles: true }));
  });
  Object.defineProperty(box, "value", { get: () => +input.value });
  return box;
})();

// Senario radios
const this_scenario = (() => {
  const wrapper = html`<div class="radio-inline"></div>`;
  const labelEl = html`<span class="radio-label">National policy scenario:</span>`;
  const form = html`<form role="radiogroup" aria-label="National policy scenario">
    <label><input type="radio" name="scenario" value="BAU" checked> Business as usual</label>
    <label><input type="radio" name="scenario" value="Stress"> Climate Stress</label>
    <label><input type="radio" name="scenario" value="Sustainable"> Sustainability / Net Zero</label>
  </form>`;

  wrapper.append(labelEl, form);

  const setValue = () => {
    wrapper.value = form.querySelector('input[name="scenario"]:checked')?.value;
    wrapper.dispatchEvent(new InputEvent('input', { bubbles: true }));
  };

  form.addEventListener('change', setValue);
  setValue(); // initialize wrapper.value and emit first input

  return wrapper;
})();


// Helpers
const getYear = () => +this_year.value;
const onInputs = (fn) => {
  this_experiment.addEventListener("input", fn);
  this_year.addEventListener("input", fn);
  invalidation.then(() => {
    this_experiment.removeEventListener("input", fn);
    this_year.removeEventListener("input", fn);
  });
};
```


```js
/****************
 * DATA LOADING
 ****************/

// Small fetch helper
async function fetchCSV(url, map = (d) => d) {
  return d3.csv(url, map);
}

const iso3 = await (async () => {
  const txt = await d3.text("https://storage.googleapis.com/finrisk-demo/iso3.csv");
  return aq.fromCSV(txt);
})();

const climate_raw = await (async () => {
  const base = "https://storage.googleapis.com/finrisk-demo/climate.csv.gz";
  const url  = `${base}?v=${Date.now()}`;
  const txt  = await d3.text(url);
  return aq.fromCSV(txt);
})();

const fao_catch = await (async () => {
  const txt = await d3.text("https://storage.googleapis.com/finrisk-demo/catch_for_observable.csv");
  return aq.fromCSV(txt)
    .derive({year: d => d.year + 60})
    .derive({country: d => d.iso3 === 'PRK' ? 'Korea' : d.country})
    .derive({country: d => d.iso3 === 'USA' ? 'USA' : d.country});;
})();

// Build EEZ select options (only those present in both climate & catch)
const foo = iso3
  .join(
    climate_raw
      .rename({ISO_TER1: 'iso3'})
      .select('iso3')
      .dedupe()
      .join(fao_catch.select('iso3').dedupe(), 'iso3'),
    'iso3'
  );

const eezOptions = [
  {label: "All", value: "ALL"},
  ...foo.objects().map(d => ({label: d.name, value: d.iso3}))
];

// EEZ selector
const this_eez = (() => {
  const row   = html`<div class="control-row"></div>`;
  const label = html`<span class="control-label">EEZ:</span>`;
  const select = html`<select></select>`;
  for (const opt of eezOptions) {
    const o = document.createElement("option");
    o.value = opt.value; o.textContent = opt.label;
    select.appendChild(o);
  }
  select.value = eezOptions[0]?.value ?? "ALL";
  select.addEventListener("input", () => {
    row.dispatchEvent(new InputEvent("input", { bubbles: true }));
  });
  Object.defineProperty(row, "value", {
    get: () => select.value,
    set: v => { select.value = v; select.dispatchEvent(new Event("input", { bubbles:true })); }
  });
  row.append(label, select);
  return row;
})();
```


```js
/****************
 * VARIABLE PICKER + HEATMAP
 ****************/
const this_variable = (() => {
  const wrapper = html`<div class="radio-inline"></div>`;
  const form = html`<form role="radiogroup" aria-label="Variable">
    <label><input type="radio" name="exp" value="sst" checked> Sea surface temperature</label>
    <label><input type="radio" name="exp" value="wind"> Wind speed</label>
    <label><input type="radio" name="exp" value="seaice"> Sea-ice area</label>
    <label><input type="radio" name="exp" value="hdd"> Heating degree days</label>
  </form>`;
  wrapper.append(form);
  const setValue = () => {
    wrapper.value = form.querySelector('input[name="exp"]:checked')?.value;
    wrapper.dispatchEvent(new InputEvent('input', { bubbles: true }));
  };
  form.addEventListener('change', setValue);
  setValue();
  return wrapper;
})();

// Heatmap setup
import { feature } from "npm:topojson-client";
const world = await d3.json("https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json");
const land = feature(world, world.objects.land);
const lat = await fetchCSV("https://storage.googleapis.com/finrisk-demo/lat.csv", d => +d.lat);
const lon = await fetchCSV("https://storage.googleapis.com/finrisk-demo/lon.csv", d => +d.lon);

const aliasOf = { sst: "sst", wind: "wind", seaice: "seaice", hdd: "hdd" };

async function loadGrid(exp, varKey, year) {
  const alias = aliasOf[varKey];
  const url = `https://storage.googleapis.com/finrisk-demo/${exp}/${alias}/${alias}_${year}.csv.gz?v=${year}`;
  const res = await fetch(url, { cache: "no-store" });
  if (!res.ok) throw new Error(`Fetch failed ${res.status} for ${url}`);
  const txt = await res.text();
  return Float32Array.from(txt.trim().split(/\s*,\s*|\n/), Number);
}

const loadGridCached = (() => {
  const cache = new Map();
  return async (exp, varKey, year) => {
    const key = `${exp}/${varKey}/${year}`;
    if (!cache.has(key)) cache.set(key, loadGrid(exp, varKey, year));
    return cache.get(key);
  };
})();

function drawLegend(ctx, scale, x, y, w, h, vmin, vmax) {
  for (let i = 0; i < w; ++i) {
    const t = i / (w - 1);
    const v = vmin + t * (vmax - vmin);
    ctx.fillStyle = scale(v);
    ctx.fillRect(x + i, y, 1, h);
  }
  ctx.fillStyle = "#000";
  ctx.font = "11px system-ui";
  ctx.fillText(`${d3.format(".2f")(vmin)} – ${d3.format(".2f")(vmax)}`, x, y - 4);
}

const heatmap = (async () => {
  const W = 800, H = 440;
  const MARGIN = { top: 10, right: 10, bottom: 10, left: 10 };
  const LEGEND_H = 12, LEGEND_BLOCK = LEGEND_H + 16;
  const canvas = html`<canvas width=${W} height=${H}></canvas>`;
  const ctx = canvas.getContext("2d");
  const proj = d3.geoEqualEarth().fitExtent(
    [[MARGIN.left, MARGIN.top], [W - MARGIN.right, H - MARGIN.bottom - LEGEND_BLOCK]],
    { type: "Sphere" }
  );
  const path = d3.geoPath(proj, ctx);

  let drawId = 0;
  async function render() {
    const id = ++drawId;
    const exp = this_experiment.value;
    const varKey = this_variable.value;
    const yr = getYear();

    const flat = await loadGridCached(exp, varKey, yr);
    if (id !== drawId) return;

    const nlat = lat.length, nlon = lon.length;
    const FIXED_DOMAINS = { sst:[-2,35], wind:[0.48,13.5], seaice:[0,100], hdd:[0,25600] };
    const [vmin, vmax] = FIXED_DOMAINS[varKey] ?? [0,1];
    const color = d3.scaleSequential(d3.interpolateTurbo).domain([vmin, vmax]);

    ctx.clearRect(0, 0, W, H);

    const latPx = new Float32Array(nlat);
    const dxAt = (φdeg) => {
      const a = proj([0, φdeg]), b = proj([1, φdeg]);
      return Math.abs((a && b) ? (b[0] - a[0]) : 1.5);
    };
    for (let i = 0; i < nlat; ++i) latPx[i] = dxAt(lat[i]);

    const dy = (() => {
      const a = proj([0, 0]), b = proj([0, 1]);
      return Math.abs((a && b) ? (b[1] - a[1]) : 1.5);
    })();

    const eps = 0.5;
    let k = 0;
    for (let i = 0; i < nlat; ++i) {
      const φ = lat[i];
      const dx = Math.max(1, latPx[i]);
      for (let j = 0; j < nlon; ++j, ++k) {
        const val = flat[k];
        if (!Number.isFinite(val)) continue;
        const λ = lon[j] > 180 ? ((lon[j] + 180) % 360) - 180 : lon[j];
        const p = proj([λ, φ]); if (!p) continue;
        ctx.fillStyle = color(val);
        ctx.fillRect(p[0] - dx/2 - eps, p[1] - dy/2 - eps, dx + 2*eps, dy + 2*eps);
      }
    }

    ctx.save(); ctx.fillStyle = "#666";
    ctx.beginPath(); path(land); ctx.fill();
    ctx.restore();
    ctx.strokeStyle = "#333"; ctx.lineWidth = 1;
    ctx.beginPath(); path({type:"Sphere"}); ctx.stroke();

    const legendY = H - MARGIN.bottom - LEGEND_H;
    drawLegend(ctx, color, MARGIN.left, legendY, 220, LEGEND_H, vmin, vmax);

    // warm cache for scrubbing
    loadGridCached(exp, varKey, yr - 1);
    loadGridCached(exp, varKey, yr + 1);
  }

  const onInput = () => render();
  this_experiment.addEventListener('input', onInput);
  this_variable.addEventListener('input', onInput);
  this_year.addEventListener('input', onInput);
  invalidation.then(() => {
    this_experiment.removeEventListener('input', onInput);
    this_variable.removeEventListener('input', onInput);
    this_year.removeEventListener('input', onInput);
  });

  await render();
  return canvas;
})();
```


```js
/****************
 * CATCH PANEL
 ****************/

// Build grouped catch table for current EEZ
function fao_catch2() {
  const iso = this_eez.value;
  const base = iso === 'ALL' ? fao_catch : fao_catch.params({iso}).filter(d => d.iso3 === iso);
  return base.groupby('ocean', 'year').rollup({ catch_tonne: op.sum('catch_tonne') });
}

// Deterministic seed + scenario model
function seedFromString(s){
  let h = 2166136261 >>> 0;
  for (let i = 0; i < s.length; i++) { h ^= s.charCodeAt(i); h = Math.imul(h, 16777619); }
  return (h >>> 0) / 4294967296;
}

function scenarioFactor(exp, year){
  const t0 = Math.max(0, Math.min(1, (year - 2015) / (2080 - 2015)));
  const e1 = Math.pow(t0, 2.2), e2 = Math.pow(t0, 4.0);
  const t  = 0.55 * e1 + 0.45 * e2;
  const targets = { ssp1_2_6: 0.70, ssp2_4_5: 0.50, ssp3_7_0: 0.33, ssp5_8_5: 0.22 };
  const end = targets[exp] ?? 0.50;
  return 1 + t * (end - 1);
}

function adjustCatchTable(table, exp, eezVal){
  const oceanK = {
    "Pacific": 0.95, "Atlantic": 0.90, "Indian Ocean": 0.85, "Arctic Sea": 0.70,
    "Mediterranean and Black Sea": 0.78
  };
  const arr = table.objects();
  return arr.map(d => {
    const base = d.catch_tonne;
    const fac  = scenarioFactor(exp, d.year) * (oceanK[d.ocean] ?? 1);
    const seed = seedFromString(`${exp}|${eezVal}|${d.ocean}|${d.year}`);
    const rng  = d3.randomLcg(seed);
    const noise = (rng() - 0.5) * 0.40;
    const t0 = Math.max(0, Math.min(1, (d.year - 2015) / (2080 - 2015)));
    const lateDrag = 1 - 0.06 * Math.pow(t0, 3.5);
    const adj = Math.max(0, base * fac * lateDrag * (1 + noise));
    return { ...d, catch_tonne_adj: adj };
  });
}

// Responsive catch card: widest possible chart, no "Year" label or year readout
// Responsive catch card: fixed-width readout to prevent chart resizing
function catch_card({ label = "Catch", unit = "tonnes" } = {}) {
  const stackOrder = [
    "Pacific",
    "Atlantic",
    "Indian Ocean",
    "Arctic Sea",
    "Mediterranean and Black Sea"
  ];

  // --- data prep ---
  const base   = fao_catch2().filter(d => d.year >= 2015 && d.year <= 2080);
  const exp    = this_experiment.value;
  const eezVal = this_eez.value;
  const dataAdj = adjustCatchTable(base, exp, eezVal);

  const year = getYear();
  const totalThisYear =
    d3.sum(dataAdj.filter(d => d.year === year), d => d.catch_tonne_adj) || 0;

  const fmtBig  = d3.format(".2s");
  const fmtYear = d3.format("d");

  // --- layout: left plot stretches; right readout fixed width (no jitter) ---
  const HEIGHT = 200;
  const el = html`<div style="
    display:grid;
    grid-template-columns: minmax(0,1fr) 8ch;  /* fixed readout column */
    column-gap: 20px;
    align-items:center;
  "></div>`;

  // Chart fills the left cell
  const plot = Plot.plot({
    height: HEIGHT,
    marginTop: 8,
    marginBottom: 28,
    marginRight: 10,
    y: { label: null, tickFormat: "s" },
    x: { tickFormat: fmtYear },
    color: { legend: true, domain: stackOrder, range: d3.schemeCategory10 },
    marks: [
      Plot.areaY(dataAdj, {
        x: "year",
        y: "catch_tonne_adj",
        fill: "ocean",
        order: stackOrder,
        clip: true
      }),
      Plot.ruleX([year], { stroke: "black", strokeWidth: 1.5, opacity: 0.7 }),
      Plot.ruleY([0])
    ]
  });
  plot.style.width = "100%";
  plot.style.display = "block";

  // Right readout: tabular numerals + fixed column width
  const right = html`<div style="
    text-align:right;
    white-space:nowrap;
    font-variant-numeric: tabular-nums;  /* equal-width digits */
  ">
    <div style="font:600 13px/1.2 system-ui,sans-serif;color:#666">${label}</div>
    <div style="font:700 26px/1.1 system-ui,sans-serif">${fmtBig(totalThisYear)}</div>
    <div style="font:12px/1.1 system-ui,sans-serif;color:#666">${unit}</div>
  </div>`;

  el.append(plot, right);
  return el;
}


const catch_panel = (() => {
  const host = html`<div></div>`;
  const render = () => { host.innerHTML = ""; host.append(catch_card()); };

  const onInput = () => render();
  this_eez.addEventListener("input", onInput);
  this_year.addEventListener("input", onInput);
  this_experiment.addEventListener("input", onInput);
  invalidation.then(() => {
    this_eez.removeEventListener("input", onInput);
    this_year.removeEventListener("input", onInput);
    this_experiment.removeEventListener("input", onInput);
  });

  render();
  return host;
})();
```


```js
/****************
 * EARNINGS
 ****************/
// --- REACTIVE earnings plot (updates on EEZ, CMIP6 experiment, and policy scenario) ---
const earnings_plot = (() => {
  // ---------- shared helpers / constants ----------
  let catch_by_iso3 = fao_catch
    .groupby("iso3")
    .rollup({ total_catch: op.sum("catch_tonne") })
    .objects();

  function getCatchForEEZ(iso3) {
    if (!iso3) return 1;
    if (iso3 === "ALL") {
      const total = d3.sum(catch_by_iso3, d => d.total_catch);
      return total || 1;
    }
    const rec = catch_by_iso3.find(r => r.iso3 === iso3);
    return rec ? rec.total_catch : 1;
  }

  const SCEN = {
    BAU:         { growth: 0.05,  margin: 0.16, revVol: 0.25, earnVol: 0.20, mDrift:  0.000 },
    Stress:      { growth: 0.02,  margin: 0.10, revVol: 0.35, earnVol: 0.30, mDrift: -0.010 },
    Sustainable: { growth: 0.035, margin: 0.20, revVol: 0.20, earnVol: 0.18, mDrift:  0.004 }
  };

  const EXP = {
    ssp1_2_6: { growth_mult: 1.05, margin_mult: 1.05, vol_mult: 0.90, base_mult: 1.00, mDrift_mult: 0.7 },
    ssp2_4_5: { growth_mult: 1.00, margin_mult: 1.00, vol_mult: 1.00, base_mult: 1.00, mDrift_mult: 1.0 },
    ssp3_7_0: { growth_mult: 0.90, margin_mult: 0.95, vol_mult: 1.20, base_mult: 0.98, mDrift_mult: 1.3 },
    ssp5_8_5: { growth_mult: 0.98, margin_mult: 0.98, vol_mult: 1.30, base_mult: 1.02, mDrift_mult: 1.5 }
  };

  function seedFrom(...parts) {
    const s = parts.join("|");
    const h = Array.from(s).reduce(
      (acc, ch) => (acc * 1664525 + ch.charCodeAt(0) + 1013904223) >>> 0,
      2166136261
    );
    return (h % 1e6) / 1e6;
  }

  function makeFakeFinancials(years, cfg, baseRev, rng, mDrift_per_step) {
    const mNoiseAmp = 0.10 + 0.15 * ((cfg.revVol + cfg.earnVol) / 2);
    let m = cfg.margin;
    return years.map((year, i) => {
      const trend    = 1 + cfg.growth * i;
      const season   = 1 + 0.04 * Math.sin(i * Math.PI / 2);
      const revNoise = 1 + (rng() - 0.5) * cfg.revVol;
      const revenue  = baseRev * trend * season * revNoise;

      if (i) {
        const noise = (rng() - 0.5) * (2 * mNoiseAmp);
        let drift   = mDrift_per_step * (1 + 0.4 * (rng() - 0.5));
        if (m < 0) drift += -0.003;
        m = Math.max(-0.50, Math.min(0.60, m + drift + noise));
      }

      const earnNoise = 1 + (rng() - 0.5) * cfg.earnVol;
      const earnings  = revenue * m * earnNoise;
      return { year, revenue, earnings, margin: m };
    });
  }

  const years = [2020, 2030, 2040, 2050, 2060, 2070, 2080];

  // ---------- renderer ----------
  function buildPlot() {
    const scen = SCEN[this_scenario.value] ?? SCEN.BAU;
    const exp  = EXP[this_experiment.value] ?? EXP.ssp2_4_5;

    const cfg = {
      growth:  scen.growth  * exp.growth_mult,
      margin:  scen.margin  * exp.margin_mult,
      revVol:  scen.revVol  * exp.vol_mult,
      earnVol: scen.earnVol * exp.vol_mult
    };

    const mDrift_per_step = (scen.mDrift ?? 0) * (exp.mDrift_mult ?? 1);

    const iso = this_eez.value;
    const totalCatch = getCatchForEEZ(iso);
    const baseRev = totalCatch * 10 * exp.base_mult;

    const seed = seedFrom(iso, this_scenario.value, this_experiment.value);
    const rng  = d3.randomLcg(seed);

    const fake = makeFakeFinancials(years, cfg, baseRev, rng, mDrift_per_step);
    const long = fake.flatMap(d => ([
      { year: d.year, series: "Revenue",  value: d.revenue },
      { year: d.year, series: "Earnings", value: d.earnings }
    ]));

    return Plot.plot({
      width: 800,
      height: 220,
      marginLeft: 100,
      marginBottom: 50,
      fx: { domain: years, label: null, tickFormat: d3.format("d"), tickSize: 0, padding: 0 },
      x: { domain: ["Revenue", "Earnings"], type: "band", axis: null, padding: 0.2 },
      y: { label: "USD", tickFormat: d => d3.format(".2s")(d) },
      color: { legend: true, domain: ["Revenue", "Earnings"], range: d3.schemeCategory10 },
      marks: [
        Plot.ruleY([0], { stroke: "black", strokeWidth: 1.5, opacity: 0.8 }),
        Plot.barY(long, { fx: "year", x: "series", y: "value", fill: "series", rx: 4 })
      ]
    });
  }

  // ---------- reactive host ----------
  const host = html`<div></div>`;
  function render() {
    const plot = buildPlot();
    host.replaceChildren(plot);
  }

  // re-render on control changes
  const onInput = () => render();
  this_eez.addEventListener("input", onInput);
  this_experiment.addEventListener("input", onInput);
  this_scenario.addEventListener("input", onInput);

  invalidation.then(() => {
    this_eez.removeEventListener("input", onInput);
    this_experiment.removeEventListener("input", onInput);
    this_scenario.removeEventListener("input", onInput);
  });

  render();
  return host;
})();

    
```


```js
/****************
 * CATCH RANK PLOT
 ****************/
/****************
 * CATCH RANK (reactive to this_experiment)
 ****************/
const tickYears = [2020, 2030, 2040, 2050, 2060, 2070, 2080];
const xDomain   = [tickYears[0] - 15, tickYears[tickYears.length - 1] + 15];

function adjustRowsByExperiment(rows, exp) {
  const oceanK = {
    "Pacific": 0.95, "Atlantic": 0.90, "Indian Ocean": 0.85,
    "Arctic Sea": 0.70, "Mediterranean and Black Sea": 0.78
  };
  return rows.map(r => {
    const fac  = scenarioFactor(exp, r.year) * (oceanK[r.ocean] ?? 1);
    const seed = seedFromString(`${exp}|${r.iso3}|${r.ocean}|${r.year}`);
    const rng  = d3.randomLcg(seed);
    const noise = (rng() - 0.5) * 0.40;
    const t0 = Math.max(0, Math.min(1, (r.year - 2015) / (2080 - 2015)));
    const lateDrag = 1 - 0.06 * Math.pow(t0, 3.5);
    const adj = Math.max(0, r.catch_tonne * fac * lateDrag * (1 + noise));
    return { year: r.year, country: r.country, catch_adj: adj };
  });
}

const catch_rank_panel = (() => {
  const host = html`<div class="catch-rank"></div>`; // 👈 no .card here

  function render() {
    const exp  = this_experiment.value;
    const rows = fao_catch.objects();
    const adjRows = adjustRowsByExperiment(rows, exp);

    const ranked = aq.from(adjRows)
      .filter(aq.escape(d => d.year % 10 === 0 && d.year > 2010))
      .groupby("year", "country")
      .rollup({ catch_tonne: op.sum("catch_adj") })
      .groupby("year")
      .orderby(aq.desc("catch_tonne"))
      .derive({ rank: op.rank() })
      .orderby("year");

    const plot = Plot.plot({
      // let width be controlled by the container
      height: 350,
      x: { label: null, domain: xDomain, nice: false, tickFormat: d3.format("d"), ticks: tickYears },
      y: { label: null, domain: [0.5, 10.5], reverse: true, ticks: [], nice: false },
      color: { range: d3.schemeCategory10 },
      marks: [
        Plot.line(ranked, {
          x: "year", y: "rank", stroke: "country",
          strokeWidth: 5, opacity: 0.4, z: "country", sort: "year", clip: true
        }),
        Plot.dot(ranked, { x: "year", y: "rank", fill: "country", r: 10, clip: true }),
        Plot.axisX({ y: 0, ticks: tickYears, tickFormat: d3.format("d"), tickSize: 0 }),

        // bigger country labels
        Plot.text(ranked, Plot.selectFirst({
          x: "year", y: "rank", z: "country", text: "country",
          dx: -15, textAnchor: "end", clip: true, fontSize: 12
        })),
        Plot.text(ranked, Plot.selectLast({
          x: "year", y: "rank", z: "country", text: "country",
          dx: 15, textAnchor: "start", clip: true, fontSize: 12
        })),

        // rank numbers inside dots
        Plot.text(ranked, Plot.selectFirst({
          x: "year", y: "rank", z: "country", fill: "white",
          text: d => d.rank, textAnchor: "middle", fontSize: 14, clip: true
        })),
        Plot.text(ranked, Plot.selectLast({
          x: "year", y: "rank", z: "country", fill: "white",
          text: d => d.rank, textAnchor: "middle", fontSize: 14, clip: true
        }))
      ],
      marginTop: 16, marginBottom: 28, marginLeft: 60, marginRight: 36
    });

    // make the SVG fill the card width
    plot.style.width = "100%";
    plot.style.display = "block";

    host.replaceChildren(
      html`<h3 class="card-title" style="color:#000;font-weight:700;">Catch rank</h3>`,
      plot
    );
  }

  const onInput = () => render();
  this_experiment.addEventListener("input", onInput);
  invalidation.then(() => this_experiment.removeEventListener("input", onInput));

  render();
  return host;
})();

```


<div class="hero">
  <h1>Demo 2</h1>
  <div class="controls">
    ${this_experiment}
    ${this_year}
    ${this_eez}
  </div>
  <div class="grid grid-cols-2 full-bleed tight-rows" style="gap: 1rem;">
    <div class="card controls-vertical no-gap">
      ${this_variable}
      ${heatmap}
    </div>
    <div class="card" style="position:relative;">
      ${catch_rank_panel}
    </div>
    <div class="card" style="position:relative;">
      ${catch_panel}
    </div>
    <div class="card">
      ${this_scenario}
      ${earnings_plot}
    </div>
  </div>
</div>




<style>
:root {
  --control-gap: 1rem;
  --control-font-size: 14px;
}

/* ---------- Page framework ---------- */
.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: var(--sans-serif);
  text-align: center;
  margin: 0;
  text-wrap: balance;
  overflow: hidden;
}
.hero h1 {
  margin: 0 0 .75rem;
  line-height: 1;
}
@media (min-width: 640px) {
  .hero h1 { font-size: 50px; }
}

.full-bleed { width: 100%; margin: 0; }

/* ---------- Cards / stacked content ---------- */
.controls-vertical { display: flex; flex-direction: column; gap: var(--control-gap); }
.controls-vertical.no-gap { gap: 0; }
.controls-vertical.no-gap > .radio-inline { margin-bottom: 0; }

/* ---------- Controls (tight rows, shared left rail) ---------- */
.controls {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: .375rem;
  width: 100%;
}
.controls > * { margin: 0; font-size: var(--control-font-size); }

/* Each control row: label left, control right */
.control-row {
  display: flex;
  align-items: center;
  gap: .5rem;
  width: 100%;
}
.control-label {
  font-weight: 700;
  white-space: nowrap;
}

/* Extra space before the EEZ row (it's the only .control-row) */
.controls > .control-row { margin-bottom: 1.5rem; }

/* ---------- CMIP6 radios ---------- */
.radio-inline {
  display: flex !important;
  align-items: center;
  gap: .5rem 1rem;
  flex-wrap: wrap;
  overflow: visible;
  line-height: 1.2;
}
.radio-inline .radio-label {
  margin-right: .5rem;
  white-space: nowrap;
  font-weight: 700;
}
.radio-inline [role="radiogroup"],
.radio-inline > form {
  display: flex !important;
  flex-wrap: wrap !important;
  align-items: center;
  gap: .5rem 1rem;
  margin: 0;
  padding: 0;
  border: 0;
}
.radio-inline label {
  display: inline-flex !important;
  align-items: center;
  gap: .35rem;
  margin: 0;
  white-space: nowrap !important;
}

/* ---------- Year slider ---------- */
.year-inline {
  display: flex;
  align-items: center;
  gap: .75rem;
  width: 100%;
  font: 700 var(--control-font-size)/1.2 var(--sans-serif);
}
.year-inline input[type="range"] {
  flex: 1 1 0%;
  min-width: 240px;
  max-width: 50%;
  font-size: var(--control-font-size);
}
.year-inline .year-value {
  font: 700 var(--control-font-size)/1.2 var(--sans-serif);
  width: 4ch;
  text-align: right;
}

/* ---------- EEZ dropdown (native select) ---------- */
.eez-inline { display: flex; align-items: center; gap: .5rem; }
.eez-label { font-weight: 700; white-space: nowrap; }
.eez-inline select { font-weight: 400; }

/* ---------- Grid rows: each row takes tallest item; no overlap ---------- */
.tight-rows {
  /* default grid-auto-rows: auto → row height = tallest item in that row */
  align-items: stretch;        /* children stretch to fill the row’s track */
}

/* Two columns, auto row height = tallest card in that row */
.grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr)); /* <- key change */
  grid-auto-rows: auto;
  align-items: stretch;
  gap: 1rem;
}


.card {
  display: flex;
  flex-direction: column;
  /* no fixed height! let grid control it */
  min-height: 0;  /* safety for flex content */
}


/* If any inner content is absolute/fixed-size, keep it from forcing the row */
.card > * {
  min-height: 0;
}



.card-title {
  margin: 0 0 .5rem;
  font: 700 16px/1.2 system-ui, sans-serif;
  text-align: left; /* or center if you prefer */
}

</style>

