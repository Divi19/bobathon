# HeatSense — Heat-Safety Planner for People Who Work and Play Outdoors

Hackathon: Build with BOB — Mini Bob-a-thon (IBM × TD SYNNEX / Tec D × Developer Kaki)
Build window: 40 minutes · Deliverable: working prototype in a public GitHub repo

This file is the single source of truth. IBM Bob: read this whole file before writing any code.

---

## 1. One-liner

HeatSense turns the hourly weather forecast into a heat-stress timeline (WBGT) and tells outdoor workers, schools, athletes and families when it is safe to work or exercise, how hard, and how long to rest — then generates a WhatsApp message a supervisor or teacher can forward to their group.

---

## 2. The Problem

In Malaysia it is humidity, not just temperature, that makes heat dangerous. At ~33°C and ~75% humidity, sweat cannot evaporate and the body cannot cool itself.
Heat warnings reaching the public are usually communicated as air temperature. The metric occupational-health professionals actually use is WBGT (Wet Bulb Globe Temperature), which includes humidity.
No free, simple tool translates WBGT into plain actions: "move heavy work to 7–10am", "rest 30 minutes every hour", "drink a cup of water every 15–20 minutes".
The people most exposed — delivery riders, construction and plantation crews, schoolchildren at PE or assembly, amateur runners, the elderly — never see this information.

---

## 3. Who Benefits (wide audience)

| Profile | Workload used | Threshold set | Why they care |
|---------|--------------|---------------|---------------|
| Delivery rider (Grab / Foodpanda / Lalamove) | Moderate | Acclimatized (TLV) | Schedule breaks, avoid collapse on the road |
| Construction / plantation crew | Heavy | Acclimatized (TLV) | Work/rest cycles; supervisor has a paper trail for workplace-safety duty of care |
| School PE / assembly | Moderate | Unacclimatized (Action Limit) — conservative for children | Decide whether to move outdoor activity indoors |
| Runner / futsal / sports event | Heavy | Unacclimatized (Action Limit) | When to train; whether to reschedule a run or match |
| Elderly / light outdoor chores | Light | Unacclimatized (Action Limit) — conservative | When not to garden or walk to the market |
| Tourist / hiker | Moderate | Unacclimatized (Action Limit) | Plan Penang Hill / trail hikes safely |

A toggle "Used to working in heat (acclimatized)?" lets the user override the default threshold set.

---

## 4. Why This Is NOT Slop

- No LLM in the product, nothing to hallucinate. Output is deterministic: a public weather forecast + a published heat-stress formula + a published occupational-health threshold table.
- No API key, no backend, no cost. Runs as one static page; anyone can use it on a phone.
- Distribution built in: a WhatsApp share message means a single foreman / teacher / organizer reaches their whole group without anyone installing anything.
- Sustainable: the weather API is free, and there is no data to store (no personal data collected — PDPA-friendly).

---

## 5. The Science

### 5.1 WBGT Model — Stull (2011) Wet-Bulb + Solar Correction

**Step 1 — Wet-bulb temperature** (Stull 2011, angles in radians):
```
Tw = T*atan(0.151977*sqrt(RH + 8.313659))
   + atan(T + RH)
   - atan(RH - 1.676331)
   + 0.00391838 * RH^1.5 * atan(0.023101 * RH)
   - 4.686035
```

**Step 2 — WBGT in shade:**
```
wbgtShade = 0.7 * Tw + 0.3 * T
```

**Step 3 — Solar correction (rule-of-thumb estimate):**
```
wbgt = wbgtShade + (inSun && isDay ? 3 * min(radiation / 800, 1) : 0)
```
Round to 1 decimal place. Label the sun adjustment as a rule-of-thumb estimate in all documentation and UI.

T = air temperature (°C), RH = relative humidity (%), radiation = shortwave radiation (W/m²).

**Sanity checks:**
- wetBulb(33, 75) ≈ 29.2 → wbgtShade ≈ 30.3 (realistic Orange/Yellow for a Malaysian afternoon)
- wetBulb(27, 90) ≈ 25.8 → wbgtShade ≈ 26.2 → Green for moderate, acclimatized

### 5.2 Work/Rest Thresholds — ACGIH TLV and Action Limit (WBGT, °C)

Source: OSHA Technical Manual, Section III Chapter 4 (Heat Stress), which reproduces the ACGIH table.

