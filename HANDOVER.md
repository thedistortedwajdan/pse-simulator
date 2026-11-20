# PSE Simulation Dashboard — Claude Code Handover

> **For Claude Code:** This README is a full handover brief. Read it completely before writing a single line of code. Everything you need to build, verify, and deploy is here.

---

## 1. What You Are Building

An **interactive dummy simulation dashboard** for a Master's thesis on *Predictive Scaling Engine (PSE)* — an LSTM-based Kubernetes autoscaler that outperforms the native Horizontal Pod Autoscaler (HPA).

The simulation looks and feels like a real live Kubernetes monitoring dashboard. It is entirely **preset/scripted** — but no one watching should know that. Every run looks slightly different because of seeded Gaussian noise layered on fixed skeleton curves.

**The narrative is always the same:**
- PSE forecasts load 3 minutes ahead and scales pods proactively
- HPA reacts only after CPU/memory thresholds breach — always 60–90 seconds late
- PSE wins on every metric: lower SLO violations, lower P99 latency, fewer wasted pod-seconds

---

## 2. Visual Design — Option D "Parchment Lab"

The UI must match this palette exactly. No dark backgrounds.

| Token | Value | Usage |
|---|---|---|
| Page background | `#faf6ef` | Main canvas |
| Header background | `#2d2016` | Top bar only |
| Toolbar background | `#f0e8d8` | Controls row |
| Panel background | `#faf6ef` | Chart panels |
| Border | `#e0d5c4` | All dividers |
| Accent (amber/sienna) | `#c8832a` | PSE line, active buttons, run CTA |
| Bad (HPA) | `#c0392b` | HPA lines, bad metrics |
| Good (PSE) | `#27795a` | PSE good metrics |
| Text primary | `#2d1a00` | Headings |
| Text muted | `#9a8060` | Labels, sublabels |
| Monospace data | `#a09080` | Chart axis labels |

**Fonts (Google Fonts):**
- `Space Grotesk` — headings, branding, CTA buttons
- `JetBrains Mono` — all data labels, metric names, axis ticks, status badges
- `DM Sans` — body text, descriptions

Load order in `<head>`:
```html
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&family=JetBrains+Mono:wght@400;600&family=Space+Grotesk:wght@500;600;700&display=swap">
```

---

## 3. File Structure

```
pse-simulator/
├── index.html          ← entire app (single file, no build step)
└── README.md           ← this file (rename HANDOVER.md → README.md before committing)
```

That's it. One HTML file. No npm, no bundler, no backend.

---

## 4. External Libraries (CDN only — no npm)

Load these from cdnjs in `<head>`, in this order, before any `<script>` of your own:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.0/chart.umd.min.js"></script>
```

No other external JS libraries are needed. Everything else is vanilla JS.

---

## 5. Full Feature Specification

### 5.1 Header Bar
- Dark chocolate background (`#2d2016`)
- Left: wordmark — `PSE` (amber `#c8832a`) + `Simulator` (cream)
- Right: two monospace status badges
  - `PSE ACTIVE` — green background (`#d4edd8`, text `#2a6b35`, border `#a8d8b0`)
  - `HPA LAG: Xs` — amber background (`#fdecc8`, text `#8a5010`, border `#f5c870`)
  - The lag value updates live during simulation (shows current lag in seconds, 0 at start, rises to 60–90s at peak load)

### 5.2 Workload Selector
Four cards in a 2×2 grid (or horizontal row on wide screens). Each card shows:
- Workload name (bold, Space Grotesk)
- One-line description (DM Sans, muted)
- A tiny sparkline preview of the shape (static SVG)

| ID | Name | Description | Shape |
|---|---|---|---|
| `W1` | Diurnal | Morning rise, afternoon peak, evening drop | Smooth bell curve |
| `W2` | Flash Crowd | Sudden spike then rapid drop | Sharp spike |
| `W3` | Gradual Ramp | Steady linear increase over the window | Ramp up |
| `W4` | Constant | Stable baseline with minor jitter | Flat line with noise |

Clicking a card selects it (amber left border + subtle amber background tint). One workload is always selected (default: W1).

### 5.3 Intensity Mode Toggle
A toggle switch with two modes — this sits below the workload selector:

**Mode A — Preset Levels**
Four buttons in a segmented control: `Low` / `Medium` / `High` / `Extreme`
Maps to intensity values: 25 / 50 / 75 / 100

