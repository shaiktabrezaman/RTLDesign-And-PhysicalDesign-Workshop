# Day 8: SPICE-Based CMOS Characterization & the 16-Mask Fabrication Process

## Overview

Day 8 covers Module 3 of the Sky130 VSD course, which shifts focus from digital
floorplanning/placement (Day 7) down to the transistor and device-fabrication
level. The day is split into two lecture threads and a connected set of hands-on
labs:

- **Lecture 1** introduces SPICE-based characterization of a CMOS inverter —
  how a SPICE deck is structured, and how it's used to extract both the
  **static behavior** (switching threshold, V_M) and **dynamic behavior**
  (rise delay, fall delay) of a standard cell.
- **Lecture 2** walks through the **16-mask CMOS fabrication process**,
  showing how a bare silicon wafer is transformed into a working CMOS
  inverter, one masking/processing step at a time.
- The **labs** put this into practice: modifying an IO placer floorplan
  parameter, opening a real Sky130 standard-cell layout (`sky130_inv`) in
  Magic, extracting a SPICE netlist directly from that layout, fixing the
  extracted netlist to reference the correct Sky130 device models, running
  transient simulations in ngspice to measure rise/fall delay and transition
  times, and finally exploring the DRC (Design Rule Check) rule deck for the
  Metal3 and Poly layers.

---

## 1. SPICE Deck Basics

A SPICE netlist describing a circuit is built from three core pieces of
information:

1. **Connectivity** — which nodes each component's terminals are connected to.
2. **Component values** — resistances, capacitances, voltage levels, transistor
   width/length, etc.
3. **Model identification** — which device model (e.g. a specific PMOS/NMOS
   process model) a given component instance should use.

For a CMOS inverter built from a matched PMOS/NMOS pair (both W = 0.375 µm,
L = 0.25 µm in the example used to introduce the concept), the netlist
declares the two transistors, a load capacitance at the output, the supply
and input sources, and simulation directives (`.op` for an operating-point
solve, `.dc` for a DC sweep, `.tran` for a transient run).

