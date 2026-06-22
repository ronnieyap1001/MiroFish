<template>
  <div class="forecast-container">
    <!-- Top nav -->
    <nav class="navbar">
      <router-link to="/" class="nav-brand">MIROFISH</router-link>
      <div class="nav-links">
        <router-link to="/" class="nav-link">← Home</router-link>
      </div>
    </nav>

    <div class="main-content">
      <header class="page-head">
        <span class="orange-tag">FORECAST</span>
        <h1 class="page-title">Traffic &amp; Conversion Forecast</h1>
        <p class="page-sub">
          Enter your current numbers and assumptions. Everything runs locally in your
          browser — no account, no API keys, no data leaves this page.
        </p>
      </header>

      <div class="layout">
        <!-- ===================== INPUTS ===================== -->
        <section class="inputs-panel">
          <div class="panel-block">
            <div class="block-header"><span class="diamond">◇</span> Traffic</div>

            <label class="field">
              <span class="field-label">Current monthly visitors</span>
              <input type="number" min="0" step="100" v-model.number="inputs.startVisitors" />
            </label>

            <label class="field">
              <span class="field-label">
                Monthly growth rate
                <em>{{ inputs.growthRate }}%</em>
              </span>
              <input type="range" min="-20" max="50" step="0.5" v-model.number="inputs.growthRate" />
            </label>

            <label class="field">
              <span class="field-label">
                Growth uncertainty (±)
                <em>±{{ inputs.growthUncertainty }} pts</em>
              </span>
              <input type="range" min="0" max="20" step="0.5" v-model.number="inputs.growthUncertainty" />
              <span class="hint">Widens the optimistic / conservative band around growth.</span>
            </label>

            <label class="field">
              <span class="field-label">
                Forecast horizon
                <em>{{ inputs.horizon }} months</em>
              </span>
              <input type="range" min="1" max="36" step="1" v-model.number="inputs.horizon" />
            </label>
          </div>

          <div class="panel-block">
            <div class="block-header"><span class="diamond">◇</span> Conversion funnel</div>

            <label class="field">
              <span class="field-label">
                Visitor → lead rate
                <em>{{ inputs.leadRate }}%</em>
              </span>
              <input type="range" min="0" max="100" step="0.5" v-model.number="inputs.leadRate" />
              <span class="hint">Visitors who take a key action (sign up, add to cart, enquiry…).</span>
            </label>

            <label class="field">
              <span class="field-label">
                Lead → customer rate
                <em>{{ inputs.customerRate }}%</em>
              </span>
              <input type="range" min="0" max="100" step="0.5" v-model.number="inputs.customerRate" />
              <span class="hint">Leads who become paying customers.</span>
            </label>

            <label class="field">
              <span class="field-label">
                Conversion uncertainty (±)
                <em>±{{ inputs.conversionUncertainty }}%</em>
              </span>
              <input type="range" min="0" max="60" step="1" v-model.number="inputs.conversionUncertainty" />
            </label>

            <div class="field-row">
              <label class="field grow">
                <span class="field-label">Average order value</span>
                <input type="number" min="0" step="1" v-model.number="inputs.aov" />
              </label>
              <label class="field currency-field">
                <span class="field-label">Currency</span>
                <input type="text" maxlength="3" v-model="inputs.currency" />
              </label>
            </div>

            <div class="overall-conv">
              Overall conversion rate:
              <strong>{{ pct(overallConv * 100, 2) }}</strong>
            </div>
          </div>

          <div class="panel-block">
            <div class="block-header"><span class="diamond">◇</span> Traffic sources (%)</div>
            <label v-for="key in sourceKeys" :key="key" class="field source-field">
              <span class="field-label">
                <span class="swatch" :style="{ background: sourceColors[key] }"></span>
                {{ sourceLabels[key] }}
                <em>{{ normalizedSources[key].toFixed(0) }}%</em>
              </span>
              <input type="number" min="0" step="1" v-model.number="inputs.sources[key]" />
            </label>
            <span class="hint">Values are normalised to 100% automatically.</span>
          </div>

          <div class="panel-actions">
            <button class="ghost-btn" @click="resetDefaults">Reset</button>
            <button class="ghost-btn" @click="downloadCsv">Export CSV</button>
          </div>
        </section>

        <!-- ===================== RESULTS ===================== -->
        <section class="results-panel">
          <!-- Headline -->
          <div class="headline">
            Over the next <strong>{{ inputs.horizon }}</strong> months you're projected to attract
            <strong>{{ fmt(totals.visitors) }}</strong> visitors,
            convert <strong>{{ fmt(totals.customers) }}</strong> customers, and generate
            <strong>{{ money(totals.revenue) }}</strong> in revenue.
          </div>

          <!-- KPI cards -->
          <div class="kpi-grid">
            <div class="kpi">
              <div class="kpi-label">Visitors / mo (final)</div>
              <div class="kpi-value">{{ fmt(finalMonth.visitors) }}</div>
            </div>
            <div class="kpi">
              <div class="kpi-label">Customers (total)</div>
              <div class="kpi-value">{{ fmt(totals.customers) }}</div>
            </div>
            <div class="kpi">
              <div class="kpi-label">Revenue (total)</div>
              <div class="kpi-value">{{ money(totals.revenue) }}</div>
            </div>
            <div class="kpi">
              <div class="kpi-label">Avg revenue / mo</div>
              <div class="kpi-value">{{ money(totals.revenue / inputs.horizon) }}</div>
            </div>
            <div class="kpi">
              <div class="kpi-label">Overall conv. rate</div>
              <div class="kpi-value">{{ pct(overallConv * 100, 2) }}</div>
            </div>
            <div class="kpi">
              <div class="kpi-label">Revenue range</div>
              <div class="kpi-value small">{{ money(totalsRange.low) }} – {{ money(totalsRange.high) }}</div>
            </div>
          </div>

          <!-- Traffic chart -->
          <div class="chart-card">
            <div class="chart-title">Monthly visitors</div>
            <svg :viewBox="`0 0 ${C.W} ${C.H}`" class="chart" preserveAspectRatio="xMidYMid meet">
              <!-- y grid + labels -->
              <g v-for="t in trafficChart.yTicks" :key="'ty'+t.label">
                <line :x1="C.PL" :x2="C.W - C.PR" :y1="t.y" :y2="t.y" class="grid" />
                <text :x="C.PL - 8" :y="t.y + 4" class="axis-label end">{{ t.label }}</text>
              </g>
              <!-- x labels -->
              <text
                v-for="t in trafficChart.xTicks" :key="'tx'+t.label"
                :x="t.x" :y="C.H - 12" class="axis-label mid"
              >{{ t.label }}</text>
              <!-- uncertainty band -->
              <path :d="trafficChart.bandPath" class="band" />
              <!-- expected line -->
              <path :d="trafficChart.linePath" class="line" />
              <circle
                v-for="p in trafficChart.points" :key="'pt'+p.m"
                :cx="p.x" :cy="p.y" r="2.5" class="dot"
              />
            </svg>
            <div class="legend">
              <span class="legend-item"><span class="legend-line"></span> Expected</span>
              <span class="legend-item"><span class="legend-band"></span> Conservative ↔ Optimistic</span>
            </div>
          </div>

          <!-- Revenue chart -->
          <div class="chart-card">
            <div class="chart-title">Projected revenue per month</div>
            <svg :viewBox="`0 0 ${C.W} ${C.H}`" class="chart" preserveAspectRatio="xMidYMid meet">
              <g v-for="t in revenueChart.yTicks" :key="'ry'+t.label">
                <line :x1="C.PL" :x2="C.W - C.PR" :y1="t.y" :y2="t.y" class="grid" />
                <text :x="C.PL - 8" :y="t.y + 4" class="axis-label end">{{ t.label }}</text>
              </g>
              <text
                v-for="t in revenueChart.xTicks" :key="'rx'+t.label"
                :x="t.x" :y="C.H - 12" class="axis-label mid"
              >{{ t.label }}</text>
              <rect
                v-for="b in revenueChart.bars" :key="'bar'+b.m"
                :x="b.x" :y="b.y" :width="b.w" :height="b.h" class="bar"
              />
            </svg>
          </div>

          <!-- Funnel + sources -->
          <div class="dual-card">
            <div class="chart-card flex1">
              <div class="chart-title">Conversion funnel (total over horizon)</div>
              <div class="funnel">
                <div v-for="stage in funnel" :key="stage.label" class="funnel-stage">
                  <div class="funnel-bar" :style="{ width: stage.width + '%', background: stage.color }">
                    <span class="funnel-count">{{ fmt(stage.value) }}</span>
                  </div>
                  <div class="funnel-meta">
                    <span class="funnel-label">{{ stage.label }}</span>
                    <span class="funnel-rate" v-if="stage.rate !== null">{{ stage.rate }}</span>
                  </div>
                </div>
              </div>
            </div>

            <div class="chart-card flex1">
              <div class="chart-title">Traffic source mix</div>
              <div class="source-bar">
                <div
                  v-for="key in sourceKeys" :key="'sb'+key"
                  class="source-seg"
                  :style="{ width: normalizedSources[key] + '%', background: sourceColors[key] }"
                  :title="`${sourceLabels[key]} ${normalizedSources[key].toFixed(0)}%`"
                ></div>
              </div>
              <ul class="source-legend">
                <li v-for="key in sourceKeys" :key="'sl'+key">
                  <span class="swatch" :style="{ background: sourceColors[key] }"></span>
                  {{ sourceLabels[key] }}
                  <span class="src-pct">{{ normalizedSources[key].toFixed(0) }}%</span>
                  <span class="src-vis">≈ {{ fmt(finalMonth.visitors * normalizedSources[key] / 100) }} /mo</span>
                </li>
              </ul>
            </div>
          </div>

          <!-- Monthly table -->
          <div class="chart-card">
            <div class="chart-title">Month-by-month detail</div>
            <div class="table-wrap">
              <table class="data-table">
                <thead>
                  <tr>
                    <th>Month</th>
                    <th>Visitors</th>
                    <th>Leads</th>
                    <th>Customers</th>
                    <th>Revenue</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="row in monthly" :key="row.m">
                    <td>{{ row.m }}</td>
                    <td>{{ fmt(row.visitors) }}</td>
                    <td>{{ fmt(row.leads) }}</td>
                    <td>{{ fmt(row.customers) }}</td>
                    <td>{{ money(row.revenue) }}</td>
                  </tr>
                </tbody>
                <tfoot>
                  <tr>
                    <td>Total</td>
                    <td>{{ fmt(totals.visitors) }}</td>
                    <td>{{ fmt(totals.leads) }}</td>
                    <td>{{ fmt(totals.customers) }}</td>
                    <td>{{ money(totals.revenue) }}</td>
                  </tr>
                </tfoot>
              </table>
            </div>
          </div>

          <p class="disclaimer">
            This is a deterministic projection based on the assumptions you entered (compound
            monthly growth and fixed funnel rates). It is a planning aid, not a guarantee — real
            traffic is affected by seasonality, campaigns and market shifts.
          </p>
        </section>
      </div>
    </div>
  </div>
