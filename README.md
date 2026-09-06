# BLE Wearable Node- RF Hardware Design, Antenna Integration & Bench Characterisation

**Nordic nRF52840 · Bluetooth Low Energy 5.x · printed 2.4 GHz MIFA · 4-layer PCB · Zephyr / nRF Connect SDK · RF verification and pre-compliance planning**

> **Project status:** design and analysis phase. Requirements, schematic package, RF calculations, antenna starting geometry, PCB floorplan/routing plan, firmware framework, and the verification plan are available. PCB fabrication and measured RF results are still pending.
>
> **Important:** this repository is an engineering portfolio/design-control exercise. It is **not** a certified medical device and it does not claim FCC, ETSI, Bluetooth SIG, IEC 60601-1-2, or other regulatory compliance. Regulatory limits referenced here are used as design and pre-compliance targets; final compliance requires the appropriate accredited test process and final product configuration.

---

## Overview

This repository develops a compact **body-worn Bluetooth Low Energy sensor node** from RF requirements through schematic design, RF analysis, PCB/antenna integration, firmware, and laboratory verification planning.

The design uses a **bare Nordic nRF52840** rather than a pre-certified radio module so that the complete RF path can be engineered and documented:

```text
CR2032
  │
  ▼
VDD / decoupling
  │
  ▼
nRF52840
  │ ANT
  ▼
π matching network
(C20 / L20 / C21)
  │
  ▼
u.FL RF test point / 0 Ω isolation link
  │
  ▼
Low-capacitance ESD protection
  │
  ▼
Printed 2.4 GHz MIFA
  │
  ▼
Air interface / body-worn propagation
```

The same board also contains:

- 32 MHz high-frequency crystal
- 32.768 kHz low-frequency crystal
- Sensirion SHT40 temperature/humidity sensor over I²C
- SWD programming/debug interface
- UART for console and Direct Test Mode
- LED and push button
- CR2032 coin-cell power

The project is organized around **traceability**: RF/electrical/firmware requirements are given IDs, and the planned M-01…M-12 measurements map back to those requirements.

---

## What this project demonstrates

This repository is intended to demonstrate practical RF-system and board-level engineering skills, including:

- RF requirement definition and verification traceability
- 2.4 GHz BLE RF architecture
- bare-SoC RF hardware integration
- antenna matching and Smith-chart based tuning workflow
- printed MIFA integration and body-loading considerations
- 50 Ω grounded coplanar-waveguide design
- 4-layer RF PCB stack-up and grounding strategy
- RF placement, routing, keep-out, and via-stitching rules
- link-budget analysis
- harmonic/spurious-emission risk budgeting
- conducted TX/RX test planning
- VNA, spectrum-analyzer, and signal-generator procedures
- BLE Direct Test Mode planning
- Zephyr / nRF Connect SDK firmware
- design-risk management and revision planning
- FCC / ETSI-oriented pre-compliance thinking

---

## Repository structure

The repository is organized into three main engineering areas:

```text
BLE-Wearable-Node-RF-Design/
│
├── README.md
├── LICENSE
├── .gitignore
│
├── 01_Hardware/
│   ├── requirements/
│   │   └── requirements.md
│   ├── schematics/
│   │   ├── sch1_power_clock.png
│   │   ├── sch2_rf.png
│   │   ├── sch3_io.png
│   │   ├── schematic_all_sheets.pdf
│   │   ├── BOM.csv
│   │   └── ble_node.net
│   ├── kicad/
│   │   ├── ble_node.kicad_pcb
│   │   ├── MIFA_2G4.kicad_mod
│   │   ├── MIFA_2G4_preview.png
│   │   └── pcb_skeleton_preview.png
│   ├── placement_routing/
│   │   ├── placement.csv
│   │   ├── routing_plan.md
│   │   └── floorplan.png
│   └── fabrication/
│       └── fabrication_order.md
│
├── 02_RF_Analysis/
│   ├── link_budget/
│   │   ├── link_budget.py
│   │   └── link_budget.png
│   ├── matching/
│   │   ├── pi_match_design.py
│   │   └── pi_match_s11.png
│   ├── cpwg/
│   │   ├── cpwg_calc.py
│   │   ├── cpwg_sweep.py
│   │   └── cpwg_z0.png
│   ├── harmonics/
│   │   ├── harmonic_budget.py
│   │   └── harmonic_budget.png
│   └── cst/
│       └── cst_simulation_setup.md
│
└── 03_Firmware_and_Test/
    ├── firmware/
    │   ├── README.md
    │   ├── app/
    │   │   ├── CMakeLists.txt
    │   │   ├── prj.conf
    │   │   ├── boards/ble_node.overlay
    │   │   └── src/main.c
    │   └── dtm/
    │       └── README.md
    ├── test_plan/
    │   └── testplan.md
    ├── lab_procedures/
    │   └── lab_procedures.md
    ├── results/
    │   └── README.md
    └── docs/
        ├── Design_Report_revA.pdf
        ├── Design_Report_revA.docx
        ├── design_methodology.md
        └── risk_register.md
```

