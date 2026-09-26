# Day 10 — Detailed Routing, DRC & TritonRoute (RTL to GDS)

## Contents
1. Introduction to Maze Routing – Lee's Algorithm
2. Lee's Algorithm – Conclusion
3. Design Rule Check (DRC)
4. Basics of Global and Detail Routing & Configuring TritonRoute
5. TritonRoute Feature 1 – Honors Pre-processed Route Guides
6. TritonRoute Features 2 & 3 – Inter-guide Connectivity and Intra- & Inter-layer Routing
7. TritonRoute Method to Handle Connectivity
8. Routing Topology Algorithm

This module moves from the abstract placement stage into the physical realization of interconnects. It explains the classic maze-routing algorithm that still underpins modern detailed routers, the manufacturing rules that must be obeyed (DRC), and how OpenROAD's TritonRoute implements a production-quality detailed router on top of global-route guides.

---

### 1. Introduction to Maze Routing – Lee's Algorithm

**What it is:** A classic grid-based shortest-path algorithm (breadth-first search / wavefront expansion) used to find a route between two pins while avoiding obstacles.

**How it works (high level):**
- The routing surface is modelled as a uniform grid.
- From the source pin, a "wave" expands cell by cell, labelling each reachable cell with its distance from the source.
- Expansion continues until the target pin is reached (or the wavefront dies out).
- A path is traced back from the target to the source by following decreasing labels.

**Key properties:**
- Guarantees a shortest path in terms of grid steps (if one exists).
- Naturally handles arbitrary obstacles.
- Memory- and time-intensive on large modern designs, so it's used today mainly as a conceptual foundation or for small regions.

---

### 2. Lee's Algorithm – Conclusion

- Lee's algorithm is optimal for maze routing on a grid but scales poorly (O(N) memory and time, where N is the number of grid cells).
- Modern detailed routers replace pure Lee's search with more sophisticated techniques (A*, multi-source multi-sink search, rip-up-and-reroute, parallel wavefronts, etc.) while retaining the same core idea of exploring free space until a legal path is found.
- Understanding Lee's algorithm remains essential, since every later improvement (including TritonRoute's internal search) is built on the same foundation.

---

### 3. Design Rule Check (DRC)

**What it is:** A set of geometric manufacturing constraints imposed by the foundry that every drawn shape (wires, vias, pins) must satisfy.

**Typical DRC rules enforced during routing:**
- Minimum width of metal wires
- Minimum spacing between same-layer wires (wire pitch, typical design rules for pairs of wires)
- Via width and via spacing rules
- Via enclosure and cut size rules
- End-of-line spacing, notch rules, antenna rules, etc.

**Why it matters:** A route that is electrically correct but violates DRC cannot be fabricated. Detailed routers must treat DRC as hard constraints while searching for paths — the goal is always a DRC-clean solution.

---

### 4. Basics of Global and Detail Routing & Configuring TritonRoute

**Global Routing:**
Produces a coarse "guide" for each net — a sequence of GCells (global cells) that the net should travel through. It optimizes congestion and approximate wirelength but does not produce actual metal geometry.

**Detailed Routing:**
Takes the global-route guides and produces real metal wires and vias that obey all DRC rules and exactly connect the pins.

**TritonRoute** is OpenROAD's detailed router. Typical configuration points:
- Layer range (which metal layers may be used)
- Track patterns and preferred directions
- Via generation rules
- Congestion-driven rip-up-and-reroute iterations
- DRC cleanup passes

---

### 5. TritonRoute Feature 1 – Honors Pre-processed Route Guides

TritonRoute treats the guides produced by the global router as soft constraints. It performs the initial detail route while attempting, as much as possible, to route within these preprocessed route guides.

Preprocessing of the raw guides happens in stages: initial route guides → splitting → merging → bridging → final preprocessed guides. Preprocessed guides must:
- Have unit width
- Be in the preferred direction (e.g. vertical for M1, horizontal for M2)

This "guide-aware" behaviour keeps the detailed solution close to the global plan while still giving the router freedom to fix local problems.

---

### 6. TritonRoute Features 2 & 3 – Inter-guide Connectivity and Intra- & Inter-layer Routing

**Inter-guide connectivity:** Route guides for each net must satisfy inter-guide connectivity. Two guides are considered connected if:
- They are on the same metal layer with touching edges, or
- They are on neighboring metal layers with a nonzero vertically overlapped area

Additionally, every unconnected terminal (i.e. a pin of a standard-cell instance) must have its pin shape overlapped by a route guide.

**Intra-layer parallel & inter-layer sequential panel routing:** TritonRoute works on a proposed MILP-based panel routing scheme:
- Panels on a given layer are routed in parallel with each other (intra-layer parallel)
- Layers are then processed one after another, alternating even- and odd-indexed panels (inter-layer sequential)

This lets the router parallelize work within a layer while still respecting the sequencing needed across layers.

---

### 7. TritonRoute Method to Handle Connectivity

**Problem statement:**
- **Inputs:** LEF, DEF, preprocessed route guides
- **Output:** Detailed routing solution with optimized wire-length and via count
- **Constraints:** Route guide honoring, connectivity constraints, and design rules

**Access Points and Access Point Clusters:**
- **Access Point (AP):** An on-grid point on the metal layer of the route guide, used to connect to lower-layer segments, upper-layer segments, pins, or IO ports.
- **Access Point Cluster (APC):** A union of all APs derived from the same lower-layer segment, upper-layer guide, a pin, or an IO port.

Access points are illustrated for three cases: connecting to a lower-layer segment, connecting to a pin shape, and connecting to an upper layer — in each case, APs mark where a route guide can legally hand off connectivity to the next layer or terminal.

---

### 8. Routing Topology Algorithm

Once access point clusters (APCs) are known for a net, TritonRoute optimizes the routing topology between them:

**Algorithm — Optimization of Routing Topology**
1. For all pairs of APCs (i, j), compute `cost[i,j] = dist(APC_i, APC_j)`
2. Build a minimum spanning tree `T` over the APCs using these pairwise costs: `T ← MST(APCs, COSTs)`
3. Return the edges `e[i,j] ∈ T` as the chosen routing topology

This MST-based topology becomes the skeleton that the detailed maze-search/track-assignment engine then realizes with legal geometry, minimizing total connection cost (wirelength/vias) between access points.

---

## Lab: Power Distribution Network (PDN) Construction & Routing

### Contents
1. Lab: Steps to Build the Power Distribution Network
2. Lab: From Power Straps to Standard-Cell Power Rails
3. Lab: Routing



---

## Conclusion