</template>

<script setup>
import { reactive, computed, watch, onMounted } from 'vue'

const STORAGE_KEY = 'mirofish_forecast_inputs'

const DEFAULTS = {
  startVisitors: 10000,
  growthRate: 5,
  growthUncertainty: 3,
  horizon: 12,
  leadRate: 10,
  customerRate: 20,
  conversionUncertainty: 20,
  aov: 50,
  currency: '$',
  sources: { organic: 45, paid: 20, social: 15, direct: 12, referral: 8 }
}

const inputs = reactive(JSON.parse(JSON.stringify(DEFAULTS)))

// Restore persisted inputs
onMounted(() => {
  try {
    const saved = JSON.parse(localStorage.getItem(STORAGE_KEY) || 'null')
    if (saved && typeof saved === 'object') {
      Object.assign(inputs, saved)
      if (!inputs.sources) inputs.sources = { ...DEFAULTS.sources }
    }
  } catch (_) { /* ignore corrupt storage */ }
})

watch(inputs, (val) => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(val))
}, { deep: true })

function resetDefaults() {
  Object.assign(inputs, JSON.parse(JSON.stringify(DEFAULTS)))
}

/* ---------- Source mix ---------- */
const sourceKeys = ['organic', 'paid', 'social', 'direct', 'referral']
const sourceLabels = {
  organic: 'Organic search',
  paid: 'Paid ads',
  social: 'Social',
  direct: 'Direct',
  referral: 'Referral'
}
const sourceColors = {
  organic: '#FF4500',
  paid: '#1A1A1A',
  social: '#00A6A6',
  direct: '#FFB000',
  referral: '#6C5CE7'
}
const normalizedSources = computed(() => {
  const total = sourceKeys.reduce((s, k) => s + (Number(inputs.sources[k]) || 0), 0)
  const out = {}
  for (const k of sourceKeys) {
    out[k] = total > 0 ? (Number(inputs.sources[k]) || 0) / total * 100 : 0
  }
  return out
})

