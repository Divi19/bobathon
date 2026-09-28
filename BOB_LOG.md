# BOB_LOG.md — How IBM Bob was used to build HeatSense

Every entry below records a distinct step where IBM Bob contributed to the build.
Format: mode used, short prompt summary, what Bob produced, and any human verification or changes.

---

### 1. Architecture Planning
- **Mode:** Plan
- **Prompt (short):** "Read BAHANG_SPEC.md completely. Produce a short implementation plan: pure function signatures, THRESHOLDS constant structure, build order, and list any ambiguities in the spec. Also create BOB_LOG.md entry #1. Do not write index.html yet."
- **What Bob produced:** `heatsense-plan.md` — a structured plan covering function signatures (`vapourPressure`, `wbgt`, `classify`, `bestWindow`, `buildWhatsAppMessage`), the full `THRESHOLDS` constant keyed by `acclimatized → workload → [row0..row3]`, a 5-phase build order with minute targets, sanity checks, profile-to-workload mapping table, and a resolution table for 6 spec ambiguities.
- **What we verified:** Confirmed threshold numbers against spec table. Confirmed sanity check expectations are consistent with the WBGT formula and decision logic. Agreed on `null` sentinel for "not permitted" rows and Purple sentinel for "above all thresholds".

---

### 2. Caught and Fixed WBGT Overestimation in Humid Conditions
- **Mode:** Plan / Code
- **What was wrong:** The original BoM formula `WBGT = 0.567*T + 0.393*e + 3.94` was flagged for severe overestimation in tropical humidity. At T=33°C and RH=75% (a routine Malaysian afternoon), it yields a vapour pressure e ≈ 37.8 hPa and a WBGT of ≈ 37.4°C — far above every ACGIH threshold, meaning the app would show Purple (danger/stop) for essentially every daytime hour in Malaysia regardless of actual heat-stress risk. This would destroy the product's credibility and usefulness.
- **Evidence:** Formula check: `e = (75/100)*6.105*exp(17.27*33/(237.7+33)) = 37.8`; `WBGT = 0.567*33 + 0.393*37.8 + 3.94 = 18.7 + 14.9 + 3.9 = 37.5`. Every Malaysian afternoon at typical RH would be purple.
- **New model adopted:**
  - **Wet-bulb temperature** — Stull (2011) empirical formula (angles in radians):
    `Tw = T*atan(0.151977*sqrt(RH+8.313659)) + atan(T+RH) - atan(RH-1.676331) + 0.00391838*RH^1.5*atan(0.023101*RH) - 4.686035`
  - **WBGT shade:** `wbgtShade = 0.7*Tw + 0.3*T`
  - **WBGT in sun:** `wbgt = wbgtShade + (inSun && isDay ? 3*min(radiation/800, 1) : 0)` — rule-of-thumb solar correction labelled as an estimate in docs.
  - At T=33, RH=75: `Tw ≈ 29.2`, `wbgtShade ≈ 30.3` — a realistic Orange/Yellow result, not Purple.
- **What we changed:** Removed `vapourPressure()` entirely. Replaced with `wetBulb(T, RH)`, `wbgtShade(T, RH)`, and updated `wbgt(T, RH, radiation, isDay, inSun)`. Updated plan, sanity checks, and How-It-Works documentation accordingly.

---

### 3. Project Rename: Bahang → HeatSense
- **Mode:** Agent
- **Prompt (short):** "Rename the project from Bahang to HeatSense in all files, page title, header, WhatsApp message prefix, README and BOB_LOG. Tagline stays unchanged."
- **What Bob produced:** Renamed `bahang-plan.md` → `heatsense-plan.md`, updated all occurrences of "Bahang" to "HeatSense" in plan, BOB_LOG, and spec references. WhatsApp message prefix updated to `🌡 HeatSense heat alert — …`. Page `<title>` and `<h1>` updated accordingly.
- **What we verified:** Grep confirmed no remaining "Bahang" references in any tracked file.

---

### 4. Sub-task 1 — HTML Scaffold
- **Mode:** Agent / Code
- **Prompt (short):** "Scaffold index.html: full HTML structure, mobile-first CSS with colour variables for all 5 heat levels, header, controls card shell (location dropdown, geo button, profile dropdown, toggles), empty section containers for right-now card, timeline, best windows, share, how-it-works, footer."
- **What Bob produced:** Complete HTML5 skeleton with all CSS inlined; CSS custom properties for all 5 level colours and borders; all section containers with correct IDs; controls markup including both acclimatized and inSun checkboxes; WhatsApp SVG icon inline; accessible timeline cell structure; collapsible How-It-Works `<details>`; footer with disclaimer.
- **What we verified:** Markup structure matched spec Section 7. All CSS colour classes (level-green/yellow/orange/red/purple and border variants) confirmed present.

---

