# Physical Design Implementation: Mapping Theory to Execution Scripts

This report bridges the theoretical concepts of Physical Design—starting from Static Timing Analysis and Multi-Mode Multi-Corner (MMMC) setup, through Placement, Clock Tree Synthesis (CTS), and Routing—directly to the provided execution scripts. 

The flow utilizes Cadence Genus for synthesis and Cadence Innovus for physical implementation.

---
# Logic Synthesis Flow: Script Mapping Overview

This document provides a concise mapping of the Cadence Genus logic synthesis script to the theoretical stages of physical design [cite: 17].

## 1. Setup and Elaboration
Before optimization, the tool loads physical and timing libraries and translates the raw RTL into a basic Boolean structure [cite: 17].

```tcl
read_libs typical.lib
read_physical -lef { ../lef/all.lef}
set_db / .cap_table_file ../captable/t018s6mlv.capTbl

read_hdl {cic.v}
elaborate $DESIGN
```
* **Explanation:** `read_libs` and `read_physical` load the timing (.lib) and geometric (.lef) boundaries for the standard cells [cite: 17]. `read_hdl` and `elaborate` check the Verilog code for syntax errors and convert it into a technology-independent logical structure [cite: 17].

## 2. Constraints and Power Optimization
The tool requires target parameters to guide the synthesis process, including clock speeds and power-saving directives [cite: 17].

```tcl
set_db / .lp_insert_clock_gating true 
create_clock -period 100  -name clk  [get_ports clk]
set_false_path -from [get_ports {rst}] -to [all_registers]
```
* **Explanation:** The `lp_insert_clock_gating` command enables automatic insertion of clock gating cells to save dynamic power [cite: 17]. `create_clock` sets the fundamental performance target (100 ns period), and `set_false_path` tells the timing engine to ignore asynchronous reset paths [cite: 17].

## 3. The Synthesis Execution
Genus executes synthesis in three distinct phases, controlled by effort levels [cite: 17].

```tcl
syn_generic
syn_map
syn_opt
```
* **Explanation:** 
  * `syn_generic`: Performs initial logic minimization without tying the logic to specific standard cells [cite: 17].
  * `syn_map`: Maps the generic logic into the actual standard cells provided in the `.lib` files [cite: 17].
  * `syn_opt`: Iteratively resizes gates and inserts buffers to resolve any remaining timing or design rule violations [cite: 17].

## 4. Handoff and Verification
Once synthesis is complete, the final gate-level netlist is exported for physical implementation and verified [cite: 17].

```tcl
write_hdl  > ${_OUTPUTS_PATH}/${DESIGN}_m.v
write_sdc > ${_OUTPUTS_PATH}/${DESIGN}_m.sdc
write_do_lec -golden_design fv_map -revised_design ${_OUTPUTS_PATH}/${DESIGN}_m.v ...
```
* **Explanation:** `write_hdl` and `write_sdc` export the optimized netlist and the timing constraints needed for Place & Route tools (like Innovus) [cite: 17]. `write_do_lec` generates scripts for Logical Equivalence Checking (LEC) to mathematically prove the final netlist matches the original RTL functionality [cite: 17].


## 1. Multi-Mode Multi-Corner (MMMC) Setup & STA Fundamentals

Before physical design begins, the tool must understand the various Process, Voltage, and Temperature (PVT) conditions the chip will face. This is handled by the MMMC script, which defines the analysis views used by the Static Timing Analysis (STA) engine during placement and routing.

### 1.1 Library Sets and Timing Conditions
The script imports Liberty (`.lib`) files to define the timing behavior of standard cells across different PVT corners.
```tcl
create_library_set -name nom -timing [list ../lib/typical.lib]
create_library_set -name fast -timing [list ../lib/fast.lib]

create_timing_condition -name nom_timing -library_set nom
create_timing_condition -name best_timing -library_set fast
```
* **Explanation:** `create_library_set` groups the `.lib` files, representing nominal and fast silicon profiles. `create_timing_condition` wraps these libraries into a functional timing profile.

### 1.2 RC Corners
Interconnect delays are calculated using extraction rules defined in the RC corners.
```tcl
create_rc_corner -name rc_nom \
   -cap_table ../captable/t018s6mlv.capTbl \
   -qrc_tech ../qrc/t018s6mm.tch \
   -T 25
```
* **Explanation:** This snippet pairs the capacitance table (`.capTbl`) and QRC technology file (`.tch`) to calculate parasitic RC values at an operating temperature of 25°C.

### 1.3 Delay Corners and Analysis Views
Delay corners combine timing conditions (gate delays) with RC corners (wire delays). Analysis views pair these delay corners with the constraints defined in the SDC file.
```tcl
create_constraint_mode -name func -sdc_files [list ./output_Feb06-12:16:56/cic_m.sdc]

create_delay_corner -name nom_nom -timing_condition nom_timing -rc_corner rc_nom
create_delay_corner -name fast_min -timing_condition best_timing -rc_corner rc_nom

create_analysis_view -name func_nom_nom -constraint_mode func -delay_corner nom_nom
create_analysis_view -name func_fast_min -constraint_mode func -delay_corner fast_min

set_analysis_view -setup [list func_nom_nom] -hold [list func_nom_nom func_fast_min]
```
* **Explanation:** The `set_analysis_view` command is critical. It instructs the STA engine to check Max Delay (Setup) constraints against the nominal corner, but to aggressively check Min Delay (Hold) constraints against *both* the nominal and fast corners. This guarantees the chip will not suffer from hold violations if the silicon is manufactured at the fast process limit.

