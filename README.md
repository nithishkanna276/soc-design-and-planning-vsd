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

```bash
ngspice sky130_inv.spice
```
```ngspice
plot y vs time a
```
#### Rise transition time calculation

Rise transition time = Time taken for output to rise to 80% - Time taken for output to rise to 20%

20% of output = 660 mV

80% of output = 2.64 V

#### Fall transition time calculation

Fall transition time = Time taken for output to fall to 20% - Time taken for output to fall to 80%

20% of output = 660 mV

80% of output = 2.64 V   

Incorrectly implemented poly.9 simple rule correction

Incorrectly implemented poly.9 rule no drc violation even though spacing < 0.48u

Find problem in the DRC section of the old magic tech file for the skywater process and fix them.


##Day 4 — Pre-Layout Timing & Clock Tree Synthesis (CTS)

#### Abstracting the Layout with LEF

The placement engine doesn't need to see the internal transistor layout of a cell; it only needs the physical boundary, metal layer routing data, and pin locations. This is abstracted into a LEF file. For seamless placement, the custom cell must be grid-aligned with its ports landing exactly on intersection points of the horizontal and vertical tracks.

#### Lab Execution — Custom Cell Integration & OpenSTA

#### Referencing the tracks info for grid alignment:

#### Applying grid spacing in Magic:

```tcl
help grid
grid 0.46um 0.34um 0.23um 0.17um
```
#### Writing the custom cell LEF:

Moving LEF and lib files to the src directory of the design:

Configuring config.tcl to point to the new libraries:

```tcl
set ::env(LIB_SYNTH)      "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)     [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

Validating abutment and successful placement of the custom inverter cell in Magic:

Expanding internal views:

```tcl
expand
```

#### Pre-CTS OpenSTA Analysis:
Setting up custom SDC constraints (my_base.sdc):

```bash
sta pre_sta.conf
```

#### Clock Tree Synthesis Execution:

```tcl
run_cts
```

#### Generating expanded timing reports via OpenROAD database:

```tcl
openroad
read_lef /OpenLane/designs/picorv32a/runs/24-03_10-03/tmp/merged.nom.lef
read_def /OpenLane/designs/picorv32a/runs/24-03_10-03/results/cts/picorv32a.def
write_db pico_cts.db
read_db pico_cts.db
read_verilog /OpenLane/designs/picorv32a/runs/24-03_10-03/results/synthesis/picorv32a.v
read_liberty $::env(LIB_SYNTH_COMPLETE)
link_design picorv32a
read_sdc /OpenLane/designs/picorv32a/src/my_base.sdc
set_propagated_clock [all_clocks]
report_checks -path_delay min_max -fields {slew trans net cap input_pins} -format full_clock_expanded -digits 4
exit
```

#### Get syntax for grid command
help grid

#### Set grid values accordingly
grid 0.46um 0.34um 0.23um 0.17um

Generate lef from the layout.

Command for tkcon window to write lef

Copy the newly generated lef and associated required lib files to 'picorv32a' design 'src' directory.

Commands to copy necessary files to 'picorv32a' design 'src' directory

#### Editing `config.tcl` to Include Custom Cell

```tcl
set ::env(LIB_SYNTH)      "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)     [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

#### Command to view internal connectivity layers
expand

#### Running OpenSTA (Pre-CTS Timing)
Newly created pre_sta.conf for STA analysis in openlane directory

Newly created my_base.sdc for STA analysis in openlane/designs/picorv32a/src directory based on the file openlane/scripts/base.sd

```bash
sta pre_sta.conf
```

#### Running CTS

```tcl
run_cts
```

#### Command to run OpenROAD tool
openroad

Reading lef file:
read_lef /OpenLane/designs/picorv32a/runs/24-03_10-03/tmp/merged.nom.lef

Reading def file:
read_def /OpenLane/designs/picorv32a/runs/24-03_10-03/results/cts/picorv32a.def

Creating an OpenROAD database to work with:
write_db pico_cts.db

