# Testbed R-2R DAC/Streamer — System Architecture Specification
**Revision:** 1.1  
**Date:** 2026-05-15  
**Author:** Wayne  
**Status:** Draft

---

## 1. Purpose and Scope

This document defines the architecture and design requirements for a stereo R-2R ladder DAC/Streamer testbed. The primary goals are:

1. Validate the sign-magnitude R-2R converter architecture as a proof of concept — "can we get good sound out of this?"
2. Serve as a learning platform for digital audio processing in an FPGA
3. Produce a finished, usable product suitable as a gift (Ward) with RPi4 and Volumio

**Note on device character:** integrating an RPi4 as the primary I2S source (running Volumio) means this is properly a *network streamer with an integrated R-2R DAC*, not a standalone DAC. The RPi also hosts a UART control link to the FPGA and a web interface for real-time DAC parameter control and diagnostics — capabilities that reinforce the streamer framing.

This testbed intentionally uses a simplified converter architecture (74HC595 shift registers, single resistor per bit) compared to the intended final product (discrete MOSFET switches, PMV16XN, separated gate/Vref domains). The testbed validates the signal chain and digital processing; the final product implements the full MSB-style architecture with all identified improvements.

---

## 2. System Overview

### 2.1 Architecture Summary

The device accepts digital audio via two input paths (RPi4 I2S and S/PDIF coax), processes it entirely in an FPGA, and drives four R-2R converter modules in a sign-magnitude single-ended topology. A UART link between the integrated RPi4 and the FPGA provides real-time parameter control and diagnostics via a web interface hosted on the RPi. All analog processing is passive, and it makes sense to include an output buffer and balanced converter from the Soekris design. The system is self-contained with an internal power supply.

### 2.2 Block Diagram

See attached block diagram (testbed_dac_block_diagram_v2).

### 2.3 Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| FPGA platform | Digilent Cmod A7-35T | $99, Artix-7, 48-pin DIP, onboard SRAM, USB programming |
| Converter resolution | 17-bit (16-bit magnitude + sign) | Exceeds CD quality by 1 bit with high def sources; adequate for architecture validation |
| Converter switch | 74HC595 shift register | Simple, available, sufficient for testbed |
| Reference voltage | ±2.5V | Standard audio level, compatible with R=150Ω ladder |
| Ladder R value | 150Ω | Low enough for adequate output level; scalable (300/600/1200Ω future) |
| Oversampling | 8× in FPGA | Pushes image to ~352kHz; allows simple passive analog filter |
| Output filter | Passive 2nd order RC | No active components in signal path; DC coupled |
| Output coupling | DC direct | No capacitor in signal path; relies on mute relay for protection |
| Clock architecture | Dual oscillator + FIFO | Complete clock domain isolation; OCXO owns the sample clock |
| FPGA control link | UART (RPi ↔ FPGA) | Real-time parameter control and diagnostics; RPi hosts web UI |

---

## 3. Input Section

### 3.1 Input Sources

Two input paths share a common downstream processing chain after the input mux:

**Path 1 — RPi4 I2S (primary)**
- Interface: 3-wire I2S (BCLK, LRCLK, DATA) via header connector
- Source: Raspberry Pi 4 running Volumio
- Word length: 32-bit frames, 24-bit audio data
- Sample rates: 44.1kHz, 48kHz, 88.2kHz, 96kHz, 176.4kHz, 192kHz
- Note: RPi BCLK may gate between tracks — FPGA must handle clock absence gracefully

**Path 2 — S/PDIF Coax (secondary)**
- Connector: BNC, 75Ω terminated
- Input buffer: Schmitt trigger (e.g. 74LVC1G17) converts 0.5V SPDIF to 3.3V LVCMOS
- Decoded entirely in FPGA — no receiver IC
- Compatible with Logitech Transporter coax output


### 3.2 Input Selection

Software-controlled mux in FPGA selects active input. Default: RPi4 I2S.

### 3.3 S/PDIF Decoder (FPGA, Path 2)

Implemented entirely in SystemVerilog, approximately 150 lines:

1. Transition detector
2. Biphase mark (BMC) decoder
3. Preamble detector — X (channel A), Y (channel B), Z (block start)
4. Frame counter and subframe extractor
5. Sample rate estimator — counts fabric clock cycles between Z preambles
6. CLK mux select output — routes to 44.1kHz or 48kHz oscillator

