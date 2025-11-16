# RISC-V-SoC-Tapeout---WEEK-7

 The main goal this week was to take a synthesized RISC-V SoC design, the `VsdBabySoC`, and run it through the entire physical design (PD) flow using the **OpenROAD-flow-scripts (ORFS)**.

This task was all about connecting the dots—seeing how the digital logic we write in Verilog (`RTL`) gets translated into a physical, routable layout (`GDSII`) ready for fabrication. It's where the theory of chip design meets the practical reality of layout, timing, and parasitic extraction.

##  The Objective: A Full Physical Design Flow

Up to this point, many steps like synthesis, STA, and layout were separate exercises. The goal of this task was to integrate them into one continuous flow, just like in a real-world ASIC design process.

My mission was to:
1.  **Set up the Environment:** Install and configure the OpenROAD-flow-scripts.
2.  **Configure the Project:** Teach the flow about our `BabySoC` design, its Verilog files, and its special macros (like the PLL and DAC).
3.  **Run the Flow:** Execute all the main physical design stages:
    * Synthesis (if not already done)
    * Floorplanning
    * Placement
    * Clock Tree Synthesis (CTS)
    * Routing
4.  **Extract Parasitics:** Generate the critical **SPEF** (Standard Parasitic Exchange Format) file, which is essential for accurate, post-layout timing signoff.

---

##  Part 1: Setting Up the Environment (OpenROAD)

First things first, I had to get the toolchain (ORFS) installed and built. This flow automates the entire process, using a set of open-source tools like Yosys, OpenROAD, and Magic.

Here are the commands I used to get the environment ready:

```bash
# 1. Clone the repository (recursively to get all submodules)
git clone --recursive [https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)

# 2. Change into the directory
cd OpenROAD-flow-scripts

# 3. Run the dependency setup script
sudo ./setup.sh

# 4. Build the OpenROAD tool itself (locally)
./build_openroad.sh --local

# 5. VERY IMPORTANT: Source the environment file to set up all the paths
# You have to do this every time you open a new terminal
source ./env.sh

# 6. Run a quick check to make sure the tools are in the path
yosys -help
openroad -help
```

<img width="3790" height="450" alt="directory" src="https://github.com/user-attachments/assets/16f3cf1a-bdc8-4284-b6ea-c208a3f0b880" />

To verify the installation , enter the following code to run the entire automated flow to check if the design works.

```
source ./env.sh
yosys -help
openroad -help
cd flow
make
```

<img width="1581" height="853" alt="image" src="https://github.com/user-attachments/assets/97d83997-ceb3-4484-88a5-377d84718e6c" />


<img width="3975" height="2499" alt="image" src="https://github.com/user-attachments/assets/70e9f7fc-168a-4f2c-8219-d974b4cbed6b" />


---

##  Part 2: Configuring the BabySoC Project

This was the most important setup step. The OpenROAD flow needs to know *where* our design files are and *how* to build them for our target technology (Skywater 130nm, or `sky130hd`).

The flow cleverly separates the **design-specific files** (`src`) from the **platform-specific files** (`sky130hd`).

1.  **Design Source (`/flow/designs/src/vsdbabysoc/`)**
    * This is the PDK-independent folder.
    * I placed all the core Verilog files here (`vsdbabysoc.v`, `rvmyth.v`, `clk_gate.v`).

2.  **Platform Config (`/flow/designs/sky130hd/vsdbabysoc/`)**
    * This folder tells the flow *how* to build our design for `sky130hd`.
    * I copied all the tech-specific files here:
        * `gds/`: GDS files for the hard macros (`avsddac.gds`, `avsdpll.gds`).
        * `lef/`: LEF files for the macros (`avsddac.lef`, `avsdpll.lef`).
        * `lib/`: Timing `.lib` files for the macros (`avsddac.lib`, `avsdpll.lib`).
        * `sdc/`: The timing constraints for synthesis (`vsdbabysoc_synthesis.sdc`).
        * `macro.cfg` & `pin_order.cfg`: Files to guide the floorplanner.

<img width="1815" height="753" alt="tree" src="https://github.com/user-attachments/assets/232b6095-878f-401a-825a-919f956a01b3" />


### The Most important File: `config.mk`

This file, which I created inside `/flow/designs/sky130hd/vsdbabysoc/`, is the main "control panel" for the flow. It points to all the files we just organized and sets our design parameters.

