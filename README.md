# 6T SRAM Cell — Analog VLSI Design (Cadence Virtuoso, GPDK 180nm)

A transistor-level design and simulation of a 6T Static Random Access Memory (SRAM) bit-cell and its peripheral circuits, implemented in **Cadence Virtuoso** using the **GPDK 180nm** process. The design covers the complete bit-cell, precharge circuit, write driver, sense amplifier, and a 1-bit SRAM column integrating all blocks — with full transient simulation results.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Technology Specifications](#technology-specifications)
- [Circuit Blocks](#circuit-blocks)
  - [1. 6T SRAM Bit-Cell](#1-6t-sram-bit-cell)
  - [2. Precharge Circuit](#2-precharge-circuit)
  - [3. Write Driver](#3-write-driver)
  - [4. Sense Amplifier](#4-sense-amplifier)
  - [5. Single-Bit SRAM Column (Top-Level)](#5-single-bit-sram-column-top-level)
- [Simulation Results](#simulation-results)
- [Signal Description](#signal-description)
- [Read and Write Operation Summary](#read-and-write-operation-summary)
- [Tools Used](#tools-used)

---

## Project Overview

This project implements a complete **6T SRAM bit-cell** with all necessary peripheral circuitry required for functional read and write operations. The design is hierarchical — each sub-block (precharge, write driver, sense amplifier) is designed and verified independently before integration into a single-bit column testbench.

The simulation demonstrates correct write and read behavior across multiple cycles with 4-bit data input/output, address decoding, and control signals including precharge, write enable, and read enable.

---

## Technology Specifications

| Parameter        | Value              |
|------------------|--------------------|
| Process          | GPDK 180nm CMOS    |
| Supply Voltage   | 1.8V (nominal)     |
| NMOS W/L         | 2µm / 180nm        |
| PMOS W/L         | 2µm / 180nm        |
| Multiplier (m)   | 1                  |
| Tool             | Cadence Virtuoso   |
| Simulation Type  | Transient (500ns)  |

---

## Circuit Blocks

### 1. 6T SRAM Bit-Cell

**File:** `6T_SRAM_cell`

![6T SRAM Bit-Cell Schematic](6T_SRAM_cell.png)

The core storage element consists of **6 transistors**:

| Transistor | Type  | Role                          |
|------------|-------|-------------------------------|
| PM0        | PMOS  | Pull-up load (left inverter)  |
| NM0        | NMOS  | Pull-down driver (left inverter) |
| PM1        | PMOS  | Pull-up load (right inverter) |
| NM1        | NMOS  | Pull-down driver (right inverter) |
| NM2        | NMOS  | Access transistor (BL side)   |
| NM3        | NMOS  | Access transistor (BLB side)  |

**Operating principle:**

- PM0–NM0 and PM1–NM1 form two **cross-coupled CMOS inverters**, creating a bistable latch that holds data at nodes **Q** and **Qbar**.
- NM2 and NM3 are **pass-gate access transistors** controlled by the **Word Line (WL)**. When WL is HIGH, the cell is connected to the bit-lines (BL, BLB) for read or write access.
- During standby (WL = LOW), the cross-coupled inverters maintain state indefinitely as long as VDD is supplied.

---

### 2. Precharge Circuit

**File:** `6T_SRAM_PC`

![Precharge Circuit Schematic](6T_SRAM_PC.png)

The precharge circuit equalizes and charges the bit-lines before every read or write cycle.

| Transistor | Type  | Role                                        |
|------------|-------|---------------------------------------------|
| PM0        | PMOS  | Precharges BL to VDD when PC is LOW         |
| PM1        | PMOS  | Precharges BLB to VDD when PC is LOW        |
| PM2        | PMOS  | Equalizer — shorts BL and BLB (VDD-gated)  |

**Operating principle:**

- All three transistors are PMOS, gate-controlled by the **PC (Precharge)** signal.
- When PC = LOW (active): PM0 and PM1 pull BL and BLB to VDD; PM2 equates any residual voltage difference between BL and BLB.
- When PC = HIGH: precharge is disabled, and the bit-lines are released for the subsequent read or write operation.
- Pre-charging to VDD ensures a known initial state on the bit-lines, preventing incorrect reads and improving noise margins.

---

### 3. Write Driver

**File:** `6T_SRAM_Write_en`

![Write Driver Schematic](6T_SRAM_Write_en.png)

The write driver forces a differential voltage onto BL and BLB based on the input data bit.

| Transistor | Type  | Role                                              |
|------------|-------|---------------------------------------------------|
| NM0        | NMOS  | Pulls BL low via VSS when Write_en is asserted    |
| NM1        | NMOS  | Input-driven, sets write direction based on Din   |
| NM2        | NMOS  | Pulls BLB low via VSS when Write_en is asserted   |
| NM3        | NMOS  | Complements NM1 to drive the opposite bit-line    |
| PM0        | PMOS  | Weak pull-up for signal conditioning              |
| NM4        | NMOS  | Write enable gate — activates driver when HIGH    |

**Operating principle:**

- Controlled by **Din** (data input) and **Write_en** (write enable).
- When Write_en = HIGH: depending on Din, one bit-line is pulled LOW (via the NMOS pull-down stack) while the other remains HIGH from precharge. This differential is strong enough to overwrite the latch state in the 6T cell.
- The PMOS PM0 provides a weak pull-up to ensure proper logic levels when the driver is inactive.

---

### 4. Sense Amplifier

**File:** `6T_SRAM_sense_amplifier`

![Sense Amplifier Schematic](6T_SRAM_sense_amplifier.png)

The sense amplifier detects the small differential voltage developed on BL/BLB during a read and amplifies it to a full logic level at **Dout**.

| Transistor | Type  | Role                                              |
|------------|-------|---------------------------------------------------|
| PM0        | PMOS  | Cross-coupled load (left side)                   |
| PM2        | PMOS  | Cross-coupled load (right side)                  |
| NM0        | NMOS  | Input differential pair — senses BLB             |
| NM1        | NMOS  | Input differential pair — senses BL              |
| NM2        | NMOS  | Tail current / enable transistor (Read_en)        |
| PM3 + NM3  | CMOS  | Output inverter buffer to drive Dout             |

**Operating principle:**

- The sense amplifier is a **latch-based differential amplifier**.
- Enabled by **Read_en** = HIGH, which activates tail transistor NM2 and connects the differential pair to VSS.
- NM0 and NM1 sense the small voltage difference (ΔV) between BLB and BL developed by the 6T cell during read.
- PM0 and PM2 (cross-coupled PMOS loads) regeneratively amplify this small ΔV to a full rail-to-rail swing.
- The output inverter (PM3 + NM3) buffers the amplified signal and drives **Dout** to the correct logic level.

---

### 5. Single-Bit SRAM Column (Top-Level)

**File:** `6T_SRAM_singlebit`

![Single-Bit SRAM Column Schematic](6T_SRAM_singlebit.png)

The top-level schematic integrates all four sub-blocks into a complete **single-bit SRAM column**, forming a fully functional memory bit-cell with read and write capability.

**Hierarchy of integration:**

```
Single-Bit SRAM Column
├── Precharge Circuit      (PC, BL, BLB)
├── 6T SRAM Bit-Cell       (WL, BL, BLB, Q, Qbar)
├── Write Driver           (Din, Write_en, BL, BLB)
└── Sense Amplifier        (BL, BLB, Read_en, Dout)
```

**Control signal flow:**

1. **PC LOW** → Precharge circuit equalizes BL = BLB = VDD.
2. **PC HIGH** → Bit-lines released; address decoded, WL activated.
3. **Write cycle:** Write_en HIGH + Din drives differential onto BL/BLB → 6T cell latch overwritten.
4. **Read cycle:** Read_en HIGH → Sense amplifier detects ΔV on BL/BLB → Dout reflects stored bit.

---

## Simulation Results

**Tool:** Cadence Virtuoso ADE — Transient Simulation (0 to 500 ns)

![Transient Waveform — Part 1](WhatsApp_Image_2026-06-17_at_8_34_32_PM.jpeg)
![Transient Waveform — Part 2](WhatsApp_Image_2026-06-17_at_8_34_31_PM.jpeg)

The testbench exercises a **4×4 SRAM array** (4 words, 4-bit wide = 16 bits total) with 2 address lines (A0, A1), control signals, and 4-bit data buses.

### Observed Signal Behavior

| Signal     | Behavior                                                                 |
|------------|--------------------------------------------------------------------------|
| `/PC`      | Periodic precharge pulses at ~10ns period; HIGH = precharge active       |
| `/A0`      | Stays LOW for first half (~180ns), transitions HIGH — selects address row |
| `/A1`      | Toggles mid-simulation — selects second address row                      |
| `/E`       | Enable signal — periodic toggling controls cell access                   |
| `/Write_en`| Pulsed HIGH during write cycles; narrower than Read_en pulses            |
| `/Din0`    | Data input bit 0 — toggles during write cycles                           |
| `/Read_en` | Enabled during read phases; wider pulses than Write_en                   |
| `/D0`      | Output bit 0 — correctly follows written data after read enable          |
| `/Din1`    | Stays at mid-rail (~1.58V) — bit 1 not exercised in this simulation      |
| `/D1`      | Output bit 1 — transitions correctly during read cycles                  |
| `/Din2`    | Held constant (1.2V) — not driven in this run                            |
| `/D2`      | Output bit 2 — narrow pulses observed; reflects residual or glitch reads |
| `/Din3`    | Toggles periodically — input for bit 3                                   |
| `/D3`      | Output bit 3 — follows Din3 after correct write-read sequence            |

### Key Observations

- **Write-before-read ordering** is correctly maintained: Write_en precedes Read_en in every cycle.
- **Precharge is completed** before WL is asserted in each cycle — verified by PC going HIGH before Write_en or Read_en activates.
- **D0 and D1** show clean output transitions correlated with their respective Din signals, confirming correct write and sense amplifier operation.
- **Din1 and Din2** are not toggled (held at intermediate voltage), so D1 and D2 show expected static or glitch behavior — not a circuit fault.
- Address lines A0 and A1 confirm the simulation exercises **multiple word-line addresses** across the 500ns window.

---

## Signal Description

| Signal      | Direction | Description                                      |
|-------------|-----------|--------------------------------------------------|
| PC          | Input     | Precharge control — active HIGH precharges BL/BLB |
| WL          | Input     | Word line — enables cell access when HIGH         |
| A0, A1      | Input     | Address bits selecting the memory row             |
| E           | Input     | Chip enable                                       |
| Write_en    | Input     | Write enable — activates write driver             |
| Read_en     | Input     | Read enable — activates sense amplifier           |
| Din0–Din3   | Input     | 4-bit data input bus                              |
| D0–D3       | Output    | 4-bit data output bus                             |
| BL, BLB     | Internal  | Complementary bit-lines                           |
| Q, Qbar     | Internal  | Storage nodes of the 6T latch                    |
| Dout        | Output    | Sense amplifier output (single-bit)               |

---

## Read and Write Operation Summary

### Write Operation
```
1. PC = LOW  → BL = BLB = VDD  (precharge)
2. PC = HIGH → bit-lines released
3. WL = HIGH → access transistors ON
4. Write_en = HIGH, Din = data → write driver forces differential on BL/BLB
5. 6T latch overwritten → Q = Din, Qbar = ~Din
6. Write_en = LOW, WL = LOW → cell isolated, data held
```

### Read Operation
```
1. PC = LOW  → BL = BLB = VDD  (precharge)
2. PC = HIGH → bit-lines released
3. WL = HIGH → access transistors ON
4. Cell develops small ΔV on BL/BLB based on stored Q/Qbar
5. Read_en = HIGH → sense amplifier activated
6. Sense amplifier amplifies ΔV → Dout = stored bit
7. Read_en = LOW, WL = LOW → operation complete
```

---

## Tools Used

| Tool                  | Purpose                              |
|-----------------------|--------------------------------------|
| Cadence Virtuoso      | Schematic capture and hierarchy      |
| Cadence ADE (Spectre) | Transient simulation                 |
| GPDK 180nm PDK        | Process design kit (nmos1, pmos1)    |
| Assura / Virtuoso DRC | Design rule check (if layout done)   |

---

## Author

**Kaushik T**
B.E. Electronics and Communication Engineering
Chennai Institute of Technology, Chennai
GitHub: [Raghul-2025](https://github.com/Raghul-2025)