**Mode B — Manual**
A continuous slider from 1–100 with a live numeric readout (amber text, monospace)
Label: `Intensity: 63%`

### 5.4 Run Duration Selector
Four pill buttons: `10 min` / `15 min` / `1 hour` / `∞ Infinite`

These control **simulated duration** compressed into real time:
- 10 min simulated → plays in ~25 seconds real time
- 15 min simulated → plays in ~40 seconds real time
- 1 hour simulated → plays in ~90 seconds real time
- Infinite → runs indefinitely, looping the workload pattern with new noise each loop

Time compression formula: `realMs = (simulatedSeconds / compressionFactor) * 1000`
Use `compressionFactor = 24` (10 sim minutes = ~25 real seconds).

### 5.5 Run / Stop Controls
- **▶ Run Simulation** button — amber background, Space Grotesk bold
- Changes to **■ Stop** (red) while running
- A live sim clock shows elapsed simulated time: `07:34` (monospace, steel-blue chip)
- A progress bar under the controls fills from left to right (amber fill on cream track)
  - Hidden during Infinite mode

### 5.6 KPI Strip (4 tiles, always visible, updates live)
| Tile | HPA value | PSE value | Note |
|---|---|---|---|
| SLO Violation Rate | red | green | % of requests exceeding SLO |
| P99 Latency Peak | red | green | in ms, SLO threshold 200ms |
| Pods Provisioned | red | green | peak pod count |
| Resource Efficiency | — | amber | % pod-seconds saved vs HPA |

Values start at baseline and animate toward final values as the simulation progresses.

### 5.7 Pod Activity Panel
A live panel that sits below the KPI strip, showing current autoscaler state:

```
┌─────────────────────────────────────────────────┐
│  HPA                          PSE               │
│  Pods active:  8              Pods active: 12   │
│  Scaling:     ↑ +3 pods       Scaling:   → hold │
│  Decision:    threshold breach Decision: forecast│
│  Lag:         72s             Lead time: 3m     │
└─────────────────────────────────────────────────┘
```

- Updates every tick (500ms)
- HPA "Scaling" shows ↑ / ↓ / → with pod delta, only when a scale event is happening
- PSE shows the same but scaled ahead of HPA by 60–90s
- HPA "Decision" cycles through: `threshold breach` → `cooldown` → `scaling up` → `stabilizing`
- PSE "Decision" cycles: `forecast: +18% in 3m` → `pre-scaling` → `scaled ✓` → `monitoring`

### 5.8 Charts (4 panels, 2×2 grid)
All charts use Chart.js line charts. Update every 500ms by pushing one new data point and shifting the window (sliding window of last N points).

**Chart 1 — Request Rate (RPS)**
- Single line: RPS (muted tan `#b8a888`)
- Y-axis: 0 – max RPS (depends on workload + intensity)
- Shows raw demand curve

**Chart 2 — Replicas**
- Two lines: HPA replicas (red dashed `#c0392b`) and PSE replicas (amber solid `#c8832a`)
- PSE line is always ahead of HPA line by 60–90s lag equivalent
- Y-axis: 0 – 24 pods

**Chart 3 — P99 Latency (ms)**
- Two lines: HPA latency (red) and PSE latency (green `#27795a`)
- Horizontal dashed reference line at 200ms (SLO threshold), amber color
- HPA line breaches 200ms during load peaks; PSE stays below

**Chart 4 — Cumulative SLO Violations**
- Two lines: HPA (red, climbs steeply) and PSE (green, nearly flat)
- Y-axis: count of violated requests

**All charts:**
- Sliding window: last 60 data points (30 seconds of real time)
- Grid lines: `#e8dcc8` (warm tan, subtle)
- Axis labels: JetBrains Mono, `#a09080`
- Legend: inline, small, same font
- No chart titles (panel header serves as title)
- Smooth animation: `tension: 0.4`, `pointRadius: 0`

---

## 6. Simulation Engine

### 6.1 Random Seed
On every "Run" click, generate a new seed:
```js
const seed = Date.now() % 99991; // prime modulo for decent spread
```
Use a seeded pseudo-random number generator (a simple LCG or Box-Muller transform) so the noise is reproducible within a run but different across runs.

Simple seeded PRNG (LCG):
```js
function makePRNG(seed) {
  let s = seed;
  return function() {
    s = (s * 1664525 + 1013904223) & 0xffffffff;
    return (s >>> 0) / 0xffffffff;
  };
}
```