When this netlist is instead **extracted directly from a Magic layout** (as
done in this day's labs), the same three pieces of information are still
present, but the component values are filled in automatically from the drawn
geometry (width, length, source/drain diffusion area and perimeter), and the
model names default to Magic's own naming convention rather than a
course-provided model file — which is exactly what made the later "fix the
netlist" lab necessary (see the Labs section).

---

## 2. Static Behavior — Switching Threshold (V_M)

The DC transfer characteristic of a CMOS inverter (input voltage on the
x-axis, output voltage on the y-axis) traces an S-shaped curve as the input
sweeps from 0 to V_DD. As the input rises, the inverter passes through five
operating regions in sequence:

1. PMOS in linear region, NMOS off
2. PMOS in linear region, NMOS in saturation
3. Both PMOS and NMOS in saturation
4. PMOS in saturation, NMOS in linear region
5. PMOS off, NMOS in linear region

**The switching threshold, V_M**, is defined as the point where V_in = V_out
on this curve — equivalently, the point where the current through the PMOS
and NMOS are equal and opposite (I_dsP = −I_dsN), since both devices are in
saturation there. It's a direct measure of how "balanced" the inverter is.

V_M is not fixed — it shifts with the relative sizing of the PMOS and NMOS.
For a matched device pair (W_n/L_n = W_p/L_p = 1.5), V_M sits close to the
midpoint of the supply (~0.98 V on a 2.5 V analysis). Widening the PMOS
relative to the NMOS (e.g. W_p/L_p = 3.75 while W_n/L_n stays at 1.5) pushes
V_M higher (~1.2 V), because the stronger PMOS pulls the output high for a
larger range of input voltages before the NMOS can win the fight. This
device-sizing dependency is the practical reason PMOS transistors are
conventionally drawn wider than NMOS in a balanced inverter — PMOS carriers
(holes) have lower mobility than NMOS carriers (electrons), so extra width
compensates for that mobility gap.

<img width="1920" height="1080" alt="vm" src="https://github.com/user-attachments/assets/e9041d27-91c5-47b2-8ff0-a5f3c3c2f63e" />

---

## 3. Dynamic Behavior — Rise Delay, Fall Delay, and Transition Times

Where V_M characterizes the inverter's DC behavior, the **transient
response** characterizes how fast it switches — this is measured by driving
the input with a pulse and observing the output in a `.tran` simulation.

Two related but distinct sets of timing metrics come out of this:

- **Transition time (rise/fall time)**: how long the output takes to swing
  through the "middle" of its voltage range on a single edge, typically
  measured between the 20% and 80% points of the full swing.
- **Propagation delay (cell rise/fall delay)**: how long it takes the output
  to respond to a change at the input, typically measured from the 50%
  crossing point of the input transition to the 50% crossing point of the
  corresponding output transition.

Both metrics matter for different reasons: transition time affects how
"sharp" a signal edge looks to the next stage (slow edges waste power and
can cause timing issues downstream), while propagation delay is the number
that directly feeds into static timing analysis of a digital path.

---

## 4. The 16-Mask CMOS Fabrication Process

Fabricating a CMOS inverter (or any CMOS logic) starts from a bare silicon
wafer and builds up transistors and interconnect one lithographic mask at a
time. The process can be broken into eight major stages, each of which may
use one or more of the 16 total masks.

### 4.1 Substrate Selection

The process starts with a **p-type silicon wafer**, chosen for high
resistivity (~10¹⁵ cm⁻³ doping) and a `<100>` crystal orientation — this
crystal orientation is preferred because it produces a lower density of
interface traps at the silicon–oxide boundary compared to other
orientations, which matters for transistor performance.

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/953a361a-1874-4d16-8515-3df920358082" />

### 4.2 Creating the Active Region (Mask 1)

A stack of ~40 nm SiO₂, ~80 nm Si₃N₄, and ~1 µm photoresist is deposited on
the wafer and patterned with Mask 1. The nitride/oxide layers act as a
barrier during the subsequent oxidation step. Field oxide is then grown in
the exposed regions through **LOCOS** (Local Oxidation of Silicon), which
also produces the characteristic "bird's beak" — a tapered lateral
encroachment of oxide under the edge of the nitride mask, a well-known
side-effect of this technique. The nitride layer is stripped afterward using
hot phosphoric acid.

<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/ddfe6b9e-19f3-4517-811d-de998b7ee8f5" />

### 4.3 N-Well and P-Well Formation (Mask 2)

With Mask 2, boron is implanted to form the p-well and phosphorus is
implanted to form the n-well side by side on the same wafer (this twin-tub
approach is what allows both NMOS and PMOS devices to be built on a single
substrate). The wafer then goes through a **high-temperature drive-in
diffusion** step in a furnace, which drives the implanted dopants deeper
into the substrate and activates them, forming the finished N-well/P-well
regions.

<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/4af1f527-d194-406e-8b67-2665d2199856" />
<img width="1920" height="1080" alt="5" src="https://github.com/user-attachments/assets/4fa34611-6536-4b76-aad8-7a67c158b665" />

### 4.4 Gate Formation (Mask 6)

Threshold voltage tuning happens here — it's controlled by the substrate
doping concentration (N_A) and the gate oxide capacitance (C_ox). A
sacrificial oxide is stripped with dilute HF, and the transistor's gate
region is defined with Mask 4 (boron) and Mask 5 (arsenic) doping steps
ahead of gate patterning. Polysilicon is deposited and doped n-type (to keep
its resistance low), and Mask 6 defines and etches the final gate shape from
that polysilicon layer.

<img width="1920" height="1080" alt="6" src="https://github.com/user-attachments/assets/2cc59149-f2c0-48a9-ba99-90a2b0f350ea" />


### 4.5 Lightly Doped Drain (LDD) Formation (Masks 7 & 8)

Before the full source/drain implant, a lighter dose is implanted close to
the gate edge — this is the LDD. It exists to counter two effects that
become more severe as devices shrink:

- **Hot-electron effect**: high-energy carriers accelerated by the strong
  electric field near the drain can gain enough energy to break Si–Si bonds
  (crossing roughly a 3.2 eV barrier between silicon and the SiO₂ conduction
  band), leading to long-term device degradation.
- **Short-channel effect**: the drain's electric field can penetrate far
  enough into the channel to interfere with the gate's control over it.

Mask 7 implants phosphorus and Mask 8 implants boron to form the lightly
doped N⁻/P⁻ regions. **Side-wall spacers** (~0.1 µm of Si₃N₄/SiO₂, deposited
and then anisotropically plasma-etched) are formed alongside the gate — these
physically set back the heavier source/drain implant that follows, which is
what creates the "lightly doped" grading at the drain edge.

<img width="1920" height="1080" alt="8" src="https://github.com/user-attachments/assets/3694ecbb-c917-4087-b735-f1331d5cce10" />


### 4.6 Source and Drain Formation (Masks 9 & 10)

A thin screen oxide is grown first to prevent channeling during implant.
Two implants — arsenic on the NMOS/p-well side and boron on the PMOS/n-well
side — form the full-strength source/drain regions, followed by a
high-temperature furnace anneal to activate the dopants and repair implant
damage.

<img width="1920" height="1080" alt="9" src="https://github.com/user-attachments/assets/8a77f115-ab59-48da-83fa-d56d39aa7e07" />


### 4.7 Local Interconnect / Contact Formation (Mask 11)

A thin oxide layer over the source/drain/gate regions is etched away in HF
to expose bare silicon, titanium is sputtered over the wafer, and the wafer
is annealed at **650–700 °C in a N₂ ambient for 60 seconds**. Where the
titanium was in direct contact with silicon, this reaction forms **titanium
nitride (TiN)**, which is used specifically for **local interconnect** —
short-range connections rather than full-chip routing. Mask 11 then defines
where these local contacts/plugs are etched and filled.

<img width="1920" height="1080" alt="10" src="https://github.com/user-attachments/assets/eb32177d-1459-4e11-bdca-fab4be80289d" />
<img width="1920" height="1080" alt="11" src="https://github.com/user-attachments/assets/c03a4b2c-6d84-45da-b60b-48a004302286" />
<img width="1920" height="980" alt="12" src="https://github.com/user-attachments/assets/ed28e2dd-0149-4b5b-9ee9-b162ca614d81" />
<img width="1920" height="998" alt="13" src="https://github.com/user-attachments/assets/61563921-29b2-4cbc-a3b1-6702a64b66d3" />

### 4.8 Higher-Level Metal Formation (Mask 12 and beyond, through Mask 16)

Building the higher metal levels is a repeated cycle of dielectric
deposition, planarization, and metal patterning:

- ~1 µm of PSG/BPSG dielectric is deposited and planarized with CMP.
- A TiN barrier layer plus a blanket tungsten fill forms the contact plugs
  (Mask 12), followed by another CMP step.
- Aluminum is deposited and plasma-etched to form the metal1 pattern.
- A further SiO₂ layer is deposited and CMP-planarized, then Mask 14 defines
  new contact holes, filled again with TiN/tungsten.
- Mask 15 defines the next metal pattern.
- A final Si₃N₄ passivation layer is deposited over the whole stack, and
  **Mask 16 — the final mask in the process** — opens the last contact holes
  through that passivation, exposing the bond pads.

The completed stack, from substrate to top metal, is what makes the finished
device fabricable, with source, gate, and drain terminals of each transistor
now accessible from outside the chip.


<img width="1920" height="997" alt="Screenshot 2026-09-15 230644" src="https://github.com/user-attachments/assets/29717eb9-87d2-4ca3-aae4-ba9c671a99c5" />
<img width="1920" height="986" alt="14" src="https://github.com/user-attachments/assets/be20a4ec-fd24-43a5-9bd5-d7cbc88d5d5a" />
<img width="1920" height="997" alt="Screenshot 2026-09-15 230655" src="https://github.com/user-attachments/assets/e56836f3-a5d6-4f89-970c-66313407cc80" />


---

## Labs

### Lab 1 — Modifying the Floorplan / IO Placer Configuration

Re-ran the floorplan step with a different IO placer mode to compare pin
placement strategies.
<img width="1920" height="983" alt="io placer equi distance" src="https://github.com/user-attachments/assets/4a7ae227-5ad5-4bae-8821-903fcf729087" />
<img width="1920" height="983" alt="zoomed io placer equi distance" src="https://github.com/user-attachments/assets/2830dee4-8a77-4da5-bdb0-2d0dc4596937" />
<img width="1920" height="983" alt="io placer after mode set to 2" src="https://github.com/user-attachments/assets/9695ad32-b6e5-4119-9c88-1dd5ca9215f8" />
<img width="1920" height="983" alt="zoomed io placer after mode set to 2" src="https://github.com/user-attachments/assets/405ed811-699e-4f9f-bb3a-9daa36eea458" />

With the IO placer left in its default (equidistant) mode, pins are spaced
uniformly along each side of the die boundary. Setting the mode parameter to
`2` changes this to a different distribution strategy, visibly clustering
pins differently along the boundary compared to the uniform baseline.

### Lab 2 — Setting Up Magic with the Sky130 Standard-Cell Design Repo

<img width="1920" height="983" alt="git clonned vsdstdcelldesign" src="https://github.com/user-attachments/assets/db652168-1e2e-464b-a044-94ae4585dcf8" />
<img width="1920" height="983" alt="copied sky130A tech to the directory clonned" src="https://github.com/user-attachments/assets/ca24c556-4172-4685-b192-f6d37c6c6c66" />
<img width="1920" height="983" alt="magic cmd" src="https://github.com/user-attachments/assets/673c71f0-e71d-4e68-a58f-b3e796b77f13" />
<img width="1920" height="983" alt="inverter layout" src="https://github.com/user-attachments/assets/ce662f30-6b04-4f39-a24c-7b8f46578e84" />

Cloned the `vsdstdcelldesign` repository (containing the reference
`sky130_inv.mag` layout) into the OpenLane working directory, copied the
`sky130A.tech` technology file from the PDK's `libs.tech/magic` folder into
that repo so Magic could resolve it locally, and launched Magic with:

```
magic -T sky130A.tech sky130_inv.mag &
```

This opened the reference Sky130 inverter standard-cell layout — VPWR/VGND
power rails, input pin **A**, output pin **Y**, and the PMOS/NMOS diffusion
regions all visible in the layout window.

### Lab 3 — Identifying Transistor Layers
<img width="1920" height="983" alt="nmos" src="https://github.com/user-attachments/assets/b2d1f6ce-cf62-4498-9578-d6fdd7928c6d" />
<img width="1920" height="983" alt="pmos" src="https://github.com/user-attachments/assets/1da18003-1631-48df-83e6-467e6f751bcf" />

Selected each transistor region in turn and ran the `what` command in
Magic's `tkcon` console to confirm the mask layer under the cursor —
verifying the left device as `nmos` and the right device as `pmos`.

### Lab 4 — Extracting a SPICE Netlist from the Layout
<img width="1920" height="983" alt="extraxct all cmd in tcon" src="https://github.com/user-attachments/assets/a13e8bb1-95c8-4c88-b90f-752aae26b395" />
<img width="1920" height="983" alt="exttospicecmd" src="https://github.com/user-attachments/assets/9b0e9537-3224-4d31-afc3-7088e776dfc9" />
<img width="1920" height="983" alt="created successfully" src="https://github.com/user-attachments/assets/56897573-d67f-413a-b404-fc2f65a53d76" />
<img width="1920" height="983" alt="sky130_inv spice file" src="https://github.com/user-attachments/assets/e51a20f6-d6d2-4c9b-9573-727daf11e123" />

With the reference inverter layout loaded, ran:

```
extract all
ext2spice cthresh 0 zthresh 0
ext2spice
```

This produced `sky130_inv.ext` and then `sky130_inv.spice`. Opening the
auto-generated SPICE file showed it referencing the transistors through
Magic's raw internal model names (`sky130_fd_pr__nfet_01v8` /
`sky130_fd_pr__pfet_01v8`) — not yet matched to any local model library, and
missing the stimulus/analysis statements needed to actually simulate it.

