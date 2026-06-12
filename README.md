# CV32E40P — ASAP7 Frequency Sweep Synthesis

Automated RTL-to-GDSII frequency-sweep synthesis for the **CV32E40P RISC-V core**
using the [OpenROAD-flow-scripts (ORFS)](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)
tool chain and the **ASAP7 7nm FinFET Predictive PDK**.

Three clock frequencies are benchmarked: **500 MHz**, **600 MHz**, and **700 MHz**.

## Results at a Glance

| Target | Period | WNS | TNS | Fmax | Area |
|--------|--------|-----|-----|------|------|
| 500 MHz | 2000 ps | +671.77 ps | 0.00 ps | 1035 MHz | 3146.87 µm² |
| 600 MHz | 1667 ps | +505.27 ps | 0.00 ps | 1035 MHz | 3146.87 µm² |
| 700 MHz | 1429 ps | +386.27 ps | 0.00 ps | 1035 MHz | 3146.87 µm² |

> **TNS = 0** means timing is fully met — Total Negative Slack only accumulates
> when endpoints are *violating* (WNS < 0). See Section 3 for details.

## Repository Structure

```
cv32e40p_submission_v2/
│
├── README.md                    ← This file
├── REPORT.md                    ← Full technical report with metrics & critical-path analysis
├── LICENSE
├── .gitignore
├── .gitattributes               ← Git LFS tracking for *.v netlists
│
├── rtl/                         ← RTL source (SystemVerilog)
│   ├── include/                 ← Package files (cv32e40p_pkg.sv, etc.)
│   └── cv32e40p_*.sv            ← Core RTL modules
│
├── flow_config/                 ← ORFS design configuration
│   ├── config_500.mk            ← ORFS Makefile config for 500 MHz
│   ├── config_600.mk
│   ├── config_700.mk
│   ├── constraint_500.sdc       ← SDC timing constraint (2000 ps)
│   ├── constraint_600.sdc       ← SDC timing constraint (1667 ps)
│   └── constraint_700.sdc       ← SDC timing constraint (1429 ps)
│
├── synthesis_500/               ← 500 MHz synthesis & floorplan results
│   ├── 1_2_yosys.v              ← Gate-level netlist (Yosys mapped, via Git LFS)
│   ├── 1_2_yosys.log            ← Yosys run log
│   ├── synth_stat.txt           ← Cell-count & area statistics
│   ├── 2_floorplan_final.rpt    ← OpenROAD STA floorplan timing report
│   └── config.json              ← OpenLane-style configuration wrapper
│
├── synthesis_600/               ← 600 MHz results (same structure)
└── synthesis_700/               ← 700 MHz results (same structure)
```

## Prerequisites

| Tool | Version used | Install |
|------|-------------|---------|
| Docker | any recent | `sudo apt install docker.io` |
| `openroad/orfs` image | `latest` (43d097) | `docker pull openroad/orfs` |
| git-lfs | any | `sudo apt install git-lfs && git lfs install` |

## How to Reproduce

Full step-by-step instructions with exact terminal input/output are in **Section 6** below.

Quick summary:

```bash
# 1. Clone this repo
git clone https://github.com/azrafinam/cv32e40p_submission_v2.git
cd cv32e40p_submission_v2

# 2. Copy RTL and config into your ORFS workspace
#    (assumes OpenROAD-flow-scripts is cloned alongside)
cp -r rtl/* ../OpenROAD-flow-scripts/flow/designs/src/cv32e40p/
cp flow_config/* ../OpenROAD-flow-scripts/flow/designs/asap7/cv32e40p/

# 3. Run the frequency sweep
cd ../OpenROAD-flow-scripts
docker run --name cv32e40p_run \
  -v $(pwd)/flow:/OpenROAD-flow-scripts/flow \
  openroad/orfs bash -c "
    cd /OpenROAD-flow-scripts/flow
    for FREQ in 500 600 700; do
      make DESIGN_CONFIG=./designs/asap7/cv32e40p/config_\${FREQ}.mk clean_synth
      make DESIGN_CONFIG=./designs/asap7/cv32e40p/config_\${FREQ}.mk clean_floorplan
      make DESIGN_CONFIG=./designs/asap7/cv32e40p/config_\${FREQ}.mk floorplan
    done
  "
```

