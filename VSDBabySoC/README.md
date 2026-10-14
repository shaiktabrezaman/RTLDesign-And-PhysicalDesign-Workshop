# VSDBabySoC — RTL Simulation, Synthesis, Gate-Level Simulation, STA and Physical Design
 
## Overview
 
This folder documents the functional verification flow carried out on **VSDBabySoC**, a small SoC built around the `rvmyth` RISC-V core along with an analog PLL (`avsdpll`) and DAC (`avsddac`). The flow followed here is: clone the design, run RTL simulation, synthesize it in Yosys against the SKY130 standard-cell library, inspect the resulting netlist, run gate-level simulation (GLS) on that netlist, and finally compare the RTL and GLS waveforms to confirm they match.
 
Part 2 of this README continues from the synthesized netlist into **static timing analysis (OpenSTA)** across 16 PVT corners and the **physical design flow** (OpenROAD) up to GDSII.
 
<p align="center">
  <img width="601" height="332" alt="VSDBabySoC" src="https://github.com/user-attachments/assets/21bffbe6-56b3-4b3c-a8b4-f26b8e32911c" />
</p>
<p align="center">
  <sub>Source: VLSI System Design (VSD)</sub>
</p>

 
## Contents
 
**Part 1 — RTL to GLS**
- [1. Setting Up the Project Folder](#1-setting-up-the-project-folder)
- [2. RTL and Testbench](#2-rtl-and-testbench)
- [3. RTL Simulation](#3-rtl-simulation)
- [4. Synthesis Using Yosys](#4-synthesis-using-yosys)
- [5. Netlist Views](#5-netlist-views)
- [6. Gate-Level Simulation (GLS)](#6-gate-level-simulation-gls)
- [7. RTL vs GLS Comparison](#7-rtl-vs-gls-comparison)
- [8. Conclusion (RTL to GLS)](#8-conclusion-rtl-to-gls)
**Part 2 — Static Timing Analysis and Physical Design**
- [9. Static Timing Analysis Using OpenSTA](#9-static-timing-analysis-using-opensta)
- [10. STA on VSDBabySoC](#10-sta-on-vsdbabysoc)
- [11. PVT Corner-Based STA (16 Corners)](#11-pvt-corner-based-sta-16-corners)
- [12. Physical Design Flow (OpenROAD)](#12-physical-design-flow-openroad)
---

## 1. Setting Up the Project Folder

A dedicated folder was created and the BabySoC simulation repository was cloned into it.

```bash
mkdir baby_soc
cd baby_soc/
git clone https://github.com/Subhasis-Sahu/BabySoC_Simulation
cd BabySoC_Simulation
```

<img width="1536" height="319" alt="image" src="https://github.com/user-attachments/assets/64279b89-a20b-4455-8715-b34ca584a5b1" />

---

## 2. RTL and Testbench

The design consists of the top-level `vsdbabysoc` module instantiating the `rvmyth` core, along with the PLL and DAC models. A dedicated testbench drives `CLK` and `reset` and observes the DAC output path.

<img width="1536" height="787" alt="VirtualBox_vsdworkshop_30_08_2026_18_34_12" src="https://github.com/user-attachments/assets/21aa5837-7f28-49ee-9637-770f1f8a9d18" />

---

## 3. RTL Simulation

RTL simulation was run first, using a `PRE_SYNTH_SIM` define so the testbench selects the pre-synthesis simulation path.

```bash
iverilog -o ./pre_synth_sim.out -DPRE_SYNTH_SIM src/module/testbench.v -I src/include/ -I src/module/

./pre_synth_sim.out

gtkwave pre_synth_sim.vcd
```

This produces `pre_synth_sim.vcd`, opened in GTKWave to observe `CLK`, `reset`, `OUT`, and the `RV_TO_DAC[9:0]` bus.

<img width="1536" height="787" alt="9" src="https://github.com/user-attachments/assets/e1010ee3-bb93-4f25-b81a-c16e6b7d9955" />

---

## 4. Synthesis Using Yosys

The following commands were used to synthesize the design in Yosys, mapping it onto the SKY130 standard-cell library along with the PLL and DAC liberty models.
 
### Reading the Top-Level Design
 
```tcl
read_verilog src/module/vsdbabysoc.v
```
 
This reads the main top-level Verilog file into Yosys. The `vsdbabysoc` module connects the major blocks of the design.
 
---
 
### Reading the RVMYTH Module
 
```tcl
read_verilog -I src/include/ src/module/rvmyth.v
```
 
This reads the RVMYTH processor core. The `-I src/include/` option lets Yosys locate the include files that `rvmyth.v` depends on.
 
---
 
### Reading the Clock Gate Module
 
```tcl
read_verilog -I src/include/ src/module/clk_gate.v
```
 
This reads the clock gate module, again with the include directory provided. Once all three RTL files are read, Yosys has the full design hierarchy.
 
---
 
### Reading the PLL Library
 
```tcl
read_liberty -lib src/lib/avsdpll.lib
```
 
This loads the Liberty file describing the PLL block, so Yosys knows its pins and timing behavior during mapping.
 
---
 
### Reading the DAC Library
 
```tcl
read_liberty -lib src/lib/avsddac.lib
```
 
This loads the Liberty file describing the DAC block, for the same reason as above.
 
---
 
### Reading the SKY130 Standard Cell Library
 
```tcl
read_liberty -lib src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
 
This loads the main SKY130 standard-cell library, containing the logic gates, buffers, inverters, and flip-flops that the design will eventually be mapped onto.
 
---
 
### Selecting and Synthesizing the Top Module
 
```tcl
synth -top vsdbabysoc
```
 
This tells Yosys that `vsdbabysoc` is the top module of the design and runs Yosys's generic synthesis flow on it: elaboration, coarse-grain optimization, and technology-independent mapping into an internal logic representation.
 
---
 
### Mapping Flip-Flops
 
```tcl
dfflibmap -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
 
This maps the generic flip-flops produced by `synth` onto actual SKY130 flip-flop cells.
 
```text
RTL Flip-Flop
      ↓
dfflibmap
      ↓
Sky130 Flip-Flop Cell
```
 
---
 
### Generic Logic Optimization
 
```tcl
opt
```
 
This runs a round of technology-independent optimization, constant propagation and redundant logic removal, cleaning up the design before it goes into technology mapping.
 
---
 
### Technology Mapping Using ABC
 
```tcl
abc -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
 
The `abc` command maps the remaining combinational logic (AND, OR, NAND, NOR, inverters, and so on) onto actual SKY130 cells. This is the step that turns generic logic into technology-specific logic.
 
---
 
### Replacing Undefined Values
 
```tcl
setundef -zero
```
 
Any undefined (`x`) signals left in the design are replaced with a defined logic `0`, so the netlist doesn't carry unknown values into simulation.
 
---
 
### Cleaning the Design
 
```tcl
clean -purge
```
 
This removes unused wires, cells, and dangling nets left over after optimization and mapping.
 
---
 
### Renaming Generated Objects
 
```tcl
rename -enumerate
```
 
Yosys often generates hard-to-read internal names during synthesis. This command renames those objects with a simple enumerated scheme, making the resulting netlist easier to read.
 
---
 
### Reviewing the Synthesis Summary
 
```tcl
stat
```
 
This prints a summary of the final design (cell counts by type, wire counts, and estimated area), used here to confirm the design mapped cleanly onto the SKY130 cells.
 
---
 
### Writing Out the Netlist
 
```tcl
write_verilog -noattr baby_soc_net.v
```
 
This writes the final synthesized netlist out as a Verilog file, `baby_soc_net.v`. The `-noattr` flag strips Yosys's internal attributes so the file stays clean for gate-level simulation.

The `stat` command was used at the end to review the final cell count and confirm the design mapped cleanly onto the SKY130 cells.

<div align="center">

<table>
  <tr>
    <td align="center">
      <img width="317" height="690" alt="stat1" src="https://github.com/user-attachments/assets/4f59dca9-661f-4297-82ac-c938d4bc5253" />
    </td>
    <td align="center">
      <img width="391" height="715" alt="stat2" src="https://github.com/user-attachments/assets/9b1a4b23-23d7-413f-be06-8990e89d291c" />
    </td>
    <td align="center">
      <img width="347" height="714" alt="stat3" src="https://github.com/user-attachments/assets/2ecd0f6d-c212-4f5e-937a-a1a8cbab287b" />
    </td>
  </tr>
</table>

</div>

<div align="center">

<table>
  <tr>
    <td align="center">
      <img width="324" height="714" alt="stat4" src="https://github.com/user-attachments/assets/e408bae6-27bc-4965-8382-514fd3beec96" />
    </td>
    <td align="center">
      <img width="389" height="712" alt="stat5" src="https://github.com/user-attachments/assets/8d3f90f7-d2ff-49b5-a5e8-93aad80c888d" />
    </td>
  </tr>
</table>

</div>
  
---
## 5. Netlist Views

The synthesized netlist was viewed using the Yosys show command (rendered via the Dot Viewer). Two levels of the same netlist were captured — a top-level block view and the fully gate-mapped internal view.

### Top-Level / Block-Level Netlist

This view shows the vsdbabysoc hierarchy at the block level — the avsdpll instance driving CLK, feeding into the rvmyth core, which in turn drives the avsddac instance through the RV_TO_DAC bus, out to OUT. Port-level connections between the three sub-blocks are clearly visible here.

<img width="1536" height="787" alt="netlist" src="https://github.com/user-attachments/assets/9094fc42-f46b-44b7-868f-af4af9751cd2" />

### Gate-Level Netlist (Post-abc Mapping)

This is the same design after abc has mapped everything down to SKY130 standard cells, the dense mesh of wires reflects the actual gate-level implementation rather than the clean block hierarchy above. It's not meant to be read cell-by-cell. it's included to show the scale/density of the mapped logic compared to the block-level view.

<img width="1536" height="787" alt="6" src="https://github.com/user-attachments/assets/29a5324f-1d99-42e6-b8f1-095c919b31ec" />

---

## 6. Gate-Level Simulation (GLS)

The synthesized netlist was then simulated with the standard-cell Verilog models included, using `POST_SYNTH_SIM` and `FUNCTIONAL` defines so the testbench exercises the post-synthesis functional path.

```bash
iverilog -DPOST_SYNTH_SIM -DFUNCTIONAL -I src/include/ -I ../../sky130RTLDesignAndSynthesisWorkshop/my_lib/verilog_model/ -I src/module/ src/module/testbench.v

./a.out

gtkwave post_synth_sim.vcd
```

<img width="1536" height="787" alt="11" src="https://github.com/user-attachments/assets/470432d5-ea09-468f-b290-6108a5f523fc" />

---

## 7. RTL vs GLS Comparison

The `RV_TO_DAC[9:0]` bus, `CLK`, `reset`, and `OUT` were compared between `pre_synth_sim.vcd` (RTL) and `post_synth_sim.vcd` (GLS). Both waveforms line up bit-for-bit across the full simulation window, confirming the synthesized netlist behaves identically to the original RTL.

<img width="1536" height="787" alt="10" src="https://github.com/user-attachments/assets/00c9cf4c-5340-4532-894f-4915272c6425" />

---

## 8. Conclusion

The BabySoC design was taken through the complete verification loop, covering RTL simulation, Yosys synthesis onto the SKY130 library, netlist inspection, and gate-level simulation. The RTL and GLS waveforms matched exactly on the `RV_TO_DAC` bus and other observed signals. This confirms that synthesis preserved the intended functional behavior of the design with no mismatch introduced during the RTL-to-gate-level translation.
---
---

# Part 2 — Static Timing Analysis and Physical Design
 
Functional correctness (Part 1) says the netlist *behaves* correctly. It says nothing about whether it does so **fast enough**. Static timing analysis (STA) checks every timing path in the design against the clock without needing any simulation vectors. This part uses **OpenSTA** for STA, first on a small example to learn the commands, then on the synthesized VSDBabySoC netlist across 16 PVT corners, and finally moves into the OpenROAD physical design flow.
 
---
 
## 9. Static Timing Analysis Using OpenSTA
 
Before running STA on VSDBabySoC, the OpenSTA command flow was practised on the small example design (`example1.v`) that ships with OpenSTA.
 
### 9.1 Building and Starting OpenSTA in Docker
 
```bash
cd OpenSTA
```
Moves into the cloned OpenSTA directory that contains the Dockerfile.
 
```bash
docker build --file Dockerfile.ubuntu22.04 --tag opensta .
```
Builds a Docker image named `opensta` from the Ubuntu 22.04 Dockerfile, so OpenSTA runs without installing anything on the host.
 
```bash
docker run -i -v $HOME:/data opensta
```
Starts the container interactively and mounts the home directory at `/data`, so design files on the host are visible inside OpenSTA.
 
> **In my run**, the tool was started from the `efabless/openlane:v0.21` image with the OpenSTA folder mounted as `/data`:
> ```bash
> docker run -it -v /home/vsduser/Desktop/OpenSTA:/data efabless/openlane:v0.21 sta
> ```

<img width="1920" height="983" alt="run_cmd" src="https://github.com/user-attachments/assets/a3edb6ff-a868-408a-86cb-8eb71b041259" />

 
### 9.2 Loading the Library, Design and Constraints
 
```tcl
read_liberty /data/examples/nangate45_slow.lib
```
Loads the Liberty (`.lib`) timing library that holds cell delays, setup/hold times and capacitances.
 
```tcl
read_verilog /data/examples/example1.v
```
Reads the gate-level Verilog netlist to be analysed.
 
```tcl
link_design top
```
Links the netlist with the library cells, with `top` as the top-level module.
 
```tcl
create_clock -name clk -period 10 {clk1 clk2 clk3}
```
Defines a clock named `clk` with a 10 ns period on the three clock ports.
 
```tcl
set_input_delay -clock clk 0 {in1 in2}
```
Tells the tool that inputs `in1` and `in2` arrive 0 ns after the clock edge.
 
> 📷 **Add image here — Heading: "Loading liberty, netlist and constraints in OpenSTA"**
> Suggested file: `images/sta_load_design.png`
 
### 9.3 Reporting Timing
 
```tcl
report_checks
```
Reports the worst timing path (setup / max by default) from startpoint to endpoint.
 
**Output obtained:**
 
- Startpoint `r2` and endpoint `r3`, both rising-edge flip-flops clocked by `clk`, path type `max`
- Path: `r2/CK` → `r2/Q` (DFF_X1) → `u1/Z` (BUF_X1) → `u2/ZN` (AND2_X1) → `r3/D`
- Data arrival time: 0.42 ns
- Data required time: 9.84 ns (10 ns clock edge minus 0.16 ns library setup time)
- **Slack: 9.43 ns (MET)**, so the path meets setup timing with a large margin

<img width="1920" height="983" alt="previous lecture lab sta report" src="https://github.com/user-attachments/assets/1033d19b-8381-4a2d-8f7d-4e5756cbf79f" />

 
### 9.4 Next Two Commands
 
```tcl
create_clock -name clk -period 0 {clk1 clk2 clk3}
```
Redefines the clock with a 0 ns period, a deliberately extreme constraint used to see how the report changes when the timing budget is removed.
 
```tcl
report_checks -path_delay min_max
```
Reports both the min (hold) and max (setup) paths in one go.

---
 
## 10. STA on VSDBabySoC
 
### 10.1 Running the Commands from a TCL File
 
Instead of typing each command into the OpenSTA prompt, the commands were kept in a TCL file so the whole STA run happens in one go.
 
```tcl
set base_path /data/Desktop/OpenSTA/vsdbabysoc/
set lib_name sky130_fd_sc_hd__tt_025C_1v80.lib
 
read_liberty -min $base_path/lib/$lib_name
read_liberty -max /data/Desktop/OpenSTA/vsdbabysoc/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
 
read_liberty -min /data/Desktop/OpenSTA/vsdbabysoc/lib/avsdpll.lib
read_liberty -max /data/Desktop/OpenSTA/vsdbabysoc/lib/avsdpll.lib
 
read_liberty -min /data/Desktop/OpenSTA/vsdbabysoc/lib/avsddac.lib
read_liberty -max /data/Desktop/OpenSTA/vsdbabysoc/lib/avsddac.lib
 
read_verilog /data/Desktop/OpenSTA/vsdbabysoc/vsdbabysoc.synth.v
 
link_design vsdbabysoc
 
read_sdc /data/Desktop/OpenSTA/vsdbabysoc/sdc/vsdbabysoc_synthesis.sdc
 
report_checks
```
 
| Command | What it does |
|---|---|
| `set base_path`, `set lib_name` | Store the directory and library file name in variables so paths aren't repeated |
| `read_liberty -min` / `-max` | Loads the library for the min (hold) and max (setup) analysis |
| `avsdpll.lib`, `avsddac.lib` | Timing models for the analog PLL and DAC blocks |
| `read_verilog vsdbabysoc.synth.v` | Reads the Yosys-synthesized netlist from Part 1 |
| `link_design vsdbabysoc` | Links the netlist to the loaded libraries with `vsdbabysoc` as top |
| `read_sdc vsdbabysoc_synthesis.sdc` | Applies the clock and I/O timing constraints |
| `report_checks` | Reports the worst timing path |
 
<img width="1920" height="983" alt="run tcl file" src="https://github.com/user-attachments/assets/23865cba-f2d2-433f-94d0-f88f692611a8" />

<img width="1920" height="983" alt="1" src="https://github.com/user-attachments/assets/970ea9fc-aa4c-48ab-a58d-a118144e11a9" />

<img width="1920" height="983" alt="2" src="https://github.com/user-attachments/assets/eb637c22-5a9f-4260-ac02-5fb368ee9139" />

 
### 10.2 SKY130 Timing Libraries (PVT Corners)
 
The SKY130 standard-cell package ships a separate Liberty file for every process-voltage-temperature (PVT) combination, in the `timing` folder of the `sky130_fd_sc_hd` library. The file name encodes the corner:
 
```text
sky130_fd_sc_hd__<process>_<temperature>_<voltage>.lib
e.g. sky130_fd_sc_hd__ss_n40C_1v28.lib   → slow-slow process, -40 °C, 1.28 V
```
 
<img width="1920" height="983" alt="timing files" src="https://github.com/user-attachments/assets/5307c8b2-e9b4-4458-9bc3-ad546bc5cb48" />

 
---
 
## 11. PVT Corner-Based STA (16 Corners)
 
### 11.1 Objective and Configuration
 
A chip must work across manufacturing variation, supply voltage and temperature, not only at the typical corner. STA was repeated on VSDBabySoC across **16 PVT corners** to evaluate setup timing, hold timing, worst negative slack (WNS), total negative slack (TNS), worst setup slack, worst hold slack and the status of each corner.
 
| Parameter | Value |
|---|---|
| Design | VSDBabySoC (synthesized netlist `vsdbabysoc.synth.v`) |
| STA tool | OpenSTA |
| Analysis | Setup (MAX path) and Hold (MIN path) |
| Clock | `clk`, period 11 ns |
| Number of PVT corners | 16 |
 
**The 16 corners:**
 
| Process | Corners |
|---|---|
| FF (fast-fast) | FF_100C_1v65, FF_100C_1v95, FF_n40C_1v56, FF_n40C_1v65, FF_n40C_1v76, FF_n40C_1v95 |
| SS (slow-slow) | SS_100C_1v40, SS_100C_1v60, SS_n40C_1v28, SS_n40C_1v35, SS_n40C_1v40, SS_n40C_1v44, SS_n40C_1v60, SS_n40C_1v76 |
| TT (typical) | TT_025C_1v80, TT_100C_1v80 |
 
Corner names read as *process_temperature_voltage*, for example `SS_n40C_1v28` is slow-slow, −40 °C, 1.28 V (`n40C` means negative 40 °C).
 
### 11.2 How the Script Works
 
All 16 corners were handled in one TCL script (`run.tcl`), run inside the container:
 
| Step | What the script does |
|---|---|
| `define_corners` | Declares the 16 corner names |
| `read_liberty -corner <name> ...` | Loads the matching SKY130 library for each corner |
| PLL and DAC libraries | Loads `avsdpll.lib` and `avsddac.lib` for every corner |
| `read_verilog` / `link_design` / `read_sdc` | Reads the synthesized netlist, links it, applies constraints |
| `foreach corner $corners` | Loops over every corner |
| `report_checks -corner $corner -path_delay max` | Reports the worst setup path for that corner |
| `report_checks -corner $corner -path_delay min` | Reports the worst hold path for that corner |
| `worst_slack -min` / `-max` | Records worst hold and worst setup slack |
| `total_negative_slack -max` | Records the TNS |
| WNS / status logic | WNS = worst setup slack if negative, else 0. Status = `VIOLATED` if WNS < 0, otherwise `MET` |
| Final `puts` table | Prints one comparison row per corner |
 
<details>
<summary><b>Complete OpenSTA TCL script (16 PVT corners)</b></summary>
```tcl
# VSDBabySoC - ALL 16 PVT CORNERS : SETUP + HOLD PATH ANALYSIS
set base_path /data/OpenSTA/vsdbabysoc
 
# DEFINE ALL 16 TIMING CORNERS
define_corners \
  FF_100C_1v65 FF_100C_1v95 FF_n40C_1v56 FF_n40C_1v65 FF_n40C_1v76 FF_n40C_1v95 \
  SS_100C_1v40 SS_100C_1v60 SS_n40C_1v28 SS_n40C_1v35 SS_n40C_1v40 SS_n40C_1v44 \
  SS_n40C_1v60 SS_n40C_1v76 TT_025C_1v80 TT_100C_1v80
 
# FF CORNERS
read_liberty -corner FF_100C_1v65 $base_path/sky130_fd_sc_hd__ff_100C_1v65.lib
read_liberty -corner FF_100C_1v95 $base_path/sky130_fd_sc_hd__ff_100C_1v95.lib
read_liberty -corner FF_n40C_1v56 $base_path/sky130_fd_sc_hd__ff_n40C_1v56.lib
read_liberty -corner FF_n40C_1v65 $base_path/sky130_fd_sc_hd__ff_n40C_1v65.lib
read_liberty -corner FF_n40C_1v76 $base_path/sky130_fd_sc_hd__ff_n40C_1v76.lib
read_liberty -corner FF_n40C_1v95 $base_path/sky130_fd_sc_hd__ff_n40C_1v95.lib
 
# SS CORNERS
read_liberty -corner SS_100C_1v40 $base_path/sky130_fd_sc_hd__ss_100C_1v40.lib
read_liberty -corner SS_100C_1v60 $base_path/sky130_fd_sc_hd__ss_100C_1v60.lib
read_liberty -corner SS_n40C_1v28 $base_path/sky130_fd_sc_hd__ss_n40C_1v28.lib
read_liberty -corner SS_n40C_1v35 $base_path/sky130_fd_sc_hd__ss_n40C_1v35.lib
read_liberty -corner SS_n40C_1v40 $base_path/sky130_fd_sc_hd__ss_n40C_1v40.lib
read_liberty -corner SS_n40C_1v44 $base_path/sky130_fd_sc_hd__ss_n40C_1v44.lib
read_liberty -corner SS_n40C_1v60 $base_path/sky130_fd_sc_hd__ss_n40C_1v60.lib
read_liberty -corner SS_n40C_1v76 $base_path/sky130_fd_sc_hd__ss_n40C_1v76.lib
 
# TT CORNERS
read_liberty -corner TT_025C_1v80 $base_path/sky130_fd_sc_hd__tt_025C_1v80.lib
read_liberty -corner TT_100C_1v80 $base_path/sky130_fd_sc_hd__tt_100C_1v80.lib
 
# PLL AND DAC LIBRARIES (repeated for every corner)
read_liberty -corner FF_100C_1v65 $base_path/lib/avsdpll.lib
read_liberty -corner FF_100C_1v65 $base_path/lib/avsddac.lib
# ... same two lines for each of the remaining 15 corners ...
 
# READ NETLIST AND CONSTRAINTS
read_verilog $base_path/vsdbabysoc.synth.v
link_design vsdbabysoc
read_sdc $base_path/sdc/vsdbabysoc_synthesis.sdc
 
set corners {
  FF_100C_1v65 FF_100C_1v95 FF_n40C_1v56 FF_n40C_1v65 FF_n40C_1v76 FF_n40C_1v95
  SS_100C_1v40 SS_100C_1v60 SS_n40C_1v28 SS_n40C_1v35 SS_n40C_1v40 SS_n40C_1v44
  SS_n40C_1v60 SS_n40C_1v76 TT_025C_1v80 TT_100C_1v80
}
 
array set min_slack {}
array set max_slack {}
array set wns {}
array set tns {}
array set status {}
 
foreach corner $corners {
  puts "# PVT CORNER: $corner"
 
  puts "SETUP PATH ANALYSIS / CORNER: $corner / PATH TYPE: MAX (SETUP)"
  report_checks -corner $corner -path_delay max -format full \
                -group_path_count 1 -endpoint_path_count 1
 
  puts "HOLD PATH ANALYSIS / CORNER: $corner / PATH TYPE: MIN (HOLD)"
  report_checks -corner $corner -path_delay min -format full \
                -group_path_count 1 -endpoint_path_count 1
 
  set min_slack($corner) [worst_slack -corner $corner -min]
  set max_slack($corner) [worst_slack -corner $corner -max]
 
  if {$max_slack($corner) < 0.0} { set wns($corner) $max_slack($corner) } else { set wns($corner) 0.0 }
 
  set tns($corner) [total_negative_slack -corner $corner -max]
 
  if {$wns($corner) < 0.0} { set status($corner) "VIOLATED" } else { set status($corner) "MET" }
}
 
# FINAL 16-CORNER COMPARISON TABLE
puts [format "%-20s %12s %12s %12s %12s %-12s" "CORNER" "MIN SLACK" "MAX SLACK" "WNS" "TNS" "STATUS"]
foreach corner $corners {
  puts [format "%-20s %12.3f %12.3f %12.3f %12.3f %-12s" \
        $corner $min_slack($corner) $max_slack($corner) $wns($corner) $tns($corner) $status($corner)]
}
puts "Total PVT corners analyzed : [llength $corners]"
puts "ALL 16 PVT CORNERS COMPLETED"
---
 
</details>
### 11.3 Execution Evidence
 
<img width="1004" height="514" alt="image" src="https://github.com/user-attachments/assets/5bb07625-d39f-4349-832b-4b29ac720648" />

 
<img width="1004" height="514" alt="image" src="https://github.com/user-attachments/assets/0475523d-039a-4860-98e1-54b5c512b68d" />

 
<img width="1004" height="514" alt="image" src="https://github.com/user-attachments/assets/020b4bfa-75a6-4b36-acf0-bdd7fe1d7dfa" />

 
### 11.4 Setup Path Evidence
 
For every corner the worst setup (MAX) path was reported. In all 16 corners the startpoint is `core/CPU_is_addi_a3_reg` and the endpoint is a bit of `core/CPU_Xreg_value_a4_reg`, so the same long combinational path limits setup timing everywhere. The data path runs through a long chain of SKY130 cells (`a21oi`, `o21ai`, `fa` full-adder cells) before reaching the register.
 
<img width="1920" height="983" alt="1" src="https://github.com/user-attachments/assets/75d1cb65-454b-46af-8954-52b50b7df8ee" />
<img width="1920" height="983" alt="2" src="https://github.com/user-attachments/assets/0b21227e-eb69-47c0-a751-2fe4f76a13f3" />
<img width="1920" height="983" alt="3" src="https://github.com/user-attachments/assets/b8f4a915-ac89-4a20-a1cf-8c9ead859167" />
<img width="1920" height="983" alt="4" src="https://github.com/user-attachments/assets/e2029051-d712-4463-b21e-1293e46ce266" />

All 16 PVT Corners Evidence is mentioned in the reports

### 11.5 Hold Path Evidence
 
For every corner the worst hold (MIN) path was reported. In all 16 corners the startpoint is `core/CPU_imm_a2_reg[10]` and the endpoint is `core/CPU_imm_a3_reg[10]`, a direct register-to-register path.
 
<img width="1920" height="983" alt="1 1" src="https://github.com/user-attachments/assets/7eb2bd6a-5b4e-414d-a32f-801c82da05a3" />
<img width="1920" height="983" alt="2 2" src="https://github.com/user-attachments/assets/6eb1e62f-3d7d-4497-9f11-f8ff274487dc" />
<img width="1920" height="983" alt="3 3" src="https://github.com/user-attachments/assets/02c35249-f3ab-40e8-9fec-98df6337dfcd" />
<img width="1920" height="983" alt="4 4" src="https://github.com/user-attachments/assets/61b55b4c-fd41-4457-ad8e-ff1f98e54799" />

 
### 11.6 PVT Comparison Table
 
Values are taken from the final comparison table printed by the script.
 
- **MIN SLACK** = worst hold slack
- **MAX SLACK** = worst setup slack
- **WNS** = worst negative setup slack
- **TNS** = total negative setup slack
- **STATUS** = MET if WNS ≥ 0, otherwise VIOLATED
| Corner | Min Slack | Max Slack | WNS | TNS | Status |
|---|---:|---:|---:|---:|---|
| FF_100C_1v65 | 0.249 | 4.055 | 0.000 | 0.000 | MET |
| FF_100C_1v95 | 0.196 | 5.567 | 0.000 | 0.000 | MET |
| FF_n40C_1v56 | 0.292 | 2.278 | 0.000 | 0.000 | MET |
| FF_n40C_1v65 | 0.255 | 3.411 | 0.000 | 0.000 | MET |
| FF_n40C_1v76 | 0.224 | 4.430 | 0.000 | 0.000 | MET |
| FF_n40C_1v95 | 0.187 | 5.592 | 0.000 | 0.000 | MET |
| SS_100C_1v40 | 0.905 | -17.122 | -17.122 | -2599.514 | VIOLATED |
| SS_100C_1v60 | 0.642 | -7.874 | -7.874 | -892.705 | VIOLATED |
| SS_n40C_1v28 | 1.830 | -62.563 | -62.563 | -13957.588 | VIOLATED |
| SS_n40C_1v35 | 1.348 | -39.697 | -39.697 | -7993.793 | VIOLATED |
| SS_n40C_1v40 | 1.125 | -29.694 | -29.694 | -5466.484 | VIOLATED |
| SS_n40C_1v44 | 0.991 | -23.908 | -23.908 | -4107.187 | VIOLATED |
| SS_n40C_1v60 | 0.663 | -10.609 | -10.609 | -1355.797 | VIOLATED |
| SS_n40C_1v76 | 0.504 | -4.382 | -4.382 | -367.928 | VIOLATED |
| TT_025C_1v80 | 0.310 | 1.939 | 0.000 | 0.000 | MET |
| TT_100C_1v80 | 0.314 | 2.094 | 0.000 | 0.000 | MET |
 
### 11.7 PVT Corner Timing Graphs
 
**WNS vs PVT corner**
 
<img width="1004" height="528" alt="image" src="https://github.com/user-attachments/assets/4e70d122-19bf-47e3-a9b3-c4acdb9dd39e" />

 
WNS is 0.000 at all six FF corners and both TT corners. At the SS corners it is negative, from −4.382 (SS_n40C_1v76) to −62.563 (SS_n40C_1v28).
 
**TNS vs PVT corner**
 
<img width="1004" height="516" alt="image" src="https://github.com/user-attachments/assets/442d6ec9-9beb-47f3-b3b4-2eb21aac07ba" />
 
TNS is 0.000 at the FF and TT corners. At the SS corners it ranges from −367.928 (SS_n40C_1v76) to −13957.588 (SS_n40C_1v28).
 
**Setup slack vs PVT corner**
 
<img width="1004" height="522" alt="image" src="https://github.com/user-attachments/assets/f50626e1-96f7-4a22-b599-3f7f58f832b6" />

 
Worst setup slack is positive at the FF corners (2.278 to 5.592) and TT corners (1.939 and 2.094), and negative at all eight SS corners, with the lowest value of −62.563 at SS_n40C_1v28.
 
**Hold slack vs PVT corner**
 
<img width="1004" height="529" alt="image" src="https://github.com/user-attachments/assets/7c6f3383-6f31-4ab3-93a0-1443b067fcda" />

 
Worst hold slack is positive at all 16 corners, from 0.187 (FF_n40C_1v95) to 1.830 (SS_n40C_1v28).
 
### 11.8 Observations
 
- FF and TT corners meet setup timing; **all eight SS corners violate setup**.
- Hold slack is positive at all 16 corners.
- The worst setup violation is at **SS_n40C_1v28** (WNS −62.563 ns, TNS −13957.588 ns), where the data arrival time is 71.70 ns against a required time of 9.14 ns.
- Within the SS group, setup slack improves steadily as the supply voltage rises (for example −62.563 at 1.28 V to −4.382 at 1.76 V at −40 °C), so low voltage combined with slow process is the most severe condition.
- This is pre-layout analysis on the synthesized netlist with an ideal clock network. The design is **not timing-clean across all corners**, and no timing violations were fixed in this analysis.
---
 
## 12. Physical Design Flow (OpenROAD)
 
*To be added: moving to the `VSDPD` Codespace and running the OpenROAD flow scripts through the `flow` file: synthesis, floorplan, placement, CTS, global routing (congestion issue and its fix), final routing, post-route STA and the final GDSII layout.*
