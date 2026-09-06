# Day 7 — Floorplanning and Placement

## Overview

Day 7 continues from the synthesized netlist produced on Day 6 and covers the next two stages of physical design: floorplanning and placement. It covers how the core/die area is sized, how pre-placed IP blocks, decoupling capacitors, power planning, and pin placement fit into a floorplan, how placement binds the netlist to physical cells and optimizes it, and a look at how standard cells themselves are characterized in the first place. The lab runs both stages in OpenLane on `picorv32a` and inspects the results in Magic.

---

## Contents

- [1. Core and Die Dimensions, Utilization, and Aspect Ratio](#1-core-and-die-dimensions-utilization-and-aspect-ratio)
- [2. Pre-Placed Cells](#2-pre-placed-cells)
- [3. Decoupling Capacitors and Noise Margins](#3-decoupling-capacitors-and-noise-margins)
- [4. Power Planning](#4-power-planning)
- [5. Pin Placement](#5-pin-placement)
- [6. Logical Cell Placement Blockage](#6-logical-cell-placement-blockage)
- [7. Placement and Routing](#7-placement-and-routing)
- [8. Library Characterization and Modeling](#8-library-characterization-and-modeling)
- [9. Labs](#9-labs)
- [10. Conclusion](#10-conclusion)

---

## 1. Core and Die Dimensions, Utilization, and Aspect Ratio

Floorplanning starts by fixing two rectangles: the **die** (the full silicon area) and the **core** inside it (where the actual logic goes, leaving margin for I/O and power structures).

**Aspect ratio** is height divided by width. An aspect ratio of 1 gives a square chip; any other value gives a rectangular one.

**Utilization** is how much of the core area is actually occupied by the netlist's cells:

```text
Utilization = (Area occupied by cells / Total core area) × 100
```

In practice, utilization is usually kept around 50–60% or lower — packing the core much tighter than that leaves too little room for routing and buffering later on.

---

## 2. Pre-Placed Cells

Not every block in a design goes through automated placement. Larger reusable IPs — memory, a clock-gating cell, a comparator, a mux, or a full sub-block of logic — are often carved out, given a fixed boundary with clearly defined external I/O pins, and treated as a single reusable block ("black box") that can be instantiated wherever it's needed.

These blocks are given **fixed locations** before the automated flow runs — this arrangement of IPs on the chip is exactly what floorplanning decides. Because their positions are fixed ahead of time, they're called **pre-placed cells**. The automated placement and routing tools then work around them, placing the remaining standard cells in whatever space is left.

---

## 3. Decoupling Capacitors and Noise Margins

Any digital signal has three interpretation zones: a valid logic-0 region, a valid logic-1 region, and an **undefined region** in between. If switching noise pushes a signal's voltage into that undefined band, the receiving gate can no longer reliably tell whether it was meant to be a 0 or a 1.

A common source of that noise is **IR drop** — when many cells switch at once, they briefly demand a burst of current from the power network, and the local supply voltage can sag. **Decoupling capacitors** placed near the cells act as a local charge reservoir: they supply that burst of current locally, then recharge from the main power supply once the circuit's demand drops. This keeps the local supply voltage stable and keeps signals out of the undefined region.

Decaps are placed during floorplanning, typically above, below, or between other blocks on the core.

---

## 4. Power Planning

Relying on a single power and ground connection for the whole chip is risky — it concentrates all the current through one path, increasing both **voltage drop** (on VDD) and **ground bounce** (on VSS), either of which can push signals toward that undefined region again.

Power planning solves this by distributing **multiple** VDD and VSS connections across the chip, in the form of power rings and power straps that feed the core from several points rather than one:

```text
Power Network
     ↓
VDD / VSS
     ↓
Power Rings
     ↓
Power Straps
     ↓
Standard Cells
```

---

## 5. Pin Placement

The netlist (written in Verilog/VHDL) captures the logical connectivity between gates, but says nothing about physical location. Pin placement decides where each I/O pin actually sits on the die — commonly along the left, right, top, or bottom edge — based on the design's connectivity and the location of whatever pre-placed blocks it connects to.

---

## 6. Logical Cell Placement Blockage

Once pre-placed blocks and pins are set, certain regions of the core can be explicitly marked off-limits to automated standard-cell placement — a **placement blockage**. This keeps standard cells from being dropped into space that's reserved for something else (routing channels around a macro, for instance), giving the floorplan tighter control over the final layout.

---

## 7. Placement and Routing

With the floorplan fixed, placement takes over — deciding where each standard cell in the netlist physically sits inside the core.

### 7.1 Binding the Netlist to Physical Cells

Every gate in the netlist has to be matched to an actual physical cell from the standard-cell library. That library defines each cell's physical footprint (essentially a rectangle with a fixed width and height) along with its electrical characteristics. Cells come in multiple "flavors" of the same logic function at different drive strengths — a physically larger cell of the same function generally switches faster (lower delay) at the cost of more area and power.

### 7.2 Placement

Once bound to physical cells, the netlist is laid out inside the core area defined by the floorplan — this is what actually turns the abstract netlist into a physical arrangement of cells.

### 7.3 Placement Optimization

After an initial pass, the tool estimates wire length, capacitance, and resulting delay for each net, and inserts buffers ("repeaters") on any signal path that's grown too long for a healthy transition time:

```text
Cell ─────── Buffer ─────── Cell
```

---

## 8. Library Characterization and Modeling

### 8.1 Concepts

The timing/power numbers used throughout synthesis, placement, and STA come from **characterizing** each standard cell in advance. Two modeling approaches used for this are the **Non-Linear Delay Model (NLDM)** and **CCS (Composite Current Source) timing**, alongside separate **power** and **noise** characterization.

### 8.2 Standard Cell Design Flow

A cell library contains cells of varying function, size, and threshold voltage. Building one involves:

- **Inputs**: the PDK, DRC/LVS rules, SPICE models, an existing reference library, and user-defined specifications for the new cell.
- **Design steps**: circuit design, physical layout design, and extraction of the layout's parasitics.
- **Outputs**: a CDL (circuit description) netlist, a GDSII layout, a LEF file (defining the cell's width/height and pin geometry for place-and-route tools), an extracted SPICE netlist (including parasitic resistance/capacitance), and the characterized `.lib` files describing timing, noise, and power.

### 8.3 Characterization Flow

Characterizing a cell (using a tool such as GUNA, referenced in the course) follows a fixed sequence:

1. Read in the process/SPICE models.
2. Read the extracted SPICE netlist for the cell.
3. Identify the cell's behavior (e.g. recognizing it as a buffer, inverter, etc.).
4. Read the subcircuit definition for that gate.
5. Attach the necessary power supplies.
6. Apply the input stimulus.
7. Specify the output load capacitance.
8. Specify the simulation command (typically a `.TRAN` transient analysis).

These eight items are assembled into a configuration deck and fed to the characterization tool, which produces the resulting timing, noise, and power models.

### 8.4 Timing Characterization

Delay and transition-time measurements depend on a set of threshold points defined per cell: `in_rise_thr`, `in_fall_thr`, `out_rise_thr`, `out_fall_thr` for delay, and `slew_low_rise_thr`, `slew_high_rise_thr`, `slew_low_fall_thr`, `slew_high_fall_thr` for transition time.

**Propagation delay:**

```text
delay = time(out_threshold) − time(in_threshold)
```

If the threshold points are chosen poorly, this can come out negative.

**Transition time (slew):**

```text
rise transition = time(slew_high_rise_thr) − time(slew_low_rise_thr)
fall transition = time(slew_low_fall_thr) − time(slew_high_fall_thr)
```

---

## 9. Labs

The lab continues the same `picorv32a` OpenLane run from Day 6 — first running the floorplan stage and inspecting it in Magic, then running placement and inspecting that result as well.

### 9.1 Exploring OpenLane's Floorplan Configuration Options

Before running the stage, it's worth checking what floorplan-related variables OpenLane actually exposes. The `configuration/` folder in the OpenLane install holds one `.tcl` file per stage, plus a `README.md` documenting every variable and its default.

<img width="1920" height="983" alt="config ls" src="https://github.com/user-attachments/assets/d7b9fda4-179f-44d3-9df7-efb9d77118ba" />

The README describes the required variables (`DESIGN_NAME`, `VERILOG_FILES`, `CLOCK_PERIOD`, `CLOCK_PORT`, `CLOCK_NET`) and the optional synthesis-related ones.

<img width="1920" height="983" alt="readme file" src="https://github.com/user-attachments/assets/0c5a338f-81b6-4b8e-b5da-beeff6edd847" />

Further down, the floorplanning-specific variables are documented — `FP_CORE_UTIL`, `FP_ASPECT_RATIO`, `FP_SIZING`, `DIE_AREA`, the I/O metal layers, and the power-distribution-network pitch/offset variables.

<img width="1920" height="983" alt="readme file floor plan" src="https://github.com/user-attachments/assets/006d727e-15eb-4a2d-ab56-9854436afca9" />

`configuration/floorplan.tcl` itself sets the actual defaults used unless a design overrides them — `FP_CORE_UTIL` defaulting to 50%, `FP_ASPECT_RATIO` to 1, and so on.

<img width="1920" height="983" alt="floorplan tcl file" src="https://github.com/user-attachments/assets/5fd4d082-6bd9-4f0b-975d-e2dcfa1a2a76" />

### 9.2 Running the Floorplan Stage

```tcl
run_floorplan
```

This runs OpenROAD's floorplanning steps: converting the netlist/DEF, tap-cell insertion, I/O pin placement, and PDN (power distribution network) generation. The tail of the run shows the PDN step relocating several power-stripe locations onto the nearest actual stripe, then confirms PDN generation succeeded.

<img width="1920" height="983" alt="floor plan run successful" src="https://github.com/user-attachments/assets/059c143c-3e41-4976-9cec-9801f2ab2b26" />

### 9.3 Reviewing the Floorplan Logs

Each run directory keeps a `logs/floorplan/` folder with one log per sub-step (`verilog2def`, `ioPlacer`, `tapcell`, `pdn`). The `ioPlacer` log confirms the LEF/DEF files it read, the pin count, and that it placed pins using a random/even distribution since no macro blocks were found in this design.

<img width="1920" height="983" alt="cmds 2" src="https://github.com/user-attachments/assets/f8884b71-7a39-41a6-83d3-1a68d5e351e1" />

<img width="1920" height="983" alt="ioPlacer log file" src="https://github.com/user-attachments/assets/a266a7ed-4e90-49ce-b46f-67d26f00975d" />

### 9.4 Utilization Override for This Run

The design's library-specific config (`sky130A_sky130_fd_sc_hd_config.tcl`) overrides the default 50% utilization down to 35% for this particular run, and derives a target placement density from it.

<img width="1920" height="983" alt="sky130A config tcl file" src="https://github.com/user-attachments/assets/b2b384c4-b3a4-40fc-ad25-5193eafae478" />

The run's fully expanded `config.tcl` confirms `FP_CORE_UTIL` actually took the value `"35"` for this run, alongside the rest of the floorplan/PDN-related settings inherited from the defaults.

<img width="1920" height="983" alt="overided deafult with sky130A prove 35 utilization factor" src="https://github.com/user-attachments/assets/6d455ce8-d7db-426f-9846-7529fbfa5007" />

### 9.5 Inspecting the Generated Floorplan DEF

`results/floorplan/picorv32a.floorplan.def` records the actual die area and row layout OpenROAD produced. Die-area coordinates are given in DEF database units, and `UNITS DISTANCE MICRONS 1000` means 1000 of those units equal 1 micron — so a die area of `(0 0) (660685 671405)` works out to roughly **660.7 µm × 671.4 µm**.

<img width="1920" height="983" alt="floorplan results demonstrating die area too" src="https://github.com/user-attachments/assets/6885cbdb-a401-4b2d-a0b3-ce349e60eda1" />

### 9.6 Viewing the Floorplan in Magic

The floorplan DEF (together with the merged LEF) can be opened directly in Magic to see it visually.

```bash
magic -T /home/vsduser/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech lef read ../../tmp/merged.lef def read picorv32a.floorplan.def &

```

<img width="1920" height="983" alt="floorplan view through magic" src="https://github.com/user-attachments/assets/f48d0eee-9a8e-4ddb-af57-670cfd377db8" />

### 9.7 Floorplan Layout — Pins, Tap Cells, and Decap Cells

At this stage the core is still mostly empty of actual logic — what's visible is the row structure, I/O pin columns, and the tap/decap cells inserted during floorplanning.

<img width="1920" height="983" alt="floorplan layout" src="https://github.com/user-attachments/assets/df57b424-31e7-4544-914c-d75799979b48" />

Zooming in shows individual decap cells (`sky130_fd_sc_hd__decap_3`) and labeled I/O pins such as `mem_la_wstrb[0]`, attached to the metal3 layer.

<img width="1920" height="983" alt="floorplan detail layout" src="https://github.com/user-attachments/assets/7819f5ce-bf42-4f1b-bfd2-5fe5cb9621c1" />

Further along the same row, more decap/tap cells and other labeled pins (`mem_rdata[3]`, `trace_data[26]`) appear on metal2/metal3.

<img width="1920" height="983" alt="floor plan layout 3" src="https://github.com/user-attachments/assets/2fdc2ae3-e38f-4639-80bf-2dd61e79273f" />

A closer inspection of one particular fixed cell (a `sky130_fd_sc_hd__buf_1` instance) shows it selected in Magic's console, alongside nearby pin columns — this is still the floorplan-stage layout, before global cell placement fills in the rest of the core.

<img width="1920" height="983" alt="detail layout floor plan" src="https://github.com/user-attachments/assets/2120b95d-9017-450e-98c4-fd1d53981e95" />

### 9.8 Running Placement

```tcl
run_placement
```

This runs global and detailed placement, followed by a resizing/optimization pass. The completion log shows the design stats: 21,699 total instances (6,354 of them fixed — the tap/decap/pin cells from floorplanning), 15,449 nets, a design area of about 420,473 µm², roughly 36% utilization (55% once padding is included), and 238 placement rows. The HPWL (half-perimeter wire length) is reported before and after legalization, and the layout is written out with a screenshot taken via KLayout.

<img width="1920" height="983" alt="run placement done" src="https://github.com/user-attachments/assets/9faa4529-33ad-4b0c-9a33-6ebabef5f148" />

### 9.9 Opening the Placement Result in Magic

The new `results/placement/picorv32a.placement.def` can be opened the same way the floorplan was, this time showing the fully placed design.

```bash
magic -T /home/vsduser/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech lef read ../../tmp/merged.lef def read picorv32a.placement.def &
```

<img width="1920" height="983" alt="run placement magic layout cmd" src="https://github.com/user-attachments/assets/4cc30dba-9f40-4788-bc75-82c1dfc1499e" />

### 9.10 Placement Layout — Full and Detailed Views

The full-core view now shows the core densely packed with standard cells rather than the sparse floorplan-only layout from section 9.7.

<img width="1920" height="983" alt="placement layout" src="https://github.com/user-attachments/assets/598b0149-b998-403a-8260-efc683fd3216" />

Zooming into a section of the placed design shows individual standard-cell instances — flip-flops (`dfxtp_4`), muxes (`mux2_8`), buffers, and AND-OR-invert cells — each labeled with its net/instance name.

<img width="1920" height="983" alt="placement layout detail view" src="https://github.com/user-attachments/assets/63fd7b48-0ca0-46b4-8a7d-f6c5085ca16b" />

---

## 10. Conclusion

Day 7 covered how a floorplan is actually built up — sizing the core and die, reserving space for pre-placed IPs, surrounding them with decoupling capacitors, planning a multi-point power network, placing I/O pins, and blocking off regions from automated placement — followed by how placement itself binds the netlist to physical cells and optimizes the result. A short look at library characterization also showed where the timing/power numbers used throughout the flow actually come from. The lab carried `picorv32a` through both `run_floorplan` and `run_placement` in OpenLane, confirming a roughly 35% target utilization and inspecting both the sparse floorplan and the fully placed layout directly in Magic.