/* ---------- Core projection ---------- */
const overallConv = computed(() => (inputs.leadRate / 100) * (inputs.customerRate / 100))

// Visitor series, months 0..horizon, with conservative/optimistic bands
const series = computed(() => {
  const start = Math.max(0, Number(inputs.startVisitors) || 0)
  const g = inputs.growthRate / 100
  const gLow = (inputs.growthRate - inputs.growthUncertainty) / 100
  const gHigh = (inputs.growthRate + inputs.growthUncertainty) / 100
  const f = (rate) => Math.max(0, 1 + rate)
  const rows = []
  for (let m = 0; m <= inputs.horizon; m++) {
    rows.push({
      m,
      expV: start * Math.pow(f(g), m),
      lowV: start * Math.pow(f(gLow), m),
      highV: start * Math.pow(f(gHigh), m)
    })
  }
  return rows
})

// Forecast months only (1..horizon)
const monthly = computed(() =>
  series.value.filter(r => r.m >= 1).map(r => {
    const visitors = r.expV
    const leads = visitors * inputs.leadRate / 100
    const customers = leads * inputs.customerRate / 100
    return { m: r.m, visitors, leads, customers, revenue: customers * inputs.aov }
  })
)

const totals = computed(() =>
  monthly.value.reduce((a, r) => ({
    visitors: a.visitors + r.visitors,
    leads: a.leads + r.leads,
    customers: a.customers + r.customers,
    revenue: a.revenue + r.revenue
  }), { visitors: 0, leads: 0, customers: 0, revenue: 0 })
)

