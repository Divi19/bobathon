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
<!-- To be filled in during build -->

---

### 5. Sub-task 2 — Pure Functions
<!-- To be filled in during build -->

---

### 6. Sub-task 3 — fetchForecast + Right-Now Card
<!-- To be filled in during build -->

---

### 7. Sub-task 4 — 48-Hour Timeline
<!-- To be filled in during build -->

---

### 8. Sub-task 5 — Controls (location, profile, toggles)
<!-- To be filled in during build -->

---

### 9. Sub-task 6 — bestWindow + Best Windows Box
<!-- To be filled in during build -->

---

### 10. Sub-task 7 — WhatsApp Share
<!-- To be filled in during build -->

---

### 11. Sub-task 8 — How-It-Works, Error State, Footer
<!-- To be filled in during build -->

---

### 12. Sub-task 9 — Sanity Checks, README, Final Polish
<!-- To be filled in during build -->