At 196.608 MHz fabric clock, each SPDIF bit at 3.072 MHz maximum rate occupies ~64 clock cycles — trivial to decode reliably.

---

## 4. Clock Architecture

### 4.1 Philosophy

The DAC owns its master clock. All internal processing and converter timing is synchronous to a local precision oscillator. The input data stream is decoupled from the internal clock via an asynchronous FIFO. This eliminates jitter from the RPi or any external S/PDIF source from affecting the analog output.

### 4.2 Oscillators

Two crystal oscillators, one per sample rate family:

| Oscillator | Frequency | Family | Part (candidate) |
|---|---|---|---|
| OSC1 | 45.1584 MHz | 44.1kHz (×1024) | NDK NZ2520SDA or Crystek CCHD-957 |
| OSC2 | 49.152 MHz | 48kHz (×1024) | NDK NZ2520SDA or Crystek CCHD-957 |

Note: NDK NZ2520SDA preferred for superior close-in phase noise below 25kHz (audio band). Crystek CCHD-957 superior above 25kHz (irrelevant for audio).

### 4.3 Clock Mux

Single logic gate (74LVC1G157 or equivalent) selects active oscillator based on FPGA sample rate detection output. FPGA detects sample rate family from LRCLK frequency (I2S) or Z-preamble rate (S/PDIF).

### 4.4 FPGA PLL (MMCM/PLLE2)

Artix-7 PLLE2 multiplies selected oscillator 4x to 196.608 MHz fabric clock for 48kHz family and 180.6336MHz for 44.1kHz family.


### 4.5 Clock Output (Logitech Transporter)

OCXO-derived clock buffered and output via BNC connector for Logitech Transporter word clock input. Transporter locks to this clock and its S/PDIF output becomes synchronous to the DAC — near-zero drift across the async FIFO when Transporter is the source.

- Buffer: 74LVC1G125 or equivalent, 75Ω series termination
- Connector: BNC

---

## 5. Asynchronous FIFO

### 5.1 Purpose

Crosses from the input clock domain (RPi BCLK or recovered S/PDIF clock) to the OCXO clock domain. Decouples jitter and frequency offset of the input source from the DAC output.

### 5.2 Implementation

- Memory: IS61WV5128BLL-10BLI SRAM on Cmod A7 (512KB, 8-bit wide, 10ns)
- Write port: clocked by input BCLK or recovered S/PDIF clock
- Read port: clocked by OCXO-derived fabric clock
- Pointer width: 14-bit (16,384 sample depth)
- Initial pointer offset: half depth (8,192 samples) — maximum margin for clock drift

### 5.3 Depth Rationale

Worst case clock delta (RPi 100ppm + OCXO 1ppm = 101ppm) over 10 minutes at 96kHz:
- Drift = 96,000 × 600 × 101×10⁻⁶ = 5,818 samples
- Required depth = 2 × 5,818 = 11,636 → round to 16,384 (14-bit)
- SRAM capacity is vastly larger; depth chosen for latency minimisation

### 5.4 FIFO Management

- On input clock loss (>1ms BCLK absence): declare stream inactive, mute output, reset pointers
- On stream resume: fill to half depth, then start read pointer and un-mute
- Gray code pointer synchronisation for safe clock domain crossing

---

## 6. FPGA Digital Processing Chain

All processing in the OCXO clock domain at 196.608 MHz. Approximately 4,459 clock cycles available per sample at 44.1kHz.

### 6.1 I2S Deserializer

- Accepts 32-bit I2S frames from FIFO output (or direct from RPi path)
- Extracts 24-bit audio data, discards padding
- Outputs left and right 24-bit words aligned to LRCLK

### 6.2 Format Conversion: 2s Complement → Sign-Magnitude

- Performed immediately after deserialiser — earliest possible point
- 24-bit 2s complement → 1-bit sign + 23-bit magnitude
- Pure combinational logic: sign = MSB; magnitude = (sign ? ~data[22:0]+1 : data[22:0])
- All downstream processing operates on sign-magnitude

### 6.3 FIR Oversampling Filter

- Oversampling ratio: 8× (adaptive by sample rate — see table below)
- Filter length: 64–128 taps (TBD based on stopband requirements)
- Implementation: Artix-7 DSP48E1 blocks (90 available; filter uses ~8–16)
- Coefficients: computed offline in Python (scipy.signal), stored in FPGA BRAM
- Operates at 24-bit precision; internal accumulator 48-bit (DSP48E1 native)