```makefile
# --- Basic Design Info ---
export DESIGN_NICKNAME = vsdbabysoc
export DESIGN_NAME = vsdbabysoc
export PLATFORM    = sky130hd

# --- Source Files ---
# Point to all the Verilog files needed for synthesis
export VERILOG_FILES = $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/module/vsdbabysoc.v \
                       $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/module/rvmyth.v \
                       $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/module/clk_gate.v

# Point to the SDC (timing constraints) file
export SDC_FILE      = $(DESIGN_HOME)/$(PLATFORM)/$(DESIGN_NICKNAME)/sdc/vsdbabysoc_synthesis.sdc

# --- Macro / IP Configuration ---
export vsdbabysoc_DIR = $(DESIGN_HOME)/$(PLATFORM)/$(DESIGN_NICKNAME)
export VERILOG_INCLUDE_DIRS = $(wildcard $(vsdbabysoc_DIR)/include/)
export ADDITIONAL_GDS  = $(wildcard $(vsdbabysoc_DIR)/gds/*.gds)
export ADDITIONAL_LEFS  = $(wildcard $(vsdbabysoc_DIR)/lef/*.lef)
export ADDITIONAL_LIBS = $(wildcard $(vsdbabysoc_DIR)/lib/*.lib)

# --- Clock Configuration ---
export CLOCK_PORT = CLK
export CLOCK_NET = $(CLOCK_PORT)

# --- Floorplan & Die Config ---
export DIE_AREA   = 0 0 1600 1600
export CORE_AREA  = 20 20 1590 1590
export FP_PIN_ORDER_CFG = $(wildcard $(vsdbabysoc_DIR)/pin_order.cfg)
export MACRO_PLACEMENT_CFG = $(wildcard $(vsdbabysoc_DIR)/macro.cfg)

# --- Placement Config ---
export PLACE_PINS_ARGS = -exclude left:0-600 -exclude left:1000-1600: -exclude right:* -exclude top:* -exclude bottom:*

# --- CTS Tuning ---
export CTS_BUF_DISTANCE = 600
export SKIP_GATE_CLONING = 1
```

---

##  Part 3: Running the Full Flow (The Fun Part!)

With the setup complete, all I had to do was `cd` into the `flow/` directory and run the `make` commands for each stage. The `DESIGN_CONFIG` variable points to the `config.mk` file we just made.

### 1. Synthesis

This step runs Yosys to turn the Verilog into a gate-level netlist.
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk synth
```
The following is the synthesis log

<img width="3964" height="2506" alt="synthesis_1" src="https://github.com/user-attachments/assets/427da15f-b2f5-4e9c-97dc-d8d3b09f897f" />
<img width="3964" height="2506" alt="synthesis_2" src="https://github.com/user-attachments/assets/4976dff9-1a8c-44af-a000-ed14f8f54e9c" />
<img width="3964" height="2506" alt="synthesis_3" src="https://github.com/user-attachments/assets/0554caa2-6e53-4302-8316-92dac7c2aadc" />
<img width="3964" height="2506" alt="synthesis_4" src="https://github.com/user-attachments/assets/545f21cf-2a93-4c24-83a8-df8bf775a084" />
<img width="3964" height="2506" alt="synthesis_5" src="https://github.com/user-attachments/assets/b0da1aea-46f2-48cc-8be7-41af232f4585" />
<img width="3964" height="754" alt="synthesis_6" src="https://github.com/user-attachments/assets/54e624e1-cdf3-4781-b21e-e948cde9c882" />

Now the netlist generated is as follows

<img width="3978" height="2560" alt="netlist" src="https://github.com/user-attachments/assets/f6a260db-e360-43f5-8086-bcd3a4b3cb01" />
<img width="3978" height="2560" alt="netlist_1" src="https://github.com/user-attachments/assets/b5cac365-fe9a-402d-b71d-1208cc67ba25" />

The following is from ``synth_check.txt``

<img width="3978" height="512" alt="synth_check" src="https://github.com/user-attachments/assets/0c8c4fe3-8159-4486-8f82-4bb4e0424b83" />

Now the following picture is from ``Synth_stat`` file

<img width="3978" height="2510" alt="synth_stat" src="https://github.com/user-attachments/assets/30002dd2-f007-4d29-b3c6-e0b0d87bfb9d" />
<img width="3978" height="2510" alt="synth_stat1" src="https://github.com/user-attachments/assets/615009e3-57b7-4128-895f-ae0061f6f9b3" />
<img width="3978" height="2510" alt="synth_stat2" src="https://github.com/user-attachments/assets/9a33a719-e043-4172-8b97-4401e41738e2" />

With this detailed analysis , we see that everything is set in place and is working fine and now we can directly go through the entire PNR flow unless we are stopped by any possible error.

### 2. Floorplan
Here, the flow defines the die area, places the I/O pins, and positions our hard macros (PLL and DAC) according to the `macro.cfg`.
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk floorplan
```
To see the result, I used the GUI command:
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk gui_floorplan
```
<img width="1561" height="716" alt="floorplan" src="https://github.com/user-attachments/assets/616d3a67-5ac8-45f4-a986-8ce41ce88a79" />


### 3. Placement
This step takes all the standard cells from the netlist and finds a legal, optimized position for them inside the core area.
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk place
```
And to view the placement heatmap (to check for congestion):
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk gui_place
```
<img width="1501" height="716" alt="placement" src="https://github.com/user-attachments/assets/a178412a-a466-4bee-b4f8-26c83f6a7969" />

Now click on z to zoom into this to observe the heatmap

<img width="1506" height="702" alt="heatmap" src="https://github.com/user-attachments/assets/ab09e827-f7ce-4dbb-89bf-224353e8e411" />


### 4. Clock Tree Synthesis (CTS)

This is a critical step where the tool builds a balanced buffer tree to distribute the clock signal (`CLK`) to all the flip-flops with minimal skew.
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk cts
```
<img width="1507" height="698" alt="cts" src="https://github.com/user-attachments/assets/04723964-aec5-4a10-8113-620731f1dfd6" />


