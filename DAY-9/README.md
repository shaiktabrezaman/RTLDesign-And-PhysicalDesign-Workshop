# Day 9: Pre-layout Timing Analysis and Importance of Good Clock Tree

## Module 4

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



---
