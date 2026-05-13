# Phoenix Transit Orbits — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file HTML canvas visualization that renders Phoenix light rail vehicles as energy pulses orbiting a central sun.

**Architecture:** One `index.html` file containing all CSS, HTML, and vanilla JS. Canvas 2D render loop at 60fps. Periodic fetch from GTFS-Realtime proxy, filtered to light rail routes (MERC, VENU, EART, MARS, JUPI, STRN). Each vehicle is tracked by ID and advanced along its route's orbit based on real speed data.

**Tech Stack:** HTML5 Canvas 2D, Fetch API, requestAnimationFrame. Zero dependencies.

**Key Design Note on Position Mapping:** Since we don't have route geometry from the GTFS feed, vehicles are NOT spatially mapped from lat/lng. Instead, each vehicle is tracked by its unique `vehicle.id`. On first appearance, it's placed at a random angle on its route's orbit. Each poll advances it along the orbit by `speed × elapsed_seconds`, in the direction indicated by `bearing`. This keeps the real-data connection (route, speed, bearing) while producing smooth, continuous motion on abstract orbits.

---

### Task 1: Project Skeleton

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create minimal HTML shell**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Phoenix Transit Orbits</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  html, body { width: 100%; height: 100%; overflow: hidden; background: #000011; }
  canvas { display: block; }
</style>
</head>
<body>
<canvas id="c"></canvas>
<script>
// All code goes here
</script>
</body>
</html>
```

- [ ] **Step 2: Add canvas resize handler**

```js
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');

function resize() {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
}
window.addEventListener('resize', resize);
resize();
```

- [ ] **Step 3: Verify the file opens and shows a black screen**

Open `index.html` in a browser. Should see a full black canvas.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add project skeleton with full-screen canvas"
```

---

### Task 2: Route Configuration

**Files:**
- Modify: `index.html` (add route config block inside `<script>`)

- [ ] **Step 1: Define route configuration after the resize handler**

```js
const ROUTES = [
  {
    id: 'MERC', name: 'Mercury', color: '#ff6600', glow: '#ffaa33',
    rx: 0.08, ry: 0.05, tilt: -10
  },
  {
    id: 'VENU', name: 'Venus', color: '#ffcc00', glow: '#ffee44',
    rx: 0.14, ry: 0.09, tilt: 15
  },
  {
    id: 'EART', name: 'Earth', color: '#44ddff', glow: '#88eeff',
    rx: 0.22, ry: 0.14, tilt: -5
  },
  {
    id: 'MARS', name: 'Mars', color: '#ff3333', glow: '#ff7777',
    rx: 0.30, ry: 0.19, tilt: 8
  },
  {
    id: 'JUPI', name: 'Jupiter', color: '#ddaa66', glow: '#eebb77',
    rx: 0.38, ry: 0.24, tilt: -8
  },
  {
    id: 'STRN', name: 'Saturn', color: '#ccaa88', glow: '#ddbb99',
    rx: 0.46, ry: 0.29, tilt: 12
  }
];

const LIGHT_RAIL_ROUTES = new Set(ROUTES.map(r => r.id));
const POLL_INTERVAL = 7000; // ms
```

Note: `rx` and `ry` are fractions of the smaller canvas dimension. They'll be multiplied by `Math.min(canvas.width, canvas.height)` at render time so the orbits scale with the viewport.

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add route configuration with orbit sizes and planet colors"
```

---

### Task 3: Star Field

**Files:**
- Modify: `index.html` (add star generation after route config)

- [ ] **Step 1: Add star field generation**

```js
const STARS = [];
const STAR_COUNT = 200;

function generateStars() {
  STARS.length = 0;
  for (let i = 0; i < STAR_COUNT; i++) {
    STARS.push({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      r: Math.random() * 1.2 + 0.3,
      opacity: Math.random() * 0.5 + 0.1
    });
  }
}

// Regenerate on resize
window.addEventListener('resize', generateStars);
generateStars();

function drawStars(ctx) {
  for (const s of STARS) {
    ctx.globalAlpha = s.opacity;
    ctx.fillStyle = '#ffffff';
    ctx.beginPath();
    ctx.arc(s.x, s.y, s.r, 0, Math.PI * 2);
    ctx.fill();
  }
  ctx.globalAlpha = 1;
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add procedural star field"
```

---

### Task 4: Data Fetching

**Files:**
- Modify: `index.html` (add data fetching + vehicle state after star generator)

- [ ] **Step 1: Add fetcher function**

```js
const VEHICLES = {}; // keyed by vehicle.id
const DATA_ENDPOINT = 'https://valley-metro-proxy.ccarse.workers.dev/';

async function fetchVehicleData() {
  try {
    const res = await fetch(DATA_ENDPOINT);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const data = await res.json();
    return data.entity || [];
  } catch (err) {
    console.warn('Feed fetch failed:', err.message);
    return null;
  }
}
```

- [ ] **Step 2: Add polling loop**

```js
let lastFetchTime = 0;

async function poll() {
  const entities = await fetchVehicleData();
  const now = Date.now();

  if (entities) {
    const dt = lastFetchTime ? (now - lastFetchTime) / 1000 : 0;
    lastFetchTime = now;
    updateVehicles(entities, dt);
  }
  // else: entities is null (fetch failed) — vehicles keep last state
}

setInterval(poll, POLL_INTERVAL);
poll(); // initial fetch
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add GTFS-RT data fetching and polling"
```

---

### Task 5: Vehicle State Management

**Files:**
- Modify: `index.html` (add vehicle update logic before the poll function)

- [ ] **Step 1: Add vehicle update function**

```js
function updateVehicles(entities, dt) {
  const seen = new Set();

  for (const entity of entities) {
    const v = entity.vehicle;
    if (!v || !v.vehicle || !v.trip) continue;

    const routeId = v.trip.routeId;
    if (!LIGHT_RAIL_ROUTES.has(routeId)) continue;

    const route = ROUTES.find(r => r.id === routeId);
    if (!route) continue;

    const vid = v.vehicle.id;
    seen.add(vid);

    if (!VEHICLES[vid]) {
      // New vehicle: spawn at random angle on its orbit
      VEHICLES[vid] = {
        id: vid,
        routeId: routeId,
        angle: Math.random() * Math.PI * 2,
        speed: v.position?.speed || 0,
        label: v.vehicle.label || '',
        bearing: v.position?.bearing || 0,
      };
    }

    const veh = VEHICLES[vid];

    // Calculate angular velocity from speed
    // 1 m/s = ~0.03 rad/s along orbit (tuned for visual effect)
    const angularVelocity = veh.speed * 0.03;

    // Direction from bearing (0=North=UP, 90=East=RIGHT)
    // Map to clockwise: North/UP = clockwise if on right side of canvas
    // Simplify: use bearing to determine direction sign
    const bearing = v.position?.bearing ?? veh.bearing;
    const dir = bearing >= 0 && bearing <= 180 ? 1 : -1;

    // Advance angle
    veh.angle += angularVelocity * dir * dt;
    if (veh.angle > Math.PI * 2) veh.angle -= Math.PI * 2;
    if (veh.angle < 0) veh.angle += Math.PI * 2;

    veh.speed = v.position?.speed || 0;
    veh.label = v.vehicle.label || veh.label;
    veh.bearing = bearing;
  }

  // Remove vehicles no longer in the feed
  for (const vid of Object.keys(VEHICLES)) {
    if (!seen.has(vid)) delete VEHICLES[vid];
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add vehicle state tracking with speed-based orbit movement"
```

---

### Task 6: Central Sun Pulse

**Files:**
- Modify: `index.html` (add sun drawing after drawStars)

- [ ] **Step 1: Add sun drawing function**

```js
function drawSun(ctx, cx, cy, scale, t) {
  const pulse = 1 + Math.sin(t / 3000 * Math.PI * 2) * 0.3; // 3s cycle
  const r = scale * 0.012 * pulse;
  const opac = 0.6 + Math.sin(t / 3000 * Math.PI * 2) * 0.3;

  // Outer glow rings
  ctx.globalAlpha = opac * 0.3;
  ctx.strokeStyle = '#ffffff';
  ctx.lineWidth = 0.5;
  ctx.beginPath();
  ctx.arc(cx, cy, r * 1.8, 0, Math.PI * 2);
  ctx.stroke();
  ctx.beginPath();
  ctx.arc(cx, cy, r * 2.5, 0, Math.PI * 2);
  ctx.stroke();

  // Core
  ctx.globalAlpha = opac;
  ctx.fillStyle = '#ffffff';
  ctx.beginPath();
  ctx.arc(cx, cy, r, 0, Math.PI * 2);
  ctx.fill();

  ctx.globalAlpha = 1;
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add central sun with pulse animation"
```

---

### Task 7: Orbit Ring Rendering

**Files:**
- Modify: `index.html` (add orbit drawing after drawSun)

- [ ] **Step 1: Add orbit drawing function**

```js
function drawOrbits(ctx, cx, cy, scale) {
  for (const route of ROUTES) {
    const rx = route.rx * scale;
    const ry = route.ry * scale;

    ctx.save();
    ctx.translate(cx, cy);
    ctx.rotate((route.tilt * Math.PI) / 180);

    ctx.globalAlpha = 0.35;
    ctx.strokeStyle = route.color;
    ctx.lineWidth = 0.8;
    ctx.setLineDash([4, 3]);
    ctx.beginPath();
    ctx.ellipse(0, 0, rx, ry, 0, 0, Math.PI * 2);
    ctx.stroke();
    ctx.setLineDash([]);
    ctx.globalAlpha = 1;

    ctx.restore();
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add orbit ring rendering"
```

---

### Task 8: Train Rendering with Trails

**Files:**
- Modify: `index.html` (add train drawing after drawOrbits)

- [ ] **Step 1: Add train drawing function**

```js
function drawTrains(ctx, cx, cy, scale) {
  for (const vid of Object.keys(VEHICLES)) {
    const v = VEHICLES[vid];
    const route = ROUTES.find(r => r.id === v.routeId);
    if (!route) continue;

    const rx = route.rx * scale;
    const ry = route.ry * scale;
    const tiltRad = (route.tilt * Math.PI) / 180;
    const angle = v.angle;

    // Position on rotated ellipse
    const x = cx + Math.cos(angle) * rx * Math.cos(tiltRad) - Math.sin(angle) * ry * Math.sin(tiltRad);
    const y = cy + Math.cos(angle) * rx * Math.sin(tiltRad) + Math.sin(angle) * ry * Math.cos(tiltRad);

    const speed = v.speed;

    // Glow intensity and trail length based on speed thresholds
    let glowAlpha, trailLen, dotRadius;
    if (speed === 0) {
      glowAlpha = 0.3; trailLen = 0; dotRadius = 1.5;
    } else if (speed < 5) {
      glowAlpha = 0.4; trailLen = 4; dotRadius = 2;
    } else if (speed < 15) {
      glowAlpha = 0.6; trailLen = 10; dotRadius = 2.5;
    } else {
      glowAlpha = 0.9; trailLen = 20; dotRadius = 3;
    }

    // Draw trail (motion line behind the train)
    if (trailLen > 0) {
      const dir = v.bearing >= 0 && v.bearing <= 180 ? 1 : -1;
      const trailAngle = angle - dir * 0.1; // slightly behind

      const tx = cx + Math.cos(trailAngle) * rx;
      const ty = cy + Math.sin(trailAngle) * ry;

      // Multi-segment trail with fading opacity
      for (let i = 3; i >= 0; i--) {
        const segAngle = angle - dir * (i + 1) * 0.015;
        const segX = cx + Math.cos(segAngle) * rx * Math.cos(tiltRad) - Math.sin(segAngle) * ry * Math.sin(tiltRad);
        const segY = cy + Math.cos(segAngle) * rx * Math.sin(tiltRad) + Math.sin(segAngle) * ry * Math.cos(tiltRad);
        ctx.globalAlpha = glowAlpha * (0.05 + 0.1 * (3 - i));
        ctx.strokeStyle = route.glow;
        ctx.lineWidth = 0.5;
        ctx.beginPath();
        ctx.moveTo(x, y);
        ctx.lineTo(segX, segY);
        ctx.stroke();
      }
    }

    // Outer glow
    ctx.globalAlpha = glowAlpha * 0.4;
    ctx.fillStyle = route.glow;
    ctx.beginPath();
    ctx.arc(x, y, dotRadius * 1.8, 0, Math.PI * 2);
    ctx.fill();

    // Inner bright core
    ctx.globalAlpha = glowAlpha;
    ctx.fillStyle = '#ffffff';
    ctx.beginPath();
    ctx.arc(x, y, dotRadius * 0.5, 0, Math.PI * 2);
    ctx.fill();

    ctx.globalAlpha = 1;
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add train rendering with speed-based glow and trails"
```

---

### Task 9: Legend Overlay

**Files:**
- Modify: `index.html` (add legend HTML after `<canvas>`)

- [ ] **Step 1: Add legend in the DOM**

```html
<div id="legend" style="position:fixed;bottom:16px;right:16px;color:#ffffff;font-family:monospace;font-size:11px;opacity:0.5;pointer-events:none;">
  <div style="margin-bottom:4px;opacity:0.7">PHOENIX TRANSIT ORBITS</div>
  <div id="legend-entries"></div>
</div>
```

- [ ] **Step 2: Populate legend entries from route config**

```js
function buildLegend() {
  const container = document.getElementById('legend-entries');
  container.innerHTML = ROUTES.map(r =>
    `<div style="display:flex;align-items:center;gap:6px;margin-bottom:2px">
      <span style="width:8px;height:8px;border-radius:50%;background:${r.color};display:inline-block;"></span>
      <span>${r.name}</span>
    </div>`
  ).join('');
}
buildLegend();
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add legend overlay with route names and colors"
```

---

### Task 10: Render Loop

**Files:**
- Modify: `index.html` (add the main render loop)

- [ ] **Step 1: Add the render loop**

```js
function render() {
  const w = canvas.width;
  const h = canvas.height;
  const cx = w / 2;
  const cy = h / 2;
  const scale = Math.min(w, h);
  const t = performance.now();

  ctx.clearRect(0, 0, w, h);

  drawStars(ctx);         // static star field
  drawOrbits(ctx, cx, cy, scale);  // orbit rings
  drawTrains(ctx, cx, cy, scale);  // vehicles on orbits
  drawSun(ctx, cx, cy, scale, t);  // sun on top

  requestAnimationFrame(render);
}

requestAnimationFrame(render);
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add main render loop"
```

---

### Task 11: Cursor Hiding

**Files:**
- Modify: `index.html` (add cursor hide in `<style>` and a timer in `<script>`)

- [ ] **Step 1: Add CSS for cursor hiding**

Add inside `<style>`:
```css
  html.cursor-hidden { cursor: none; }
  html.cursor-hidden #legend { opacity: 0; transition: opacity 0.5s; }
```

- [ ] **Step 2: Add idle timer to hide cursor after 3 seconds**

```js
let idleTimer;
function resetIdleTimer() {
  document.documentElement.classList.remove('cursor-hidden');
  const legend = document.getElementById('legend');
  if (legend) legend.style.opacity = '';
  clearTimeout(idleTimer);
  idleTimer = setTimeout(() => {
    document.documentElement.classList.add('cursor-hidden');
    const legend = document.getElementById('legend');
    if (legend) legend.style.opacity = '0.3';
  }, 3000);
}
document.addEventListener('mousemove', resetIdleTimer);
document.addEventListener('touchstart', resetIdleTimer);
resetIdleTimer();
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: hide cursor and dim legend after 3s idle"
```

---

### Task 12: Error Handling & Graceful Degradation

**Files:**
- Modify: `index.html` (add error state handling)

- [ ] **Step 1: Add fetch failure tracking**

Modify the `poll` function to track consecutive failures and dim the display:

```js
let consecutiveFailures = 0;
const MAX_FAILURES_BEFORE_DIM = 3;

async function poll() {
  const entities = await fetchVehicleData();
  const now = Date.now();

  if (entities) {
    consecutiveFailures = 0;
    const dt = lastFetchTime ? (now - lastFetchTime) / 1000 : 0;
    lastFetchTime = now;
    updateVehicles(entities, dt);
  } else {
    consecutiveFailures++;
  }
}
```

- [ ] **Step 2: Pass dim state to render**

```js
function isDimmed() {
  return consecutiveFailures >= MAX_FAILURES_BEFORE_DIM;
}
```

- [ ] **Step 3: Apply dimming in drawTrains and drawOrbits**

In `drawTrains`, check `isDimmed()` and multiply `glowAlpha` by 0.4 if dimmed. In `drawOrbits`, multiply `ctx.globalAlpha` by 0.2 if dimmed.

Add the dim check at the top of both functions:
```js
const dim = isDimmed();
```

Then apply:
```js
// In drawOrbits: ctx.globalAlpha = dim ? 0.07 : 0.35;
// In drawTrains glowAlpha calculations: multiply by (dim ? 0.3 : 1) at the end
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add graceful degradation when feed is unavailable"
```

---

### Task 13: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README**

```markdown
# Phoenix Transit Orbits

A full-screen ambient visualization of Phoenix's light rail and streetcar network as a living orbital system. Each route becomes a planetary orbit, each train an energy pulse moving through space.

Built for [Sibi Vibe-Off Zero](https://artifact.sibi.fun/public/a/vibe-off).

## Data

Live GTFS-Realtime vehicle positions from Valley Metro, served via [valley-metro-proxy](https://valley-metro-proxy.ccarse.workers.dev/).

## Run

Open `index.html` in any modern browser. No build step, no dependencies.

## Routes

| Orbit | Route | Planet |
|-------|-------|--------|
| Innermost | MERC | Mercury |
| Small | VENU | Venus |
| Medium | EART | Earth |
| Medium-Large | MARS | Mars |
| Large | JUPI | Jupiter |
| Outermost | STRN | Saturn |

## License

MIT
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with project description and data credit"
```

---

### Task 14: Verify & Test

- [ ] **Step 1: Open index.html in browser**

Expected: Dark space background with star field, 6 colored orbit ellipses centered on screen, a pulsing white sun at center. After ~7 seconds, light rail vehicles should appear as glowing dots moving along their orbits.

- [ ] **Step 2: Verify data fetch works**

Open browser console. Should see no errors. If a vehicle is moving, the speed should cause visible trail + brightness variation.

- [ ] **Step 3: Verify idle cursor hide**

Don't move mouse for 3 seconds. Cursor should disappear. Move mouse — cursor reappears.

- [ ] **Step 4: Test graceful degradation**

Add an inline fetch mock to test failure:
```js
// Temporarily replace fetchVehicleData to test
async function fetchVehicleData() { return null; }
```
Expected: orbits and sun remain visible, no trains, no errors. Visuals dim after a few polls.

- [ ] **Step 5: Verify no console errors on sustained run**

Let it run for 2-3 minutes. No console errors, no memory leaks visible in Chrome task manager.

- [ ] **Step 6: Commit any final tweaks**

```bash
git add index.html
git commit -m "chore: final verification tweaks"
```