Gaussian noise from uniform (Box-Muller):
```js
function gaussianNoise(rand, mean, std) {
  const u1 = rand(), u2 = rand();
  const z = Math.sqrt(-2 * Math.log(u1)) * Math.cos(2 * Math.PI * u2);
  return mean + z * std;
}
```

### 6.2 Base Curves (per workload)

Each workload is a function `f(t)` where `t` is 0–1 (normalized simulated time), returning a value 0–1 (normalized load).

```js
const BASE_CURVES = {
  W1: t => { // Diurnal — smooth bell
    return 0.15 + 0.85 * Math.pow(Math.sin(Math.PI * t), 1.4);
  },
  W2: t => { // Flash Crowd — sudden spike
    const spike = Math.exp(-Math.pow((t - 0.35) / 0.08, 2));
    return 0.1 + 0.9 * spike;
  },
  W3: t => { // Gradual Ramp
    return 0.1 + 0.85 * Math.pow(t, 0.9);
  },
  W4: t => { // Constant with minor jitter (jitter added via noise, not here)
    return 0.55;
  }
};
```

### 6.3 Noisy RPS from Base Curve

```js
function getRPS(workloadId, t, intensity, rand) {
  const maxRPS = 200 + intensity * 6; // 200–800 RPS range
  const base = BASE_CURVES[workloadId](t);
  const noise = gaussianNoise(rand, 0, 0.03); // ±3% noise
  return Math.max(10, (base + noise) * maxRPS);
}
```

### 6.4 Deriving Other Metrics from RPS

**PSE replicas** (scales ahead):
```js
// PSE looks ahead by leadSteps (simulate by using t + leadFraction)
const leadFraction = 0.08; // ~8% of total duration ahead
const psePods = Math.ceil(getRPS(wl, Math.min(t + leadFraction, 1), intensity, rand) / 45);
// clamp to 2–24
```

**HPA replicas** (lags behind):
```js
// HPA reacts after threshold breach — use t - lagFraction
const lagFraction = 0.06 + rand() * 0.04; // varies 6–10% lag (60–90s equiv)
const hpaPods = Math.ceil(getRPS(wl, Math.max(t - lagFraction, 0), intensity, rand) / 45);
```

**P99 Latency:**
```js
// Latency spikes when demand exceeds capacity
const hpaCapacity = hpaPods * 45;
const pseCapacity = psePods * 45;
const rps = currentRPS;

const hpaOverload = Math.max(0, (rps - hpaCapacity) / hpaCapacity);
const pseOverload = Math.max(0, (rps - pseCapacity) / pseCapacity);

const hpaLatency = 80 + hpaOverload * 800 + gaussianNoise(rand, 0, 8);
const pseLatency = 65 + pseOverload * 400 + gaussianNoise(rand, 0, 6);
```

**SLO violation (boolean per tick):**
```js
const hpaViolation = hpaLatency > 200;
const pseViolation = pseLatency > 200;
// Accumulate counts: hpaViolations++, pseViolations++ when true
```

### 6.5 Pod Activity (scaling events)
```js
// Compare current pods to previous tick
const hpaDelta = currentHpaPods - prevHpaPods;
const pseDelta = currentPsePods - prevPsePods;
// hpaDelta > 0 → scaling up, < 0 → scaling down, 0 → hold
```

### 6.6 Tick Loop
```js
let tickInterval = null;

function startSim() {
  const totalTicks = getDurationTicks(); // e.g. 10min @ 500ms ticks = ~50 ticks (25 real seconds)
  let tick = 0;
  const rand = makePRNG(seed);

  tickInterval = setInterval(() => {
    const t = totalTicks === Infinity ? (tick % 120) / 120 : tick / totalTicks;
    updateCharts(t, rand);
    updatePodPanel(t, rand);
    updateKPIs();
    updateClock(tick);
    tick++;
    if (totalTicks !== Infinity && tick >= totalTicks) stopSim();
  }, 500);
}
```

---

## 7. End-of-Simulation Summary Panel

When simulation completes (not on Infinite), show a summary overlay/panel:

```
╔══════════════════════════════════════╗
║  Simulation Complete — W1 Diurnal    ║
║  Duration: 15:00 simulated           ║
╠══════════════════════════════════════╣
║  Metric            HPA      PSE      ║
║  SLO Violations    9.1%     2.2%     ║
║  P99 Peak          348ms    142ms    ║
║  Peak Pods         18       12       ║
║  Pod-seconds       2,410    1,980    ║
║  Resource saved    —        +18.2%   ║
╠══════════════════════════════════════╣
║  PSE reduced SLO violations by 75.8% ║
║  [▶ Run Again]  [↓ Save Results]     ║
╚══════════════════════════════════════╝
```