---

## 1. Design targets

### 1.1 RF requirements

| ID | Requirement | Design target | Verification |
|---|---|---:|---|
| RF-01 | Conducted TX power at antenna-feed reference plane | 0 dBm ±1 dB default; +8 dBm maximum test mode | M-03 |
| RF-02 | Antenna + match return loss, free space, 2400–2483.5 MHz | S11 ≤ −10 dB | M-01 |
| RF-03 | Antenna + match return loss in on-body configuration | S11 ≤ −6 dB after tuning; frequency shift documented | M-02 |
| RF-04 | 99% occupied bandwidth on highest BLE channel | Remain inside 2400–2483.5 MHz | M-04 |
| RF-05 | 6 dB bandwidth | ≥ 500 kHz | M-04 |
| RF-06 | Upper band-edge emission at 2483.5 MHz | ≥ 20 dB below in-band reference | M-07 |
| RF-07 | 2nd/3rd harmonic design-screening target | approximately ≤ −41 dBm EIRP-equivalent screening level | M-05 / M-06 |
| RF-08 | LE 1M receiver sensitivity | ≤ −94 dBm at 30.8% PER | M-08 |
| RF-09 | Receiver blocking with CW blocker near 2.4 GHz | PER ≤ 10% at defined wanted level | M-09 |
| RF-10 | Carrier-frequency accuracy over −20…+60 °C | within ±20 ppm | M-10 |
| RF-11 | On-body link to a phone at 0 dBm | ≥ 10 m | M-11 |

### 1.2 Electrical and mechanical requirements

| ID | Requirement | Target |
|---|---|---:|
| EL-01 | Average current, 1 s advertising interval at 0 dBm | ≤ 30 µA |
| EL-02 | Sleep current | ≤ 5 µA |
| EL-03 | Operating voltage | 1.8–3.6 V |
| EL-04 | ESD robustness at exposed/RF interface | bench evaluation where equipment is available |
| ME-01 | Antenna keep-out | no copper on any layer in defined antenna keep-out |
| ME-02 | RF ground reference | continuous L2 ground plane under RF feed region |
| ME-03 | PCB size | ≤ 35 × 25 mm |

### 1.3 Firmware requirements

| ID | Requirement |
|---|---|
| FW-01 | BLE Direct Test Mode over UART for controlled TX/RX bench testing |
| FW-02 | BLE advertiser carrying sensor data with configurable advertising interval/TX power |
| FW-03 | TX-power control mechanism available for emission/band-edge mitigation experiments |

The complete design-input document is maintained under `01_Hardware/requirements/`.

---

## 2. Hardware architecture

### 2.1 Main components

| Function | Component / implementation |
|---|---|
| BLE SoC | Nordic nRF52840-QIAA-R, aQFN-73 |
| Sensor | Sensirion SHT40 temperature/humidity sensor |
| Main clock | 32 MHz crystal, ±10 ppm target |
| Low-frequency clock | 32.768 kHz crystal |
| Antenna | PCB meander inverted-F antenna (MIFA) |
| RF matching | 0402 π-network footprints C20 / L20 / C21 |
| RF access | Hirose u.FL test connector plus 0 Ω isolation link |
| RF ESD | low-capacitance RF TVS/ESD device, target ≤ 0.3 pF |
| Power | CR2032 coin-cell holder |
| Debug | SWD, UART, VDD/GND/UART test points |

A direct count of `ble_node.net` gives **40 component instances and 25 electrical nets**. The BOM quantity total is also **40 physical component instances**.