| Input Rate | Oversample | Output Rate | First Image |
|---|---|---|---|
| 44.1 kHz | 8× | 352.8 kHz | 332 kHz |
| 48 kHz | 8× | 384 kHz | 364 kHz |
| 96 kHz | 4× | 384 kHz | 364 kHz |
| 192 kHz | 2× | 384 kHz | 364 kHz |

### 6.4 TPDF Dither

Applied immediately before truncation:

- Two independent LFSRs (17-bit and 23-bit, different lengths for statistical independence)
- LSB of each LFSR summed and subtracted by 1: result is −1, 0, or +1 with triangular distribution
- Added to audio sample at bit position 8 (the 16-bit LSB boundary within 24-bit word)
- Completely decorrelates quantisation error from signal

### 6.5 Truncation

- 24-bit magnitude → 16-bit magnitude
- Applied after dither addition
- Result: 1-bit sign + 16-bit magnitude per channel

### 6.6 Zero Crossing Detection and Converter Routing

Note - not sure any of this is needed:
- Monitors sign bit each sample
- On sign change: sequence idle converter pair to zero before activating; active pair latches zeros
- Manages handoff to prevent glitch at crossover
- Routes positive samples to +L/+R converters; negative samples to −L/−R converters

### 6.7 SR Serialisers (×4)

One serialiser per converter module:

- Independent DATA and LATCH signals per converter (enables per-converter timing and future BDT implementation)
- Shared SCLK across all converters
- Shift rate: up to 20 MHz SCLK → 8 clocks per update → ~2.5 MHz maximum conversion rate
- Both 74HC595 chips in each module shift simultaneously (parallel, not daisy-chain) — doubles effective update rate

---

## 7. Converter Modules

### 7.1 Overview

Four identical converter modules in sign-magnitude topology:
- Converter +L (positive, left channel)
- Converter −L (negative, left channel)  
- Converter +R (positive, right channel)
- Converter −R (negative, right channel)

At any instant, only one converter per channel carries the audio signal; the complementary converter holds zero.

### 7.2 Shift Register

- IC: 74HC595 (8-bit serial-in, parallel-out with output latch)
- Count per module: 2× (16-bit output total)
- Data: 16-bit magnitude, MSB first
- Latch: simultaneous on both SRs via independent LATCH signal from FPGA

### 7.3 R-2R Ladder

- Topology: standard binary-weighted R-2R
- R value: 150Ω (2R = 300Ω)
- Resistor tolerance: 0.01% thin-film (required for 16-bit linearity)
- Bits: 16 (15 magnitude + termination)
- Resistor count per module: 31 (15× R + 16× 2R)

> **Note on scalability:** R=150Ω base supports future parallel module configurations: 2 modules parallel → effective R=75Ω (add 600Ω); 4 modules → R=37.5Ω (add 1200Ω). Native resistor progression: 150, 300, 600, 1200Ω — all standard E96 values.

> **Note on parts availability** 5/15/26 on JLCPCB site, 100 ohm and 200 ohm R available, parts are 0.02% / 10PPM/C / 0805 package; PTFR0805Q100RN9 and PTFR0805Q200RN9.  It was found that 0.01% parts were not available or prohibitive cost: $8 to $65 per part.  This is probably why Soekris started with 0.05% and there was discussion about going to 0.02% and a goal of 0.01% but I don't know if it was ever resolved.  In addition, these parts are available at Mouser: 708-RNCF0603TKW100R and 708-RNCF0603TKW200R 100 ohms / 200 ohms which are 0.01%, 2ppm, 0603 package, and $1.76 each in volume of 100.  

### 7.4 R-2R Ladder Leg Trim (Invention — Wayne)

In a standard R-2R ladder DAC using discrete MOSFET switches, switch Ron adds in series with each 2R rung, making the effective rung impedance 2R+Ron instead of 2R. This breaks the exact 2:1 ratio required for linearity-free operation, producing a gain error of Ron/(2R+Ron). This is a pure gain error — INL and DNL are unaffected because all switches carry identical current regardless of code.

**The invention:** Add a trim resistor of value Ron/2 in series with each ladder leg (the horizontal R resistors). The rungs are left unchanged.

