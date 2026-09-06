# BLE Wearable Node — RF Design

**Status: Design phase.** Schematic, RF analysis, PCB stack-up and test plan are complete. Board layout, fabrication and bench measurement are in progress. The measured-results table below is intentionally empty and will only be filled with real lab data.

A small Bluetooth Low Energy sensor node designed end-to-end as an RF hardware exercise: bare nRF52840 SoC, printed meandered inverted-F antenna (MIFA), 4-layer 35 × 25 mm PCB with a controlled-impedance CPWG feed, a coin-cell supply and a temperature/humidity sensor. The goal is a board that can be characterised on a VNA and spectrum analyser and checked against the radiated-emission limits of **EN 300 328** and **FCC §15.247 / §15.209**.

---

## 1. Why this project exists

Wearable medical and consumer radios live or die on three things: antenna efficiency in a tiny form factor, a clean 50 Ω path from the chip to that antenna, and staying inside the regulatory mask at full transmit power. This repository documents each of those decisions with calculations, simulation set-ups and a measurement plan, so every design choice can be traced from requirement → analysis → layout → test.

---

## 2. Key specifications

| Item | Value |
|---|---|
| SoC / radio | Nordic nRF52840 (bare chip, 2.4 GHz BLE 5) |
| Antenna | Printed MIFA, 2.40–2.48 GHz, with ground keep-out |
| PCB | 4-layer, 35 × 25 mm, FR-4, controlled-impedance top layer |
| RF feed | Coplanar waveguide with ground (CPWG), W = 0.20 mm, gap = 0.25 mm, 50 Ω |
| Matching | π-network, series L = 3.3 nH / shunt C = 0.8 pF for measured antenna impedance target Za ≈ 35 − j25 Ω |
| RF test point | u.FL connector in the feed line for conducted measurements |
| ESD | Low-capacitance TVS diode at the antenna port |
| Sensor | Sensirion SHT40 (I²C temperature / humidity) |
| Power | CR2032 coin cell, LDO/DC-DC as per schematic sheet 2 |
| Firmware | Zephyr RTOS — Direct Test Mode (DTM) app + BLE advertiser with per-channel TX-power table |
| Target TX power | +8 dBm (harmonic budget evaluated at this level) |

---

## 3. Repository layout

```
01_Hardware/            schematics, KiCad project, footprints, placement, fabrication files
02_RF_Analysis/         link budget, matching-network design, CPWG sweep, harmonic budget, CST set-up
03_Firmware_and_Test/   Zephyr firmware, test plan, lab procedures, results (empty), design docs
```

### 01_Hardware
| Folder | Contents |
|---|---|
| `requirements/` | Requirement tables RF-01…RF-11 (radio), EL-xx (electrical), ME-xx (mechanical), FW-xx (firmware), each with an acceptance criterion and the test ID that verifies it |
| `schematics/` | Three sheets — (1) SoC, crystal, RF front-end and antenna; (2) power; (3) sensor and debug — plus BOM |
| `kicad/` | KiCad 8 project, netlist, custom MIFA footprint with keep-out, 4-layer stack-up |
| `placement_routing/` | `placement.csv`, routing plan (RF path first, then crystal, then power) |
| `fabrication/` | Fab order notes (JLCPCB / PCBWay, 5 bare boards, 2–3 assembled); Gerbers added once layout is signed off |

### 02_RF_Analysis
| Folder | Contents |
|---|---|
| `link_budget/` | Python link-budget script: TX power, antenna gains, path loss, body-loss allowance, receiver sensitivity → range margin |
| `matching/` | π-match synthesis with Smith-chart plot (scikit-rf), component values and sensitivity to ±5 % tolerance |
| `cpwg/` | Trace-width / gap sweep for 50 Ω on the chosen stack-up |
| `harmonics/` | Harmonic-emissions budget vs FCC §15.209 and EN 300 328 spurious limits — flags 2nd harmonic ≈ 9 dB over the FCC limit at +8 dBm, driving the low-pass filter decision |
| `cst/` | CST Studio set-up guide for MIFA return loss, efficiency and pattern; results to be added |

### 03_Firmware_and_Test
| Folder | Contents |
|---|---|
| `firmware/` | Zephyr DTM application (for conducted TX/RX tests with a BLE tester or SA) and a BLE advertiser with a per-channel TX-power table |
| `test_plan/` | Tests M-01…M-12 mapped to requirements and to EN 300 328 / FCC §15.247 clauses |
| `lab_procedures/` | Step-by-step VNA / spectrum-analyser / signal-generator procedures for the University of Oulu RF laboratory |
| `results/` | Measured-results table — **empty until real measurements exist** |
| `docs/` | Design report rev A, risk register, design-controls note mapping the workflow to ISO 13485 / ISO 14971 practice |