### 5. Sub-task 2 — Pure Functions
- **Mode:** Agent / Code
- **Prompt (short):** "Implement all pure functions: wetBulb (Stull 2011), wbgtShade, wbgt (with solar correction), THRESHOLDS constant, classify, bestWindow, buildWhatsAppMessage."
- **What Bob produced:** `wetBulb(T,RH)` — full Stull 2011 formula, five terms, angles in radians via `Math.atan`; `wbgtShade` — 0.7·Tw+0.3·T; `wbgt` — shade plus rule-of-thumb solar correction, rounded to 1dp; `THRESHOLDS` with null sentinels; `classify()` walks rows 0→3 returning Purple if above all non-null thresholds; `bestWindow()` cascades maxLevel 0→3; `buildWhatsAppMessage()` — 7-line spec-compliant template.
- **What we verified:** Sanity checks via `console.assert` block: wetBulb(33,75)≈29.2 ✓, wbgtShade(33,75)≈30.3 ✓, wetBulb(27,90)≈25.8 ✓, classify(29.5,'moderate',true)=orange ✓, classify(28.5,'heavy',false)=purple ✓.

---

### 6. Sub-task 3 — fetchForecast + Right-Now Card
- **Mode:** Agent / Code
- **Prompt (short):** "Implement fetchForecast with Open-Meteo (fields: temperature_2m, relative_humidity_2m, shortwave_radiation, is_day; timezone Asia/KL; forecast_days=3). Parse into state.hours. Find current hour. Render right-now card."
- **What Bob produced:** `fetchForecast()` async/await; `parseForecast()` mapping all four hourly arrays to structured objects; current-hour detection by ISO string after rounding `new Date()` to the hour; `renderRightNow()` with badge, large WBGT, label, stats row, contrast sentence, hydration box; `showLoading()` and `showError()` with Retry.
- **What we verified:** Open-Meteo URL confirmed with all four fields and correct timezone. Error path tested with disconnected state — shows Retry button.

---

### 7. Sub-task 4 — 48-Hour Timeline
- **Mode:** Agent / Code
- **Prompt (short):** "Implement renderTimeline(): 48-hour horizontal scroll of coloured cells, night marker 🌙, hover/tap tooltip."
- **What Bob produced:** Horizontally scrollable flex container; each cell coloured by `level-{key}` class; `tl-hour`, `tl-wbgt`, `tl-night` spans; inline tooltip shown on `:hover` and `.active` (keyboard Enter/Space); `tabindex=0`, `role=button`, `aria-label` for accessibility; CSS tooltip arrow via `::after`.
- **What we verified:** Night cells (isDay===0) show 🌙. Tooltip appears on hover. Keyboard activation toggles `.active` class.

---

### 8. Sub-task 5 — Controls (location, profile, toggles)
- **Mode:** Agent / Code
- **Prompt (short):** "Wire all 5 controls: location re-fetches; geo falls back silently; profile resets both toggles to defaults then re-renders; individual toggles update state and re-render."
- **What Bob produced:** Event listeners for all 5 controls. Profile change resets both `acclimatized` and `inSun` toggles from `PROFILES` constant. Geolocation falls back silently. Location change updates `state.cityName` from `option.text`. Toggle changes call `render()` only (no re-fetch).
- **What we verified:** Confirmed profile change resets both toggles to profile defaults per spec.

---

### 9. Sub-task 6 — bestWindow + Best Windows Box
- **Mode:** Agent / Code
- **Prompt (short):** "Implement bestWindow() and renderBestWindows(): find today's and tomorrow's best consecutive daytime block."
- **What Bob produced:** `bestWindow()` — outer loop over maxLevel 0→3, inner loop over daytime hours for target date, tracks run length, returns first level with any consecutive block. `renderBestWindows()` — two rows (today/tomorrow) with time range and colour badge.
- **What we verified:** Cascade logic confirmed: tries Green first, falls back Yellow → Orange. Returns null when all hours Red/Purple.

---

### 10. Sub-task 7 — WhatsApp Share
- **Mode:** Agent / Code
- **Prompt (short):** "Implement renderShare(): compute worst level, build WhatsApp message, set href."
- **What Bob produced:** `renderShare()` — worst classification over next 12 daytime hours; calls `buildWhatsAppMessage()`; sets `link.href` to `https://wa.me/?text=` + `encodeURIComponent(msg)`.
- **What we verified:** `encodeURIComponent` applied before wa.me. Message matches spec 7-line template.

---

### 11. Sub-task 8 — How-It-Works, Error State, Footer
- **Mode:** Agent / Code
- **Prompt (short):** "Add collapsible How-It-Works with Stull formula, sun disclaimer, threshold table, source links; error state with Retry; footer."
- **What Bob produced:** `<details>/<summary>` collapsible with CSS triangle; Stull formula in `<code>`; limitation note box; colour-coded threshold table; three source links (Stull 2011, OSHA TM Ch.4, Open-Meteo docs); Retry wired to `fetchForecast()`; footer with disclaimer and "Built with IBM Bob".
- **What we verified:** Limitation note clearly labels solar correction as rule-of-thumb estimate per spec.

---

### 12. Sub-task 9 — Sanity Checks, README, Final Polish
<!-- To be filled in during sub-task 9 -->
