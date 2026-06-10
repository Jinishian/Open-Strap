# OpenStrap

**A free, open-source WHOOP companion app for Windows — no subscription required.**

OpenStrap connects directly to your WHOOP 4.0 or 5.0 strap over Bluetooth and gives you live biometrics, HRV analysis, sleep staging, and long-term trends — all in a single HTML file that runs in your browser. Your data stays on your device. No account, no cloud, no monthly fee.

![OpenStrap Live Tab](screenshots/live.png)

---

## Features

| | |
|---|---|
| **Live metrics** | Heart rate, HRV (RMSSD), recovery score, strain (Edwards TRIMP), SpO₂, respiration rate, battery |
| **ECG animation** | Real-time P-QRS-T waveform synced to your actual R-R intervals |
| **HRV analysis** | Poincaré plot with SD1/SD2 ellipse, frequency domain (LF/HF via FFT), tachogram, SDNN, pNN50 |
| **Sleep staging** | Automatic Wake/Light/Deep/REM classification from HRV + HR patterns, live hypnogram |
| **Trend charts** | 7/14/30-day HRV, recovery, resting HR, SpO₂, strain, sleep duration, sleep efficiency |
| **Full protocol** | WHOOP 4.0 BLE protocol — CRC8/CRC32 framing, bonding handshake, historical data offload |
| **WHOOP import** | Drag-and-drop `physiological_cycles.csv` and `sleeps.csv` from your WHOOP data export |
| **CSV export** | Export any live session to CSV with timestamp, HR, R-R, HRV, recovery, strain, SpO₂ |
| **Persistence** | Sessions and daily metrics saved to IndexedDB — survives browser restarts |
| **Auto-reconnect** | Automatically reconnects on drop, up to 5 attempts with exponential backoff |

---

## Quick Start

### Requirements
- Windows 10 or 11
- Google Chrome or Microsoft Edge (version 56+)
- Bluetooth 4.0+ adapter
- WHOOP 4.0 or 5.0 strap

### Run

**Option 1 — Double-click launcher (easiest)**

Download both files into the same folder, then double-click `serve.bat`. It starts a local web server and opens Chrome automatically.

**Option 2 — Manual**

```bat
python -m http.server 8080
```

Then open Chrome and go to: `http://localhost:8080/openstrap.html`

> **Why a local server?** Web Bluetooth requires a secure context. Browsers block it when opening HTML files directly via `file://`. Serving via `localhost` satisfies the requirement without needing HTTPS.

---

## Connecting Your Strap

1. **Enable Broadcast Heart Rate** in the WHOOP app:
   `Profile → App Settings → Device Settings → Broadcast Heart Rate` → ON

2. **Close the WHOOP app** on your phone (or disconnect the strap from it in Bluetooth settings). WHOOP straps hold one BLE connection at a time — if your phone is connected, OpenStrap gets dropped.

3. Click **Connect** in OpenStrap, select your WHOOP from the browser picker, and click Pair if Windows prompts you.

4. HR and vitals appear within a few seconds. HRV, recovery, and the Poincaré plot populate once R-R interval data starts flowing (~10–15 seconds). The frequency domain (LF/HF) requires ~2 continuous minutes of R-R data.

---

## Tabs

### Live
Real-time dashboard. HR ring color-codes by zone (green → yellow → red). Six metric cards update continuously. The ECG canvas renders a P-QRS-T waveform paced to your actual R-R intervals. A sleep banner appears automatically when sleep is detected and shows current sleep stage and a mini hypnogram.

### Trends
Long-term charts: HRV, recovery score, resting HR, SpO₂, and strain over your chosen range (7/14/30 days). Session history list at the bottom with HRV, recovery, and strain badges per session.

### Sleep
Last-night summary with stage breakdown bar, full hypnogram, deep/REM/latency/disruption stats. Multi-day sleep duration, efficiency, and respiratory rate trend charts.

### HRV
Deep HRV panel:
- **Poincaré plot** — scatter of (RRₙ, RRₙ₊₁) pairs with SD1/SD2 ellipse. Recent points are brighter; the latest beat glows green.
- **Frequency domain** — power spectrum with LF (0.04–0.15 Hz) and HF (0.15–0.40 Hz) bands, LF/HF ratio with sympathovagal interpretation.
- **Tachogram** — scrolling 60-second R-R time series.
- **Six metrics** — RMSSD, SDNN, pNN50, SD1, SD2, LF/HF.

### Import
Drag and drop your WHOOP data export files:
- `physiological_cycles.csv` — daily recovery, HRV, strain, SpO₂, respiratory rate
- `sleeps.csv` — sleep sessions with stage durations, efficiency, latency, disruptions

Export from the WHOOP app: **Profile → App Settings → Export Data**

---

## Technical Details