**Proof of exact correction:**
- Rungs: 2R + Ron (unchanged)
- Legs: R + Ron/2 (trimmed)
- Ratio: (2R + Ron) / (R + Ron/2) = 2(R + Ron/2) / (R + Ron/2) = **exactly 2**

The correction is algebraically exact for any value of Ron. Trim resistors need only 1% tolerance because they are correcting a systematic deterministic error, not a random matching error.

**Partial trim:** Trimming only the upper N bits (e.g. top 8 of 16) captures essentially all the benefit. The absolute voltage error on lower bits scales with their bit weight and falls below the noise floor rapidly — component count and layout complexity in the lower ladder section can be reduced without meaningful performance penalty.

**Simulation confirmed:** MSB transition discontinuity present without trim, completely eliminated with trim. INL/DNL flat to numerical noise floor in both cases.

**Applicability:** This correction applies specifically to the final product architecture using discrete MOSFET switches (PMV16XN). The testbed uses 74HC595 shift registers which have no discrete switch Ron in this sense; the trim is not required here but the principle is validated by simulation and ready to apply to the final product PCB.

**Status:** Independently invented by Wayne during DAC3 design. Not previously encountered in published R-2R DAC literature.

---

### 7.5 Reference Voltage

- Positive: +2.5V from LT3042 LDO (0.8 nV/√Hz)
- Negative: −2.5V from LT3094 LDO (2 nV/√Hz)
- Input to LDOs: ±5V from power supply
- LDO dissipation: (5−2.5) × ~50mA = 125mW per device — no heatsink required
- All four converter modules share one Vref rail pair — noise is correlated, not averaged (acceptable for testbed)

> **Critical architectural note:** 74HC595 VCC is powered from the **digital supply**, NOT from Vref. This separates the digital switching domain from the Vref domain. This is the same architectural separation that gives MSB its performance advantage over Denafrips (which powers 74LV595 from Vref, injecting ~400mA switching transients onto the reference).

### 7.6 Module Connectors

**Status: TBD — connector selection deferred pending further research.**

The connector choice is more consequential than it first appears: if the prototype motherboard is designed from the outset as a **testbench for final product converter modules**, then the connector fitted here becomes the de-facto standard for the final product. Changing it later means redesigning both the motherboard and every converter module. This warrants choosing carefully rather than defaulting to a convenience header.

Design intent:
- Backplane plugin style (module plugs vertically into motherboard), not discrete pin headers
- MSB Technology's Cascade architecture uses a similar physical arrangement — high-density edge or board-to-board connector with good mechanical retention and low contact resistance; exact part to be identified from product photos and teardown references
- Two connectors per module (one near each end) for mechanical stability, matching MSB physical layout
- Gold contacts preferred for audio-grade reliability
- Connector must be available through standard distributors (LCSC, Mouser, DigiKey) in small quantities

Candidate families to evaluate:
- Samtec high-density board-to-board (e.g. LSHM, QSE/QTE series)
- TE Connectivity ERNI or DIN 41612 Eurocard-style (if pitch and depth suit the module height)
- Custom edge connector (if MSB approach is confirmed to be gold-finger style)

Fallback (simple pin header, adequate for testbed-only use):
- Würth Elektronik WR-PHD 2.54mm, 12-pin 2-row
  - Module (right-angle pin header): 61301221021 — $0.96
  - Motherboard (vertical socket): 61301221821 — $0.97

**Action required:** Research MSB connector type from available photos/teardowns before committing to PCB footprint. See open items.

### 7.7 Output-Referred Kelvin Calibration (Invention — Wayne)

#### Background

In a sign-magnitude R-2R DAC with discrete MOSFET switches, the positive and negative half-cycles of the output waveform are governed by Vref+ and Vref− respectively. Any asymmetry between these rails — whether from LDO trim error, ladder resistor mismatch, switch Ron variation, or connector resistance — produces a gain difference between the two half-cycles. In single-ended mode this appears as second harmonic distortion; in balanced mode it is partially but not fully rejected by the downstream differential receiver.

The naive correction — measuring Vref+ and Vref− directly and applying a digital scale factor from the ratio — is problematic for two reasons: measuring a negative rail with an FPGA ADC requires additional level-shifting hardware; and more fundamentally, it captures only one of many asymmetry sources.

#### The Invention

Rather than measuring the supply rails, calibration measures the **DAC output directly**, closing the correction loop around the entire analog signal chain. This is the classical Kelvin sensing principle applied at the system level: measure at the load, not at the source.

