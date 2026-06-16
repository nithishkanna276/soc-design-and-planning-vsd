# soc-design-and-planning-vsd
"Code to Chip: An open-source RTL-to-GDSII journey with OpenLANE &amp; Sky130 PDK | VSD SoC Design and Planning Workshop"
# Digital VLSI SoC Design: Complete RTL to GDSII Flow
>A deep dive into the complete RTL-to-GDSII pipeline for digital VLSI System-on-Chip (SoC) design, guided by the VSD (VLSI System Design) and NASSCOM workshop.
This repository serves as a thorough log of lab executions, technical concepts, and standard cell characterizations over the course of the program.

## Day 1 — The Open-Source EDA Revolution & Sky130 PDK

#### Deconstructing the Chip

When examining a standard embedded system, the visible black square on the PCB is merely the package. This housing protects the delicate silicon and facilitates connectivity. Inside, the actual silicon die interfaces with the external pins through microscopic wire bonds connecting to the chip's pads.

#### The Die: Pads and the Core

Focusing on the bare silicon, the pads form a perimeter ring routing signals in and out of the chip. Inside this boundary is the core, the heart of the chip where the intricate digital logic arrays reside. The combination of the core and the pad ring constitutes the die.

- Foundry: The physical fabrication plant where the silicon wafers are processed.

- Foundry IPs: Highly specialized, process-node-dependent blocks (e.g., SRAM, ADC, PLL).

- Macros: Reusable, synthesized digital logic components.

#### Bridging Software and Silicon

A standard C program traverses several abstraction layers before physical execution:

1. The compiler translates the C code into the target Instruction Set Architecture (ISA) (e.g., RISC-V).

2. The assembler translates these instructions into binary machine code.

3. The hardware requires an RTL (Register Transfer Level) design that understands this specific ISA.

4. The RTL undergoes Synthesis and the Place & Route (PnR) flow to become a physical silicon layout.

#### The Significance of Open-Source EDA

Historically, custom ASIC design was blocked by expensive proprietary tools and strict NDAs for Process Design Kits (PDKs). True open-source chip design requires:

1. Open RTL (e.g., RISC-V cores).

2. Open EDA Tools (for synthesis, routing, and simulation).

3. Open PDKs (foundry-specific data and standard cells).

Google and SkyWater Technology's release of the Sky130 PDK in 2020 broke the final barrier, granting global access to a manufacturable 130nm process node.

The OpenLANE Flow
OpenLANE is an automated script framework that orchestrates several open-source EDA tools to take an RTL netlist to a manufacturable GDSII file.

| Stage | Tool(s) Used |
|---|---|
| Synthesis | "Yosys, ABC" |
| Floorplanning & PDN | OpenROAD |
| Placement | OpenROAD |
| Clock Tree Synthesis (CTS) | TritonCTS |
| Routing | "FastRoute, TritonRoute" |
| Parasitic Extraction | OpenRCX |
| Timing Analysis | OpenSTA |
| Layout & DRC/LVS | "Magic, Netgen, KLayout" |

#### Lab Execution — Synthesizing picorv32a

Initializing the Environment:
Working within the OpenLANE directory, the tool is launched interactively to allow step-by-step execution.
```bash
cd /home/vscode/Desktop/OpenLane
make mount
./flow.tcl -interactive
package require openlane 1.0.2
```

#### Design Setup:
Merging the LEF files and preparing the working directories.

```tcl
prep -design picorv32a
```

#### Synthesis:

```tcl
run_synthesis
```

```
Flop Ratio Calculation:
Flop Ratio = (Total D Flip-Flops) / (Total Cells) = 1613 / 15762 ≈ 10.23%
```

## Day 2 — Floorplanning and Standard Cells

#### Macro Placement and Chip Geometry

Floorplanning defines the physical constraints of the design:

- Utilization Factor: The area of the active netlist relative to the total core area. Starting around 0.5 (50%) leaves vital space for clock buffers and complex routing.

- Aspect Ratio: The core's height divided by its width. A ratio of 1.0 yields a perfect square.

#### Managing Pre-placed Cells and Decap Injection

Large blocks like SRAMs or analog IP are established as pre-placed cells. They are dropped into the floorplan manually before auto-placement. To stabilize their power supply during high-frequency switching, decoupling capacitors are packed around them to provide localized charge buffers.

#### Robust Power Distribution

To prevent severe IR drop across the silicon, a multi-layered grid of power (VDD) and ground (VSS) rails is constructed. This grid ensures every standard cell has immediate access to power, preventing logical faults and electromigration.

### Lab Execution — Floorplan and Placement

#### Generating Floorplan:

```tcl
run_floorplan
```
#### Reviewing via Magic VLSI:

```bash
cd results/floorplan/
magic -T /home/vsduser/Desktop/OpenLane/designs/picorv32a/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.nom.lef \
      def read picorv32a.def &
```

#### Running Standard Cell Placement:

```tcl
run_floorplan
```
After this completes, we can inspect the DEF file that was generated:

```bash
cd results/floorplan/
less picorv32a.def
```

#### Viewing the Floorplan in Magic

```bash
magic -T /home/vsduser/Desktop/OpenLane/designs/picorv32a/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.nom.lef \
      def read picorv32a.def &
```
#### Running Placement

```tcl
run_placement
```

## Day 3 — Cell Characterization (Magic & ngspice)

#### SPICE Deck Construction

Standard cell characterization relies on highly accurate SPICE simulations. A deck specifies the PMOS/NMOS geometries, input stimulus, parasitic loads, and power supply. We specifically evaluate:

- Rise/Fall Times: Measured between the 20% and 80% voltage thresholds.

- Propagation Delay: Measured at the 50% threshold from input to output.

#### Lab Execution — Custom Inverter Layout and Extraction
Pulling the Layout:

```bash
git clone https://github.com/nickson-jose/vsdstdcelldesign.git
magic -T sky130A.tech sky130_inv.mag &
```
#### Extracting Netlist in Magic tkcon:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

#### Prepping the file by observing grid parameters:

#### Transient Analysis with ngspice:
                    
