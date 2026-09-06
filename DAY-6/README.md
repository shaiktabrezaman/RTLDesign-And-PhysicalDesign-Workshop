# Day 6 — Physical Design

## Overview

Day 6 starts the Physical Design portion of the course. It covers how a chip is physically organized (pads, die, core), what lives inside the core (foundry IPs and macros), how the RISC-V ISA eventually becomes a physical layout, the software-to-hardware abstraction stack, and the two flows that tie everything together — the Digital ASIC design flow and the RTL-to-GDSII flow. The lab puts the first stage of that flow into practice by taking picorv32a through an OpenLane run up to synthesis.

---

## Contents

- [1. Chip Anatomy — Pads, Die, and Core](#1-chip-anatomy--pads-die-and-core)
- [2. Foundry IPs and Macros](#2-foundry-ips-and-macros)
- [3. RISC-V: From ISA to Layout](#3-risc-v-from-isa-to-layout)
- [4. Software-to-Hardware Abstraction](#4-software-to-hardware-abstraction)
- [5. Digital ASIC Design — RTL, EDA Tools, PDK Data](#5-digital-asic-design--rtl-eda-tools-pdk-data)
- [6. RTL to GDSII Flow](#6-rtl-to-gdsii-flow)
- [7. Additional References](#7-additional-references)
- [8. Labs](#8-labs)

---

## 1. Chip Anatomy — Pads, Die, and Core

A packaged chip is built up in layers: the outermost **pads** are where external signals and power enter and leave the chip, the **die** is the full piece of silicon, and the **core** is the region inside the die that actually holds the logic.

The pad-level view for a real chip shows how different peripheral interfaces (I2C, QSPI, UART, JTAG, GPIO, ADC, power/ground) are grouped and routed around the edge of the die before reaching the core.

The same chip, redrawn to explicitly label the **die**, **PADS**, and **core** regions.

<img width="1033" height="543" alt="image" src="https://github.com/user-attachments/assets/209a6b32-fa1d-42b2-b962-d1a79a258d5d" />

---

## 2. Foundry IPs and Macros

Inside the core, the design is made up of two kinds of pre-built blocks:

- **Foundry IPs** — analog/mixed-signal blocks provided by the foundry, such as the **PLL** and **DAC/ADC** used here.
- **Macros** — larger reusable digital blocks, such as the **RISC-V core** and an **SRAM** block, that get instantiated as-is rather than synthesized from scratch each time.

Together with a GPIO bank and SPI interface, these blocks make up the internal structure of the example RISC-V SoC used in this course.

<img width="1541" height="802" alt="image" src="https://github.com/user-attachments/assets/578cdd69-dc65-4612-93f8-fbc3cf373389" />

---

## 3. RISC-V: From ISA to Layout

RISC-V is an **Instruction Set Architecture (ISA)** — a specification of instructions, registers, and behavior, independent of any particular hardware implementation. The same ISA can be implemented many different ways in RTL.

The path shown goes:

```
RISC-V ISA
     ↓
RTL Implementation (e.g. picorv32 core)
     ↓
Physical Layout (via Qflow)
```

A concrete example: a small C program is compiled and disassembled into RISC-V instructions, the `picorv32` RTL implements those instructions in Verilog, and that RTL is eventually carried through to a physical layout.

<img width="1600" height="720" alt="pd3" src="https://github.com/user-attachments/assets/7db0aa65-1513-4013-9e4c-a9782e46573e" />

---

## 4. Software-to-Hardware Abstraction

Before getting to the physical flow, it helps to see where hardware sits relative to the software stack above it. Application software doesn't talk to hardware directly — it goes through several layers first:

```
Application Software
        ↓
System Software (O/S, Compiler, Assembler)
        ↓
Hardware
```

Zooming into just the compilation path for a piece of code:

```
Source Code
     ↓
Compiler            (translates code to ISA-level instructions)
     ↓
Instruction Set Architecture (RISC-V)
     ↓
Assembler           (translates instructions to machine code)
     ↓
Hardware (binary / machine language)
```

And on the hardware design side, this mirrors a similar chain:

```
RTL (Verilog description of hardware)
     ↓
Synthesized Netlist
     ↓
Physical Design Implementation (layout)
```

---

## 5. Digital ASIC Design — RTL, EDA Tools, PDK Data

A Digital ASIC design flow takes three essential inputs and combines them, using automated software, to produce a manufacturable chip.

The three key inputs:

- **RTL (Register Transfer Level):** The Verilog design describing the chip's intended functional behavior — this is what the designer writes.
- **EDA Tools (Electronic Design Automation):** The software toolchain that automates the entire flow — synthesis (e.g. Yosys), placement & routing (e.g. OpenROAD), timing analysis (e.g. OpenSTA), and physical verification. These tools convert abstract RTL into a manufacturable layout without requiring engineers to manually place billions of transistors.
- **PDK (Process Design Kit):** Foundry-provided data describing exactly how a specific manufacturing process works — includes the standard cell libraries (`.lib`), layout design rules, layer stack information, and electrical models specific to that fabrication node (e.g. the open-source **SkyWater 130nm (SKY130)** PDK used throughout this program).

**How they combine:** EDA tools take the RTL as input, consult the PDK to understand what's physically realizable and how to characterize timing/power, and produce a manufacturable design as output — a GDSII file ready for fabrication.

<img width="768" height="670" alt="image" src="https://github.com/user-attachments/assets/6d2aa669-b3ee-4030-8903-5668a1915cc6" />

---

## 6. RTL to GDSII Flow

The complete, automated pipeline that transforms a Verilog RTL design into **GDSII** — the industry-standard file format a semiconductor foundry uses to physically fabricate a chip.

The major stages:

1. **RTL Design** — write and functionally verify the Verilog design.
2. **Synthesis** — convert RTL into a gate-level netlist mapped to the PDK's standard cell library (using Yosys).
3. **Floorplanning** — define the chip's physical boundary, core area, and placement of I/O pads and macros.
4. **Placement** — position all standard cells within the floorplan, optimizing for area and wire length.
5. **Clock Tree Synthesis (CTS)** — build a balanced clock distribution network so the clock signal reaches every flip-flop with minimal skew.
6. **Routing** — physically connect all placed cells using available metal layers, respecting the PDK's design rules.
7. **Static Timing Analysis (STA)** — verify the routed design meets all timing constraints (setup/hold) across process, voltage, and temperature corners.
8. **Physical Verification (DRC/LVS)** — confirm the layout obeys the foundry's manufacturing design rules (DRC) and matches the original netlist (LVS).
9. **GDSII Generation** — export the final, verified physical layout as a GDSII file, ready to be sent to the foundry for fabrication.

<img width="1518" height="670" alt="image" src="https://github.com/user-attachments/assets/f4497716-e074-497e-9e3a-8f4719599546" />

---

## 7. Additional References

### Real Fabricated Open-Source SoCs (striVe Series)

A set of real, taped-out chips (striVe, striVe2, striVe3, striVe5) built using the open-source SKY130 flow, each varying in standard-cell library and SRAM configuration.

<img width="957" height="667" alt="image" src="https://github.com/user-attachments/assets/22fd458f-3fd3-4264-ae6c-44326b10e4c9" />

Source: SkyWater, Google, OpenROAD, efabless

### OpenROAD/OpenLANE Detailed Flow

A more detailed breakdown of the RTL-to-GDSII flow, showing the specific tools used at each stage (Yosys + abc for synthesis, OpenROAD app for floorplanning/placement/CTS, TritonRoute for detailed routing, magic/netgen for physical verification).

<img width="1078" height="667" alt="image" src="https://github.com/user-attachments/assets/feca1485-c066-4f6e-a7be-4d2f08fe1d29" />

Source: OpenROAD / OpenLANE project documentation

---

## 8. Labs

The lab for Day 6 sets up a fresh OpenLane run for the `picorv32a` design (already introduced in section 3 above) and carries it through synthesis, using the SKY130 PDK.

### 8.1 PDK Directory Structure

Under `sky130A`, `libs.ref` holds the reference libraries for each standard-cell flavor (`sky130_fd_sc_hd`, `sky130_fd_sc_hs`, SRAM macros, and so on), while `libs.tech` holds tool-specific tech files (magic, klayout, openlane, qflow, etc.). One level further into `sky130_fd_sc_hd` are the actual `.lib` (timing), `.lef` (abstract layout), and `.tlef` (technology LEF) files used later during synthesis and place-and-route.

<img width="1920" height="983" alt="cmds_pdks" src="https://github.com/user-attachments/assets/0739c565-d35f-4009-a465-0f31dbdb7f17" />

<img width="1920" height="983" alt="cmds_pdks_2" src="https://github.com/user-attachments/assets/4494f0d2-74cb-4506-9065-6af3483a09d3" />

### 8.2 OpenLane Design Directory

`designs/` in the OpenLane install holds one folder per example design. `picorv32a` contains its own `config.tcl`, several PDK/library-specific config overrides, and a `src/` folder with the actual RTL (`picorv32a.v`) and timing constraints (`picorv32a.sdc`).

<img width="1920" height="983" alt="cmds_openlane" src="https://github.com/user-attachments/assets/087fa1dc-1c7f-4374-9825-216c86010512" />

### 8.3 Design-Level Config

`config.tcl` sets the design-specific basics: design name, source files, clock port/period, and then sources a PDK/library-specific config file if one exists for the chosen standard-cell library.

<img width="1920" height="983" alt="picorv_description" src="https://github.com/user-attachments/assets/c18a094c-9c66-4b87-a97a-6c4518a17ae1" />

### 8.4 Library-Specific Config

The sourced file for this run, `sky130A_sky130_fd_sc_hd_config.tcl`, overrides a few defaults for the `sky130_fd_sc_hd` library specifically — synthesis fanout limits, a recalculated clock period, and target core utilization/placement density.

<img width="1920" height="983" alt="openlane_picorv_sky130A_config_tcl" src="https://github.com/user-attachments/assets/bf66b20d-8dc0-4cc1-944c-b3af0dd0f3fd" />

### 8.5 Checking the OpenLane Docker Image

```bash
docker images
```

Lists locally available Docker images, used here to confirm the `efabless/openlane:v0.21` image is already pulled before starting a container from it.

<img width="1920" height="983" alt="invoking_bash_before-cmds" src="https://github.com/user-attachments/assets/88136f92-d04c-46c3-aa87-3107d38711dc" />

### 8.6 Starting the OpenLane Container

```bash
docker run -it \
  -v $PWD:/openLANE_flow \
  -v $PDK_ROOT:$PDK_ROOT \
  -e PDK_ROOT=$PDK_ROOT \
  -u $(id -u $USER):$(id -g $USER) \
  efabless/openlane:v0.21
```

Launches the OpenLane container interactively. `-v $PWD:/openLANE_flow` mounts the current OpenLane folder into the container; `-v $PDK_ROOT:$PDK_ROOT` mounts the PDK at the *same absolute path* it has on the host, which matters because OpenLane's saved run configs can store absolute PDK paths; `-e PDK_ROOT=$PDK_ROOT` passes that path in as an environment variable; `-u ...` runs the container as the current host user instead of root.

```bash
./flow.tcl -interactive
```

Starts OpenLane's interactive Tcl shell inside the container.

```tcl
package require openlane 0.9
```

Loads the OpenLane Tcl package (version 0.9) into the interactive shell, making all the flow commands (`prep`, `run_synthesis`, etc.) available.

<img width="1920" height="983" alt="invoke_bash_1" src="https://github.com/user-attachments/assets/567df147-4dd9-4a38-801b-6ee05709815d" />

### 8.7 Preparing the Design

```tcl
prep -design picorv32a
```

Reads the design's `config.tcl`, resolves the PDK path, selects the standard-cell library, and creates a brand-new timestamped run directory under `designs/picorv32a/runs/` for this run.

<img width="1920" height="983" alt="invoke_bash2" src="https://github.com/user-attachments/assets/ca5b156f-5fa6-48f6-aed7-0d852c6fa836" />

### 8.8 The New Run Directory

Once `prep` completes, the new run folder (e.g. `04-09_12-25`) contains `tmp/`, `results/`, `reports/`, `logs/`, a copy of `config.tcl`, and a running command log.

<img width="1920" height="983" alt="runs_appear_after_invoke" src="https://github.com/user-attachments/assets/5a5d8222-71cd-4dc9-8e3f-f19b3cc1a0bc" />

Inside `tmp/`, OpenLane has already merged the standard-cell LEF files into a single `merged.lef`, alongside a trimmed liberty file and empty stage subfolders (`synthesis`, `placement`, `cts`, `routing`, etc.) waiting to be filled in as the flow progresses.

<img width="1920" height="983" alt="runs_appear_after_invoke2" src="https://github.com/user-attachments/assets/1c4b7bde-27e6-4bb3-bb07-99f3f60ebc13" />

### 8.9 Inspecting the Merged LEF

`merged.lef` defines the physical abstract views (pin shapes, layers, macro boundaries) for every standard cell, such as the `sky130_fd_sc_hd__dfstp_4` flip-flop shown here with its `D`, `Q`, and `SET_B` pins.

<img width="1920" height="983" alt="Merged_lef_dscrp" src="https://github.com/user-attachments/assets/7fabb474-1edb-408d-bc9f-5a9db0d9765d" />

### 8.10 Full Run Configuration

The run's own `config.tcl` (auto-generated by `prep`) expands every setting OpenLane will use for this run — PDK paths, cell padding, clock buffer choices, diode insertion strategy, routing tool selection, and dozens of other environment variables inherited from the design and library configs.

<img width="1920" height="983" alt="second_config_tcl" src="https://github.com/user-attachments/assets/8b29590b-f478-46f1-8832-3169d4e51a78" />

### 8.11 Running Synthesis

```tcl
run_synthesis
```

Runs Yosys + `abc` synthesis on the RTL, followed by an OpenSTA timing check on the resulting netlist. The log reports the final chip area, confirms the netlist was written out, and prints the timing summary (`tns`/`wns`) before declaring synthesis successful.

<img width="1920" height="983" alt="run_synthesis_cmd_completion" src="https://github.com/user-attachments/assets/d7ab7f22-a3e8-43e1-8ab4-9466a60bc0c2" />

### 8.12 Synthesis Statistics and Flip-Flop Ratio

The Yosys `stat` output lists the total wire/cell counts and a full breakdown by standard-cell type. Of particular interest is `sky130_fd_sc_hd__dfxtp_2`, the flip-flop cell — its count against the total cell count gives the flip-flop ratio for this synthesis.

<img width="1920" height="1080" alt="flop_ratio_percentage" src="https://github.com/user-attachments/assets/fb3477f4-1285-41a9-ae07-08facaa7e063" />

```text
Flip-Flop Ratio = (Flip-Flops / Total Cells) × 100
                = (1613 / 14876) × 100
                ≈ 10.84%
```


### 8.13 Viewing the Synthesized Netlist

The actual gate-level netlist Yosys wrote out, `picorv32a.synthesis.v`, can be opened directly — it's a much larger file than the original RTL, with auto-generated wire names for every internal net.

<img width="1920" height="983" alt="synthesis_results_file" src="https://github.com/user-attachments/assets/7c743b4f-cdea-4854-9fc2-ab32a4b5daaa" />

### 8.14 Full Synthesis Statistics Report

The complete `stat` report (saved as a synthesis report file) lists every standard-cell type instantiated and how many of each were used, ending with the total chip area for the module.

<img width="1920" height="983" alt="synth_STAT_rpt1" src="https://github.com/user-attachments/assets/92a88240-f8a1-4dd4-b6f6-db03f6fb811d" />

<img width="1920" height="983" alt="synth_STAT_rpt2" src="https://github.com/user-attachments/assets/bedc4a00-f66a-46bd-ae5b-2b5578581069" />

### 8.15 Synthesis and STA Report Files

The `reports/synthesis/` folder collects everything generated during this stage: Yosys check/stat reports (`1-yosys_*.rpt`) and OpenSTA reports (`2-opensta_*.rpt`) covering timing, slew, and min/max delay.

<img width="1920" height="983" alt="rpts" src="https://github.com/user-attachments/assets/0d0a123f-15c4-47d8-91d1-98fc43f0c123" />

One of those OpenSTA reports, the detailed timing path report, breaks down a specific worst-case path cell by cell — showing fanout, capacitance, slew, and incremental delay building up from one flip-flop to the next through several standard cells.

<img width="1920" height="983" alt="opensta_timing_rpt" src="https://github.com/user-attachments/assets/e9ed703a-bf19-4937-be8e-6f5a203b7c2c" />

---

## 9. Conclusion

Day 6 covered how a chip is physically organized (pads, die, core), what foundry IPs and macros sit inside that core, how the RISC-V ISA eventually becomes a physical layout, and where the RTL-to-GDSII flow fits between raw RTL and a fabrication-ready GDSII file. The lab put the first stage of that flow into practice — setting up a fresh OpenLane run for picorv32a inside Docker, walking through its design and PDK configuration files, and running synthesis through to a verified gate-level netlist with a flip-flop ratio of roughly 10.84%. Floorplanning and the remaining physical stages continue from this same run next.
