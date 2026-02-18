# Wi-Fi RTT Triangulation Engine

A real-time indoor/urban positioning system that estimates location using Wi-Fi signal strength (RSSI), network round-trip time (RTT), and browser GPS. Demonstrates how Wi-Fi triangulation can supplement or outperform GPS in environments where satellite signals are weak or unreliable — dense urban areas, indoors, parking garages, tunnels, etc.

**Live demo:** [wifi-location-test.vercel.app](https://wifi-location-test.vercel.app)

---

## What This Does

### The Problem

GPS relies on line-of-sight to satellites. In urban canyons (tall buildings), indoors, or underground, GPS accuracy degrades from ~3m to 50-100m+ or fails entirely. Devices often fall back to Wi-Fi-based positioning (using known AP databases), but that's a black box controlled by Google/Apple with no transparency or user control.

### How It Works

The app uses three independent positioning techniques and combines them:

**1. RSSI-to-Distance (Signal Strength)**

Every Wi-Fi access point broadcasts radio signals. Signal strength decays predictably with distance following the log-distance path loss model:

```
distance = 10^((TxPower - RSSI) / (10 * n))
```

We use frequency-aware path loss exponents because higher frequencies attenuate faster through walls and air:

| Band | Path Loss (n) | Tx Power | Characteristics |
|------|--------------|----------|-----------------|
| 2.4 GHz | 2.8 | -38 dBm | Best wall penetration, most interference |
| 5 GHz | 3.2 | -40 dBm | Moderate penetration, less congestion |
| 6 GHz | 3.5 | -42 dBm | Worst penetration, cleanest spectrum |

**2. RTT (Round-Trip Time)**

Network-layer round-trip time is measured using:
- **TCP SYN timing** — Nanosecond-precision timestamps via `process.hrtime.bigint()`
- **ICMP ping** — Standard ping with statistical outlier trimming

At the speed of light, radio signals travel ~30cm per nanosecond. A 3ms network RTT would imply ~450km — clearly dominated by processing overhead, not signal flight time. The app uses statistical minimum RTT to estimate and subtract this overhead, extracting the true signal flight component.

The real promise is **IEEE 802.11mc Fine Time Measurement (FTM)**, which measures RTT at the physical layer with <100 picosecond precision, enabling **sub-1 meter** accuracy. Our network-layer RTT serves as a demonstration of the concept and provides useful signal quality metrics (jitter, stability).

**3. GPS Anchor + Wi-Fi Offset**

The browser's Geolocation API provides a GPS fix as the global reference point. Wi-Fi trilateration calculates meter-level offsets from this anchor, which are converted to real GPS coordinates using geodetic math:
- 1° latitude = 111,320 meters
- 1° longitude = 111,320 × cos(latitude) meters

**4. Trilateration (Combining It All)**

With 3+ APs at known positions, the app solves for your location using Weighted Nonlinear Least Squares (WNLS) with momentum-based gradient descent. Each AP is weighted by its signal-to-noise ratio (SNR) — stronger, cleaner signals get more influence. A 1D Kalman filter per axis smooths the result over time, reducing jitter from noisy readings.

**5. GPS Calibration Walk**

The app includes a calibration mode where you physically walk to each router and save your GPS coordinates at that location. This gives each AP a real-world GPS position. After calibrating 3+ APs, the system trilaterates your position directly in GPS coordinate space — no manual canvas placement needed. Calibrations persist in localStorage across page reloads.

### Pages

| Page | URL | Description |
|------|-----|-------------|
| **Triangulation Engine** | `/` | Interactive canvas showing AP positions, distance circles, trilateration math, RSSI/RTT data, Kalman-filtered position estimate, GPS coordinates, and GPS calibration mode |
| **Live Map** | `/map.html` | Real-time OpenStreetMap with GPS tracking, accuracy circles, movement trail, speed/altitude, and optional Wi-Fi position overlay |

---

## Accuracy: What to Expect

### Wi-Fi RTT Accuracy by Method

| Method | Precision | Accuracy | Requirements |
|--------|-----------|----------|-------------|
| **RSSI only** (signal strength) | 2-5m per AP | 3-15m trilaterated | 3+ APs with known positions |
| **Network-layer RTT** (TCP/ICMP) | ~1ms (~150km raw) | Quality metric only | Gateway reachable |
| **802.11mc FTM** (physical-layer RTT) | <100 ps (~1.5cm) | **0.5-2m** per AP | FTM-capable APs + client |
| **RSSI + GPS Calibration** | 2-5m per AP | **3-8m** trilaterated | 3+ calibrated APs |
| **FTM + GPS Calibration** | 0.5-2m per AP | **0.5-3m** trilaterated | FTM hardware + calibration |

### Compared to GPS

| Environment | GPS Alone | Wi-Fi Triangulation | Combined | Improvement |
|-------------|-----------|-------------------|----------|-------------|
| Outdoors, clear sky | 3-5m | 5-15m | 3-5m | Marginal |
| Urban canyon (tall buildings) | 15-50m+ | 3-8m | **3-8m** | **2-6x better** |
| Indoors (office, mall) | 30-100m+ or fails | 3-10m | **3-10m** | **5-10x better** |
| Underground / parking garage | Fails entirely | 5-15m | **5-15m** | **GPS unusable** |
| Dense AP environment (10+) | 5-15m | 2-5m | **2-5m** | **3-5x better** |

### Why Wi-Fi RTT Can Beat GPS Indoors

GPS signals travel ~20,200km from satellites and arrive at your device with roughly **-130 dBm** power — extremely weak. Any obstruction (roof, walls, metal) attenuates the signal further or causes multipath reflections where the signal bounces off surfaces and arrives multiple times, confusing the receiver.

Wi-Fi signals travel 1-50m from nearby routers and arrive at **-30 to -80 dBm** — orders of magnitude stronger. The physics:

- **Signal-to-noise ratio**: A -35 dBm Wi-Fi signal has ~55 dB SNR. A GPS signal indoors might have 0 dB SNR or worse.
- **Geometry**: GPS satellites are all above you (poor vertical geometry indoors). Wi-Fi APs surround you horizontally (ideal for 2D positioning).
- **Multipath**: GPS multipath errors are hard to detect. Wi-Fi multipath can be mitigated by using multiple APs from different directions — errors average out.
- **Availability**: GPS requires sky view. Wi-Fi works anywhere with routers.

### The 802.11mc Promise

IEEE 802.11mc (Wi-Fi Round-Trip Time / Fine Time Measurement) measures signal flight time at the physical layer, bypassing all network processing delays. At light speed:

```
1 nanosecond = 0.3 meters (30 cm)
100 picoseconds = 0.03 meters (3 cm)
```

With FTM-capable hardware achieving <100 ps precision, each individual AP measurement gives **0.5-2m** accuracy. With 3+ FTM APs and trilateration, the theoretical limit is **sub-meter** indoor positioning — competitive with ultra-wideband (UWB) at a fraction of the infrastructure cost since it uses existing Wi-Fi hardware.

Our app currently uses network-layer RTT (TCP SYN / ICMP), which includes ~1-3ms of kernel and firmware processing overhead. We statistically estimate and subtract this overhead to extract the signal flight component, but the precision is limited to ~5-50m. The app architecture is designed to drop in FTM measurements when the hardware is available — the trilateration math works identically regardless of how distances are estimated.

### Factors That Affect Accuracy

| Factor | Impact | Mitigation |
|--------|--------|-----------|
| **Number of APs** | 3 minimum, 6+ ideal | More APs = more equations = better solution |
| **AP geometry** | APs surrounding you > all on one side | Calibrate APs in different directions |
| **Walls and obstacles** | Add 2-5m error per wall | Use per-band path loss exponents; more APs average out errors |
| **GPS anchor quality** | Directly affects final coordinates | Calibrate outdoors for best anchor accuracy |
| **Signal interference** | 2.4 GHz most affected | 5/6 GHz bands weighted higher automatically |
| **Movement** | RSSI fluctuates while moving | Kalman filter smooths temporal jitter |

---

## Understanding Access Points (APs)

### What Are the Access Points?

The app detects nearby Wi-Fi access points (routers, hotspots, mesh nodes) and measures their signal properties. Each AP entry shows:

| Field | Meaning |
|-------|---------|
| **SSID** | Network name (e.g. `AP_5GHz_ch48_A`). May be redacted on macOS without a developer certificate. |
| **Band** | Radio frequency — 2.4GHz (longest range), 5GHz (medium), or 6GHz (shortest range, newest). |
| **Channel** | The specific radio channel the AP operates on. |
| **RSSI** | Received Signal Strength Indicator in dBm. Closer to 0 = stronger. -32 dBm is very strong (nearby), -80 dBm is weak (far away). |
| **SNR** | Signal-to-Noise Ratio in dB. Higher = cleaner signal. APs with SNR > 40 dB give the most reliable distance estimates. |
| **Bandwidth** | Channel width (20/40/80/160 MHz). Wider channels carry more data but don't affect positioning. |
| **Est. distance** | Calculated distance from you to the AP using the path loss model, with uncertainty range (e.g. `0.56m ±0.1m`). |

### Two Ways to Position APs

**Option A: Manual Placement (Canvas)**

Click "Place" on any AP, then click on the canvas to mark its real-world position. Best when you know the floor plan layout.

**Option B: GPS Calibration Walk (Recommended)**

1. Tap **Calibrate** in the header to enter calibration mode
2. Walk to a physical router — the strongest AP in the list (marked **NEAREST**) is the one you're standing next to
3. Tap **Calibrate** on that AP — your current GPS coordinates are saved as its position
4. Repeat for 3+ APs from different locations
5. The app automatically trilaterates your position in real GPS coordinates

Calibrations persist across page reloads via localStorage. Use "Clear Calibrations" in the canvas HUD to start fresh.

### Do I Need to Place APs?

**Without placing APs**, the app still provides:
- GPS coordinates from your browser (the anchor point)
- All AP signal data (RSSI, SNR, distance estimates, band info)
- RTT measurements to the gateway router
- Signal environment analysis

**With 3+ APs placed or calibrated**, the app additionally provides:
- Wi-Fi-triangulated position that can refine GPS accuracy
- Combined GPS + Wi-Fi coordinate estimate
- Position tracking on the canvas with distance circles
- 95% confidence interval

---

## Deployment Modes

### Static (Vercel)

The live demo loads pre-captured Wi-Fi scan data from `scan-data.json` and uses browser GPS. No backend server required. This mode works **anywhere in the world** on any device with a browser.

The scan data includes 11 APs across 2.4GHz, 5GHz, and 6GHz bands captured at the reference location. The GPS anchor is automatically set from the saved coordinates.

### Local Server (Full Wi-Fi Scanning)

For real-time scanning, run the Node.js server on macOS. This mode:

- Scans nearby APs every 3 seconds via CoreWLAN
- Measures gateway RTT with TCP SYN timing and ICMP ping
- Streams data to the browser via WebSocket
- Works at **any location** — the scanner discovers whatever APs are nearby

### Platform Support

| Platform | Wi-Fi Scanning | GPS | Live Map |
|----------|---------------|-----|----------|
| macOS (local server) | Full | Yes | Yes |
| Windows/Linux | No (CoreWLAN is macOS-only) | Yes | Yes |
| Any browser (Vercel) | Saved data only | Yes | Yes |
| Mobile browser | Saved data only | Yes | Yes |

> **Note:** macOS requires Location Services permission for the scanner app to read SSID/BSSID data. Full SSID visibility requires an Apple Developer certificate ($99/year). Without it, SSIDs appear redacted but the scanner still detects APs by channel and band.

---

## Local Development Setup

### Prerequisites

- **macOS** (required for Wi-Fi scanning; saved data mode works on any OS)
- **Node.js 18+**
- **Xcode Command Line Tools** (for compiling the Swift scanner)

### Install

```bash
git clone <repo-url>
cd wifi-triangulation
npm install
```

### Compile the Wi-Fi Scanner (macOS only)

```bash
mkdir -p WifiScanner.app/Contents/MacOS

swiftc scanner.swift \
  -o WifiScanner.app/Contents/MacOS/wifi-scanner \
  -framework CoreWLAN \
  -framework CoreLocation \
  -framework Foundation

# Create Info.plist (required for Location Services)
cat > WifiScanner.app/Contents/Info.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleIdentifier</key>
    <string>com.wifitriangulation.scanner</string>
    <key>CFBundleName</key>
    <string>WifiScanner</string>
    <key>CFBundleExecutable</key>
    <string>wifi-scanner</string>
    <key>CFBundleVersion</key>
    <string>1.0</string>
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    <key>LSUIElement</key>
    <true/>
    <key>NSLocationUsageDescription</key>
    <string>Wi-Fi scanning requires location access to read network details.</string>
    <key>NSLocationWhenInUseUsageDescription</key>
    <string>Wi-Fi scanning requires location access to read network details.</string>
    <key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
    <string>Wi-Fi scanning requires location access to read network details.</string>
</dict>
</plist>
EOF

codesign --force --sign - WifiScanner.app
```

### Enable Location Services

1. Open **System Settings > Privacy & Security > Location Services**
2. Enable Location Services (if not already)
3. Find **WifiScanner** in the list and toggle it **ON**

### Run

```bash
npm start
```

Open http://localhost:3000 in your browser.

---

## Architecture

```text
Browser
  |
  |-- GPS ← navigator.geolocation (browser-native)
  |
  |-- Saved Data ← scan-data.json (pre-captured APs + RTT)
  |
  |-- WebSocket ← Real-time scan data (local server only)
  |       |
  |       v
  Node.js Server (macOS, optional)
    |-- Wi-Fi Scanner ← CoreWLAN via compiled Swift binary
    |-- RTT Measurement ← TCP SYN timing + ICMP ping
    |-- ARP Table ← Network device discovery
```

### Key Files

| File | Purpose |
|------|---------|
| `public/index.html` | Triangulation UI — canvas, RSSI/RTT processing, trilateration, Kalman filter, GPS anchor, calibration mode |
| `public/map.html` | Live map — Leaflet + OpenStreetMap with GPS tracking and Wi-Fi overlay |
| `public/scan-data.json` | Pre-captured Wi-Fi scan data for static deployment |
| `server.js` | Express + WebSocket server, runs scanner, measures RTT (local dev only) |
| `scanner.swift` | macOS Wi-Fi scanner using CoreWLAN + CoreLocation |

### Signal Processing Pipeline

```text
Raw RSSI (dBm) → Frequency-aware path loss model → Distance estimate (meters)
                                                          |
                                                    ± Uncertainty
                                                          |
                    GPS Calibration Walk ─────────────┐    |
                    (real AP GPS positions)            v    v
                                              3+ AP distances
                                                     |
                                        WNLS Trilateration (gradient descent)
                                              |                |
                                         SNR weighting    Kalman filter
                                                              |
                                                   GPS anchor + offset
                                                              |
                                                    Lat/Lng coordinates
                                                    (64-bit, 10 decimal places)
```

---

## Technologies

- **Frontend:** Vanilla HTML/CSS/JS, Canvas API, Leaflet.js, OpenStreetMap
- **Backend:** Node.js, Express, WebSocket (ws)
- **Scanner:** Swift, CoreWLAN, CoreLocation (macOS)
- **Math:** Log-distance path loss model, WNLS trilateration, Kalman filtering, geodetic coordinate conversion
- **Deployment:** Vercel (static), GitHub

---

## License

MIT