const totalsRange = computed(() => {
  const conv = overallConv.value
  const cu = inputs.conversionUncertainty / 100
  const convLow = conv * (1 - cu)
  const convHigh = conv * (1 + cu)
  let low = 0, high = 0
  for (const r of series.value) {
    if (r.m < 1) continue
    low += r.lowV * convLow * inputs.aov
    high += r.highV * convHigh * inputs.aov
  }
  return { low, high }
})

const finalMonth = computed(() => {
  const last = series.value[series.value.length - 1]
  const visitors = last ? last.expV : 0
  const leads = visitors * inputs.leadRate / 100
  const customers = leads * inputs.customerRate / 100
  return { visitors, leads, customers, revenue: customers * inputs.aov }
})

const funnel = computed(() => {
  const v = totals.value.visitors || 1
  const stages = [
    { label: 'Visitors', value: totals.value.visitors, color: '#1A1A1A', rate: null },
    { label: 'Leads', value: totals.value.leads, color: '#FF7A45', rate: pct(inputs.leadRate, 1) + ' of visitors' },
    { label: 'Customers', value: totals.value.customers, color: '#FF4500', rate: pct(inputs.customerRate, 1) + ' of leads' }
  ]
  return stages.map(s => ({ ...s, width: Math.max(2, s.value / v * 100) }))
})

/* ---------- Chart geometry ---------- */
const C = { W: 760, H: 300, PL: 70, PR: 18, PT: 18, PB: 34 }

function niceMax(v) {
  if (v <= 0) return 1
  const exp = Math.floor(Math.log10(v))
  const base = Math.pow(10, exp)
  const f = v / base
  let nf
  if (f <= 1) nf = 1
  else if (f <= 2) nf = 2
  else if (f <= 2.5) nf = 2.5
  else if (f <= 5) nf = 5
  else nf = 10
  return nf * base
}