### Lab 5 — Locating the Sky130 Device Models

<img width="1920" height="983" alt="lib files" src="https://github.com/user-attachments/assets/a8557a22-1863-4bec-8dee-74c971469eff" />
<img width="1920" height="983" alt="nshort_model file" src="https://github.com/user-attachments/assets/c51becb8-baca-4dc8-b443-e322123f5cda" />
<img width="1920" height="983" alt="pshort_model file" src="https://github.com/user-attachments/assets/005f68d0-989e-4661-a4f9-6c5d46bbd45f" />

Inspected the `libs` folder inside `vsdstdcelldesign`, which contains
`pshort.lib` and `nshort.lib` (BSIM4 models for the short-channel PMOS and
NMOS devices used in the standard cell library) alongside the
fast/typical/slow corner libraries for the full `sky130_fd_sc_hd` cell
library. Opening these files confirmed the actual usable model names are
**`pshort_model.0`** (pmos) and **`nshort_model.0`** (nmos) — the names that
the extracted netlist needed to be edited to reference.

### Lab 6 — Fixing the Extracted Netlist and Running the First Simulation

<img width="1920" height="983" alt="ngspice run cmd" src="https://github.com/user-attachments/assets/6c881ec6-429d-4d53-b621-a372972a3194" />
<img width="1920" height="983" alt="modified spice deck file" src="https://github.com/user-attachments/assets/71186fc0-2da7-4fc6-8b96-f5d5acefde21" />

