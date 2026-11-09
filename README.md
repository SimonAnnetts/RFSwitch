# RFSwitch

A 4-way RF switch (one input, four selectable outputs) built from conventional
through-hole parts from my parts bin: **J310 JFETs**, a **CD4104-style quad
low-to-high voltage level translator**, and a handful of resistors and
capacitors.

The RF path is switched at 9–12 V, but the channel-select lines are plain 3.3 V
logic, so the board can be driven directly from a Raspberry Pi Pico W (or any
3.3 V source).

* KiCad project: [RFSwitch/RFSwitch.kicad_pro](RFSwitch/RFSwitch.kicad_pro)
* Schematic: [RFSwitch/RFSwitch.kicad_sch](RFSwitch/RFSwitch.kicad_sch)
* PCB: [RFSwitch/RFSwitch.kicad_pcb](RFSwitch/RFSwitch.kicad_pcb)
* Symbols, footprints and 3D models used by the project are in `RFSwitch/libs` and `RFSwitch/packages3D`.

10 PCBs were ordered from JLCPCB in November 2025 costing about GBP 5.00 in total with shipping included. The boards are 50 mm × 50 mm, two-layer, ENIG finish, with a solder mask and silkscreen.

## Interfaces

### Control header J1 (1x8, 2.54 mm)