function compactNum(n) {
  const abs = Math.abs(n)
  if (abs >= 1e9) return (n / 1e9).toFixed(1).replace(/\.0$/, '') + 'B'
  if (abs >= 1e6) return (n / 1e6).toFixed(1).replace(/\.0$/, '') + 'M'
  if (abs >= 1e3) return (n / 1e3).toFixed(1).replace(/\.0$/, '') + 'k'
  return Math.round(n).toString()
}

function xTicksFor() {
  const h = inputs.horizon
  const innerW = C.W - C.PL - C.PR
  const step = h <= 12 ? 1 : Math.ceil(h / 12)
  const ticks = []
  for (let m = 0; m <= h; m += step) {
    ticks.push({ x: C.PL + (m / h) * innerW, label: m === 0 ? 'Now' : 'M' + m })
  }
  return ticks
}

const trafficChart = computed(() => {
  const h = Math.max(1, inputs.horizon)
  const innerW = C.W - C.PL - C.PR
  const innerH = C.H - C.PT - C.PB
  const maxV = Math.max(...series.value.map(r => r.highV), 1)
  const yMax = niceMax(maxV)
  const xOf = (m) => C.PL + (m / h) * innerW
  const yOf = (v) => (C.PT + innerH) - (v / yMax) * innerH

  const points = series.value.map(r => ({ m: r.m, x: xOf(r.m), y: yOf(r.expV) }))
  const linePath = points.map((p, i) => (i === 0 ? 'M' : 'L') + p.x.toFixed(1) + ',' + p.y.toFixed(1)).join(' ')

  const top = series.value.map(r => xOf(r.m).toFixed(1) + ',' + yOf(r.highV).toFixed(1))
  const bottom = series.value.map(r => xOf(r.m).toFixed(1) + ',' + yOf(r.lowV).toFixed(1)).reverse()
  const bandPath = 'M' + top.join(' L') + ' L' + bottom.join(' L') + ' Z'

  const yTicks = []
  for (let i = 0; i <= 4; i++) {
    const val = yMax * i / 4
    yTicks.push({ y: yOf(val), label: compactNum(val) })
  }
  return { points, linePath, bandPath, yTicks, xTicks: xTicksFor() }
})

const revenueChart = computed(() => {
  const h = Math.max(1, inputs.horizon)
  const innerW = C.W - C.PL - C.PR
  const innerH = C.H - C.PT - C.PB
  const data = monthly.value
  const maxR = Math.max(...data.map(r => r.revenue), 1)
  const yMax = niceMax(maxR)
  const yOf = (v) => (C.PT + innerH) - (v / yMax) * innerH
  const slot = innerW / h
  const bw = Math.max(2, slot * 0.6)

  const bars = data.map(r => {
    const x = C.PL + (r.m - 0.5) * slot - bw / 2
    const y = yOf(r.revenue)
    return { m: r.m, x, y, w: bw, h: (C.PT + innerH) - y }
  })

  const yTicks = []
  for (let i = 0; i <= 4; i++) {
    const val = yMax * i / 4
    yTicks.push({ y: yOf(val), label: (inputs.currency || '') + compactNum(val) })
  }
  return { bars, yTicks, xTicks: xTicksFor() }
})

/* ---------- Formatting ---------- */
function fmt(n) {
  return Math.round(Number(n) || 0).toLocaleString('en-US')
}
function money(n) {
  return (inputs.currency || '') + Math.round(Number(n) || 0).toLocaleString('en-US')
}
function pct(n, d = 1) {
  return (Number(n) || 0).toFixed(d) + '%'
}