### 2.2 Why a bare nRF52840?

A radio module would be a reasonable product choice when certification effort and schedule dominate. This project intentionally uses the bare nRF52840 because the engineering objective is to expose and design the RF path between the SoC antenna pin and free space.

That makes it possible to demonstrate:

- SoC RF-pin breakout
- matching-network implementation
- transmission-line design
- RF test-point placement
- antenna integration
- harmonic/band-edge risk management
- conducted TX/RX measurements
- PCB-level debug and tuning

### 2.3 Power architecture

Revision A uses **LDO mode** rather than the nRF52840 DC/DC converter. The reason is to reduce switching-spur risk during the first RF spin. The current penalty is acceptable for a portfolio/characterisation board; DC/DC operation can be evaluated in a later revision after the RF baseline is known.

---

## 3. Schematic design

The schematic is divided into three functional sheets:

| Sheet | Main content |
|---|---|
| `sch1_power_clock` | CR2032 supply, nRF52840 power/DEC network, 32 MHz and 32.768 kHz clocks |
| `sch2_rf` | nRF52840 ANT pin, π match, u.FL test point, 0 Ω isolation link, ESD protection, printed MIFA |
| `sch3_io` | SHT40 I²C interface, SWD, UART, LED, push button, reset network and test points |

The RF chain is intentionally linear and accessible:

```text
nRF52840 ANT
   │
   ├── C20 shunt footprint
   │
   ├── L20 series footprint
   │
   ├── C21 shunt footprint
   │
   ├── J1 u.FL test tap
   │
   ├── R20 0 Ω isolation link
   │
   ├── D1 low-C ESD device
   │
   └── ANT1 printed MIFA
```

This arrangement allows the antenna to be isolated from the SoC for VNA measurements and allows conducted transmitter/receiver measurements through the same defined RF access point.

---

## 4. RF analysis

All RF-analysis results in this section are **design-phase analytical/simulation results**, not final measurements.

### 4.1 Link budget

The link-budget model uses:

- frequency: 2.44 GHz
- TX power: 0 dBm
- on-body transmit antenna gain assumption: −3 dBi
- phone receive antenna gain assumption: −2 dBi
- mismatch loss: 1 dB
- body-shadowing penalty: 6 dB
- fade margin: 10 dB

The script in `02_RF_Analysis/link_budget/` produces the following approximate ranges:

| BLE PHY | Assumed sensitivity | LOS range | Body-shadowed range |
|---|---:|---:|---:|
| LE 1M | −94 dBm | ~78 m | ~39 m |
| LE 2M | −91 dBm | ~56 m | ~28 m |
| LE Coded S2 | −99 dBm | >100 m | ~70 m |
| LE Coded S8 | −103 dBm | >100 m | >100 m |

Therefore, the **10 m on-body target at 0 dBm has substantial analytical margin** in the current model. The +8 dBm mode is primarily useful for worst-case emissions and stress testing rather than for meeting the basic range requirement.

![Link-budget analysis](02_RF_Analysis/link_budget/link_budget.png)

### 4.2 Antenna matching

The matching script uses the antenna impedance measured or assumed at 2.44 GHz and synthesizes an L-section using the available π-network footprint.

Example design case:

```text
Assumed on-body antenna impedance:
Za = 35 − j25 Ω at 2.44 GHz
```

Calculated ideal values are approximately:

```text
Series inductance ≈ 3.13 nH
Shunt capacitance ≈ 0.85 pF
```

Nearest practical values used by the analysis are approximately:

```text
L20 ≈ 3.3 nH
C20 ≈ 0.82 pF
C21 = DNP for the basic L-match case
```

For the assumed antenna model, worst-case in-band S11 improves from approximately **−7.8 dB to −14.5 dB**.

![Matching analysis](02_RF_Analysis/matching/pi_match_s11.png)

These are **not final production values**. The BOM/schematic keeps tunable 0402 footprints, and the final L/C population must be selected from the measured M-01/M-02 antenna impedance.

### 4.3 50 Ω grounded coplanar waveguide

The RF feed is implemented as grounded coplanar waveguide on L1 referenced to the solid L2 ground plane.

Current design assumptions:

```text
L1 → L2 dielectric height h ≈ 0.10 mm
FR-4 relative permittivity εr ≈ 4.4
Outer copper ≈ 35 µm
Trace width W = 0.20 mm
Gap to coplanar ground S = 0.25 mm
```

