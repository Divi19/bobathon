# HeatSense 🌡

**Know when the heat becomes dangerous — hour by hour.**

> Download the repo and open `index.html` in any browser — no install needed.  
> Live demo: *(deploy to GitHub Pages for a shareable link — optional)*

---

## The Problem

In Malaysia, it is humidity — not just temperature — that makes heat deadly. At 33°C and 75% humidity, sweat cannot evaporate and the body cannot cool itself. Heat warnings in the media report air temperature; the metric occupational-health professionals actually use is **WBGT** (Wet Bulb Globe Temperature), which accounts for humidity. No free, simple tool translates WBGT into plain actions for the people most at risk: delivery riders, construction crews, schoolchildren, runners, and the elderly.

---

## Who It Helps

| Profile | Workload | Threshold | Why they care |
|---------|----------|-----------|---------------|
| Delivery Rider (Grab / Foodpanda / Lalamove) | Moderate | Acclimatized (TLV) | Schedule breaks, avoid collapse on the road |
| Construction / Plantation crew | Heavy | Acclimatized (TLV) | Work/rest cycles; supervisor paper trail for safety duty of care |
| School PE / Assembly | Moderate | Unacclimatized — conservative for children | Decide whether to move activity indoors |
| Runner / Athlete | Heavy | Unacclimatized | When to train; whether to reschedule |
| Elderly / Light outdoor | Light | Unacclimatized — conservative | When not to garden or walk to the market |
| Tourist / Hiker | Moderate | Unacclimatized | Plan trail hikes safely |

---

## How It Works

```
Open-Meteo forecast (free, no key)
  → temperature + humidity + solar radiation + day/night flag (hourly)
  → Stull (2011) wet-bulb formula → WBGT in shade
  → + solar correction if in direct sun (rule-of-thumb estimate)
  → ACGIH TLV / Action Limit thresholds
  → colour-coded work/rest recommendation per hour
  → WhatsApp share message for supervisors / teachers
```

### WBGT Calculation

**Step 1 — Wet-bulb temperature** (Stull 2011, angles in radians):
```
Tw = T·atan(0.151977·√(RH+8.313659)) + atan(T+RH)
   − atan(RH−1.676331) + 0.00391838·RH^1.5·atan(0.023101·RH) − 4.686035
```

**Step 2 — WBGT in shade:**  `wbgtShade = 0.7·Tw + 0.3·T`

**Step 3 — Solar correction (rule-of-thumb estimate):**  
`WBGT = wbgtShade + (inSun && isDay ? 3·min(radiation/800, 1) : 0)`

*Sanity check: at T=33°C, RH=75%, Tw≈29.2°C and WBGT shade≈30.3°C — a realistic Orange/Yellow result for a Malaysian afternoon.*

### Work/Rest Thresholds (ACGIH TLV & Action Limit)

| Level | Colour | Work pattern |
|-------|--------|--------------|
| 🟢 Green | Normal | Continuous work |
| 🟡 Yellow | Caution | 45 min work, 15 min rest |
| 🟠 Orange | Warning | 30 min work, 30 min rest |
| 🔴 Red | Danger | 15 min work, 45 min rest |
| 🟣 Purple | Stop | No strenuous activity — heat stroke risk |

---

## How We Used IBM Bob

IBM Bob was the primary development partner throughout this build. Bob:

1. **Planned the architecture** — read the full spec and produced the function signatures, THRESHOLDS data structure, build order, and resolved 7 spec ambiguities before a single line of code was written.
2. **Caught a critical formula error** — identified that the original BoM WBGT formula would produce ~37.5°C for a routine Malaysian afternoon (T=33, RH=75), making the app show Purple danger for essentially every daytime hour. Bob switched the model to the Stull (2011) wet-bulb approach before it could ship.
3. **Implemented every function** — `wetBulb`, `wbgtShade`, `wbgt`, `classify`, `bestWindow`, `buildWhatsAppMessage`, `fetchForecast`, all render functions, controls wiring, and the sanity-check block.
4. **Wrote all documentation** — this README, the spec, and the full BOB_LOG.

See **[BOB_LOG.md](BOB_LOG.md)** for the complete step-by-step log with prompts, outputs, and verification notes for every sub-task.

---

## Why It's Not Slop

- **No AI at runtime** — zero LLM calls in the product. Output is deterministic: public forecast data + a peer-reviewed formula + a published occupational safety standard.
- **Nothing to hallucinate** — the thresholds are a lookup table. The WBGT is arithmetic. If the numbers are wrong, the formula is wrong — and the formula is citable.
- **No API keys, no backend, no cost** — a single static HTML file anyone can open on their phone.
- **Distribution built in** — one supervisor taps Share, 30 workers are protected. No install.
- **PDPA-friendly** — no personal data collected or stored.

---

## Limitations

- **Stull wet-bulb** is an empirical approximation (±1°C error across most of the tropical range). It does not use measured solar radiation or wind speed in the wet-bulb step.
- **Solar correction** (the +3°C sun adjustment) is a rule-of-thumb, not a calibrated Liljegren model. It may over- or under-estimate in low-radiation or windy conditions.
- **Forecast uncertainty** — Open-Meteo is a numerical weather model, not a measurement. Accuracy degrades beyond 24 hours.
- **This is general guidance, not medical advice.** Follow your employer's safety rules and local authority warnings.

---

## Roadmap

- **Full Liljegren (2008) WBGT model** incorporating measured solar radiation and wind speed for higher accuracy.
- **Bahasa Malaysia, Bengali and Nepali translations** — the people most at risk (migrant construction workers, plantation workers) often don't read English.
- **Employer dashboard** — daily schedule view for a whole crew, exportable PDF for safety documentation.
- **Push / WhatsApp scheduled alerts** — automatically notify a foreman's group when morning WBGT crosses a threshold.
- **School-district view** — aggregate risk level across multiple schools for district education offices.

---

## Sources

- Stull, R. (2011). *Wet-Bulb Temperature from Relative Humidity and Air Temperature*. Journal of Applied Meteorology and Climatology, 50(11), 2267–2269.
- [OSHA Technical Manual, Section III Chapter 4 — Heat Stress](https://www.osha.gov/otm/section-3-health-hazards/chapter-4) (ACGIH TLV & Action Limit table)
- [Open-Meteo API documentation](https://open-meteo.com/en/docs) (free, no API key required)

---

## Run Locally

Download the repo and open `index.html` in any browser — no install needed. All logic runs client-side; the only network call is the Open-Meteo forecast fetch.

```bash
git clone https://github.com/<your-username>/bobathon.git
cd bobathon
open index.html   # macOS
# or: start index.html  (Windows)
# or: xdg-open index.html  (Linux)
```

---

*Built with IBM Bob · HeatSense*