/* ---------- CSV export ---------- */
function downloadCsv() {
  const header = ['Month', 'Visitors', 'Leads', 'Customers', 'Revenue']
  const lines = [header.join(',')]
  for (const r of monthly.value) {
    lines.push([r.m, Math.round(r.visitors), Math.round(r.leads), Math.round(r.customers), Math.round(r.revenue)].join(','))
  }
  lines.push(['Total', Math.round(totals.value.visitors), Math.round(totals.value.leads), Math.round(totals.value.customers), Math.round(totals.value.revenue)].join(','))
  const blob = new Blob([lines.join('\n')], { type: 'text/csv;charset=utf-8;' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'traffic-conversion-forecast.csv'
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}
</script>

<style scoped>
.forecast-container {
  --black: #000;
  --orange: #FF4500;
  --border: #E5E5E5;
  --gray-text: #666;
  --font-mono: 'JetBrains Mono', monospace;
  --font-sans: 'Space Grotesk', 'Inter', system-ui, sans-serif;
  min-height: 100vh;
  background: #fff;
  color: var(--black);
  font-family: var(--font-sans);
}

/* Nav */
.navbar {
  height: 60px;
  background: var(--black);
  color: #fff;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 40px;
}
.nav-brand {
  font-family: var(--font-mono);
  font-weight: 800;
  letter-spacing: 1px;
  font-size: 1.2rem;
  color: #fff;
  text-decoration: none;
}
.nav-link {
  color: #fff;
  text-decoration: none;
  font-family: var(--font-mono);
  font-size: 0.9rem;
  transition: opacity 0.2s;
}
.nav-link:hover { opacity: 0.7; }

.main-content {
  max-width: 1400px;
  margin: 0 auto;
  padding: 50px 40px 80px;
}

/* Header */
.page-head { margin-bottom: 40px; }
.orange-tag {
  background: var(--orange);
  color: #fff;
  padding: 4px 10px;
  font-family: var(--font-mono);
  font-weight: 700;
  font-size: 0.75rem;
  letter-spacing: 1px;
}
.page-title {
  font-size: 2.8rem;
  font-weight: 500;
  letter-spacing: -1px;
  margin: 18px 0 12px;
}
.page-sub {
  color: var(--gray-text);
  max-width: 640px;
  line-height: 1.6;
}

/* Layout */
.layout {
  display: grid;
  grid-template-columns: 380px 1fr;
  gap: 36px;
  align-items: start;
}

/* Inputs */
.inputs-panel { position: sticky; top: 20px; }
.panel-block {
  border: 1px solid var(--border);
  padding: 22px;
  margin-bottom: 18px;
}
.block-header {
  font-family: var(--font-mono);
  font-size: 0.8rem;
  color: #999;
  margin-bottom: 18px;
  display: flex;
  align-items: center;
  gap: 8px;
}
.diamond { color: var(--orange); }

.field { display: flex; flex-direction: column; margin-bottom: 18px; }
.field:last-child { margin-bottom: 0; }
.field-label {
  font-size: 0.85rem;
  font-weight: 500;
  margin-bottom: 8px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.field-label em {
  font-style: normal;
  font-family: var(--font-mono);
  color: var(--orange);
  font-size: 0.8rem;
}
.field input[type="number"],
.field input[type="text"] {
  border: 1px solid var(--border);
  padding: 10px 12px;
  font-family: var(--font-mono);
  font-size: 0.9rem;
  outline: none;
  background: #FAFAFA;
}
.field input[type="number"]:focus,
.field input[type="text"]:focus { border-color: var(--orange); }
.field input[type="range"] {
  width: 100%;
  accent-color: var(--orange);
  cursor: pointer;
}
.hint {
  font-size: 0.72rem;
  color: #999;
  margin-top: 6px;
  line-height: 1.4;
}
.field-row { display: flex; gap: 12px; }
.field-row .grow { flex: 1; }
.currency-field { width: 90px; }

.overall-conv {
  margin-top: 4px;
  padding-top: 16px;
  border-top: 1px dashed var(--border);
  font-size: 0.85rem;
  color: var(--gray-text);
}
.overall-conv strong { color: var(--black); font-family: var(--font-mono); }

.source-field .field-label { font-weight: 400; }
.swatch {
  display: inline-block;
  width: 10px; height: 10px;
  border-radius: 2px;
  margin-right: 6px;
  vertical-align: middle;
}

.panel-actions { display: flex; gap: 12px; }
.ghost-btn {
  flex: 1;
  border: 1px solid var(--black);
  background: #fff;
  padding: 12px;
  font-family: var(--font-mono);
  font-size: 0.8rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}
.ghost-btn:hover { background: var(--black); color: #fff; }

/* Results */
.headline {
  border-left: 3px solid var(--orange);
  padding: 4px 0 4px 16px;
  font-size: 1.1rem;
  line-height: 1.6;
  margin-bottom: 24px;
}
.headline strong { font-family: var(--font-mono); }

.kpi-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 14px;
  margin-bottom: 24px;
}
.kpi { border: 1px solid var(--border); padding: 16px 18px; }
.kpi-label { font-size: 0.75rem; color: #999; margin-bottom: 8px; }
.kpi-value {
  font-family: var(--font-mono);
  font-size: 1.5rem;
  font-weight: 500;
}
.kpi-value.small { font-size: 1rem; }

.chart-card {
  border: 1px solid var(--border);
  padding: 22px;
  margin-bottom: 18px;
}
.chart-title {
  font-family: var(--font-mono);
  font-size: 0.85rem;
  font-weight: 600;
  margin-bottom: 16px;
}
.chart { width: 100%; height: auto; display: block; }

.grid { stroke: #EEE; stroke-width: 1; }
.axis-label { font-family: var(--font-mono); font-size: 10px; fill: #999; }
.axis-label.end { text-anchor: end; }
.axis-label.mid { text-anchor: middle; }
.band { fill: rgba(255, 69, 0, 0.12); stroke: none; }
.line { fill: none; stroke: var(--orange); stroke-width: 2; }
.dot { fill: var(--orange); }
.bar { fill: #1A1A1A; }
.bar:hover { fill: var(--orange); }

.legend {
  display: flex;
  gap: 20px;
  margin-top: 12px;
  font-family: var(--font-mono);
  font-size: 0.72rem;
  color: #888;
}
.legend-item { display: flex; align-items: center; gap: 6px; }
.legend-line { width: 18px; height: 2px; background: var(--orange); display: inline-block; }
.legend-band { width: 18px; height: 10px; background: rgba(255, 69, 0, 0.18); display: inline-block; }

/* Funnel + sources */
.dual-card { display: flex; gap: 18px; }
.dual-card .flex1 { flex: 1; margin-bottom: 18px; }

.funnel { display: flex; flex-direction: column; gap: 14px; }
.funnel-stage { display: flex; flex-direction: column; gap: 6px; }
.funnel-bar {
  height: 38px;
  border-radius: 3px;
  display: flex;
  align-items: center;
  min-width: 60px;
  transition: width 0.3s ease;
}
.funnel-count {
  color: #fff;
  font-family: var(--font-mono);
  font-size: 0.85rem;
  padding: 0 12px;
  font-weight: 600;
}
.funnel-meta {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #888;
}
.funnel-label { font-weight: 600; color: var(--black); }
.funnel-rate { font-family: var(--font-mono); }

.source-bar {
  display: flex;
  height: 28px;
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 16px;
}
.source-seg { transition: width 0.3s ease; min-width: 1px; }
.source-legend { list-style: none; padding: 0; margin: 0; }
.source-legend li {
  display: flex;
  align-items: center;
  font-size: 0.8rem;
  padding: 5px 0;
  border-bottom: 1px dashed #F0F0F0;
}
.src-pct { font-family: var(--font-mono); margin-left: auto; font-weight: 600; }
.src-vis { font-family: var(--font-mono); color: #999; margin-left: 14px; font-size: 0.72rem; }

/* Table */
.table-wrap { overflow-x: auto; }
.data-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.82rem;
}
.data-table th, .data-table td {
  text-align: right;
  padding: 8px 12px;
  border-bottom: 1px solid #F0F0F0;
  font-family: var(--font-mono);
}
.data-table th:first-child, .data-table td:first-child { text-align: left; }
.data-table thead th {
  color: #999;
  font-weight: 600;
  border-bottom: 1px solid var(--border);
}
.data-table tfoot td {
  font-weight: 700;
  border-top: 2px solid var(--black);
  border-bottom: none;
}

.disclaimer {
  font-size: 0.75rem;
  color: #aaa;
  line-height: 1.5;
  margin-top: 8px;
}

/* Responsive */
@media (max-width: 1024px) {
  .layout { grid-template-columns: 1fr; }
  .inputs-panel { position: static; }
  .dual-card { flex-direction: column; }
}
@media (max-width: 640px) {
  .main-content { padding: 30px 20px 60px; }
  .navbar { padding: 0 20px; }
  .page-title { font-size: 2rem; }
  .kpi-grid { grid-template-columns: repeat(2, 1fr); }
}
</style>