---

## 2. Design Import & Initialization

The physical design flow begins by importing the synthesized netlist alongside physical libraries and power definitions.

### 2.1 Importing Files and Initializing
```tcl
read_mmmc ../script_innovus/mmmc.tcl
read_physical -lef {../lef/all.lef}
read_netlist -top cic "./output_Feb06-12:16:56/cic_m.v"

set_db init_power_nets VDD
set_db init_ground_nets VSS
init_design
```
* **Explanation:** The initialization script reads the MMMC configuration, the physical LEF files containing cell geometries and routing track definitions, and the gate-level Verilog netlist generated by the Genus synthesis tool. `init_design` compiles this data into the Innovus database, officially transitioning the design into the physical realm.

---

## 3. Placement

Placement assigns exact physical coordinates to all standard cells. The provided script executes global placement, optimization, and detailed legalization.

### 3.1 Global Placement and Optimization
```tcl
set_db timing_analysis_type ocv
set_db place_global_place_io_pins true
place_opt_design
```
* **Explanation:** `timing_analysis_type ocv` enables On-Chip Variation modeling to account for local process variations. The `place_opt_design` command represents the modern concurrent approach to placement: it performs global analytical placement while simultaneously executing timing-driven and congestion-driven optimizations (resizing, buffering, moving cells).

### 3.2 Legalization and Tie-Offs
```tcl
set_db add_tieoffs_cells "TIEHI TIELO"
add_tieoffs
place_detail
```
* **Explanation:** Unused gate inputs must be tied to logic 1 or 0 to prevent floating gates. `add_tieoffs` inserts physical TIEHI and TIELO cells into the netlist. `place_detail` then legally snaps all cells to the floorplan's site rows and resolves any overlapping instances.

### 3.3 Pre-CTS Timing Analysis
```tcl
route_early_global
extract_rc
time_design -pre_cts -path_report -drv_report -slack_report
```
* **Explanation:** Before building the clock tree, the tool runs a fast global route (`route_early_global`) to accurately estimate wire RC parasitics (`extract_rc`). `time_design -pre_cts` ensures the design meets setup constraints under ideal clock assumptions before moving forward.

---

## 4. Clock Tree Synthesis (CTS)

CTS transforms the ideal clock network into a physical, buffered tree to minimize skew and insertion delay.

### 4.1 Non-Default Routing Rules (NDR) for Clocks
Clock signals switch at high frequencies and are highly susceptible to resistance and crosstalk. 
```tcl
create_route_rule -width {Metal1 0.240 Metal2 0.280 Metal3 0.280 Metal4 0.280 Metal5 0.280 Metal6 0.440} \
                  -spacing {Metal1 0.280 Metal2 0.350 Metal3 0.320 Metal4 0.320 Metal5 0.320 Metal6 0.480} -name 2w2s

create_route_type -name clkroute -route_rule 2w2s -bottom_preferred_layer Metal2 -top_preferred_layer Metal5

set_db cts_route_type_trunk clkroute
set_db cts_route_type_leaf clkroute
```
* **Explanation:** To protect the clock, a Non-Default Rule (NDR) named `2w2s` (double width, double spacing) is created. This rule is applied as a routing type (`clkroute`) restricted between Metal 2 and Metal 5. Finally, `set_db` applies this robust routing type to both the primary trunks and the leaf nodes of the clock tree.

### 4.2 Clock Concurrent Optimization (CCOpt)
```tcl
set_db cts_inverter_cells {CLKINVXL CLKINVX1 ... CLKINVX20}
set_db cts_buffer_cells {CLKBUFXL CLKBUFX1 ... CLKBUFX20}

create_clock_tree_spec -out_file ccopt.spec
source ccopt.spec
ccopt_design
```
* **Explanation:** The tool is restricted to using specific, balanced clock buffers and inverters defined via `set_db`. The `ccopt_design` command executes Clock Concurrent Optimization. Instead of blindly aiming for zero skew, CCOpt integrates CTS with timing optimization, leveraging "useful skew" to borrow time across sequential stages and meet setup/hold constraints while using fewer buffers.

---

## 5. Routing & Post-Route Optimization

Routing connects all placed cells and clock trees following design rules.

### 5.1 Detailed Routing and DFM Setup
```tcl
set_db route_design_detail_fix_antenna true
set_db route_design_antenna_diode_insertion 1
set_db route_design_antenna_cell_name ANTENNA

set_db route_design_with_timing_driven true
set_db route_design_with_si_driven true

route_design -global_detail
```
* **Explanation:** The script enables Design for Manufacturing (DFM) protections by activating antenna rule fixing and assigning the `ANTENNA` diode cell to prevent charge accumulation during plasma etching. The `route_design` command executes both global and detailed maze routing while adhering to Signal Integrity (SI) limits (e.g., crosstalk mitigation via wire spreading) and timing constraints.

### 5.2 Post-Route Signoff Optimization
```tcl
set_db extract_rc_engine post_route
time_design -post_route
time_design -post_route -hold
opt_design -post_route -setup -hold
write_db ./DBS/final.db
```
* **Explanation:** After physical routing is complete, parasitic RC extraction is switched to high-accuracy `post_route` mode. Final timing is assessed for both setup and hold parameters. `opt_design -post_route` attempts to fix any remaining DRC, setup, or hold violations by resizing drivers or inserting delay buffers, concluding the physical design flow before the final database is saved.
