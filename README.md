# 6T SRAM Cell — Analog VLSI Design
### Cadence Virtuoso · GPDK 180nm CMOS · Spectre Transient Simulation

![Tool](https://img.shields.io/badge/Tool-Cadence%20Virtuoso-blue)
![Process](https://img.shields.io/badge/Process-GPDK%20180nm-green)
![Supply](https://img.shields.io/badge/VDD-1.8V-orange)
![Simulation](https://img.shields.io/badge/Simulation-Spectre%20ADE-purple)

---

## Overview

A full transistor-level design and simulation of a **6T SRAM bit-cell** with all peripheral circuits, implemented in **Cadence Virtuoso** on the **GPDK 180nm** process node.

The project is structured hierarchically — each peripheral block is designed and verified independently, then integrated into a complete **single-bit SRAM column** and validated through a 500ns transient simulation covering multiple write and read cycles.

---

## Table of Contents

- [Technology Specifications](#technology-specifications)
- [Project Hierarchy](#project-hierarchy)
- [Circuit Blocks](#circuit-blocks)
  - [1. 6T SRAM Bit-Cell](#1-6t-sram-bit-cell)
  - [2. Precharge Circuit](#2-precharge-circuit)
  - [3. Write Driver](#3-write-driver)
  - [4. Sense Amplifier](#4-sense-amplifier)
  - [5. Single-Bit SRAM Column](#5-single-bit-sram-column-top-level)
- [Simulation Results](#simulation-results)
- [Read & Write Operation](#read--write-operation)
- [Signal Reference](#signal-reference)
- [Tools Used](#tools-used)

---

## Technology Specifications

| Parameter       | Value           |
|-----------------|-----------------|
| Process         | GPDK 180nm CMOS |
| Supply Voltage  | 1.8V            |
| NMOS W/L        | 2µm / 180nm     |
| PMOS W/L        | 2µm / 180nm     |
| Multiplier (m)  | 1               |
| Simulator       | Spectre (ADE)   |
| Sim Duration    | 0 – 500 ns      |

---

## Project Hierarchy

```
6T_SRAM_singlebit  ← Top-level single-bit column
├── 6T_SRAM_PC             (Precharge Circuit)
├── 6T_SRAM_cell           (6T Bit-Cell)
├── 6T_SRAM_Write_en       (Write Driver)
└── 6T_SRAM_sense_amplifier (Sense Amplifier)
```

---

## Circuit Blocks

### 1. 6T SRAM Bit-Cell

![6T SRAM Bit-Cell Schematic](6T_SRAM_cell.png)

The core storage element is built from **6 transistors** — two cross-coupled CMOS inverters forming a bistable latch, and two NMOS access transistors for bit-line connectivity.

| Transistor | Type | Function |
|------------|------|----------|
| PM0 | PMOS | Pull-up load — left inverter |
| NM0 | NMOS | Pull-down driver — left inverter |
| PM1 | PMOS | Pull-up load — right inverter |
| NM1 | NMOS | Pull-down driver — right inverter |
| NM2 | NMOS | Access transistor — BL side |
| NM3 | NMOS | Access transistor — BLB side |

**How it works:**
- PM0–NM0 and PM1–NM1 form two **cross-coupled CMOS inverters**. Their outputs (Q and Qbar) feed back into each other's inputs, creating a stable bistable latch that retains data as long as VDD is supplied.
- NM2 and NM3 are **Word Line (WL) controlled pass gates**. When WL = HIGH, the cell connects to the bit-lines for read or write access. When WL = LOW, the cell is isolated and retains its stored value indefinitely.

---

### 2. Precharge Circuit

![Precharge Circuit Schematic](6T_SRAM_PC.png)

The precharge circuit initializes both bit-lines to VDD before every access cycle, ensuring a known starting condition for reliable read and write operations.

| Transistor | Type | Function |
|------------|------|----------|
| PM0 | PMOS | Precharges BL to VDD |
| PM1 | PMOS | Precharges BLB to VDD |
| PM2 | PMOS | Equalizer — shorts BL and BLB |

**How it works:**
- All three transistors are PMOS, activated when **PC = LOW** (active-low control).
- PM0 and PM1 pull BL and BLB up to VDD simultaneously.
- PM2 (gate tied to VDD) equalizes any voltage difference between BL and BLB, eliminating residual differential before the next cycle.
- When PC = HIGH, all three transistors are OFF and the bit-lines are released for the subsequent read or write operation.

---

### 3. Write Driver

![Write Driver Schematic](6T_SRAM_Write_en.png)

The write driver forces a strong differential voltage onto BL and BLB based on the input data bit, overwriting the existing latch state in the 6T cell.

| Transistor | Type | Function |
|------------|------|----------|
| NM0 | NMOS | Pulls BL toward VSS during write |
| NM1 | NMOS | Din-driven — sets write direction |
| NM2 | NMOS | Pulls BLB toward VSS during write |
| NM3 | NMOS | Complements NM1 — drives opposite bit-line |
| PM0 | PMOS | Weak pull-up for signal conditioning |
| NM4 | NMOS | Write enable gate — controlled by Write_en |

**How it works:**
- Activated when **Write_en = HIGH**.
- Based on **Din**: one bit-line is pulled LOW through the NMOS pull-down stack while the other stays HIGH from precharge, creating the differential needed to flip the 6T latch.
- PM0 provides a weak pull-up to maintain proper logic levels when the driver is inactive.
- The NMOS sizing ensures the write driver can overcome the regenerative feedback of the cross-coupled latch.

---

### 4. Sense Amplifier

![Sense Amplifier Schematic](6T_SRAM_sense_amplifier.png)

The sense amplifier detects the small differential voltage that develops on BL/BLB during a read operation and amplifies it to a full rail-to-rail logic level at **Dout**.

| Transistor | Type | Function |
|------------|------|----------|
| PM0 | PMOS | Cross-coupled load — left side |
| PM2 | PMOS | Cross-coupled load — right side |
| NM0 | NMOS | Differential input — senses BLB |
| NM1 | NMOS | Differential input — senses BL |
| NM2 | NMOS | Tail transistor — enabled by Read_en |
| PM3 + NM3 | CMOS | Output inverter buffer — drives Dout |

**How it works:**
- Enabled by **Read_en = HIGH**, which turns ON tail transistor NM2 and connects the differential pair to VSS.
- NM0 and NM1 sense the small ΔV between BLB and BL caused by the 6T cell during read.
- PM0 and PM2 (cross-coupled PMOS loads) regeneratively amplify this ΔV to a full VDD/VSS swing.
- The CMOS output inverter (PM3 + NM3) buffers the amplified output and drives **Dout** cleanly.

---

### 5. Single-Bit SRAM Column (Top-Level)

![Single-Bit SRAM Column Schematic](6T_SRAM_singlebit.png)

The top-level schematic integrates all four sub-blocks into a complete single-bit SRAM column. All peripheral blocks share the BL and BLB internal nets and are controlled by a common set of external signals.

**Control signal flow per cycle:**

```
Step 1 → PC = LOW       : Precharge — BL = BLB = VDD
Step 2 → PC = HIGH      : Release bit-lines
Step 3 → WL = HIGH      : Enable cell access

  [WRITE]
Step 4 → Write_en = HIGH, Din = data
         Write driver forces differential on BL/BLB
         6T latch overwritten → Q = Din, Qbar = ~Din

  [READ]
Step 4 → Read_en = HIGH
         Cell develops ΔV on BL/BLB
         Sense amplifier amplifies ΔV → Dout = stored bit

Step 5 → WL = LOW       : Isolate cell — hold state
```

---

## Simulation Results

Transient simulation run from **0 to 500 ns** in Cadence Spectre ADE, exercising a **4-word × 4-bit SRAM array** with 2 address lines and 4-bit data buses.

![Transient Waveform — Upper Signals](WhatsApp_Image_2026-06-17_at_8_34_32_PM.jpeg)

![Transient Waveform — Lower Signals](WhatsApp_Image_2026-06-17_at_8_34_31_PM.jpeg)

### Signal Behavior Summary

| Signal | Observed Behavior |
|--------|-------------------|
| `/PC` | Periodic active-HIGH pulses; precharges BL/BLB to VDD each cycle |
| `/A0` | Stays LOW for first ~180ns, then transitions HIGH — selects address row 0 then row 1 |
| `/A1` | Toggles mid-simulation — selects second address row |
| `/E` | Periodic chip enable toggling |
| `/Write_en` | Narrow HIGH pulses aligned with write cycles |
| `/Din0` | Toggles HIGH during write cycles — data input for bit 0 |
| `/Read_en` | Wider HIGH pulses following Write_en — activates sense amplifier |
| `/D0` | Correctly follows Din0 after each write-then-read sequence |
| `/Din1` | Held at ~1.58V (not driven) — bit 1 not exercised |
| `/D1` | Transitions correctly during read cycles for bit 1 |
| `/Din2` | Held at 1.2V (not driven) — bit 2 not exercised |
| `/D2` | Narrow pulses — residual behavior; not a circuit fault |
| `/Din3` | Toggles periodically — data input for bit 3 |
| `/D3` | Follows Din3 after correct write-read sequence |

### Key Verification Points

- ✅ **Precharge completes** before WL assertion in every cycle
- ✅ **Write precedes Read** in correct sequence every cycle
- ✅ **D0 and D3** show clean output transitions matching their Din inputs
- ✅ **Address decoding** confirmed — A0 and A1 transitions select different word-lines across the 500ns window
- ℹ️ **Din1 and Din2** are not toggled in this testbench run — flat output on D1/D2 is expected, not a fault

---

## Read & Write Operation

### Write Operation
```
1. PC = LOW        →  BL = BLB = VDD  (precharge + equalize)
2. PC = HIGH       →  Bit-lines released
3. WL = HIGH       →  Access transistors NM2, NM3 turn ON
4. Write_en = HIGH →  Write driver forces BL = Din, BLB = ~Din
5. 6T latch flips  →  Q = Din, Qbar = ~Din stored
6. Write_en = LOW  →  Driver disabled
7. WL = LOW        →  Cell isolated, data held
```

### Read Operation
```
1. PC = LOW        →  BL = BLB = VDD  (precharge + equalize)
2. PC = HIGH       →  Bit-lines released
3. WL = HIGH       →  Access transistors NM2, NM3 turn ON
4. Cell discharges →  Small ΔV develops on BL or BLB
5. Read_en = HIGH  →  Sense amplifier tail transistor ON
6. SA amplifies ΔV →  Dout = Q (stored bit)
7. Read_en = LOW   →  SA disabled
8. WL = LOW        →  Cell isolated
```

---

## Signal Reference

| Signal | Direction | Description |
|--------|-----------|-------------|
| PC | Input | Precharge control — active LOW |
| WL | Input | Word line — enables cell access |
| A0, A1 | Input | 2-bit address — selects memory row |
| E | Input | Chip enable |
| Write_en | Input | Write enable — activates write driver |
| Read_en | Input | Read enable — activates sense amplifier |
| Din0–Din3 | Input | 4-bit data input bus |
| D0–D3 | Output | 4-bit data output bus |
| BL | Internal | Bit-line |
| BLB | Internal | Complementary bit-line |
| Q | Internal | Storage node (true) |
| Qbar | Internal | Storage node (complement) |
| Dout | Output | Sense amplifier output |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Cadence Virtuoso | Schematic capture and hierarchy |
| Cadence ADE (Spectre) | Transient simulation |
| GPDK 180nm PDK | Process design kit (nmos1, pmos1) |
| Assura | DRC / LVS verification |

---

## Author

**Kaushik T**
B.E. Electronics and Communication Engineering — 2029
Chennai Institute of Technology, Chennai
GitHub: [@Raghul-2025](https://github.com/Raghul-2025)