### BLE Protocol
OpenStrap implements the WHOOP 4.0 BLE protocol documented by the [Noop project](https://github.com/NoopApp/noop):

- **Frame format:** `[0xAA][len u16 LE][CRC8(len)][type][seq][cmd][payload…][CRC32 u32 LE]`
- **CRC8:** poly 0x07, init 0x00
- **CRC32:** reflected, poly 0xEDB88320 (zlib)
- **Bond trigger:** confirmed write of `GET_BATTERY_LEVEL` (cmd 26) to characteristic `61080002-…`
- **Services used:** WHOOP custom (`61080001-…`), Heart Rate (`0x180D`), Battery (`0x180F`), PLX (`0x1822`)
- **Historical offload:** `SEND_HISTORICAL_DATA` (cmd 22) with trim acknowledgment via `HISTORICAL_DATA_RESULT` (cmd 23)

### HRV Algorithms
| Metric | Method |
|---|---|
| RMSSD | √(mean(ΔRR²)) — Task Force 1996 |
| SDNN | σ of all RR intervals in window |
| pNN50 | % successive differences > 50ms |
| SD1 | RMSSD / √2 (Poincaré short-term) |
| SD2 | √(2·SDNN² − 0.5·RMSSD²) |
| LF/HF | Iterative Cooley-Tukey FFT on 4 Hz resampled RR series, Hann-windowed; LF = 0.04–0.15 Hz, HF = 0.15–0.40 Hz |

### Recovery Score
Z-score of current RMSSD against a personal session baseline (first 5 readings), mapped to 0–100 via a linear sigmoid: `score = 50 + z × 17`.

### Strain
Edwards TRIMP mapped to WHOOP's 0–21 logarithmic scale:
- HR zones based on Tanaka HRmax (208 − 0.7 × age, default 30)
- Zone weights: <50% = 0, 50–60% = 1, 60–70% = 2, 70–80% = 3, 80–90% = 4, ≥90% = 5
- `strain = min(21, ln(trimp + 1) × 3.6)`

### Sleep Staging
30-second epochs classified using HR delta from resting baseline and RMSSD ratio:
- **Wake:** HR > baseline + 12
- **REM:** RMSSD ratio > 1.12, HR < baseline + 9
- **Deep:** RMSSD ratio < 0.68, HR ≤ baseline + 1
- **Light:** everything else in sleep range

Based on Beattie et al. 2017 (cardiorespiratory-only staging, ~67% 4-class accuracy without accelerometer). Labelled as estimated — not a clinical measurement.

### Respiration Rate
Estimated from respiratory sinus arrhythmia (RSA): zero-crossing frequency of mean-subtracted R-R series, valid range 6–35 br/min.

---

## Architecture

Single-file HTML app — no build step, no dependencies (Google Fonts only). All data stored locally in IndexedDB. No telemetry, no network calls except font loading.

```
openstrap.html
├── CSS (~300 lines)         Dark dashboard theme
├── HTML (~280 lines)        5-tab layout
└── JavaScript (~1500 lines)
    ├── CRC utilities        CRC8, CRC32, SFLOAT16
    ├── Reassembler          BLE MTU fragmentation handler
    ├── WhoopBLE             Full WHOOP 4.0 protocol layer
    ├── SleepStager          HRV-based sleep stage classifier
    ├── Analytics            RMSSD, recovery, strain, respiration
    ├── DB                   IndexedDB (sessions, daily, sleep)
    ├── TrendChart           Canvas line/bar charts
    ├── Hypnogram            Canvas sleep stage renderer
    ├── PoincarePlot         Canvas scatter + SD1/SD2 ellipse
    ├── FreqPlot             Canvas LF/HF power spectrum
    ├── Tachogram            Canvas R-R time series
    ├── CSVImporter          WHOOP cycles + sleeps CSV parser
    ├── ECG                  Scrolling P-QRS-T animation
    └── App                  Connection, reconnect, keepalive, UI
```

---

## Contributing

Pull requests welcome. The most useful areas:

- **WHOOP 5.0 support** — different service UUID (`fd4b0001-…`) and CRC16-Modbus header
- **Historical data parsing** — full schema-driven decoder for type-47 frames (requires `whoop_protocol.json`)
- **SpO₂ from custom protocol** — decode red/IR PPG values from type-43 realtime frames
- **Better sleep staging** — add movement detection if accelerometer data can be decoded

---

## Disclaimer

OpenStrap is not affiliated with, endorsed by, or connected to WHOOP, Inc. It is not a medical device. All metrics are estimates for informational purposes only. Do not use for clinical decisions.

This project uses reverse-engineered BLE protocol information. Using it may violate WHOOP's Terms of Service. Use at your own risk.

---

## License

MIT License — Copyright © 2026 Jinishian Industries LLC

Permission is hereby granted, free of charge, to any person obtaining a copy of this software to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies, subject to the following conditions: the above copyright notice and this permission notice shall be included in all copies or substantial portions of the software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND.
