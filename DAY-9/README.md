# Day 9: Pre-layout Timing Analysis and Importance of Good Clock Tree

### Contents
1. [Timing Modeling using Delay Tables](#1-timing-modeling-using-delay-tables)
2. [Timing Analysis with Ideal Clocks using OpenSTA](#2-timing-analysis-with-ideal-clocks-using-opensta)
3. [Clock Tree Synthesis, TritonCTS and Signal Integrity](#3-clock-tree-synthesis-tritoncts-and-signal-integrity)
4. [Timing Analysis with Real Clocks using OpenSTA](#4-timing-analysis-with-real-clocks-using-opensta)
5. [Labs](#5-labs)

---

## 1. Timing Modeling using Delay Tables

### Introduction to Delay Tables
Delay tables are 2D lookup tables stored inside Liberty (`.lib`) files. They map **input transition (slew)** and **output load capacitance** to the resulting **cell delay** and **output transition**. Running full SPICE simulation for every cell instance in a multi-million-gate design isn't feasible, so STA tools use these pre-characterized tables and interpolate between entries for speed.

### Delay Table Usage
- Axes: `input_net_transition` (slew) and `total_output_net_capacitance` (load).
- Values: `cell_rise` / `cell_fall` (propagation delay), `rise_transition` / `fall_transition` (output slew).
- Linear/bilinear interpolation is used when the actual slew-load pair falls between characterized points.
- A separate set of tables exists for each PVT (Process, Voltage, Temperature) corner, since delay varies with process variation, supply voltage, and temperature.
- Every path delay STA reports is ultimately derived from these table lookups.

---

## 2. Timing Analysis with Ideal Clocks using OpenSTA

### Flip-Flop Setup Time
Setup time is the minimum time before the active clock edge that data must be stable at a flip-flop's D input for it to be captured reliably. Violating it causes metastability.

**Setup equation (ideal clock):**
```
Tclk ≥ Tck→q + Tlogic + Tsetup + Tskew
```
- `Tck→q` – clock-to-Q delay of the launching flop
- `Tlogic` – combinational path delay
- `Tsetup` – setup requirement of the capturing flop
- `Tskew` – clock skew between launch and capture

### Clock Jitter and Uncertainty
- **Clock jitter**: cycle-to-cycle variation in clock period, caused by PLL/DLL noise and power-supply noise.
- **Clock uncertainty**: the combined margin (jitter + duty-cycle distortion + PLL phase error) subtracted from the available clock period in setup checks, so the analysis stays realistic instead of assuming a perfect clock.

### Slack
```
Slack = Required Time − Arrival Time
```
Positive slack → timing met. Negative slack → violation.

---

## 3. Clock Tree Synthesis, TritonCTS and Signal Integrity

### Clock Tree Routing and Buffering using H-Tree Algorithm
Clock Tree Synthesis (CTS) builds a balanced, low-skew distribution network from the clock source (PLL/pad) to every sequential element. The **H-tree** is a recursive, symmetric routing topology: the clock signal is split at the center of each branch into equal-length paths, repeating recursively so every leaf sees nearly identical wire length and delay. Buffers/inverters are inserted along the tree to control transition time and drive strength at each level.

Typical CTS goals:
- Minimize global skew (difference between earliest and latest leaf arrival).
- Control insertion delay (source-to-leaf latency).
- Meet max transition/capacitance limits on clock nets.

### Crosstalk and Clock Net Shielding
Clock nets switch frequently and drive many loads, making them prone to **crosstalk** from neighboring signal nets — coupling capacitance can inject noise that shifts the clock edge or causes glitches. **Shielding** mitigates this by routing grounded (or fixed) shield wires alongside clock nets, or by giving clock nets extra spacing, to reduce coupling capacitance from adjacent aggressor nets.

---

## 4. Timing Analysis with Real Clocks using OpenSTA

Once CTS is done, the clock is no longer ideal — launch and capture paths have real, distinct clock-tree delays, so setup/hold analysis is re-run using actual arrival times.

### Setup Timing Analysis using Real Clocks
```
Data Required Time (DRT) − Data Arrival Time (DAT) = Slack
Slack ≥ 0 → timing met
```
- DAT includes real launch-clock insertion delay + Tck→q + logic delay.
- DRT includes real capture-clock insertion delay, period, and setup uncertainty.
- **Useful skew** can be exploited here: making the capture clock arrive intentionally later than the launch clock gives data more time, improving setup slack.

### Hold Timing Analysis using Real Clocks
```
Data Arrival Time (DAT) − Data Required Time (DRT) = Slack
Slack ≥ 0 → timing met
```
- Hold checks use the same-edge launch and capture (no clock period involved).
- Because a real clock tree can widen skew, hold violations become far more likely on short paths — the same skew that helps setup can hurt hold.
- **Clock Reconvergence Pessimism Removal (CRPR)** is applied to avoid double-counting delay/uncertainty common to both launch and capture clock paths.


---

## 5. Labs
 
### Lab 1: Exploring Track Info and Grid Alignment in Magic
 
Open the existing inverter layout in Magic with the sky130A tech file:
 
```bash
magic -T sky130A.tech sky130_inv.mag &
```
 
Check the routing track pitch for each layer (`li1`, `met1`–`met5`) from the PDK tech directory:
 
```bash
cd ~/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/openlane/sky130_fd_sc_hd
less tracks.info
```
 
Sample output:
```
li1 X 0.23 0.46
li1 Y 0.17 0.34
met1 X 0.17 0.34
met1 Y 0.17 0.34
...
met5 X 1.70 3.40
met5 Y 1.70 3.40
```
 
Snap the Magic layout grid to the li1 track pitch from the tkcon console:
```
grid 0.46um 0.34um 0.23um 0.46um
```
This aligns the view grid to the actual routing grid, so placed geometry lines up with valid track positions.
 
 
<img width="1920" height="983" alt="open magic cmd" src="https://github.com/user-attachments/assets/cf15b2ea-a223-421f-8a4e-c9a6914acb50" />
<img width="1920" height="983" alt="inverter layout" src="https://github.com/user-attachments/assets/727c6b53-c925-4537-a570-a8d947f29f34" />
<img width="1920" height="983" alt="tracks info filw" src="https://github.com/user-attachments/assets/2c64f797-9bfb-45b3-8464-d71f2de84b84" />
<img width="1920" height="983" alt="to open tracks info cmd and its path" src="https://github.com/user-attachments/assets/c720b99b-fe34-4452-b175-5b873e17fc11" />
<img width="1920" height="983" alt="taken li1 tracks on grid and how layout grid changes to the the size" src="https://github.com/user-attachments/assets/fabf720a-4fae-403e-b025-1a2898dab84c" />
<img width="1920" height="983" alt="u can see the intersection of horizontal and vertical tracks placed for routing" src="https://github.com/user-attachments/assets/20069292-1329-473d-842f-2c9903fe8568" />
<img width="1920" height="983" alt="width covers 3 boxes ( 2boxes and half half on ends)" src="https://github.com/user-attachments/assets/4c23ca81-9123-4587-bd17-bd5569fbd32d" />
<img width="1920" height="983" alt="vetical boxes length" src="https://github.com/user-attachments/assets/adfad48b-4e92-40b8-9848-dc3be29b052d" />

---
 
### Lab 2: Building and Labeling a Custom Standard Cell (sky130_vsdinv)
 
- Zoom into the inverter layout to inspect the `A` (input) and `Y` (output) diffusion/poly regions, along with `VPWR`/`VGND` rails.
- Use **Edit → Text (label)** to attach port labels to the correct layer:
  - `VPWR` → attached to `metal1`, port class `inout`, use `power`
  - `VGND` → attached to `metal1`, port class `inout`, use `ground`
  - `A` → attached to `locali`, port class `input`, use `signal`
  - `Y` → attached to `locali`, port class `output`, use `signal`
- Verify each label/port assignment in the tkcon console using `what`:
```
  % what
  Selected label(s):
      "VPWR" is attached to metal1 in cell def sky130_inv
  % port class inout
  % port use power
```
  Repeat similarly for VGND, A, and Y.
- Save the modified cell under a new name:
```
  save sky130_vsdinv.mag
```
 
 <img width="1920" height="983" alt="we can label the ports" src="https://github.com/user-attachments/assets/4f1e558a-d044-4b1b-9ab9-83a04acb1a52" />
<img width="1920" height="983" alt="port class name set and use declaration" src="https://github.com/user-attachments/assets/55390dca-a0a7-4672-a9e3-5e1107c9e8df" />
<img width="1920" height="983" alt="saving as sky130_vsdinv mag" src="https://github.com/user-attachments/assets/575a3d59-3f33-4d2c-b128-cdffb9142be0" />
<img width="1920" height="983" alt="successfully saved as sky130_vsdinv mag" src="https://github.com/user-attachments/assets/f7518d73-b2c9-4878-a738-1378729441a9" />
---
 
### Lab 3: Generating a LEF from the Custom Cell
 
With the saved `sky130_vsdinv.mag` cell loaded, generate the abstract LEF view from the tkcon console:
```
write lef
```
This produces `sky130_vsdinv.lef`, containing the macro definition (CLASS CORE, SIZE, PIN definitions for A, Y, VPWR, VGND with their RECT geometry and antenna area values).
 
Verify the file was written:
```bash
ls -ltr
# sky130_vsdinv.mag  (2716 bytes)
```
 <img width="1920" height="983" alt="using write lef cmd to create a lef file" src="https://github.com/user-attachments/assets/c4f95a24-01d8-4a1f-8a48-a212eb6642cf" />
<img width="1920" height="983" alt="created a lef file using lef write cmd and this is the file" src="https://github.com/user-attachments/assets/43ddedae-ee7d-401f-96b0-ff4978ee01ba" />

---
 
### Lab 4: Integrating the Custom Cell into the picorv32a Design
 
Copy the generated LEF into the design's `src` folder:
```bash
cp sky130_vsdinv.lef /home/vsduser/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src
```
 
Also copy the standard cell timing libraries (fast/typical/slow) into the same `src` folder:
```bash
cd ~/Desktop/work/tools/openlane_working_dir/openlane/vsdstdcelldesign/libs
cp sky130_fd_sc_hd__* /home/vsduser/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src
```
 
Edit `designs/picorv32a/config.tcl` to point to these libraries and include the extra LEF:
```tcl
set ::env(LIB_SYNTH)   "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
 
set ::env(EXTRA_LEFS) [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```
 
<img width="1920" height="983" alt="copied lef file into picorv32a" src="https://github.com/user-attachments/assets/b68c7dca-0f82-47fc-bc9d-c527a18181f0" />
<img width="1920" height="983" alt="typical file" src="https://github.com/user-attachments/assets/ea826eb8-ad28-4f63-a2ad-5afdd7ea4ae4" />
<img width="1920" height="983" alt="slow_lib file" src="https://github.com/user-attachments/assets/13470ae3-ee28-4629-afb8-36793ef6b580" />
<img width="1920" height="983" alt="fast_lib file" src="https://github.com/user-attachments/assets/d150eb3b-7e0c-4644-a06a-c2e87366421b" />
<img width="1920" height="983" alt="successfully copied lib files into picorv32a" src="https://github.com/user-attachments/assets/fdd6f3ff-ccf8-4f7e-b14b-3450cae453e8" />
<img width="1920" height="983" alt="modified config tcl " src="https://github.com/user-attachments/assets/a1e6e7d8-afc8-4984-95c9-71ed88ebeb0f" />

---
 
### Lab 5: Running OpenLane Flow (Prep + Synthesis) with the Custom Cell
 
Launch the OpenLane Docker container and enter interactive flow mode:
```bash
docker run -it \
  -v $PWD:/openLANE_flow \
  -v $PDK_ROOT:$PDK_ROOT \
  -e PDK_ROOT=$PDK_ROOT \
  -u $(id -u $USER):$(id -g $USER) \
  efabless/openlane:v0.21
bash-4.2$ ./flow.tcl -interactive
```
 
Prepare the design (merges LEFs, including the custom `sky130_vsdinv.lef`):
```
% prep -design picorv32a -tag 11-09_13-36 -overwrite
```
Log confirms the extra LEF was merged:
```
[INFO]: Merging the following extra LEFs: /openLANE_flow/designs/picorv32a/src/sky130_vsdinv.lef
[INFO]: Preparation complete
```
 
Run synthesis:
```
% run_synthesis
```
Confirm the custom cell was used and check initial timing numbers in the synthesis log:
```
sky130_vsdinv                1554
Chip area for module '\picorv32a': 147712.918000
...
tns -711.59
wns -23.89
[INFO]: Synthesis was successful
```
1554 instances of the custom inverter cell were inferred into the synthesized netlist.
 
<img width="1920" height="983" alt="docker cmds" src="https://github.com/user-attachments/assets/024c5110-2348-4910-90e4-ca82c2032ce5" />
<img width="1920" height="983" alt="prep complete" src="https://github.com/user-attachments/assets/67fd989a-4dc2-4b99-bc6b-b041644b738f" />
<img width="1920" height="983" alt="1554 vsdinv cells are used in picorv32a" src="https://github.com/user-attachments/assets/837ab6ee-d8d0-48df-b0bb-1101e1a4181f" />
<img width="1920" height="983" alt="synthesis completed" src="https://github.com/user-attachments/assets/412bc52b-88bf-46e6-a6e1-7f0505cdac9b" />

---
 
### Lab 6: Inspecting Synthesis Strategy Variables (Trying to Reduce Negative Slack)
 
From the interactive flow shell, inspect and tweak synthesis strategy environment variables to influence WNS/TNS:
```
% echo $::env(SYNTH_STRATEGY)
AREA 0
% set ::env(SYNTH_STRATEGY) 1
% echo $::env(SYNTH_BUFFERING)
1
% echo $::env(SYNTH_SIZING)
0
% set ::env(SYNTH_SIZING) 1
% echo $::env(SYNTH_DRIVING_CELL)
sky130_fd_sc_hd__inv_8
```
These control whether synthesis applies area-based or delay-based optimization, whether cell sizing/buffering is enabled, and which cell drives primary inputs — all of which affect the reported negative slack.
 

<img width="1920" height="983" alt="trying to reduce the negative slack" src="https://github.com/user-attachments/assets/eb032599-fca7-449c-917e-681bbb6d2979" />

---
 
### Lab 7: Running Placement
 
```
% run_placement
```
Key stats from the run:
```
total instances        21696
nets                    15444
design area             420473.3 u^2
utilization             36 %
utilization padded      54 %
rows                    238
row height              2.7 u
 
original HPWL           756230.0 u
legalized HPWL          769495.6 u
delta HPWL              2 %
[INFO DPL-0020] Mirrored 6226 instances
[INFO]: Taking a Screenshot of the Layout Using Klayout...
```
 
<img width="1920" height="983" alt="run_placement finished" src="https://github.com/user-attachments/assets/bfad91cb-33bb-4262-a759-7fe029f9cd23" />
 
---
 
### Lab 8: Viewing the Placed Layout in Magic
 
Navigate to the placement results and open the DEF in Magic using the merged LEF:
```bash
cd designs/picorv32a/runs/11-09_13-36/results/placement/
magic -T /home/vsduser/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech \
  lef read ../../tmp/merged.lef \
  def read picorv32a.placement.def
```
This renders the full placed chip (dense standard-cell rows) — zooming in shows individual placed cells (e.g. `sky130_fd_sc_hd__o22a_1`, `sky130_fd_sc_hd__tapvpwrvgnd_1`) with instance names like `PHY_3136`, `_18745_`.
 
<img width="1920" height="983" alt="magic run cmds after placement" src="https://github.com/user-attachments/assets/0903b39d-418a-4d80-acad-8afbf4db29f4" />
<img width="1920" height="983" alt="placement layout" src="https://github.com/user-attachments/assets/1e9f3b05-fc50-4722-a32b-83b1e52caef6" />
<img width="1920" height="983" alt="found vsdinv cells in picorv32a" src="https://github.com/user-attachments/assets/6bab3e09-02dd-4d62-9dcb-74fab2042db5" />


---
 
### Lab 9: Pre-STA Timing Check and Creating a Base SDC
 
Run a pre-STA timing check (before a proper constraints file is in place) and observe a heavily violated setup path:
```
data arrival time      48.32
data required time      4.71
slack (VIOLATED)      -43.62
wns -43.62
tns -11872.33
```
 
To address this, create a custom `my_base.sdc` in `designs/picorv32a/src/` with proper clock and I/O delay constraints:
```tcl
set ::env(CLOCK_PORT) clk
set ::env(CLOCK_PERIOD) 5.000
set ::env(SYNTH_DRIVING_CELL) sky130_fd_sc_hd__buf_8
set ::env(SYNTH_DRIVING_CELL_PIN) X
set ::env(SYNTH_CAP_LOAD) 17.65
 
create_clock [get_ports $::env(CLOCK_PORT)] -name $::env(CLOCK_PORT) -period $::env(CLOCK_PERIOD)
 
set IO_PCT 0.9892
set input_delay_value  [expr {$::env(CLOCK_PERIOD) * $IO_PCT}]
set output_delay_value [expr {$::env(CLOCK_PERIOD) * $IO_PCT}]
 
puts "\[INFO\]: Setting output delay to: $output_delay_value"
puts "\[INFO\]: Setting input delay to: $input_delay_value"
 
set clk_indx  [lsearch [all_inputs] [get_port $::env(CLOCK_PORT)]]
set rst_indx  [lsearch [all_inputs] [get_port resetn]]
set all_inputs_wo_clk     [lreplace [all_inputs] $clk_indx $clk_indx]
set all_inputs_wo_clk_rst [lreplace $all_inputs_wo_clk $rst_indx $rst_indx]
 
# correct resetn
set_input_delay $input_delay_value -clock [get_clocks $::env(CLOCK_PORT)] $all_inputs_wo_clk_rst
set_output_delay $output_delay_value -clock [get_clocks $::env(CLOCK_PORT)] [all_outputs]
 
set cap_load [expr {$::env(SYNTH_CAP_LOAD) / 1000.0}]
puts "\[INFO\]: Setting load to: $cap_load"
set_load $cap_load [all_outputs]
```
 
<img width="1920" height="983" alt="pre_sta cong file runned" src="https://github.com/user-attachments/assets/d1940afd-d144-4134-92f4-7363f7b79d5a" />
<img width="1920" height="983" alt="my_base sdc file created" src="https://github.com/user-attachments/assets/4ca9c8b1-c7e5-41cb-8323-cedb5f6efd47" />

---
 
### Lab 10: Running Clock Tree Synthesis (CTS)
 
*(To be filled in.)*
 

---
 
### Lab 11: Setup & Hold Analysis with Real Clock Network (Post-CTS)
 
*(To be filled in.)*
 
 
---