## Key Design Notes

- **ASAP7 time unit is 1 ps** — SDC clock periods must be in picoseconds.
  `2.000` would mean 2 ps (500 GHz), not 2 ns (500 MHz).
- **HDL frontend**: `slang` (full SystemVerilog support, required for cv32e40p).
- **Cell library**: `asap7sc7p5t_AO` RVT, Best-Case (FF) corner.
- **Fmax = 1035 MHz** is the physical speed limit of the cell mapping.
  All three target frequencies are comfortably within this limit.

## RTL Attribution

RTL source files are derived from the
[OpenHW Group cv32e40p](https://github.com/openhwgroup/cv32e40p) project
(Solderpad Hardware Licence v2.0).

## Executive Summary

This document details the automated RTL-to-GDSII frequency sweep synthesis project for the **CV32E40P RISC-V core** (a 4-stage in-order core with floating-point support). The target frequencies evaluated are **500 MHz**, **600 MHz**, and **700 MHz** using the ASAP7 7nm FinFET Predictive PDK and the OpenROAD-flow-scripts (ORFS) toolchain.

**Key Finding**: All three target frequencies are **timing-compliant** with positive slack (WNS) and zero timing violations (TNS = 0). The physical speed limit of this design is **1035.20 MHz**, determined by the ASAP7 cell library constraints.

By identifying that the ASAP7 FinFET cell libraries operate under a default time unit of **1 ps (picoseconds)** rather than 1 ns (nanoseconds), we corrected the clock constraint files (`.sdc`) to specify periods in **picoseconds** — delivering fully-met timing across all three frequency targets.

## Methodology & Synthesis Setup

### Tool Flow & Environment

The OpenROAD-flow-scripts (ORFS) framework was run within the official `openroad/orfs:latest` Docker image.

- **HDL Frontend**: `slang` (SystemVerilog parser)
- **Synthesis Tool**: Yosys 0.64+post
- **Technology Mapper**: ABC
- **Floorplanning / STA**: OpenROAD (OpenSTA)

### Technology PDK

The design was mapped to Arizona State University's **ASAP7 Predictive PDK** using the **7.5-track cell library (asap7sc7p5t_AO)** under Typical-Typical (TT) or Best-Case (FF) corner conditions.
The default time unit in ASAP7 Liberty files (`.lib`) is explicitly set to **1 ps**:

```liberty
library (asap7sc7p5t_SEQ_RVT_FF_nldm_220123) {
  time_unit : "1ps";
  ...
}
```

Setting clock periods as `2.000`, `1.667`, and `1.429` in SDC files previously instructed the synthesis tools to target a clock period of **2 ps** (500 GHz), causing ABC to map for absolute maximum speed.

Correcting the periods to **2000 ps**, **1667 ps**, and **1429 ps** (respectively) aligns synthesis with the intended target frequencies.

## Synthesis & Floorplanning Results

The table below summarizes the key design and timing metrics extracted from the Yosys synthesis logs (`synth_stat.txt`) and OpenROAD floorplanning reports (`2_floorplan_final.rpt`) across all three frequency targets.

### Metrics Comparison Table

| Metric | Target: 500 MHz | Target: 600 MHz | Target: 700 MHz |
| :--- | :---: | :---: | :---: |
| **Target Clock Period** | 2000 ps | 1667 ps | 1429 ps |
| **WNS (Worst-case Slack)** | **+671.77 ps** | **+505.27 ps** | **+386.27 ps** |
| **TNS (Total Negative Slack)** | **0.00 ps** | **0.00 ps** | **0.00 ps** |
| **Timing Verdict** | **MET** | **MET** | **MET** |
| **Violated Endpoints** | 0 | 0 | 0 |
| **Minimum Period Limit** | 966.00 ps | 966.00 ps | 966.00 ps |
| **Maximum Frequency ($F_{max}$)** | 1035.20 MHz | 1035.20 MHz | 1035.20 MHz |
| **Total Gate Count** | 26,282 | 26,282 | 26,282 |
| **Sequential Elements Area** | 816.54 µm² (25.95%) | 816.54 µm² (25.95%) | 816.54 µm² (25.95%) |
| **Total Module Chip Area** | 3146.87 µm² | 3146.87 µm² | 3146.87 µm² |
| **Core Cell Utilization** | 9% | 9% | 9% |

> **Why is TNS = 0?**  
> TNS (Total Negative Slack) is defined as the **sum of the negative slack across all violating timing endpoints**. When WNS is **positive**, it means the worst path still has slack left — it is not violating. Because all paths pass timing, TNS accumulates zero.

### Cell Count Analysis

The cell counts are identical across the three target frequencies. Because the physical speed limit of this mapped design is **1035.20 MHz** (a minimum period of **966 ps**), all three target periods (**2000 ps**, **1667 ps**, **1429 ps**) are well above the minimum achievable. The ABC technology mapper therefore produces an identical gate-level netlist for all three frequency targets; only the slack margin changes as the target period shrinks.

## Critical Path Timing Analysis (700 MHz Run)

At 700 MHz (period target = 1429 ps), the worst path is a **half-cycle setup path** ending at a negative-level sensitive latch.

### Path Summary

- **Startpoint**: `id_stage_i.alu_operand_b_ex_o[6]$_DFFE_PN0P_` (rising edge-triggered DFF, clocked by `clk_i` at 0.00 ps)
- **Endpoint**: `sleep_unit_i.core_clock_gate_i.clk_en$_DLATCH_N_` (negative level-sensitive latch, clocked by `clk_i` at the falling edge)
- **Path Type**: Setup Max Delay
- **Slack**: **+386.27 ps (MET)**

### Timing Calculation Details

Because the endpoint is a negative-level latch, it captures data on the falling edge of the clock.

- For a clock period of T_clk = 1429 ps, the falling edge occurs at T_clk/2 = 714.50 ps.
- **Data Arrival Time**: 328.23 ps
- **Data Required Time**: 714.50 ps (half-period)
- **Slack Equation**:
  Slack = Required Time - Arrival Time = 714.50 ps - 328.23 ps = 386.27 ps

This explains why the slack decreases linearly as the clock period decreases:

- **500 MHz**: Required Time = 1000.00 ps → Slack = 1000.00 - 328.23 = **671.77 ps**
- **600 MHz**: Required Time = 833.50 ps → Slack = 833.50 - 328.23 = **505.27 ps**
- **700 MHz**: Required Time = 714.50 ps → Slack = 714.50 - 328.23 = **386.27 ps**

## Folder Structure of the Repository Submission

The deliverables are packaged in a standalone directory tree ready for submission:

```
cv32e40p_submission_v2/
├── REPORT.md                    ← Full Report
├── synthesis_500/
│   ├── 1_2_yosys.v              ← Mapped Verilog Netlist
│   ├── 1_2_yosys.log            ← Yosys Run Log
│   ├── synth_stat.txt           ← Yosys Cell Statistics Report
│   ├── 2_floorplan_final.rpt    ← OpenROAD Floorplan STA Timing Report
│   └── config.json              ← OpenLane-Style Configuration Wrapper
├── synthesis_600/
│   ├── 1_2_yosys.v
│   ├── 1_2_yosys.log
│   ├── synth_stat.txt
│   ├── 2_floorplan_final.rpt
│   └── config.json
└── synthesis_700/
    ├── 1_2_yosys.v
    ├── 1_2_yosys.log
    ├── synth_stat.txt
    ├── 2_floorplan_final.rpt
    └── config.json
```

## Reproducibility — Full Terminal Session

This section contains the **exact commands entered** and the **key terminal outputs** that were produced during this run. You can copy-paste this session to reproduce the results.

> The complete raw Docker terminal log (1506 lines) is saved at: `docker_terminal_full.log`

### Step 1 — Verify and Correct SDC Clock Constraints

All clock constraints must be in **picoseconds** to match the `time_unit : "1ps"` in the ASAP7 Liberty files.

**Input:**
```bash
# On host, from /home/stark/OpenROAD-flow-scripts
cat flow/designs/asap7/cv32e40p/constraint_500.sdc
cat flow/designs/asap7/cv32e40p/constraint_600.sdc
cat flow/designs/asap7/cv32e40p/constraint_700.sdc
```

**Output (after correction):**
```
create_clock -name clk_i -period 2000 [get_ports clk_i]
create_clock -name clk_i -period 1667 [get_ports clk_i]
create_clock -name clk_i -period 1429 [get_ports clk_i]
```

If the files still show the wrong values (`2.000`, `1.667`, `1.429`), fix them:
```bash
echo 'create_clock -name clk_i -period 2000 [get_ports clk_i]' > flow/designs/asap7/cv32e40p/constraint_500.sdc
echo 'create_clock -name clk_i -period 1667 [get_ports clk_i]' > flow/designs/asap7/cv32e40p/constraint_600.sdc
echo 'create_clock -name clk_i -period 1429 [get_ports clk_i]' > flow/designs/asap7/cv32e40p/constraint_700.sdc
```

### Step 2 — Confirm ASAP7 PDK Time Unit

**Input:**
```bash
grep 'time_unit' flow/platforms/asap7/lib/NLDM/asap7sc7p5t_SEQ_RVT_FF_nldm_220123.lib
```
**Output:**
```
  time_unit : "1ps";
```
This confirms the library is in **picoseconds**.

### Step 3 — Confirm Docker Image is Available

**Input:**
```bash
docker images | grep openroad/orfs
```
**Output:**
```
openroad/orfs:latest   e4e71714221b   6.49GB   1.58GB
```

### Step 4 — Run the Full Frequency Sweep in Docker

**Input:**
```bash
docker run --name cv32e40p_run \
  -v /home/stark/OpenROAD-flow-scripts/flow:/OpenROAD-flow-scripts/flow \
  -v /home/stark/OpenROAD-flow-scripts/cv32e40p:/cv32e40p \
  openroad/orfs \
  bash -c "
    cd /OpenROAD-flow-scripts/flow
    for FREQ in 500 600 700; do
      echo '=== Starting FREQ=\${FREQ} ==='
      make DESIGN_CONFIG=./designs/asap7/cv32e40p/config_\${FREQ}.mk clean_synth
      make DESIGN_CONFIG=./designs/asap7/cv32e40p/config_\${FREQ}.mk clean_floorplan
      make DESIGN_CONFIG=./designs/asap7/cv32e40p/config_\${FREQ}.mk floorplan
    done
  "
```

**Key Output Milestones (condensed from 1506-line log):**
```
=== Starting FREQ=500 ===
Setting clock period to 2000
...
abc ... -D 2000
14. Executing ABC pass (technology mapping using ABC).
...
Chip area for module '\cv32e40p_core': 3146.874300
  of which used for sequential elements: 816.538320 (25.95%)
...
[INFO RSZ-0098] No setup violations found
...
Report metrics stage 2, floorplan final...
Elapsed time: 0:48.41[h:]min:sec. CPU time: user 51.55 sys 0.61 (107%)

=== Starting FREQ=600 ===
Setting clock period to 1667
...
abc ... -D 1667
...
Chip area for module '\cv32e40p_core': 3146.874300
...
[INFO RSZ-0098] No setup violations found
Elapsed time: 0:47.93[h:]min:sec. CPU time: user 51.22 sys 0.64 (108%)

=== Starting FREQ=700 ===
Setting clock period to 1429
...
abc ... -D 1429
...
Chip area for module '\cv32e40p_core': 3146.874300
...
[INFO RSZ-0098] No setup violations found
Elapsed time: 0:47.84[h:]min:sec. CPU time: user 51.16 sys 0.68 (108%)

Exit code: 0
```

**Verify container is preserved:**
```bash
docker ps -a | grep cv32e40p_run
```
**Output:**
```
37e1ba5d35fb  openroad/orfs  "bash -c 'cd /OpenRO..."  Exited (0)  cv32e40p_run
```

### Step 5 — Verify the Clock Periods Were Applied Correctly

**Input:**
```bash
grep -i 'Setting clock period' \
  flow/logs/asap7/cv32e40p_500/base/1_2_yosys.log \
  flow/logs/asap7/cv32e40p_600/base/1_2_yosys.log \
  flow/logs/asap7/cv32e40p_700/base/1_2_yosys.log
```
**Output:**
```
flow/logs/asap7/cv32e40p_500/base/1_2_yosys.log:Setting clock period to 2000
flow/logs/asap7/cv32e40p_600/base/1_2_yosys.log:Setting clock period to 1667
flow/logs/asap7/cv32e40p_700/base/1_2_yosys.log:Setting clock period to 1429
```

### Step 6 — Extract Timing and Area Metrics

**Input:**
```bash
for FREQ in 500 600 700; do
  echo "=== ${FREQ} MHz ==="
  grep -E "cells$|Chip area|sequential" \
    flow/reports/asap7/cv32e40p_${FREQ}/base/synth_stat.txt
  echo "---"
  grep -E "tns|wns|worst slack|fmax|period_min" \
    flow/reports/asap7/cv32e40p_${FREQ}/base/2_floorplan_final.rpt
done
```
**Output:**
```
=== 500 MHz ===
    26282 3.15E+03    26282 3.15E+03 cells
   Chip area for module '\cv32e40p_core': 3146.874300
     of which used for sequential elements: 816.538320 (25.95%)
---
tns max 0.00
wns max 0.00
worst slack max 671.77
clk_i period_min = 966.00 fmax = 1035.20

=== 600 MHz ===
    26282 3.15E+03    26282 3.15E+03 cells
   Chip area for module '\cv32e40p_core': 3146.874300
     of which used for sequential elements: 816.538320 (25.95%)
---
tns max 0.00
wns max 0.00
worst slack max 505.27
clk_i period_min = 966.00 fmax = 1035.20

=== 700 MHz ===
    26282 3.15E+03    26282 3.15E+03 cells
   Chip area for module '\cv32e40p_core': 3146.874300
     of which used for sequential elements: 816.538320 (25.95%)
---
tns max 0.00
wns max 0.00
worst slack max 386.27
clk_i period_min = 966.00 fmax = 1035.20
```

### Step 7 — Package the Submission Folder

**Input:**
```bash
mkdir -p ~/cv32e40p_submission_v2/synthesis_{500,600,700}

for FREQ in 500 600 700; do
  cp flow/results/asap7/cv32e40p_${FREQ}/base/1_2_yosys.v     ~/cv32e40p_submission_v2/synthesis_${FREQ}/
  cp flow/logs/asap7/cv32e40p_${FREQ}/base/1_2_yosys.log       ~/cv32e40p_submission_v2/synthesis_${FREQ}/
  cp flow/reports/asap7/cv32e40p_${FREQ}/base/synth_stat.txt   ~/cv32e40p_submission_v2/synthesis_${FREQ}/
  cp flow/reports/asap7/cv32e40p_${FREQ}/base/2_floorplan_final.rpt ~/cv32e40p_submission_v2/synthesis_${FREQ}/
done
```

**Verify:**
```bash
find ~/cv32e40p_submission_v2 -type f | sort
```
**Output:**
```
/home/stark/cv32e40p_submission_v2/REPORT.md
/home/stark/cv32e40p_submission_v2/docker_terminal_full.log
/home/stark/cv32e40p_submission_v2/synthesis_500/1_2_yosys.log
/home/stark/cv32e40p_submission_v2/synthesis_500/1_2_yosys.v
/home/stark/cv32e40p_submission_v2/synthesis_500/2_floorplan_final.rpt
/home/stark/cv32e40p_submission_v2/synthesis_500/config.json
/home/stark/cv32e40p_submission_v2/synthesis_500/synth_stat.txt
/home/stark/cv32e40p_submission_v2/synthesis_600/1_2_yosys.log
/home/stark/cv32e40p_submission_v2/synthesis_600/1_2_yosys.v
/home/stark/cv32e40p_submission_v2/synthesis_600/2_floorplan_final.rpt
/home/stark/cv32e40p_submission_v2/synthesis_600/config.json
/home/stark/cv32e40p_submission_v2/synthesis_600/synth_stat.txt
/home/stark/cv32e40p_submission_v2/synthesis_700/1_2_yosys.log
/home/stark/cv32e40p_submission_v2/synthesis_700/1_2_yosys.v
/home/stark/cv32e40p_submission_v2/synthesis_700/2_floorplan_final.rpt
/home/stark/cv32e40p_submission_v2/synthesis_700/config.json
/home/stark/cv32e40p_submission_v2/synthesis_700/synth_stat.txt
```