#### Calibration Procedure

At power-up, before audio playback begins:

1. Output a code corresponding to (something like) +0.75 × FS on the positive half
2. Output a code corresponding to (something like) −0.75 × FS on the negative half
3. Feed both outputs into a difference amplifier
4. Adjust the codes digitally — trimming one side toward the other — until the difference amplifier output is nulled to zero
5. The code delta required to achieve null is the correction factor
6. Store the correction factor as a digital scale factor in the FPGA
7. Apply the scale factor as a multiply to all subsequent output codes during playback

The 0.75 × FS operating point avoids ladder endpoint nonlinearity while keeping the signal large enough for reliable null detection.

#### Key Properties

The difference amplifier functions as a **null detector only** — not a precision measurement device. A null detector requires only sufficient resolution to determine the sign and approximate magnitude of the error, and short-term stability over the seconds during which calibration runs. It does not require calibrated absolute accuracy, low offset, or precision gain. Any competent op-amp satisfies the requirement.

Because measurement is taken at the DAC output, **every source of positive/negative asymmetry in the entire analog chain is captured and corrected simultaneously**, including:

- Vref+ / Vref− magnitude mismatch from LDO trim
- Ladder resistor ratio errors
- PMV16XN Ron variation between positive and negative switch banks
- Switch threshold voltage spread
- Connector and PCB trace resistance asymmetry

None of these sources need to be identified or measured individually. The correction is total and automatic.

#### Hardware Cost

Zero additional connector pins are required — the DAC analog output is already present on the analog connector. The only additional hardware is a single difference amplifier: a simple dual op-amp on the motherboard, shared across all modules, not in the audio signal path during normal playback.

#### Thermal Considerations

Calibration is performed at power-up and repeated at intervals during thermal warm-up. The FPGA monitors successive calibration results and locks the scale factor when the delta between iterations falls below a threshold, indicating thermal equilibrium. This ensures the correction reflects operating temperature rather than cold-start conditions.

#### Comparison with Prior Art

Standard R-2R DAC implementations (Denafrips, Soekris) apply no asymmetry correction of this type. Supply-rail measurement approaches require bipolar ADC capability and correct only one of many asymmetry sources. The output-referred approach as described here — using the DAC itself as the measurement instrument, with a null detector as the only additional hardware, correcting all asymmetry sources simultaneously — has not been previously encountered in the published DIY or commercial DAC literature. MSB Technology is hypothesised to implement a similar approach given the architectural sophistication of their calibration system, but this has not been confirmed.

**Applicability:** Final product architecture (discrete MOSFET switches). Not applicable to the testbed (74HC595, no independent gate/Vref domain separation), but the design and firmware can be stubbed in the testbed FPGA for later activation.

**Status:** Independently invented by Wayne during DAC3/final product architecture design.

---

## 8. Analog Output Section

### 8.1 Output Filter

Passive 2nd order RC, one per channel (L and R — not per converter):

| Component | Value | Function |
|---|---|---|
| R-2R Thevenin impedance | 75Ω | First filter resistor (inherent) |
| Series resistor | 150–220Ω | Second filter resistor |
| Shunt capacitor | 3–5 nF | Filter capacitor to AGND |

- Filter pole: ~100 kHz
- Attenuation at 352 kHz (8× image): ~−22 dB (2nd order) — adequate given 8× oversampling
- DC coupled throughout — no capacitor in signal path
- Full scale output: ~1.25V peak into high-impedance load

### 8.2 Mute Relay

One relay per channel, shorts output to AGND:

- Type: Omron G6K-2F-Y or equivalent SMD DPDT, gold contacts
- Function: protects 4P1L tube preamp filament input from DAC startup/shutdown transients
- Control: RC-delayed release (~2–3 seconds after power supply stable)
- Engages immediately on power loss (relay de-energises)

### 8.3 Output Connector

- Type: RCA phono jack, one per channel
- Coupling: DC direct (no output capacitor)
- Load: 4P1L tube preamplifier, filament input (~50kΩ–several hundred kΩ — no loading concern)

### 8.4 Output Options

- Populate output buffer and balanced converter from the Soekris design as a solder-in option.

---

## 9. Power Supply

### 9.1 Overview

Self-contained internal power supply. No external wall adapter.

### 9.2 Architecture

