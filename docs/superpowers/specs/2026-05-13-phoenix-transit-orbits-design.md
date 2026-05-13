# Phoenix Transit Orbits — Design Spec

**Date:** 2026-05-13
**Context:** Vibe-Off Zero hackathon — make Phoenix's buses "look like something"

## Overview

A full-screen ambient web visualization of Phoenix's light rail / streetcar network as a living orbital system. Each light rail route becomes a planetary orbit, each train an energy pulse moving through space. Watchable-only, no interaction. Built as a single HTML file with Canvas 2D — no dependencies, no build step.

## Core Concept

**Trains as celestial energy.** The feed's light rail routes already have planet names (MERC, VENU, EART, MARS, JUPI, STRN). We lean into that: each route is an elliptical orbit ring radiating from a central "sun" (Phoenix). Trains are glowing particles moving along their orbits — speed controls glow intensity and trail length.

**Aesthetic:** Electric cyber-neural — cyans, magentas, warm planet colors on a deep black space background. TRON-like energy pulses. The system breathes via an independent central heartbeat.

## Data Source

GTFS-Realtime vehicle positions from Valley Metro proxy:
- Endpoint: `https://valley-metro-proxy.ccarse.workers.dev/`
- Poll rate: 1 request per 5-10 seconds (respects the 3-second rule with margin)
- Filter: only entities with trip.routeId in `{MERC, VENU, EART, MARS, JUPI, STRN}`
- Each entity provides: id, position (lat, lng, bearing, speed), trip (routeId, directionId), stopId, currentStatus, vehicle (label)

## Data Mapping

| Feed Field | Visual Mapping |
|---|---|
| `position.lat`, `position.lng` | Mapped to a position along the route's pre-defined orbital ellipse |
| `position.speed` (m/s) | Glow intensity + motion trail length. Fast = bright, long trail. Stopped = dim dot |
| `position.bearing` | Direction of movement along the orbit (ensures pulse flows correctly) |
| `trip.routeId` | Which orbit ring the train belongs to; determines color |
| `trip.directionId` | Which direction along the orbit (clockwise/counterclockwise) |

### Orbit Shapes

Each route has a hardcoded elliptical orbit, centered on the canvas center. Routes radiate outward by approximate real-world distance from downtown Phoenix, with some artistic rotation for visual balance:

| Route | Planet Color | Relative Orbit Size | Notes |
|---|---|---|---|
| MERC | Orange (#ff6600) | Innermost, tight | Streetcar — smallest loop |
| VENU | Gold (#ffcc00) | Small | Streetcar |
| EART | Cyan (#44ddff) | Medium | Light rail |
| MARS | Red (#ff3333) | Medium-large | Streetcar |
| JUPI | Bronze (#ddaa66) | Large | Streetcar |
| STRN | Beige (#ccaa88) | Outermost | Streetcar |

### Position Interpolation

Each train's lat/lng is normalized to a 0→1 parameter along its route's orbit. Since we don't have route geometry from the feed, we use a simplified approach:
- For each route in each direction, define anchor points based on observed lat/lng range from a sample of feed data
- Normalize the vehicle's current position between these anchor points
- Map the normalized value to an angle along the ellipse

## Visual Design

### Canvas Composition

- **Background:** Deep black (#000011) with subtle procedural star field (random small white dots with slight opacity variance, generated once — static, no animation)
- **Central Sun:** White circle at canvas center. Slow pulse animation (scale + opacity oscillation, ~3-second cycle). No data dependency — pure ambient heartbeat
- **Orbit Lines:** Thin dashed ellipses, one per route, in the route's planet color. Semi-transparent, clean stroke
- **Trains:** Small glowing circles (3-4px radius) with inner bright core (1-2px). Speed-controlled:
  - Speed 0: small dim dot, no trail
  - Speed low (0-5 m/s): small glow, short trail (3-5px)
  - Speed med (5-15 m/s): medium glow, medium trail (5-15px)
  - Speed high (15+ m/s): bright glow, long trail (15-30px)
  - Trail fades via opacity gradient (3-4 segments decreasing opacity)
- **Legend:** Bottom-right corner, small text listing route codes with color dots. Low opacity, non-intrusive

### Frame Loop

- 60fps requestAnimationFrame
- Each frame: clear canvas → draw star field (static) → draw central sun with pulse → draw orbit rings → update and draw train positions
- Train positions interpolate smoothly between data updates using lerp toward target position

## Technical Architecture

### Single File Structure

```
index.html
├── <style> — full-screen black, no scroll, no cursor (hide after 3s idle)
├── <canvas> — full viewport
└── <script>
    ├── Configuration (route definitions, colors, polling interval)
    ├── Data fetcher (poll proxy, parse JSON, filter routes)
    ├── Vehicle state store (current positions, lerp targets)
    ├── Star field generator (procedural, generated once)
    ├── Render loop (draw stars → sun → orbits → trains)
    └── Error/graceful degradation (offline = keep last state, dim slightly)
```

### Dependencies

None. Vanilla JavaScript, Canvas 2D API, Fetch API. No libraries, no frameworks, no build tools.

### Error Handling

- **Feed unavailable:** Trains stay at last-known positions with slightly dimmed glow. Sun continues pulsing. Orbits remain visible. No error overlays — the ambient experience doesn't break
- **No light rail data in response:** All orbits empty. Sun continues pulsing. The system is "asleep" but still alive
- **Network timeout:** Retry after a longer interval. No visual disruption

### Performance

- Vehicle count is bounded (~10-20 light rail vehicles)
- Canvas 2D at 60fps is well within budget
- Star field pre-generated, not recalculated per frame
- Single DOM element (canvas), minimal reflow

## Non-Goals

- No user interaction (no clicks, no hover, no controls)
- No audio / sonic component
- No geographic accuracy (abstract orbits, not Phoenix map)
- No bus routes — light rail only
- No persistence or session storage
- No mobile-specific optimizations beyond responsive canvas sizing

## Ship Artifacts

1. `index.html` — the complete application
2. `README.md` — one-liner explanation, endpoint credit, link to live demo
3. Live demo URL (deployed to a static host or served locally)