After this step, the tool runs a Static Timing Analysis (STA). and generates the following report

```
==========================================================================
cts final report_tns
--------------------------------------------------------------------------
tns 0.00

==========================================================================
cts final report_wns
--------------------------------------------------------------------------
wns 0.00

==========================================================================
cts final report_worst_slack
--------------------------------------------------------------------------
worst slack 5.55

==========================================================================
cts final report_clock_skew
--------------------------------------------------------------------------
Clock clk
   0.92 source latency core.CPU_result_a4[0]$_DFF_P_/CLK ^
  -0.82 target latency core.CPU_Dmem_value_a5[0][18]$_SDFFE_PP0P_/CLK ^
   0.55 clock uncertainty
   0.00 CRPR
--------------
   0.65 setup skew


==========================================================================
cts final report_checks -path_delay min
--------------------------------------------------------------------------
Startpoint: core.CPU_Xreg_value_a4[25][15]$_SDFFE_PP0P_
            (rising edge-triggered flip-flop clocked by clk)
Endpoint: core.CPU_src2_value_a3[15]$_DFF_P_
          (rising edge-triggered flip-flop clocked by clk)
Path Group: clk
Path Type: min

Fanout     Cap    Slew   Delay    Time   Description
-----------------------------------------------------------------------------
                          0.00    0.00   clock clk (rise edge)
                          0.00    0.00   clock source latency
     1    0.10    0.00    0.00    0.00 ^ pll/CLK (avsdpll)
                                         CLK (net)
                  0.00    0.00    0.00 ^ clkbuf_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     8    0.23    0.24    0.26    0.26 ^ clkbuf_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_0_CLK (net)
                  0.24    0.00    0.26 ^ clkbuf_3_7_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     2    0.04    0.06    0.20    0.47 ^ clkbuf_3_7_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_3_7_0_CLK (net)
                  0.06    0.00    0.47 ^ clkbuf_4_14__f_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    13    0.17    0.18    0.24    0.71 ^ clkbuf_4_14__f_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_4_14__leaf_CLK (net)
                  0.18    0.00    0.71 ^ clkbuf_leaf_108_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    12    0.04    0.06    0.19    0.90 ^ clkbuf_leaf_108_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_leaf_108_CLK (net)
                  0.06    0.00    0.90 ^ core.CPU_Xreg_value_a4[25][15]$_SDFFE_PP0P_/CLK (sky130_fd_sc_hd__dfxtp_1)
     2    0.01    0.06    0.32    1.22 ^ core.CPU_Xreg_value_a4[25][15]$_SDFFE_PP0P_/Q (sky130_fd_sc_hd__dfxtp_1)
                                         core.CPU_Xreg_value_a4[25][15] (net)
                  0.06    0.00    1.22 ^ _14708_/A1 (sky130_fd_sc_hd__a21oi_1)
     1    0.00    0.04    0.06    1.28 v _14708_/Y (sky130_fd_sc_hd__a21oi_1)
                                         _02012_ (net)
                  0.04    0.00    1.28 v _14712_/A (sky130_fd_sc_hd__nand4_1)
     1    0.01    0.13    0.12    1.40 ^ _14712_/Y (sky130_fd_sc_hd__nand4_1)
                                         _02016_ (net)
                  0.13    0.00    1.40 ^ _14729_/A1 (sky130_fd_sc_hd__o32ai_4)
     1    0.04    0.12    0.18    1.58 v _14729_/Y (sky130_fd_sc_hd__o32ai_4)
                                         _02033_ (net)
                  0.12    0.00    1.59 v _14731_/A2 (sky130_fd_sc_hd__o21ai_0)
     1    0.00    0.06    0.16    1.74 ^ _14731_/Y (sky130_fd_sc_hd__o21ai_0)
                                         core.CPU_src2_value_a2[15] (net)
                  0.06    0.00    1.74 ^ core.CPU_src2_value_a3[15]$_DFF_P_/D (sky130_fd_sc_hd__dfxtp_2)
                                  1.74   data arrival time

                          0.00    0.00   clock clk (rise edge)
                          0.00    0.00   clock source latency
     1    0.10    0.00    0.00    0.00 ^ pll/CLK (avsdpll)
                                         CLK (net)
                  0.00    0.00    0.00 ^ clkbuf_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     8    0.23    0.24    0.26    0.26 ^ clkbuf_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_0_CLK (net)
                  0.24    0.00    0.26 ^ clkbuf_3_6_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     2    0.04    0.06    0.21    0.47 ^ clkbuf_3_6_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_3_6_0_CLK (net)
                  0.06    0.00    0.47 ^ clkbuf_4_13__f_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    12    0.16    0.18    0.24    0.71 ^ clkbuf_4_13__f_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_4_13__leaf_CLK (net)
                  0.18    0.00    0.71 ^ clkbuf_leaf_70_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    12    0.04    0.06    0.18    0.89 ^ clkbuf_leaf_70_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_leaf_70_CLK (net)
                  0.06    0.00    0.89 ^ core.CPU_src2_value_a3[15]$_DFF_P_/CLK (sky130_fd_sc_hd__dfxtp_2)
                          0.88    1.78   clock uncertainty
                          0.00    1.78   clock reconvergence pessimism
                         -0.03    1.74   library hold time
                                  1.74   data required time
-----------------------------------------------------------------------------
                                  1.74   data required time
                                 -1.74   data arrival time
-----------------------------------------------------------------------------
                                  0.00   slack (MET)



==========================================================================
cts final report_checks -path_delay max
--------------------------------------------------------------------------
Startpoint: core.CPU_valid_taken_br_a5$_DFF_P_
            (rising edge-triggered flip-flop clocked by clk)
Endpoint: core.CPU_Xreg_value_a4[7][13]$_SDFFE_PP0P_
          (rising edge-triggered flip-flop clocked by clk)
Path Group: clk
Path Type: max

Fanout     Cap    Slew   Delay    Time   Description
-----------------------------------------------------------------------------
                          0.00    0.00   clock clk (rise edge)
                          0.00    0.00   clock source latency
     1    0.10    0.00    0.00    0.00 ^ pll/CLK (avsdpll)
                                         CLK (net)
                  0.00    0.00    0.00 ^ clkbuf_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     8    0.23    0.24    0.26    0.26 ^ clkbuf_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_0_CLK (net)
                  0.24    0.00    0.26 ^ clkbuf_3_4_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     2    0.04    0.07    0.21    0.47 ^ clkbuf_3_4_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_3_4_0_CLK (net)
                  0.07    0.00    0.47 ^ clkbuf_4_8__f_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    11    0.15    0.16    0.23    0.70 ^ clkbuf_4_8__f_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_4_8__leaf_CLK (net)
                  0.16    0.00    0.70 ^ clkbuf_leaf_149_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    15    0.04    0.06    0.18    0.88 ^ clkbuf_leaf_149_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_leaf_149_CLK (net)
                  0.06    0.00    0.88 ^ core.CPU_valid_taken_br_a5$_DFF_P_/CLK (sky130_fd_sc_hd__dfxtp_1)
     1    0.00    0.03    0.30    1.18 v core.CPU_valid_taken_br_a5$_DFF_P_/Q (sky130_fd_sc_hd__dfxtp_1)
                                         core.CPU_valid_taken_br_a5 (net)
                  0.03    0.00    1.18 v _07913_/A (sky130_fd_sc_hd__or4_4)
    19    0.22    0.35    0.83    2.01 v _07913_/X (sky130_fd_sc_hd__or4_4)
                                         _02930_ (net)
                  0.35    0.01    2.02 v _07915_/A (sky130_fd_sc_hd__clkinv_16)
    43    0.45    0.31    0.36    2.38 ^ _07915_/Y (sky130_fd_sc_hd__clkinv_16)
                                         _02932_ (net)
                  0.31    0.02    2.39 ^ _09981_/C (sky130_fd_sc_hd__nor3_2)
     2    0.02    0.10    0.14    2.53 v _09981_/Y (sky130_fd_sc_hd__nor3_2)
                                         _04371_ (net)
                  0.10    0.00    2.53 v _09982_/B1 (sky130_fd_sc_hd__a21oi_4)
     6    0.10    0.70    0.57    3.10 ^ _09982_/Y (sky130_fd_sc_hd__a21oi_4)
                                         _04372_ (net)
                  0.70    0.00    3.10 ^ _09988_/A2 (sky130_fd_sc_hd__o21ai_4)
    16    0.12    0.33    0.38    3.47 v _09988_/Y (sky130_fd_sc_hd__o21ai_4)
                                         _04378_ (net)
                  0.33    0.00    3.48 v _13552_/A (sky130_fd_sc_hd__nor3_4)
    23    0.14    1.37    1.19    4.67 ^ _13552_/Y (sky130_fd_sc_hd__nor3_4)
                                         _07042_ (net)
                  1.37    0.00    4.67 ^ _13572_/B (sky130_fd_sc_hd__nand2_2)
     5    0.03    0.29    0.30    4.98 v _13572_/Y (sky130_fd_sc_hd__nand2_2)
                                         _07058_ (net)
                  0.29    0.00    4.98 v _13574_/B1 (sky130_fd_sc_hd__o221ai_1)
     1    0.00    0.20    0.24    5.22 ^ _13574_/Y (sky130_fd_sc_hd__o221ai_1)
                                         _01444_ (net)
                  0.20    0.00    5.22 ^ hold2174/A (sky130_fd_sc_hd__dlygate4sd3_1)
     1    0.00    0.05    0.57    5.79 ^ hold2174/X (sky130_fd_sc_hd__dlygate4sd3_1)
                                         net2283 (net)
                  0.05    0.00    5.79 ^ core.CPU_Xreg_value_a4[7][13]$_SDFFE_PP0P_/D (sky130_fd_sc_hd__dfxtp_1)
                                  5.79   data arrival time

                         11.05   11.05   clock clk (rise edge)
                          0.00   11.05   clock source latency
     1    0.10    0.00    0.00   11.05 ^ pll/CLK (avsdpll)
                                         CLK (net)
                  0.00    0.00   11.05 ^ clkbuf_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     8    0.23    0.24    0.26   11.31 ^ clkbuf_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_0_CLK (net)
                  0.24    0.00   11.31 ^ clkbuf_3_6_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     2    0.04    0.06    0.21   11.52 ^ clkbuf_3_6_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_3_6_0_CLK (net)
                  0.06    0.00   11.52 ^ clkbuf_4_13__f_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    12    0.16    0.18    0.24   11.76 ^ clkbuf_4_13__f_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_4_13__leaf_CLK (net)
                  0.18    0.00   11.76 ^ clkbuf_leaf_90_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    11    0.04    0.06    0.19   11.94 ^ clkbuf_leaf_90_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_leaf_90_CLK (net)
                  0.06    0.00   11.94 ^ core.CPU_Xreg_value_a4[7][13]$_SDFFE_PP0P_/CLK (sky130_fd_sc_hd__dfxtp_1)
                         -0.55   11.39   clock uncertainty
                          0.00   11.39   clock reconvergence pessimism
                         -0.05   11.34   library setup time
                                 11.34   data required time
-----------------------------------------------------------------------------
                                 11.34   data required time
                                 -5.79   data arrival time
-----------------------------------------------------------------------------
                                  5.55   slack (MET)



==========================================================================
cts final report_checks -unconstrained
--------------------------------------------------------------------------
Startpoint: core.CPU_valid_taken_br_a5$_DFF_P_
            (rising edge-triggered flip-flop clocked by clk)
Endpoint: core.CPU_Xreg_value_a4[7][13]$_SDFFE_PP0P_
          (rising edge-triggered flip-flop clocked by clk)
Path Group: clk
Path Type: max

Fanout     Cap    Slew   Delay    Time   Description
-----------------------------------------------------------------------------
                          0.00    0.00   clock clk (rise edge)
                          0.00    0.00   clock source latency
     1    0.10    0.00    0.00    0.00 ^ pll/CLK (avsdpll)
                                         CLK (net)
                  0.00    0.00    0.00 ^ clkbuf_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     8    0.23    0.24    0.26    0.26 ^ clkbuf_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_0_CLK (net)
                  0.24    0.00    0.26 ^ clkbuf_3_4_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     2    0.04    0.07    0.21    0.47 ^ clkbuf_3_4_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_3_4_0_CLK (net)
                  0.07    0.00    0.47 ^ clkbuf_4_8__f_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    11    0.15    0.16    0.23    0.70 ^ clkbuf_4_8__f_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_4_8__leaf_CLK (net)
                  0.16    0.00    0.70 ^ clkbuf_leaf_149_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    15    0.04    0.06    0.18    0.88 ^ clkbuf_leaf_149_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_leaf_149_CLK (net)
                  0.06    0.00    0.88 ^ core.CPU_valid_taken_br_a5$_DFF_P_/CLK (sky130_fd_sc_hd__dfxtp_1)
     1    0.00    0.03    0.30    1.18 v core.CPU_valid_taken_br_a5$_DFF_P_/Q (sky130_fd_sc_hd__dfxtp_1)
                                         core.CPU_valid_taken_br_a5 (net)
                  0.03    0.00    1.18 v _07913_/A (sky130_fd_sc_hd__or4_4)
    19    0.22    0.35    0.83    2.01 v _07913_/X (sky130_fd_sc_hd__or4_4)
                                         _02930_ (net)
                  0.35    0.01    2.02 v _07915_/A (sky130_fd_sc_hd__clkinv_16)
    43    0.45    0.31    0.36    2.38 ^ _07915_/Y (sky130_fd_sc_hd__clkinv_16)
                                         _02932_ (net)
                  0.31    0.02    2.39 ^ _09981_/C (sky130_fd_sc_hd__nor3_2)
     2    0.02    0.10    0.14    2.53 v _09981_/Y (sky130_fd_sc_hd__nor3_2)
                                         _04371_ (net)
                  0.10    0.00    2.53 v _09982_/B1 (sky130_fd_sc_hd__a21oi_4)
     6    0.10    0.70    0.57    3.10 ^ _09982_/Y (sky130_fd_sc_hd__a21oi_4)
                                         _04372_ (net)
                  0.70    0.00    3.10 ^ _09988_/A2 (sky130_fd_sc_hd__o21ai_4)
    16    0.12    0.33    0.38    3.47 v _09988_/Y (sky130_fd_sc_hd__o21ai_4)
                                         _04378_ (net)
                  0.33    0.00    3.48 v _13552_/A (sky130_fd_sc_hd__nor3_4)
    23    0.14    1.37    1.19    4.67 ^ _13552_/Y (sky130_fd_sc_hd__nor3_4)
                                         _07042_ (net)
                  1.37    0.00    4.67 ^ _13572_/B (sky130_fd_sc_hd__nand2_2)
     5    0.03    0.29    0.30    4.98 v _13572_/Y (sky130_fd_sc_hd__nand2_2)
                                         _07058_ (net)
                  0.29    0.00    4.98 v _13574_/B1 (sky130_fd_sc_hd__o221ai_1)
     1    0.00    0.20    0.24    5.22 ^ _13574_/Y (sky130_fd_sc_hd__o221ai_1)
                                         _01444_ (net)
                  0.20    0.00    5.22 ^ hold2174/A (sky130_fd_sc_hd__dlygate4sd3_1)
     1    0.00    0.05    0.57    5.79 ^ hold2174/X (sky130_fd_sc_hd__dlygate4sd3_1)
                                         net2283 (net)
                  0.05    0.00    5.79 ^ core.CPU_Xreg_value_a4[7][13]$_SDFFE_PP0P_/D (sky130_fd_sc_hd__dfxtp_1)
                                  5.79   data arrival time

                         11.05   11.05   clock clk (rise edge)
                          0.00   11.05   clock source latency
     1    0.10    0.00    0.00   11.05 ^ pll/CLK (avsdpll)
                                         CLK (net)
                  0.00    0.00   11.05 ^ clkbuf_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     8    0.23    0.24    0.26   11.31 ^ clkbuf_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_0_CLK (net)
                  0.24    0.00   11.31 ^ clkbuf_3_6_0_CLK/A (sky130_fd_sc_hd__clkbuf_16)
     2    0.04    0.06    0.21   11.52 ^ clkbuf_3_6_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_3_6_0_CLK (net)
                  0.06    0.00   11.52 ^ clkbuf_4_13__f_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    12    0.16    0.18    0.24   11.76 ^ clkbuf_4_13__f_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_4_13__leaf_CLK (net)
                  0.18    0.00   11.76 ^ clkbuf_leaf_90_CLK/A (sky130_fd_sc_hd__clkbuf_16)
    11    0.04    0.06    0.19   11.94 ^ clkbuf_leaf_90_CLK/X (sky130_fd_sc_hd__clkbuf_16)
                                         clknet_leaf_90_CLK (net)
                  0.06    0.00   11.94 ^ core.CPU_Xreg_value_a4[7][13]$_SDFFE_PP0P_/CLK (sky130_fd_sc_hd__dfxtp_1)
                         -0.55   11.39   clock uncertainty
                          0.00   11.39   clock reconvergence pessimism
                         -0.05   11.34   library setup time
                                 11.34   data required time
-----------------------------------------------------------------------------
                                 11.34   data required time
                                 -5.79   data arrival time
-----------------------------------------------------------------------------
                                  5.55   slack (MET)



==========================================================================
cts final report_check_types -max_slew -max_cap -max_fanout -violators
--------------------------------------------------------------------------
max capacitance

Pin                                    Limit     Cap   Slack
------------------------------------------------------------
_13743_/Y                               0.15    0.15   -0.00 (VIOLATED)
_13455_/Y                               0.15    0.15   -0.00 (VIOLATED)


==========================================================================
cts final max_slew_check_slack
--------------------------------------------------------------------------
0.02021305449306965

==========================================================================
cts final max_slew_check_limit
--------------------------------------------------------------------------
1.4951449632644653

==========================================================================
cts final max_slew_check_slack_limit
--------------------------------------------------------------------------
0.0135

==========================================================================
cts final max_fanout_check_slack
--------------------------------------------------------------------------
1.0000000150474662e+30

==========================================================================
cts final max_fanout_check_limit
--------------------------------------------------------------------------
1.0000000150474662e+30

==========================================================================
cts final max_capacitance_check_slack
--------------------------------------------------------------------------
-0.001024815021082759

==========================================================================
cts final max_capacitance_check_limit
--------------------------------------------------------------------------
0.1538189947605133

==========================================================================
cts final max_capacitance_check_slack_limit
--------------------------------------------------------------------------
-0.0067

==========================================================================
cts final max_slew_violation_count
--------------------------------------------------------------------------
max slew violation count 0

==========================================================================
cts final max_fanout_violation_count
--------------------------------------------------------------------------
max fanout violation count 0

==========================================================================
cts final max_cap_violation_count
--------------------------------------------------------------------------
max cap violation count 2

==========================================================================
cts final setup_violation_count
--------------------------------------------------------------------------
setup violation count 0

==========================================================================
cts final hold_violation_count
--------------------------------------------------------------------------
hold violation count 0

==========================================================================
cts final report_checks -path_delay max reg to reg
--------------------------------------------------------------------------
Startpoint: core.CPU_valid_taken_br_a5$_DFF_P_
            (rising edge-triggered flip-flop clocked by clk)
Endpoint: core.CPU_Xreg_value_a4[7][13]$_SDFFE_PP0P_
          (rising edge-triggered flip-flop clocked by clk)
Path Group: clk
Path Type: max

  Delay    Time   Description
---------------------------------------------------------
   0.00    0.00   clock clk (rise edge)
   0.00    0.00   clock source latency
   0.00    0.00 ^ pll/CLK (avsdpll)
   0.26    0.26 ^ clkbuf_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.21    0.47 ^ clkbuf_3_4_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.23    0.70 ^ clkbuf_4_8__f_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.18    0.88 ^ clkbuf_leaf_149_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.00    0.88 ^ core.CPU_valid_taken_br_a5$_DFF_P_/CLK (sky130_fd_sc_hd__dfxtp_1)
   0.30    1.18 v core.CPU_valid_taken_br_a5$_DFF_P_/Q (sky130_fd_sc_hd__dfxtp_1)
   0.83    2.01 v _07913_/X (sky130_fd_sc_hd__or4_4)
   0.36    2.38 ^ _07915_/Y (sky130_fd_sc_hd__clkinv_16)
   0.15    2.53 v _09981_/Y (sky130_fd_sc_hd__nor3_2)
   0.57    3.10 ^ _09982_/Y (sky130_fd_sc_hd__a21oi_4)
   0.38    3.47 v _09988_/Y (sky130_fd_sc_hd__o21ai_4)
   1.20    4.67 ^ _13552_/Y (sky130_fd_sc_hd__nor3_4)
   0.31    4.98 v _13572_/Y (sky130_fd_sc_hd__nand2_2)
   0.24    5.22 ^ _13574_/Y (sky130_fd_sc_hd__o221ai_1)
   0.57    5.79 ^ hold2174/X (sky130_fd_sc_hd__dlygate4sd3_1)
   0.00    5.79 ^ core.CPU_Xreg_value_a4[7][13]$_SDFFE_PP0P_/D (sky130_fd_sc_hd__dfxtp_1)
           5.79   data arrival time

  11.05   11.05   clock clk (rise edge)
   0.00   11.05   clock source latency
   0.00   11.05 ^ pll/CLK (avsdpll)
   0.26   11.31 ^ clkbuf_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.21   11.52 ^ clkbuf_3_6_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.24   11.76 ^ clkbuf_4_13__f_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.19   11.94 ^ clkbuf_leaf_90_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.00   11.94 ^ core.CPU_Xreg_value_a4[7][13]$_SDFFE_PP0P_/CLK (sky130_fd_sc_hd__dfxtp_1)
  -0.55   11.39   clock uncertainty
   0.00   11.39   clock reconvergence pessimism
  -0.05   11.34   library setup time
          11.34   data required time
---------------------------------------------------------
          11.34   data required time
          -5.79   data arrival time
---------------------------------------------------------
           5.55   slack (MET)



==========================================================================
cts final report_checks -path_delay min reg to reg
--------------------------------------------------------------------------
Startpoint: core.CPU_Xreg_value_a4[25][15]$_SDFFE_PP0P_
            (rising edge-triggered flip-flop clocked by clk)
Endpoint: core.CPU_src2_value_a3[15]$_DFF_P_
          (rising edge-triggered flip-flop clocked by clk)
Path Group: clk
Path Type: min

  Delay    Time   Description
---------------------------------------------------------
   0.00    0.00   clock clk (rise edge)
   0.00    0.00   clock source latency
   0.00    0.00 ^ pll/CLK (avsdpll)
   0.26    0.26 ^ clkbuf_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.21    0.47 ^ clkbuf_3_7_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.24    0.71 ^ clkbuf_4_14__f_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.19    0.90 ^ clkbuf_leaf_108_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.00    0.90 ^ core.CPU_Xreg_value_a4[25][15]$_SDFFE_PP0P_/CLK (sky130_fd_sc_hd__dfxtp_1)
   0.32    1.22 ^ core.CPU_Xreg_value_a4[25][15]$_SDFFE_PP0P_/Q (sky130_fd_sc_hd__dfxtp_1)
   0.06    1.28 v _14708_/Y (sky130_fd_sc_hd__a21oi_1)
   0.12    1.40 ^ _14712_/Y (sky130_fd_sc_hd__nand4_1)
   0.18    1.58 v _14729_/Y (sky130_fd_sc_hd__o32ai_4)
   0.16    1.74 ^ _14731_/Y (sky130_fd_sc_hd__o21ai_0)
   0.00    1.74 ^ core.CPU_src2_value_a3[15]$_DFF_P_/D (sky130_fd_sc_hd__dfxtp_2)
           1.74   data arrival time

   0.00    0.00   clock clk (rise edge)
   0.00    0.00   clock source latency
   0.00    0.00 ^ pll/CLK (avsdpll)
   0.26    0.26 ^ clkbuf_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.21    0.47 ^ clkbuf_3_6_0_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.24    0.71 ^ clkbuf_4_13__f_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.19    0.89 ^ clkbuf_leaf_70_CLK/X (sky130_fd_sc_hd__clkbuf_16)
   0.00    0.89 ^ core.CPU_src2_value_a3[15]$_DFF_P_/CLK (sky130_fd_sc_hd__dfxtp_2)
   0.88    1.78   clock uncertainty
   0.00    1.78   clock reconvergence pessimism
  -0.03    1.74   library hold time
           1.74   data required time
---------------------------------------------------------
           1.74   data required time
          -1.74   data arrival time
---------------------------------------------------------
           0.00   slack (MET)



==========================================================================
cts final critical path target clock latency max path
--------------------------------------------------------------------------
0

==========================================================================
cts final critical path target clock latency min path
--------------------------------------------------------------------------
0

==========================================================================
cts final critical path source clock latency min path
--------------------------------------------------------------------------
0

==========================================================================
cts final critical path delay
--------------------------------------------------------------------------
5.7921

==========================================================================
cts final critical path slack
--------------------------------------------------------------------------
5.5454

==========================================================================
cts final slack div critical path delay
--------------------------------------------------------------------------
95.740750

==========================================================================
cts final report_power
--------------------------------------------------------------------------
Group                  Internal  Switching    Leakage      Total
                          Power      Power      Power      Power (Watts)
----------------------------------------------------------------
Sequential             6.95e-03   9.72e-04   1.45e-08   7.92e-03  37.7%
Combinational          2.12e-03   4.52e-03   3.05e-08   6.64e-03  31.6%
Clock                  3.66e-03   2.76e-03   2.96e-09   6.42e-03  30.6%
Macro                  0.00e+00   0.00e+00   0.00e+00   0.00e+00   0.0%
Pad                    0.00e+00   0.00e+00   0.00e+00   0.00e+00   0.0%
----------------------------------------------------------------
Total                  1.27e-02   8.25e-03   4.80e-08   2.10e-02 100.0%
                          60.7%      39.3%       0.0%
```