**Acclimatized workers — TLV**

| Work in each hour | Light | Moderate | Heavy | Very heavy |
|-------------------|-------|----------|-------|------------|
| 75–100% (continuous) | 31.0 | 28.0 | — | — |
| 50–75% | 31.0 | 29.0 | 27.5 | — |
| 25–50% | 32.0 | 30.0 | 29.0 | 28.0 |
| 0–25% | 32.5 | 31.5 | 30.5 | 30.0 |

**Unacclimatized — Action Limit**

| Work in each hour | Light | Moderate | Heavy | Very heavy |
|-------------------|-------|----------|-------|------------|
| 75–100% (continuous) | 28.0 | 25.0 | — | — |
| 50–75% | 28.5 | 26.0 | 24.0 | — |
| 25–50% | 29.5 | 27.0 | 25.5 | 24.5 |
| 0–25% | 30.0 | 29.0 | 28.0 | 27.0 |

"—" means that work/rest level is not permitted for that workload.

### 5.3 Decision Logic (per hour, per workload)

Walk rows from least to most restrictive; pick first row whose threshold ≥ current WBGT:

| Result | Colour | Label shown to user |
|--------|--------|---------------------|
| 75–100% row | 🟢 Green | "Normal work. Drink water regularly." |
| 50–75% row | 🟡 Yellow | "Work 45 min, rest 15 min in shade." |
| 25–50% row | 🟠 Orange | "Work 30 min, rest 30 min in shade." |
| 0–25% row | 🔴 Red | "Work 15 min, rest 45 min in shade. Essential work only." |
| Above every threshold | 🟣 Purple | "STOP strenuous activity. Danger of heat stroke." |

### 5.4 Hydration and First-Aid Copy (fixed text)

**Hydration:** "Drink about 1 cup (250 ml) of water every 15–20 minutes when working in heat — even if not thirsty. Don't drink more than about 1.5 L per hour."

**Heat-stroke warning signs:** confusion or slurred speech, hot and dry skin or heavy sweating, fainting, seizures. Call 999 immediately, move the person to shade, cool them with water and fanning.

**Footer disclaimer:** "HeatSense gives general guidance based on published heat-stress standards and a forecast estimate. It is not medical advice. Follow your employer's safety rules and local authority warnings."

---

## 6. Data Source — Open-Meteo (free, no API key, CORS-enabled)

```
https://api.open-meteo.com/v1/forecast
  ?latitude={lat}&longitude={lon}
  &hourly=temperature_2m,relative_humidity_2m,shortwave_radiation,is_day
  &timezone=Asia/Kuala_Lumpur
  &forecast_days=3
```

Call directly from browser JavaScript with fetch.
Handle errors: show a friendly message and a "Retry" button if the fetch fails.

**Preset locations:**

| City | Lat | Lon |
|------|-----|-----|
| Penang (George Town) — default | 5.4141 | 100.3288 |
| Kuala Lumpur | 3.1390 | 101.6869 |
| Johor Bahru | 1.4927 | 103.7414 |
| Kota Kinabalu | 5.9804 | 116.0735 |
| Kuching | 1.5533 | 110.3592 |
| Singapore | 1.3521 | 103.8198 |

Plus a "📍 Use my location" button (navigator.geolocation), falling back silently to Penang if denied.

---

## 7. Product Spec (UI)

Single page, mobile-first, clean and serious (safety tool, not a toy). Works at 380px width.

- **Header:** "HeatSense 🌡" + tagline "Know when the heat becomes dangerous — hour by hour."
- **Controls:** location dropdown + "Use my location" button · profile dropdown · acclimatized toggle · "Working in direct sun" toggle (defaults per profile).
- **"Right now" card:** current air temp, humidity, WBGT, colour badge, the plain-language label, hydration line. Contrast line, e.g. "Air temp 32°C sounds fine — but heat stress is in the ORANGE zone."
- **48-hour timeline:** one cell per hour, coloured by 5.3, showing hour and WBGT. Night hours visibly marked (🌙). Tap/hover a cell to see details.
- **"Best windows" box:** for today and tomorrow, the longest consecutive daytime block that is Green (or least restrictive available).
- **Share to WhatsApp button:** opens `https://wa.me/?text=` + encodeURIComponent(message). Message template:
  ```
  🌡 HeatSense heat alert — {City}, {Day}
  Profile: {Profile}
  ⚠ {Worst period}: {label}
  ✅ Best time: {best window}
  💧 Drink 1 cup of water every 15–20 min.
  🚨 Heat stroke signs (confusion, collapse, hot skin) → call 999.
  Check your area: {page URL}
  ```