The first attempt to run the raw extracted netlist in ngspice failed with:

```
Error: unknown subckt: x0 y a vgnd vgnd pshort_model.0 ...
```

because the deck didn't correctly reference the model libraries. After
editing the deck to `.include ./libs/pshort.lib` and
`.include ./libs/nshort.lib`, defining a proper `.subckt sky130_inv A Y
VPWR VGND` with M0/M1 correctly instantiating `pshort_model.0` /
`nshort_model.0`, adding a `Va` PULSE source on the input, the extracted
parasitic capacitances, and a `.tran`/`.control` block, the simulation ran
successfully — ngspice printed the initial transient solution without
errors.

### Lab 7 — Transient Analysis and Delay/Transition-Time Extraction

<img width="1920" height="983" alt="transcient analysis plot of cmps inverter (modified code)" src="https://github.com/user-attachments/assets/587558a9-e4dc-4152-a208-e33f39c9a55b" />
<img width="1917" height="987" alt="rise time transistion" src="https://github.com/user-attachments/assets/37a15742-305a-4bd8-89e2-1a381692c4b1" />
<img width="1917" height="981" alt="fall time transistion" src="https://github.com/user-attachments/assets/97e1ec5d-88f7-47c5-82e9-6ed0a2652d99" />
<img width="1912" height="982" alt="cell rise delay" src="https://github.com/user-attachments/assets/12064c53-e62a-4c83-96f4-507ee86bde61" />
<img width="1917" height="977" alt="cell fall delay" src="https://github.com/user-attachments/assets/6c18e544-6a78-42d6-820b-047769db1b27" />