| Rail | Voltage | Load | Purpose |
|---|---|---|---|
| +5V digital | +5V | Cmod A7, logic ICs | FPGA and digital logic |
| +Vref | +2.5V (LT3042) | R-2R ladders | Positive reference |
| −Vref | −2.5V (LT3094) | R-2R ladders | Negative reference |
| +5V analog input | +5V to LT3042 | LDO input | Regulated by LT3042 |
| −5V analog input | −5V to LT3094 | LDO input | Regulated by LT3094 |

### 9.3 Mains Input

- Connector: IEC C14 inlet with integrated fuse and switch
- Transformer: small toroidal or R-core, dual secondary

### 9.4 Estimated Total Power

- FPGA (Cmod A7): ~500mW typical
- Converter logic (4× 74HC595 pairs): ~200mW
- R-2R ladder (4 converters): ~130mW
- LT3042/LT3094 LDO dissipation: ~250mW
- **Total: ~1.1W** — very low dissipation, small transformer adequate

---

## 10. Mechanical Architecture

### 10.1 Module/Motherboard Split

| Board | Qty ordered | Qty used | Contents |
|---|---|---|---|
| Converter module | 5 | 4 | 74HC595×2, R-2R resistors, module connectors |
| Motherboard | 5 | 1 | Cmod A7 socket, Vref LDOs, oscillators, CLK mux, input buffers, output filter, power supply |

JLCPCB minimum order is 5 units — 1 converter module spare, 4 motherboards spare (blanks or assembled to taste).

### 10.2 Module Orientation

Converter modules mount edge-on (vertical) into motherboard, consistent with MSB Cascade physical arrangement. Final connector orientation (right-angle on module, vertical socket on motherboard, or edge connector) to be confirmed once connector type is selected — see §7.6.

### 10.3 Mechanical Retention

Spreader bar or standoffs between module PCBs to prevent connector rocking. Simple aluminium bar with M3 standoffs adequate for testbed.

---

## 11. FPGA Resource Estimate (Artix-7 XC7A35T)

| Resource | Available | Estimated Use | Margin |
|---|---|---|---|
| LUTs | 33,280 | ~1,500 | 95% |
| DSP48E1 | 90 | ~16 (FIR) | 82% |
| BRAM (36Kb) | 50 | ~10 (coefficients, small buffers) | 80% |
| I/O pins | 44 | ~20 | 55% |
| MMCM/PLL | 5 | 1 | 80% |

Artix-7 35T is very comfortably sized for this application.

---

## 12. I/O Pin Allocation (Cmod A7)

| Signal | Pins | Notes |
|---|---|---|
| I2S in (BCLK, LRCLK, DATA) | 3 | RPi4 input |
| S/PDIF coax in | 1 | After schmitt buffer |
| Toslink in | 1 | Direct from module |
| Shared SCLK to converters | 1 | All 4 converters |
| DATA per converter | 4 | Independent |
| LATCH per converter | 4 | Independent — enables timing control |
| CLK mux select | 1 | Oscillator family select |
| Mute relay control | 2 | One per channel |
| Status LED | 1 | Power/lock indicator |
| Clock out enable (optional) | 1 | BNC clock output control |
| UART TX (FPGA → RPi) | 1 | Control link — 3.3V direct, compatible with RPi GPIO |
| UART RX (RPi → FPGA) | 1 | Control link |
| **Total** | **~21** | **23 pins remaining** |

---

## 13. Software / Firmware

### 13.1 FPGA Development Environment

- Synthesis/implementation: Vivado ML Standard, Ubuntu 20.04 VM via UTM on M2 MacBook Pro
- Simulation: Icarus Verilog + Verilator + GTKWave (native macOS via Homebrew)
- Language: SystemVerilog exclusively
- Workflow: simulate first on Mac, synthesise in Vivado only when logic verified

### 13.2 FIR Coefficient Generation

- Tool: Python / scipy.signal
- Output: SystemVerilog coefficient ROM or BRAM initialisation file

### 13.3 Host Software (RPi4)

- OS: Raspberry Pi OS
- Player: Volumio (configured for bit-perfect I2S output)
- Control interface: lightweight Flask web server running alongside Volumio on a separate port (e.g. :8080); accessible from phone/tablet browser
- No display required on the unit

### 13.4 UART Control Link (RPi ↔ FPGA)

