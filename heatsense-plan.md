# HeatSense — Implementation Plan

## Overview

Single static file (`index.html`) — vanilla HTML + CSS + JS, no frameworks, no build step, no backend, no API keys.
Fetches from Open-Meteo → computes WBGT (Stull wet-bulb model) → classifies every hour against ACGIH thresholds → renders UI → WhatsApp share.
Hosting optional; README says "open index.html in any browser — no install needed."

---

## Pure Functions (revised — Stull 2011 model)

```js
wetBulb(T, RH)
  → number   // °C; Stull 2011, angles in radians
  // Tw = T*atan(0.151977*sqrt(RH+8.313659)) + atan(T+RH)
  //      - atan(RH-1.676331) + 0.00391838*RH^1.5*atan(0.023101*RH) - 4.686035

wbgtShade(T, RH)
  → number   // °C; = 0.7*Tw + 0.3*T

wbgt(T, RH, radiation, isDay, inSun)
  → number   // °C, 1dp; = wbgtShade + (inSun && isDay ? 3*min(radiation/800,1) : 0)
  // Sun adjustment is a rule-of-thumb estimate; label as such in UI and docs.

classify(wbgtVal, workload, acclimatized)
  → { level: 0–4, colour, label, workPct, restPct }

bestWindow(hours, workload, acclimatized, inSun, targetDay)
  → { startHour, endHour, label } | null

buildWhatsAppMessage(state)
  → string   // raw string; caller does encodeURIComponent before wa.me append
```

---

## THRESHOLDS Constant Structure

```js
const THRESHOLDS = {
  acclimatized: {
    light:     [31.0, 31.0, 32.0, 32.5],  // rows: 75–100%, 50–75%, 25–50%, 0–25%
    moderate:  [28.0, 29.0, 30.0, 31.5],
    heavy:     [null, 27.5, 29.0, 30.5],  // null = not permitted at that row
    veryHeavy: [null, null, 28.0, 30.0],
  },
  unacclimatized: {
    light:     [28.0, 28.5, 29.5, 30.0],
    moderate:  [25.0, 26.0, 27.0, 29.0],
    heavy:     [null, 24.0, 25.5, 28.0],
    veryHeavy: [null, null, 24.5, 27.0],
  },
};
// Row index 0 → Green  (normal work)
// Row index 1 → Yellow (45 min work / 15 min rest)
// Row index 2 → Orange (30 min / 30 min)
// Row index 3 → Red    (15 min / 45 min)
// Above all non-null thresholds → Purple (stop)
```

---

## Profile Defaults

| Profile key  | workload | acclimatized | inSun (default) |
|--------------|----------|--------------|-----------------|
| rider        | moderate | true         | true            |
| construction | heavy    | true         | true            |
| school       | moderate | false        | true            |
| runner       | heavy    | false        | true            |
| elderly      | light    | false        | false           |
| tourist      | moderate | false        | true            |

Both `acclimatized` and `inSun` toggles reset to profile default on profile change; user may override.

---

## Open-Meteo API Fields

```
hourly=temperature_2m,relative_humidity_2m,shortwave_radiation,is_day
timezone=Asia/Kuala_Lumpur
forecast_days=3
```

---

## Build Order

### Phase 1 — Data & Core Logic
1. `wetBulb`, `wbgtShade`, `wbgt` pure functions
2. `THRESHOLDS` constant
3. `classify` function
4. `fetchForecast(lat, lon)` → array of `{ time, T, RH, radiation, isDay }`

### Phase 2 — Right-Now Card + 48h Timeline
5. Find current hour index; slice next 48 hours
6. Render Right-Now card: T, RH, WBGT, colour badge, label, contrast line, hydration line
7. Render 48h timeline: coloured cells, 🌙 night, tap/hover detail

### Phase 3 — Controls, Best Windows, WhatsApp
8. Location dropdown + geolocation (fallback Penang/George Town)
9. Profile dropdown + acclimatized toggle + inSun toggle
10. `bestWindow` function
11. Best Windows box (today + tomorrow)
12. `buildWhatsAppMessage` + share button

### Phase 4 — Polish & How-It-Works
13. Collapsible "How it works": Stull formula, sun adjustment disclaimer, threshold table, source links
14. Footer disclaimer + "Built with IBM Bob"
15. Error state + Retry button
16. Mobile-first CSS (min-width 380px)

### Phase 5 — Verify & Document
17. `console.assert` sanity checks (see below)
18. Cross-check thresholds vs OSHA TM Sec. III Ch. 4
19. Write README.md, finalise BOB_LOG.md, add screenshots

---

## Sanity Checks (console.assert in script)

| Check | Expected |
|-------|----------|
| `wetBulb(33, 75)` | ≈ 29.2 |
| `wbgtShade(33, 75)` | ≈ 30.3 |
| `wetBulb(27, 90)` | ≈ 25.8 → `wbgtShade` ≈ 26.2 → Green, moderate, acclimatized |
| `classify(29.5, 'moderate', true)` | Orange |
| `classify(28.5, 'heavy', false)` | Purple |

---

## Ambiguities Resolved

| Ambiguity | Resolution |
|-----------|------------|
| Night hours | `is_day === 0`; mark 🌙; note sun adjustment not applied |
| bestWindow fallback | Cascade Yellow → Orange if no Green daytime block exists |
| Very Heavy workload | Kept in THRESHOLDS; not in profile dropdown |
| Toggle resets | Both acclimatized and inSun reset to profile defaults on profile change |
| 48h slice | From current hour index, next 48 entries (forecast_days=3 ensures no shortage) |
| WhatsApp URL | `window.location.href` |
| Hosting | Optional; README primary instruction is "open index.html in any browser" |

---

## File Structure

```
heatsense/
├── index.html        ← entire app (markup + CSS + JS inline)
├── README.md         ← written in Phase 5
├── BOB_LOG.md        ← maintained throughout
├── HEATSENSE_SPEC.md ← spec reference
└── screenshots/      ← added in Phase 5
```

---

## Sub-Tasks

| # | Sub-task | Status |
|---|----------|--------|
| 1 | Scaffold `index.html`: HTML structure, CSS, header, controls shell | [ ] pending |
| 2 | Pure functions: `wetBulb`, `wbgtShade`, `wbgt`, `THRESHOLDS`, `classify` | [ ] pending |
| 3 | `fetchForecast` + Right-Now card | [ ] pending |
| 4 | 48-hour timeline with colour cells and night markers | [ ] pending |
| 5 | Controls: location, geolocation, profile, acclimatized toggle, inSun toggle | [ ] pending |
| 6 | `bestWindow` + Best Windows box | [ ] pending |
| 7 | `buildWhatsAppMessage` + share button | [ ] pending |
| 8 | Collapsible How-It-Works, error state, footer | [ ] pending |
| 9 | Sanity checks, threshold verify, README, BOB_LOG finalise | [ ] pending |