Plotting `y` (output) against `time a` (input) in ngspice's transient plot
showed several clean switching transitions of the inverter. Using the plot
window's cursor tool to measure between the relevant threshold crossings:

| Metric | Method | Result |
|---|---|---|
| Rise transition time | 20%–80% on a rising edge | 0.06194 ns |
| Fall transition time | 20%–80% on a falling edge | 0.04267 ns |
| Cell rise delay | 50% input → 50% output, rising | 0.05953 ns |
| Cell fall delay | 50% input → 50% output, falling | 0.05941 ns |

These directly correspond to the dynamic-behavior metrics introduced in the
theory section above, now measured on a real extracted-from-layout netlist
rather than a hand-written textbook example.

### Lab 8 — Exploring the DRC Rule Deck (Metal3 and Poly)

<img width="1920" height="983" alt="met3" src="https://github.com/user-attachments/assets/f83f8c06-707d-4c40-b04d-28f64313001d" />

Loaded a reference DRC example deck containing several labeled Metal3 (m3)
test structures — m3.1 through m3.7 — split into cells demonstrating correct
designs (m3.4) versus an incorrect one (m3.7), plus two larger multi-via
test structures (m3.3c, m3.3d).

<img width="1920" height="983" alt="drc why met2" src="https://github.com/user-attachments/assets/22ed60bf-4c47-4821-bb7d-90b772be695d" />
<img width="1920" height="983" alt="drc why met 3 2" src="https://github.com/user-attachments/assets/19cee2bb-670f-4174-9ddd-6691324c6712" />