Physical layer:
- RPi UART: GPIO 14 (TX) / GPIO 15 (RX) — `/dev/ttyAMA0` with Bluetooth disabled, or `/dev/ttyS0`
- FPGA I/O: 3.3V LVCMOS — direct connection, no level shifting required (both 3.3V)
- Baud rate: 115200 8N1

Protocol (ASCII line-oriented, CR/LF terminated):
- RPi → FPGA: command strings, e.g. `SET INPUT 0`, `SET MUTE ON`, `SET DITHER ON`, `GET STATUS`
- FPGA → RPi: ACK/NAK per command, plus unsolicited periodic status packet containing FIFO fill %, detected sample rate, lock status, error counters

FPGA implementation:
- Standard UART RX/TX module in SystemVerilog (~100 lines)
- Small command parser and status reporter
- Controllable parameters: input source select, mute, dither enable/disable, future: oversampling ratio override
- Diagnostic readbacks: FIFO fill level, sample rate detected, S/PDIF lock, BCLK presence

### 13.5 Web Control Interface

Hosted on RPi4, served by Flask (Python). Communicates with the FPGA by writing to `/dev/ttyAMA0`.

Controls exposed:
- Input source select (RPi I2S / S/PDIF coax)
- Mute toggle
- Dither enable
- Status display: detected sample rate, FIFO health bar, lock indicator, uptime

Diagnostics panel:
- Real-time FIFO fill level (polls FPGA via UART every ~1s)
- Error counter display (FIFO underrun/overrun, S/PDIF lock loss events)
- FPGA firmware version readback

UI: single responsive HTML page, minimal JavaScript (no framework). Runs alongside Volumio without conflict.

---

## 14. Open Items and Action Items

| # | Item | Priority |
|---|---|---|
| 1 | Verify 44.1kHz family PLL configuration (45.1584 MHz → 196.608 MHz) | High |
| 2 | Confirm RPi BCLK behaviour between tracks using AD2 | High |
| 3 | Confirm LT3042/LT3094 availability at JLCPCB | High |
| 4 | Select and verify 0.01% resistor values (150Ω, 300Ω) on LCSC | High |
| 5 | Verify Toslink module 3.3V compatibility with Cmod A7 I/O | Medium |
| 6 | Design schmitt buffer circuit for coax S/PDIF input | Medium |
| 7 | Design mute relay RC delay circuit | Medium |
| 8 | Select oscillator parts (NDK vs Crystek) and confirm LCSC stock | Medium |
| 9 | Determine PCB dimensions for converter module and motherboard | Medium |
| 10 | Write and simulate SystemVerilog: S/PDIF decoder | Medium |
| 11 | Write and simulate SystemVerilog: async FIFO controller | Medium |
| 12 | Generate FIR coefficients and verify frequency response | Medium |
| 13 | Write and simulate SystemVerilog: full processing chain | Medium |
| 14 | Identify MSB Cascade module connector from photos/teardowns; evaluate backplane-style alternatives (Samtec, DIN 41612, edge connector) — decision gates PCB layout | High |
| 15 | Investigate Logitech Transporter clock input frequency requirement | Low |
| 16 | Confirm RPi UART port to use (ttyAMA0 vs ttyS0) — disable BT if needed | High |
| 17 | Write and simulate SystemVerilog: UART RX/TX module and command parser | Medium |
| 18 | Define UART command set and status packet format | Medium |
| 19 | Write RPi Flask web control server and HTML UI | Medium |
| 20 | Test UART link end-to-end: RPi ↔ FPGA command/response | Medium |

---

## 15. Future / Final Product Notes

The following are noted here for continuity but are NOT in scope for this testbed:

- Discrete MOSFET switches (PMV16XN) replacing 74HC595 — separates gate drive from Vref domain completely
- ICL7660 charge pump + 74LVCH16T245 level shifters for negative converter drive
- LT6657-2.5 metrology reference for FPGA ADC calibration
- Per-converter error correction LUTs (FPGA ADC calibration → pre-distortion)
- GTP transceiver fiber link (Cascade-style DD↔converter chassis)
- OCXO as master timebase with DD chassis slaving
- BDT (Bit Diffusion Technology) — MSB-first staircase with per-converter latch timing
- Spatial staggering of converter latch timing via MMCM phase
- Volume control relay attenuator (Omron G6K-2F-Y family candidate)
- Two-chassis architecture (PSU + converter) with Lemo cable

---

*End of document — Rev 1.1*