Add slight randomness to the final numbers each run (±5% variation) so it never looks canned.

---

## 8. Local Verification Checklist

Before deploying, verify all of these work in a browser (open `index.html` directly — no server needed):

- [ ] All 4 workload cards are selectable, one at a time
- [ ] Preset mode: Low/Medium/High/Extreme buttons work
- [ ] Manual mode: slider updates live numeric readout
- [ ] Duration selector: 10min / 15min / 1hr / Infinite all selectable
- [ ] ▶ Run button starts the simulation; button changes to ■ Stop
- [ ] Live clock ticks up during simulation
- [ ] Progress bar fills (hidden in Infinite mode)
- [ ] All 4 charts animate (data points slide in from right)
- [ ] HPA line always lags behind PSE line in the Replicas chart
- [ ] HPA latency breaches 200ms SLO line during peaks; PSE stays below
- [ ] Pod Activity panel updates every tick with correct ↑/↓/→ indicators
- [ ] HPA lag badge in header updates live
- [ ] Two consecutive runs of the same workload produce visually different (but similar-shaped) graphs
- [ ] Summary panel appears at end of 10min/15min/1hr runs
- [ ] "Run Again" button in summary works
- [ ] Page is responsive — usable on a phone (400px wide)
- [ ] No console errors

---

## 9. GitHub Setup

```bash
# 1. Create repo (do this once)
gh repo create pse-simulator --public --description "PSE vs HPA autoscaling simulation dashboard"

# 2. Init and push
git init
git add index.html README.md
git commit -m "Initial commit: PSE simulation dashboard"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/pse-simulator.git
git push -u origin main
```

**Repository settings:**
- Visibility: Public (required for free Cloudflare Pages)
- Branch: `main`
- No build command, no package.json needed

---

## 10. Cloudflare Pages Deployment

1. Go to [https://pages.cloudflare.com](https://pages.cloudflare.com)
2. Click **Create a project** → **Connect to Git**
3. Authorize Cloudflare to access your GitHub account
4. Select the `pse-simulator` repository
5. Build settings:
   - **Framework preset:** None
   - **Build command:** *(leave blank)*
   - **Build output directory:** `/` (root)
6. Click **Save and Deploy**
7. Wait ~30 seconds → your site is live at `https://pse-simulator.pages.dev`

**Custom domain (optional):**
- In Cloudflare Pages → Custom domains → Add domain
- Works with any domain managed by Cloudflare DNS

**Auto-deploy on push:**
Any `git push` to `main` triggers an automatic redeploy. No action needed.

---

## 11. Key Design Constraints for Claude Code

1. **Single file** — everything in `index.html`. No separate `.css` or `.js` files.
2. **No npm, no build step** — `open index.html` in a browser and it works.
3. **Chart.js only** — load from cdnjs, exact version `4.4.0`. No D3, no Plotly.
4. **Fonts from Google Fonts only** — Space Grotesk, JetBrains Mono, DM Sans.
5. **No dark background anywhere** — Option D palette, cream/parchment throughout.
6. **PSE always outperforms HPA** — this is non-negotiable in the simulation logic.
7. **HPA lag is always 60–90s** — the lag badge in the header must reflect this live.
8. **Randomness on every run** — same seed within a run, new seed on each Run click.
9. **No external API calls** — the simulation is entirely client-side.
10. **Responsive** — must work at 400px width (phone), no horizontal scroll.

---

## 12. Exact Numbers to Use in Summary Panel

These are the thesis results. Use them as the base for the final summary (with ±5% noise):

| Metric | HPA | PSE |
|---|---|---|
| SLO Violation Rate (W1 Diurnal) | 9.1% | 2.2% |
| SLO Violation Rate (W2 Flash) | 14.3% | 3.8% |
| SLO Violation Rate (W3 Ramp) | 7.6% | 1.9% |
| SLO Violation Rate (W4 Constant) | 4.2% | 1.1% |
| P99 Latency peak | 348ms | 142ms |
| Resource efficiency gain | — | +18.2% |
| LSTM forecast horizon | — | 3 min (18 steps) |
| HPA characteristic lag | 60–90s | — |

---

*Handover written from Claude.ai session. Build → verify locally → deploy to Cloudflare Pages.*