Loading the created database in OpenROAD:
read_db pico_cts.db

Read netlist post CTS:
read_verilog /OpenLane/designs/picorv32a/runs/24-03_10-03/results/synthesis/picorv32a.v

Read library for design:
read_liberty $::env(LIB_SYNTH_COMPLETE)

Link design and library:
link_design picorv32a

Read in the custom sdc we created:
read_sdc /OpenLane/designs/picorv32a/src/my_base.sdc

Setting all cloks as propagated clocks:
set_propagated_clock [all_clocks]

Check syntax of 'report_checks' command:
help report_checks

Generating custom timing report:
report_checks -path_delay min_max -fields {slew trans net cap input_pins} -format full_clock_expanded -digits 4

Exit to OpenLANE flow
exit
---


##Day 5 — Post-Route Processing & Routing Stages

#### Global Routing vs. Detailed Routing
#### Routing logic paths while adhering to complex fabrication rules is accomplished in two major stages:

1. Global Routing (FastRoute): Calculates approximate pathways across a grid of routing cells, providing rough topologies and solving broad congestion.

2. Detailed Routing (TritonRoute): Computes the exact tracks, physical wiring, and via drops necessary to connect the logic without violating node DRC constraints (like spacing, minimum metal area, or antenna rules).

#### Lab Execution — Final Power Network and Routing

#### Power Grid Generation:

```tcl
gen_pdn
```
Executing Detailed Route:

```tcl
run_routing
```
Inspecting the PDN and route output in Magic:

##### Development Environment & Toolchain

The OpenLANE pipeline utilizes a heavy Linux container stack. This flow and the respective labs were independently set up and processed natively on local hardware architectures, specifically validated on an Acer Aspire A514-54 and a Lenovo Flex 2-14 to test tooling compatibility across different host hardware configurations.

## Tools & Environment

| Tool | Purpose |
|---|---|
| **OpenLANE** | RTL-to-GDSII automation flow |
| **Yosys** | RTL synthesis |
| **OpenROAD** | Floorplan, Placement, CTS, Routing |
| **Magic** | Layout editor, DRC, LVS |
| **OpenSTA** | Static Timing Analysis |
| **ngspice** | SPICE simulation |
| **TritonRoute** | Detailed routing |
| **Netgen** | LVS (Layout vs Schematic) |
| **Sky130 PDK** | SkyWater 130nm open-source PDK |

---

## Key Learnings

- Understood how a chip moves from an idea (RTL) to a manufacturable file (GDSII) using a fully open-source toolchain
- Got hands-on with floorplanning, placement, CTS, and routing for the `picorv32a` RISC-V core
- Learned how to characterise custom standard cells and integrate them into an existing flow
- Gained practical experience with STA concepts — setup/hold slack, OCV, CRPR — using OpenSTA
- Understood how parasitics from post-route SPEF extraction affect timing sign-off

---

## Acknowledgements

A huge thank you to *Kunal Ghosh* (Co-founder, VSD Corp. Pvt. Ltd.) and *Nickson P Jose* (Physical Design Engineer, Intel) for putting together such a well-structured and genuinely practical workshop. Running a real CPU from RTL to GDSII using nothing but open-source tools is something I didn’t expect to be possible — and yet here we are.
- **Kunal Ghosh** — Co-founder, VSD (VLSI System Design)
- **Nickson Jose** — for the `vsdstdcelldesign` repository used in Day 3 labs
- **NASSCOM** — for facilitating this workshop program

---

## References

- [VSD SoC Design Workshop](https://www.vlsisystemdesign.com/)
- [OpenLANE GitHub](https://github.com/The-OpenROAD-Project/OpenLane)
- [SkyWater Sky130 PDK](https://github.com/google/skywater-pdk)
- [vsdstdcelldesign](https://github.com/nickson-jose/vsdstdcelldesign)
Documented by Nayana | VTU Electronics & Communication Engineering | VLSI & Physical Design