- **Collapsible "How it works" section:** Stull WBGT formula, sun adjustment disclaimer, threshold table, limitations, and source links.
- **Footer:** disclaimer (5.4) + "Built with IBM Bob".

---

## 8. Technical Architecture

```
heatsense/
├── index.html      # markup + styles + script inline (single file, no build step)
├── README.md
├── BOB_LOG.md
├── HEATSENSE_SPEC.md
└── screenshots/
```

Plain HTML + CSS + vanilla JavaScript. No frameworks, no build tools, no backend, no API keys.

**Pure functions:**
```js
wetBulb(T, RH)
wbgtShade(T, RH)
wbgt(T, RH, radiation, isDay, inSun)
classify(wbgt, workload, acclimatized) → {level, colour, label}
bestWindow(hours, workload, acclimatized, inSun, day)
buildWhatsAppMessage(state)
```

**Profile defaults:**

| Profile | workload | acclimatized | inSun |
|---------|----------|--------------|-------|
| rider | moderate | true | true |
| construction | heavy | true | true |
| school | moderate | false | true |
| runner | heavy | false | true |
| elderly | light | false | false |
| tourist | moderate | false | true |

**Sanity checks (console.assert):**
- wetBulb(33,75) ≈ 29.2 and wbgtShade(33,75) ≈ 30.3
- wetBulb(27,90) ≈ 25.8 → wbgtShade ≈ 26.2 → Green for moderate, acclimatized
- WBGT 29.5, moderate, acclimatized → Orange
- WBGT 28.5, heavy, unacclimatized → Purple

---

## 9. IBM Bob Usage Log

See BOB_LOG.md.

---

## 10. README Requirements

1. Title, tagline, live demo link (optional placeholder), one screenshot.
2. The problem (3 sentences).
3. Who it helps (table from Section 3).
4. How it works — data → WBGT → thresholds → action, with the formula.
5. How we used IBM Bob — summary + link to BOB_LOG.md.
6. Why it's not slop — deterministic, published standards, no hallucination, no keys, free to run.
7. Limitations — simplified WBGT (Stull approximation, rule-of-thumb solar correction), forecast uncertainty, guidance not medical advice.
8. Roadmap — full Liljegren WBGT model; Bahasa Malaysia, Bengali and Nepali translations for migrant workers; employer dashboard; push/WhatsApp scheduled alerts; school-district view.
9. Sources — Stull (2011) wet-bulb formula; ACGIH TLV via OSHA Technical Manual Sec. III Ch. 4; Open-Meteo API.
10. Run locally: "Download the repo and open index.html in any browser — no install needed."

---

## 11. Submission

**Title:** HeatSense — Hour-by-hour heat-safety planner for outdoor workers, schools and athletes

**Description:**
Heat warnings usually report air temperature, but in humid Malaysia it's humidity that makes heat deadly. HeatSense converts the free Open-Meteo forecast into WBGT — the heat-stress index occupational-health experts use — and applies the published ACGIH work/rest thresholds to tell delivery riders, construction crews, schools, runners and the elderly exactly when it's safe to work or exercise, how long to rest, and how much to drink. One tap shares the plan to WhatsApp so a supervisor or teacher can protect a whole group. No AI hallucination, no API keys, no backend: a free static page anyone can open. Built with IBM Bob (see BOB_LOG.md).

---

## 12. Pitch (~60 seconds)

**Hook:** "Right now in Penang it's about 32 degrees. Sounds normal. But with this humidity, heat stress for a delivery rider is already in the danger zone — and nothing they see tells them that."

**Demo:** open the live link on a phone → Right-now card → switch profile Rider → Construction (colours change) → show best window → tap WhatsApp share.

**Trust:** "Nothing here is generated by an AI at runtime. It's a published formula and a published safety standard, so it can't hallucinate."

**Reach:** "One foreman shares it, 30 workers are protected. Free, no install."

**Bob:** "Bob planned the architecture, caught a critical formula error before it shipped, implemented and cross-checked the thresholds, and wrote our docs — every step is logged in the repo."

**Close:** roadmap — migrant-worker languages and an employer dashboard.