Using the `drc why` command on flagged regions returned the specific rule
being violated in each case:

- `Metal3 spacing < 0.3um (met3.2)`
- `Metal3 minimum area < 0.24um^2 (met3.6)`

<img width="1920" height="983" alt="cif see VIA2 cmd full view" src="https://github.com/user-attachments/assets/3094af63-0b52-43de-b6fc-360b89278dde" />
<img width="1920" height="983" alt="cif see VIA2 cmd" src="https://github.com/user-attachments/assets/2a803c93-5686-4559-b2bb-1710e3dc4f14" />
<img width="1920" height="983" alt="measurement" src="https://github.com/user-attachments/assets/2afd3f72-2eeb-40d3-9e84-3d3d3682c10a" />


Zooming into the m3.3c test structure (a grid array of via/contact cuts
inside a met3 region, showing DRC=22 total violations), used `paint
m3contact` and `cif see VIA2` to isolate and inspect the VIA2 mask layer
directly, then queried a selected via/contact region with `box`, which
reported dimensions of 0.190 × 0.200 µm (0.038 µm² area).

<img width="1920" height="983" alt="rule violation" src="https://github.com/user-attachments/assets/d80dfd77-e13a-40bb-be33-398927d40db2" />

Switched the editing layer to **poly** (DRC count changed to DRC=10 on this
layer) and queried a selected poly shape, which reported dimensions of
0.335 × 0.210 µm (0.070 µm² area) — labeled "poly.9" in the reference deck.

---

## Conclusion

Day 8 connected the transistor-level theory of Module 3 to hands-on practice
across two closely related threads. On the fabrication side, walking through
all 16 masks made it clear how each structural feature of a finished CMOS
inverter — wells, gate, LDD, source/drain, local interconnect, and the
multiple metal levels — traces back to a specific masking and processing
step, and how techniques like LOCOS and self-aligned TiN interconnect exist
specifically to solve process-level problems (bird's-beak control,
local-vs-global routing) rather than being arbitrary choices.

On the characterization side, going from a reference Magic layout to a
working SPICE netlist end-to-end — extracting it, discovering it referenced
the wrong model names, fixing the deck against the actual `pshort_model.0`
/ `nshort_model.0` BSIM4 models, and finally getting clean rise/fall delay
and transition-time numbers out of a real transient simulation — made the
abstract V_M/delay definitions from the lecture concrete. The DRC labs
closed the loop by showing what happens when a layout *doesn't* meet the
rules those same masks are built to respect, and how Magic's `drc why`
command turns an abstract rule violation into a specific, actionable
spacing or area number.