| Pin | Signal     | Description                                              |
|-----|------------|----------------------------------------------------------|
| 1   | GND        | Ground                                                   |
| 2   | VCC        | 9–12 V supply for the FET bias and switching             |
| 3   | +3V3       | 3.3 V logic supply (from the Pico's 3V3 out)             |
| 4   | EN         | Enables the level-shifter outputs                        |
| 5   | RF1_SEL    | Select channel 1 (3.3 V logic)                           |
| 6   | RF2_SEL    | Select channel 2                                         |
| 7   | RF3_SEL    | Select channel 3                                         |
| 8   | RF4_SEL    | Select channel 4                                         |

### RF connectors

Five edge-mount SMA connectors: `RFIN` (common) and `RFOUT1` … `RFOUT4`.

## How it works

### 1. Signal path

```
                       RF5 (common node, DC-biased to VCC via R18 10k)
                        │
 RFIN ──L1──C4(100n)────┤
                        ├── Q1 (series J310) ── RF1 ──C9(100n)──┬── RFOUT1
                        │                         │             C13 2.2p to GND
                        │                    Q5 (shunt J310)
                        │                         │
                        │                 "Net-(Q5-S)" node ── R19 10k to VCC
                        │                         │             C5..C8 100n to GND
                        ├── Q2 ── RF2 ── (Q6 shunt) ... C10 ── RFOUT2
                        ├── Q3 ── RF3 ── (Q7 shunt) ... C11 ── RFOUT3
                        └── Q4 ── RF4 ── (Q8 shunt) ... C12 ── RFOUT4
```

Each channel has two J310 JFETs:

| Channel | Series FET (common ↔ channel) | Shunt FET (channel ↔ RF ground) |
|---------|-------------------------------|---------------------------------|
| 1       | Q1                            | Q5                              |
| 2       | Q2                            | Q6                              |
| 3       | Q3                            | Q7                              |
| 4       | Q4                            | Q8                              |

* The **series** FET connects the common node `RF5` to the channel node
  (`RF1`…`RF4`). When it is on, the signal passes through; when it is off, the
  channel is disconnected.
* The **shunt** FET connects the channel node to the shunt-source net
  (`Net-(Q5-S)`). That net is held at VCC by R19 (10 k) and decoupled to ground
  by C5–C8 (100 nF), so it is an **RF ground**. When the shunt FET is on it
  shorts any leakage from an unselected channel to ground, which improves
  off-channel isolation.
* `RFIN` reaches the common node through L1 and the DC-blocking capacitor C4;
  each output has its own DC-blocking capacitor (C9–C12, 100 nF).

### 2. Why a J310 can be a switch here

A J310 is a depletion-mode N-channel JFET: with V<sub>GS</sub> = 0 V it is fully
**on** (low channel resistance), and it turns **off** when the gate is taken
several volts below the source (pinch-off is roughly −2 to −6 V).

The schematic takes advantage of this by putting the whole RF path at a **DC
level of VCC**:

* `RF5` is pulled to VCC through R18 (10 k).
* The shunt-source net is pulled to VCC through R19 (10 k).
* Because a channel node is always connected either to `RF5` (series FET on) or
  to the shunt-source net (shunt FET on), every FET source and drain sits at
  about VCC in DC terms.
* The C4 and C9–C12 capacitors block that DC from the connectors.

The FET gates are then driven between **VCC** and **0 V**:

* Gate = VCC → V<sub>GS</sub> = 0 V → FET **on**.
* Gate = 0 V → V<sub>GS</sub> ≈ −VCC (−9 to −12 V) → FET firmly **off**.

Pinching off with the full supply (far beyond pinch-off voltage) keeps the
off-FETs well clear of conduction on RF signal peaks. This is why the board
wants a 9 V or 12 V supply rather than 3.3 V: a 3.3 V gate swing could not
reliably pinch off a J310.

### 3. Level shifting with the 4104

U1 is a quad low-voltage to high-voltage translator with tri-state outputs. It
is supplied from two rails:

| Pin | Name  | Connection                                                   |
|-----|-------|--------------------------------------------------------------|
| 1   | Vddo  | `VCC` (9–12 V) – output side supply                          |
| 16  | Vddi  | `VDDi` – 3.3 V through R1 (1 k), decoupled by C1 (100 nF)    |
| 8   | Vss   | GND                                                          |
| 15  | EN    | `EN` from J1                                                 |

Each of the four sections has one 3.3 V input (`RFn_SEL`) and **two** outputs at
VCC level: a "true" output `RFn_T` and a complementary output `RFn_S`.

| Section | Input (U1 pin) | `_T` output (pin) | `_S` output (pin) |
|---------|----------------|-------------------|-------------------|
| 1       | RF1_SEL (4)    | RF1_T (3)         | RF1_S (2)         |
| 2       | RF2_SEL (12)   | RF2_T (13)        | RF2_S (14)        |
| 3       | RF3_SEL (11)   | RF3_T (10)        | RF3_S (9)         |
| 4       | RF4_SEL (5)    | RF4_T (6)         | RF4_S (7)         |

Each output reaches its FET gate through a 15 k resistor (R10–R17). The `_T`
output drives the **series** FET and the `_S` output drives the **shunt** FET:

| Series FET | Gate net | Shunt FET | Gate net |
|------------|----------|-----------|----------|
| Q1 | RF1_T via R10 | Q5 | RF1_S via R11 |
| Q2 | RF2_T via R12 | Q6 | RF2_S via R13 |
| Q3 | RF3_T via R14 | Q7 | RF3_S via R15 |
| Q4 | RF4_T via R16 | Q8 | RF4_S via R17 |

### 4. Truth table

Because `_T` and `_S` are complementary, a single logic line makes a channel
either "series on / shunt off" or "series off / shunt on":

| RFn_SEL | RFn_T | RFn_S | Series FET | Shunt FET | Channel state |
|---------|-------|-------|------------|-----------|---------------|
| 1 (3.3 V) | VCC | 0 V | on  | off | **Connected** to RFIN |
| 0         | 0 V | VCC | off | on  | **Isolated**, shunted to RF ground |

Normally exactly one `RFn_SEL` should be high at a time. (The hardware does not
enforce this, so setting two high connects two outputs to the input together.)

### 5. Safe state when disabled

Each `_T` line has a 100 k pull-down to ground (R2, R4, R6, R8) and each `_S`
line has a 100 k pull-up to VCC (R3, R5, R7, R9). When the 4104 outputs are
tri-stated (`EN` in the disabling state, or the logic side unpowered) these
resistors define the levels, so **all series FETs turn off and all shunt FETs
turn on**: every output is isolated and shorted to RF ground. The
100 k/15 k values also keep the gates from loading the RF path and slow the
gate edges slightly.

### 6. Power supply decoupling

* C2 (100 nF) – VCC at the connector.
* C3 (100 nF) – VCC at the switch area.
* R1 + C1 – RC filter on the 3.3 V feed into the 4104 `Vddi` pin.

### 7. Other RF components

* **L1 (25–33 nH, 4–5 turns at 3 mm, air-core) in series with C4** at the
  input, and **C13–C16 (2.2 pF) from each output to ground**, are not part of
  the switching mechanism. They look like frequency-response/impedance
  compensation for the J310 capacitances. I have not confirmed this from a
  design note, so treat it as an inference from the schematic.

## Bill of materials (summary)

| Ref        | Value                | Function                                      |
|------------|----------------------|-----------------------------------------------|
| Q1–Q4      | J310                 | Series switches                               |
| Q5–Q8      | J310                 | Shunt switches                                |
| U1         | 4104                 | 3.3 V → VCC quad level translator, tri-state  |
| R1         | 1 k                  | `Vddi` series resistor                        |
| R2–R9      | 100 k                | Pull-downs on `_T`, pull-ups on `_S`          |
| R10–R17    | 15 k                 | Gate resistors                                |
| R18, R19   | 10 k                 | DC bias of the common and shunt-source nodes to VCC |
| C1–C3      | 100 nF               | Supply decoupling                             |
| C4         | 100 nF               | Input DC block                                |
| C5–C8      | 100 nF               | RF decoupling of the shunt-source node        |
| C9–C12     | 100 nF               | Output DC blocks                              |
| C13–C16    | 2.2 pF               | Output shunt capacitors                       |
| L1         | 25–33 nH             | Input series inductor (air-core)              |
| J1         | 1x8 pin header       | Power and control                             |
| 5 × SMA    | edge mount           | RFIN, RFOUT1–4                                |

## Usage

1. Connect GND, a 9–12 V supply to VCC, and 3.3 V to +3V3 (J1 pins 1–3).
2. Drive `EN` to the enabling level.
3. Take one `RFn_SEL` line high to route `RFIN` to `RFOUTn`; take it low to
   isolate that channel.

## Notes and caveats

* The schematic was analysed by reading the KiCad files, not by simulation or
  measurement. No bandwidth, insertion-loss or isolation figures are claimed.
* Check the 4104 datasheet to confirm which logic level of `EN` enables the
  outputs and which output pin is the true/complement for each section; the
  description above is derived from how the board is wired.
* Respect the 4104's maximum supply voltage and keep VCC above the J310's
  pinch-off voltage.