The closed-form calculator gives:

```text
Z0 ≈ 50.1 Ω
```

![CPWG impedance sweep](02_RF_Analysis/cpwg/cpwg_z0.png)

This value must still be checked against the **actual PCB-fabricator stack-up and impedance calculator** before ordering because dielectric thickness, Dk, solder mask, copper thickness and etch compensation affect the final impedance.

### 4.4 Harmonic-emission budget

The design includes an early harmonic-risk calculation for +8 dBm TX using assumed SoC harmonic levels and estimated antenna gain at harmonic frequencies.

Current analytical estimate:

| Component | Estimated EIRP | Screening margin vs approximately −41.2 dBm FCC §15.209-equivalent line | Screening margin vs −30 dBm EN 300 328 value used in the model |
|---|---:|---:|---:|
| 2f, ~4.88 GHz | ~−32 dBm | **+9.2 dB — risk** | −2 dB |
| 3f, ~7.32 GHz | ~−41 dBm | **+0.2 dB — marginal** | −11 dB |

![Harmonic budget](02_RF_Analysis/harmonics/harmonic_budget.png)

The 2nd harmonic is therefore treated as a **known design risk**, not a pass result.

Mitigation provisions already included in the design are:

1. the spare shunt capacitor C21, allowing the matching network to become a low-pass π structure;
2. the option to fit a stronger LC low-pass network in a later revision;
3. TX-power reduction during worst-case emission experiments;
4. M-05 conducted-harmonic measurement and M-06 near-field pre-scan before any formal certification activity.

### 4.5 Optional CST antenna simulation

The CST setup guide defines a pre-fabrication EM study for:

- free-space resonance and efficiency
- feed-tap sensitivity
- antenna open-end length sensitivity
- body-phantom loading
- frequency shift caused by body proximity
- radiation efficiency and gain changes

The body-phantom example uses a simplified high-permittivity/lossy material model appropriate for a first-order 2.45 GHz body-loading study. Final wearable behaviour must be validated by measurement.

---

## 5. Printed MIFA antenna

The repository includes a parametric KiCad footprint generator for a starting 2.4 GHz meander inverted-F antenna.

Current generated geometry:

| Parameter | Value |
|---|---:|
| Antenna envelope | ~9.0 × 5.5 mm |
| Generated copper path | ~29.9 mm |
| Trace width | 0.5 mm |
| Feed position | ~2.0 mm from shorting stub |
| All-layer keep-out | ~13.0 × 7.2 mm |

The footprint automatically carries an all-layer copper keep-out around the radiating region.

The printed antenna is intentionally treated as a **starting geometry**, not a guaranteed tuned antenna. Resonance depends on:

- actual FR-4 dielectric properties
- solder mask
- board ground dimensions
- nearby battery/mechanics
- wearable enclosure
- user/body proximity
- component and feed parasitics

The planned tuning flow is:

```text
Generate starting MIFA
      ↓
Fabricate board
      ↓
M-01 VNA measurement in free space
      ↓
Adjust open end / feed position if necessary
      ↓
M-02 measure on-body impedance
      ↓
Run matching synthesis with measured Za
      ↓
Fit L20/C20/C21
      ↓
Re-measure free-space and on-body S11
```

---

## 6. PCB architecture and RF layout

### 6.1 Board

```text
Board size: approximately 35 × 25 mm
Layers:     4
Thickness:  approximately 1.6 mm
Finish:     ENIG planned
```

Proposed stack-up usage:

| Layer | Function |
|---|---|
| L1 / F.Cu | RF feed, critical components and signals |
| L2 / In1.Cu | continuous RF/digital ground reference |
| L3 / In2.Cu | VDD distribution |
| L4 / B.Cu | low-priority signals / battery-side routing |

The generated KiCad PCB skeleton establishes the RF-critical geometry before general routing, including:

- board outline
- stack-up metadata
- antenna footprint and keep-out
- RF feed geometry
- ground/power zones
- initial ground-stitching via pattern
- component placement markers

### 6.2 RF routing rules

The RF route is handled first.

Key rules:

1. RF feed remains on the top layer.
2. Target width/gap is **0.20 mm / 0.25 mm** pending fabricator confirmation.
3. No RF-feed vias.
4. Avoid 90° corners; use short 45° segments/arcs.
5. Maintain an uninterrupted L2 ground reference under the feed.
6. Shunt match/ESD components get extremely short ground returns with vias at the pads.
7. Stitch the ground along both sides of the RF region.
8. Keep all copper out of the antenna keep-out on **every layer**.
9. Keep the battery and unrelated components away from the antenna corner.
10. Do not cross the RF feed with other signals on another layer.

### 6.3 Clock and decoupling rules

- Y1 32 MHz crystal is placed very close to XC1/XC2.
- crystal traces are short and approximately symmetric.
- no unrelated routing is allowed under the high-frequency crystal region.
- DEC/VDD capacitor loops are kept as short as possible.
- DEC4/DEC5 paths receive highest priority.
- the nRF52840 exposed ground region is connected into the solid ground plane with a via array.

---

## 7. Firmware

The firmware examples in this repository are pinned to **nRF Connect SDK (NCS) v2.6.2** for reproducibility.

For NCS v2.6.2, the nRF52840 DK board target is:

```text
nrf52840dk_nrf52840
```

NCS v2.7.0 and later use Zephyr Hardware Model v2 board qualifiers, where the equivalent target is:

```text
nrf52840dk/nrf52840
```

The commands below therefore use the **v2.6.2** target form. If the project is migrated to NCS v2.7+, update the board target and re-validate the Devicetree/Kconfig files rather than assuming drop-in compatibility.

### 7.1 Application firmware

The Zephyr application:

- enables BLE advertising;
- reads the SHT40 sensor;
- places temperature/humidity data into manufacturer-specific advertising data;
- uses a 1 s BLE advertising interval;
- provides software TX-power control;
- cycles test TX-power values using the push button;
- blinks an LED as an activity indication;
- exposes UART for logging/debug.

Configured TX power levels in the application are:

```text
−20, −8, 0, +4, +8 dBm
```

#### TX-power table limitation

The source currently contains a three-entry array labelled for advertising channels 37/38/39. The Nordic vendor-specific HCI command used by the application is applied to the **advertising handle as a whole**, not independently to channels 37, 38 and 39.

Therefore, the current implementation is a **TX-power control / mitigation hook**, not proof of independent per-advertising-channel power control. If RF-06 band-edge testing requires channel-specific back-off, the firmware must be revised to use a controller/test sequence that explicitly controls the active RF channel.

### 7.2 UART configuration: application vs DTM

The custom application overlay `ble_node.overlay` explicitly configures:

```text
Application UART: 115200 8N1
TX: P0.06
RX: P0.08
```

Nordic's NCS v2.6.2 `direct_test_mode` sample, however, defaults to:

```text
DTM UART: 19200 8N1
No hardware flow control
```

The Nordic DTM sample's own `app.overlay` also selects `uart0` as `ncs,dtm-uart`.

**Important:** the current application overlay sets `uart0` to 115200. It should therefore **not be reused unchanged for DTM** if the standard Nordic 19200-baud DTM setup is desired. For the custom board, use a DTM-specific overlay that:

1. keeps `ncs,dtm-uart = &uart0`;
2. maps TX/RX to P0.06/P0.08;
3. sets `current-speed = <19200>`;
4. leaves the application UART at 115200 in the application image.

Until that DTM-specific overlay is added and built on the selected SDK version, the DTM integration should be treated as **planned/configuration-pending**, not build-verified.

### 7.3 Direct Test Mode use on the RF bench

Nordic BLE Direct Test Mode is planned for:

- modulated transmitter testing;
- selected BLE channel;
- selected PHY;
- selected TX-power level;
- receiver packet counting;
- sensitivity testing;
- blocking testing;
- frequency-offset/drift characterisation;
- unmodulated-carrier tests where supported by the selected DTM command set.

### 7.4 Build examples- NCS v2.6.2

From the repository root, the application can be built using the pinned NCS v2.6.2 board target:

```bash
west build -b nrf52840dk_nrf52840 \
  03_Firmware_and_Test/firmware/app -- \
  -DDTC_OVERLAY_FILE=boards/ble_node.overlay
west flash
```

For Nordic's DTM sample, first add/verify a **DTM-specific custom-board overlay** as described above. The baseline NCS v2.6.2 sample target is:

```bash
west build -b nrf52840dk_nrf52840 \
  $NCS_ROOT/nrf/samples/bluetooth/direct_test_mode \
  -d build_dtm
```

Do not claim the custom-board DTM image is ready until its P0.06/P0.08 + 19200-baud overlay has been built and tested.

---

## 8. Verification strategy

All conducted RF measurements use the **u.FL/J1 reference plane** unless a test specifically states otherwise.

### Measurement matrix

| ID | Measurement | Main instrument/setup | Requirement |
|---|---|---|---|
| M-01 | Antenna S11, free space | VNA, 2–3 GHz, calibrated to RF cable reference plane | RF-02 |
| M-02 | Antenna S11 / impedance on body or phantom | VNA | RF-03 |
| M-03 | Conducted TX power, ch 0/19/39 and PHY/power variants | Spectrum analyzer + DTM | RF-01 |
| M-04 | 99% OBW and 6 dB bandwidth | Spectrum analyzer OBW measurement | RF-04 / RF-05 |
| M-05 | 2f / 3f conducted harmonics | Spectrum analyzer to ≥8 GHz | RF-07 |
| M-06 | Near-field radiated pre-scan | H/E probes + spectrum analyzer | design-risk screening |
| M-07 | Upper band-edge behaviour at 2483.5 MHz | Spectrum analyzer, ch 39 | RF-06 |
| M-08 | LE 1M receiver sensitivity | BLE-capable generator + DTM RX | RF-08 |
| M-09 | Receiver blocking | wanted source + CW blocker + combiner | RF-09 |
| M-10 | Carrier frequency vs temperature | DTM carrier + spectrum analyzer | RF-10 |
| M-11 | On-body range / RSSI | phone + BLE logging | RF-11 |
| M-12 | Sleep and advertising current | PPK2 or precision DMM | EL-01 / EL-02 |

### Worst-case test dimensions

The verification plan considers combinations of:

- BLE channels 0 / 19 / 39
- LE 1M / LE 2M / LE Coded S8 where relevant
- maximum configured TX power
- supply-voltage variation
- temperature variation
- board orientation
- free-space versus on-body condition

---

## 9. Laboratory procedure highlights

### M-01- free-space antenna S11

1. Isolate the antenna from the SoC using the RF link arrangement.
2. Perform a 1-port VNA calibration at the cable/reference plane.
3. Sweep approximately 2.0–3.0 GHz.
4. Keep the board away from metal, hands, cables and other loading objects.
5. Record:
   - resonance frequency;
   - S11 at 2400/2440/2480 MHz;
   - −10 dB bandwidth;
   - antenna impedance at 2440 MHz.
6. Tune radiator length/feed geometry if resonance is significantly displaced.

### M-02- on-body tuning

1. Repeat S11 measurement with the board in the representative wearable orientation.
2. Read complex antenna impedance near 2.44 GHz.
3. Insert the measured impedance into `pi_match_design.py`.
4. Fit the calculated matching components.
5. Re-measure both free-space and body-loaded cases.
6. Document the final values and measured frequency shift.

### M-03 to M-07 — transmitter characterisation

The planned measurements include:

- conducted TX power
- occupied bandwidth
- 6 dB bandwidth
- second/third harmonics
- near-field spurious scan
- top-channel band-edge behaviour

### M-08 / M-09- receiver characterisation

Receiver verification is based on DTM packet counting and PER:

```text
PER = 1 − received_packets / transmitted_packets
```

The planned LE 1M sensitivity point is evaluated around **30.8% PER**.

---

## 10. Current design-phase results

The following results are available before fabrication:

| Quantity | Method | Current result | Interpretation |
|---|---|---|---|
| LE 1M body-shadowed range at 0 dBm | link budget | ~39 m with 10 dB fade margin | analytical margin above 10 m target |
| Example body-loaded antenna matching | L-section synthesis for 35−j25 Ω | 3.3 nH / 0.82 pF practical example | final values require VNA data |
| Example in-band S11 improvement | antenna equivalent model | worst ~−7.8 dB → ~−14.5 dB | analytical only |
| RF feed impedance | CPWG closed-form calculation | ~50.1 Ω at W=0.20 mm, gap=0.25 mm | fab confirmation required |
| 2nd harmonic at +8 dBm | emissions budget | ~−32 dBm estimated EIRP | known FCC-oriented screening risk |
| 3rd harmonic at +8 dBm | emissions budget | ~−41 dBm estimated EIRP | marginal in FCC-oriented screening model |