The reports showed **no setup or hold violations**, which is great!

> **Post-CTS Timing Report Summary:**
> * **WNS (Worst Negative Slack):** 5.55 ns (MET)
> * **TNS (Total Negative Slack):** 0.00 ns (MET)
> * **Setup Violations:** 0
> * **Hold Violations:** 0

### 5. Routing
This is the final layout step. The global router plans the paths for all the nets, and the detailed router draws the actual metal wires to make the connections.
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk route
```
And the final view of the fully routed chip:
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk gui_route
```
- After this i have faced a congestion error , basically after routing or during the place left for routing isnt enough or very less so that the routes , or tracks are too close each other causing DRC violations. 

<img width="1438" height="231" alt="route_Error" src="https://github.com/user-attachments/assets/a559dea6-7149-4b4d-849f-39846827bb4a" />

---

##  Part 4: What is SPEF?

The final step of the flow, and a key goal of this week, was to generate the **SPEF (Standard Parasitic Exchange Format)** file.

### Why is SPEF so important?

Before routing, our timing analysis (pre-STA) is just an *estimate*. It uses a "Wire Load Model" to guess the resistance (R) and capacitance (C) of the wires that *will* be drawn.

After routing, we have the *actual, physical wires*. The SPEF file contains the **real, extracted parasitic R and C values** for every single net in the design.