---

## 4. Design workflow

1. **Requirements** — radio, electrical, mechanical and firmware requirements written first, each with a measurable acceptance criterion.
2. **Architecture and analysis** — link budget sets the TX power and sensitivity targets; harmonic budget decides whether a filter is needed; CPWG sweep fixes the trace geometry; matching network designed for the expected antenna impedance.
3. **Schematic and netlist** — three sheets, netlist generated programmatically and imported into KiCad.
4. **Layout** — RF path routed first: chip → matching → u.FL → ESD → antenna, on one layer with continuous ground and stitching vias; crystal and DC-DC kept away from the feed.
5. **Simulation** — MIFA and feed modelled in CST before ordering.
6. **Fabrication** — 5 boards, 2–3 assembled.
7. **Measurement** — conducted and radiated tests per M-01…M-12; results compared against the analysis.
8. **Design review** — deviations fed back into rev B.

---

## 5. Measurement plan (summary)

| ID | Test | Instrument | Requirement | Standard clause |
|---|---|---|---|---|
| M-01 | Antenna return loss / bandwidth | VNA | RF-01 | — |
| M-02 | Feed-line impedance (TDR / VNA) | VNA | RF-02 | — |
| M-03 | Matching-network verification | VNA | RF-03 | — |
| M-04 | Conducted output power per channel | SA via u.FL | RF-04 | EN 300 328 §4.3.2.2 / FCC §15.247(b) |
| M-05 | Occupied bandwidth | SA | RF-05 | FCC §15.247(a) |
| M-06 | Conducted spurious / harmonics | SA | RF-06 | FCC §15.247(d), §15.209 |
| M-07 | Radiated spurious (pre-scan) | SA + antenna | RF-07 | EN 300 328 §4.3.2.9 |
| M-08 | RX sensitivity (DTM, PER) | SG / BLE tester | RF-08 | — |
| M-09 | Crystal frequency accuracy | SA | RF-09 | — |
| M-10 | Current consumption (TX / RX / sleep) | DMM / power analyser | EL-xx | — |
| M-11 | Advertising range test | Phone / sniffer | RF-10 | — |
| M-12 | Body-loading effect on antenna | VNA | RF-11 | — |

Full procedures, pass/fail limits and setup photos live in `03_Firmware_and_Test/`.

---

## 6. Measured results

| Test | Expected (analysis) | Measured | Pass / Fail | Date |
|---|---|---|---|---|
| — | — | — | — | — |

*This table is deliberately blank. It will be populated only with data measured on fabricated boards.*

---

## 7. Tools used

- **KiCad 8** — schematic capture, PCB layout, Gerber export
- **Python** — scikit-rf, NumPy, matplotlib for RF analysis; SKiDL for netlist generation; schemdraw for schematic figures
- **CST Studio Suite** — antenna and feed simulation
- **Zephyr RTOS / nRF Connect SDK** — firmware
- **Rohde & Schwarz VNA / spectrum analyser, signal generator** — measurement (University of Oulu RF lab)

---

## 8. Standards referenced

- ETSI EN 300 328 — Wideband transmission systems in the 2.4 GHz band
- FCC 47 CFR §15.247 and §15.209 — Digital modulation in the ISM band; general radiated-emission limits
- IEC 60601-1-2 — EMC for medical electrical equipment (used as a design-margin reference only; this board is not a medical device)
- ISO 13485 / ISO 14971 — design-controls and risk-management practice followed in the documentation structure

---

## 9. Roadmap

- [x] Requirements, architecture, link budget
- [x] Schematics, BOM, netlist
- [x] Matching network, CPWG geometry, harmonic budget
- [x] MIFA footprint and 4-layer stack-up
- [x] Firmware (DTM + advertiser)
- [x] Test plan and lab procedures
- [ ] KiCad placement and routing
- [ ] CST simulation of antenna and feed
- [ ] Gerber release and fabrication order
- [ ] Assembly and bring-up
- [ ] Bench measurements M-01…M-12
- [ ] Results, deviations and rev B notes

---

## 10. Related work

- [RF-Microwave-Circuit-Design-Portfolio](https://github.com/dipucwc) — Keysight ADS / CST exercises and a KiCad solar home power-control board
- MIMO-OFDM PHY simulator and RF transceiver algorithm-to-firmware repositories (see profile)

---

## 11. Author

**Md Moklesur Rahman** — RF / PHY systems engineer, Oulu, Finland
MSc Wireless Communications Engineering, University of Oulu (CWC)
GitHub: [dipucwc](https://github.com/dipucwc) · Email: moklesur.eee@gmail.com

## License

MIT — see `LICENSE`. Hardware design files are shared for educational and portfolio purposes; no certification claims are made for this board.