### Measured results

Measured values are intentionally left blank until real hardware data exists.

| ID | Result | Pass/Fail |
|---|---|---|
| M-01 S11 free space | pending | — |
| M-02 S11 on body / final match | pending | — |
| M-03 conducted TX power | pending | — |
| M-04 OBW / 6 dB BW | pending | — |
| M-05 harmonics | pending | — |
| M-06 near-field pre-scan | pending | — |
| M-07 upper band edge | pending | — |
| M-08 receiver sensitivity | pending | — |
| M-09 receiver blocking | pending | — |
| M-10 frequency drift | pending | — |
| M-11 on-body range | pending | — |
| M-12 current consumption | pending | — |

Measurement data should be committed as raw CSV/Touchstone files wherever possible, with generated plots added separately. Raw data is preferred to screenshot-only evidence because plots can then be reproduced.

---

## 11. Risk register

The design identifies risks before fabrication rather than presenting every analysis result as a success.

| Risk | Likelihood | Impact | Planned mitigation / verification |
|---|---|---|---|
| 2nd harmonic exceeds intended regulatory screening level at max TX power | High in current estimate | possible emissions failure | C21/LC filtering, TX-power reduction, M-05/M-06 |
| MIFA detunes in proximity to the body | High | mismatch/range loss | antenna tuning + π match, M-01/M-02 |
| upper band-edge issue on highest BLE channel | Medium | regulatory margin loss | TX-power mitigation experiment, M-07 |
| crystal drift over temperature | Low/Medium | frequency/OBW impact | ±10 ppm crystal target, M-10 |
| RF ESD event disturbs radio/SoC | Medium | reset/link interruption | low-C ESD protection + bench testing |
| receiver blocking from nearby interferers | Medium | increased PER | M-09, possible Rev-B filtering |
| aQFN assembly defects | Medium | failed bring-up | professional assembly, ENIG, multiple boards |

A strong outcome for this project is not “everything passed first time”; it is a documented **measure → diagnose → modify → re-test** loop.

---

## 12. Fabrication plan

Planned PCB fabrication parameters:

| Parameter | Planned value |
|---|---|
| Layers | 4 |
| Thickness | 1.6 mm |
| Surface finish | ENIG |
| Outer copper | ~1 oz |
| L1→L2 RF dielectric | ~0.10 mm target stack-up |
| Minimum general track/space | 0.15/0.15 mm |
| Via | ~0.3 mm drill / 0.6 mm pad |
| RF impedance | 50 Ω grounded CPWG, fabricator-verified |
| Quantity | 5 PCBs, with 2–3 assembled initially |

Before ordering:

- DRC must report zero intended connectivity/rule errors.
- verify nRF52840 aQFN footprint against the current Nordic package drawing;
- verify crystal load-cap values against the actual crystal parts;
- verify the 4-layer stack-up with the PCB fabricator;
- confirm controlled-impedance geometry;
- inspect all copper layers around the antenna keep-out;
- verify assembly availability for the nRF52840 and fine-pitch parts;
- review Gerbers and drill files independently before release.

---

## 13. Bring-up sequence

The planned first-power procedure is:

```text
1. Visual inspection
2. VDD-to-GND continuity/short check
3. Current-limited 3 V bench supply
4. Verify idle current
5. Connect SWD/J-Link
6. Recover/read nRF52840
7. Flash DTM
8. Generate carrier/modulated BLE signal on a mid-band channel
9. Verify conducted RF output through u.FL
10. Perform M-01/M-02 antenna characterisation
11. Tune matching network
12. Flash application firmware
13. Verify BLE advertisement and SHT40 data on a phone
14. Continue M-03…M-12 verification
```

---

## 14. Reproducing the RF analysis

Recommended Python environment:

```bash
python -m venv .venv

# Linux/macOS
source .venv/bin/activate

# Windows PowerShell
# .venv\Scripts\Activate.ps1

pip install numpy matplotlib scikit-rf
```

Example runs from the appropriate directories:

```bash
python link_budget.py
python pi_match_design.py
python cpwg_calc.py
python cpwg_sweep.py
python harmonic_budget.py
```