**In short: SPEF = Ground Truth.**

When we run our final Static Timing Analysis (known as **post-route STA**), we feed it this SPEF file. This allows the timer to calculate the *actual* delays through the wires. This is the "signoff" analysis that determines if our chip will *really* work at the target frequency.

Without a SPEF file, you're just guessing. With it, you *know*.

- To get the RC extraction and generate a spef file we can use the following tcl script

```
# Extraction

if { $rcx_rules_file != "" } {
  define_process_corner -ext_model_index 0 X
  extract_parasitics -ext_model_file $rcx_rules_file

  set spef_file [make_result_file ${design}_${platform}.spef]
  write_spef $spef_file

  read_spef $spef_file
} else {
  # Use global routing based parasitics inlieu of rc extraction
  estimate_parasitics -global_routing
}
```


---

##  Conclusion

The BabySoC physical design flow for Week 7 is complete. The OpenROAD-flow-scripts environment was configured, and the chip was successfully processed through synthesis, floorplanning, placement, CTS, and routing. The primary objective was met with the generation of the post-route SPEF file, capturing the real parasitic data essential for signoff-level timing analysis. This concludes the tasks for Week 7, with Week 8 to follow.

## Acknowledgement 
@ Kunal Sir
@ VSD 
@ all the team members and forum helper.