The analysis scripts generate plots that can be committed with the design report. Whenever measured S-parameter data becomes available, analytical assumptions should be replaced by measured values rather than tuned to make the original prediction look correct.

---

## 15. Standards and engineering references

The project uses the following documents as **engineering context and pre-compliance references**:

- Bluetooth Core Specification / relevant BLE RF-PHY test material
- FCC 47 CFR Part 15, including §15.247 and §15.209 concepts
- ETSI EN 300 328 V2.2.2
- IEC 60601-1-2 Ed. 4.1 as EMC/immunity context for wearable/medical-electronics thinking
- Nordic Semiconductor nRF52840 reference circuitry and hardware documentation
- PCB-fabricator stack-up/controlled-impedance guidance
- printed-MIFA application-note/reference geometries as starting points

---

## 16. Tools

### Hardware / RF

- KiCad 8
- CST Studio Suite — optional antenna EM study
- VNA
- spectrum analyzer
- RF signal generator
- near-field H/E probes
- Nordic Power Profiler Kit II / precision DMM

### Software

- Python
- NumPy
- Matplotlib
- scikit-rf
- nRF Connect SDK
- Zephyr RTOS
- Git / GitHub

---

## 17. Design-control / traceability approach

Although this is a portfolio project, the workflow deliberately borrows formal design-control ideas:

| Design-control concept | Repository evidence |
|---|---|
| Design inputs | RF/EL/ME/FW requirements |
| Design outputs | schematics, BOM, netlist, PCB files, antenna footprint, firmware |
| Design review | pre-order PCB/fabrication checklist |
| Verification | M-01…M-12 test plan |
| Traceability | requirement IDs mapped to measurement IDs |
| Risk management | risk register |
| Design change | planned Rev-A → Rev-B change record based on measured results |

The intended workflow is:

```text
Requirements
   ↓
Architecture
   ↓
Schematic
   ↓
RF analysis
   ↓
PCB + antenna integration
   ↓
Firmware / DTM
   ↓
Fabrication
   ↓
Measurement
   ↓
Root-cause analysis
   ↓
Design change
   ↓
Re-test
```

---

## 18. Planned Rev-B decisions

Revision B should be driven by measured evidence rather than assumptions. Candidate changes include:

- final MIFA geometry/feed tap from measured resonance;
- final C20/L20/C21 population from measured on-body impedance;
- additional low-pass filtering if M-05 shows insufficient harmonic margin;
- improved RF ESD network if antenna-port capacitance or ESD robustness is inadequate;
- optional SAW filtering if receiver blocking requires it;
- DC/DC-mode evaluation after the emissions baseline is established;
- layout changes based on near-field scan results;
- antenna/battery/mechanical spacing changes based on body-loading measurements.

---

## 19. Project status

| Phase | Deliverable | Status |
|---|---|---|
| Requirements | RF/EL/ME/FW design inputs and verification IDs | ✅ complete |
| Schematic | 3-sheet schematic package, netlist and BOM | ✅ complete |
| RF analysis | link budget, matching, CPWG and harmonic budget | ✅ complete |
| Antenna | parametric MIFA starting geometry and keep-out | ✅ design complete / tuning pending |
| PCB layout | floorplan, placement coordinates, routing plan and KiCad skeleton | 🔧 implementation/final DRC pending |
| Firmware | application source available; custom-board DTM 19200-baud overlay | 🔧 DTM overlay/build verification pending |
| Fabrication | Gerbers, assembly and physical Rev-A boards | ⏳ pending |
| Bring-up | power/SWD/RF sanity checks | ⏳ pending |
| RF measurements | M-01…M-12 | ⏳ pending |
| Pre-compliance | near-field scan and mitigation loop | ⏳ pending |
| Rev-B | measurement-driven design update | ⏳ pending |

---

## 20. Author

**Md Moklesur Rahman**  
Wireless communications · RF systems · RF hardware/PCB · BLE/IoT · PHY/DSP · embedded RF calibration

GitHub: `dipucwc`

---

## License

See [`LICENSE`](LICENSE) for the repository license.

---


The final technical value of the repository comes from closing that loop with real VNA, spectrum-analyzer, signal-generator and power measurements and documenting what changed between Rev-A and Rev-B.
