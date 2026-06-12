# CV32E40P — ASAP7 Frequency Sweep Synthesis

Frequency-sweep **synthesis and floorplan-stage timing analysis** of the
**CV32E40P RISC-V core** using
[OpenROAD-flow-scripts (ORFS)](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)
on the **ASAP7 7nm FinFET predictive PDK**.

Three clock targets are benchmarked: **500 MHz**, **600 MHz**, and **700 MHz**.

> **Flow scope.** The full RTL-to-GDSII flow is
> RTL → synthesis → floorplan → placement → CTS → routing → GDSII.
> This repository covers the first stage (synthesis) plus floorplan
> initialization — run only because ORFS produces its first STA report at the
> floorplan step. All timing below is therefore **pre-placement**: every net
> delay is 0.00, so the numbers measure pure logic speed with zero wire
> contribution. Placement and routing are out of scope for this week.

## Results at a Glance

| Target | Period | Worst slack | TNS | Violated endpoints | Fmax | Area |
|--------|--------|-------------|-----|--------------------|------|------|
| 500 MHz | 2000 ps | **+671.77 ps (MET)** | 0.00 ps | 0 | 1035.20 MHz | 3146.87 µm² |
| 600 MHz | 1667 ps | **+505.27 ps (MET)** | 0.00 ps | 0 | 1035.20 MHz | 3146.87 µm² |
| 700 MHz | 1429 ps | **+386.27 ps (MET)** | 0.00 ps | 0 | 1035.20 MHz | 3146.87 µm² |

Identical netlist at all three targets: **26,282 cells**, 2,154 flip-flops,
sequential area 816.54 µm² (25.95%). TNS = 0 because TNS only accumulates over
*violating* endpoints — with positive worst slack everywhere, there are none.

## The Units Lesson (read this before reusing the configs)

**ASAP7 Liberty files use `time_unit : "1ps"`, and OpenSTA reads SDC values in
library units.** A constraint written as `create_clock -period 2.000` (meant as
ns) is silently interpreted as **2 ps — a 500 GHz clock** — producing huge fake
negative slack (≈ −887 ps at every frequency) and driving ABC to map for
absolute maximum speed. The SDCs in `flow_config/` are already correct, in
picoseconds:

````tcl
create_clock -name clk_i -period 2000 [get_ports clk_i]   # 500 MHz
create_clock -name clk_i -period 1667 [get_ports clk_i]   # 600 MHz
create_clock -name clk_i -period 1429 [get_ports clk_i]   # 700 MHz
````

## Critical Path Findings

- **Worst-slack printed path** (all three runs): an ID/EX ALU operand register
  (`id_stage_i.alu_operand_b_ex_o[6]`) → the sleep unit's **clock-gate enable
  latch**, 15 logic levels, arrival 328.23 ps. The latch captures on the
  **falling** clock edge, so its deadline is period/2 — which is exactly why
  the slack steps 671.77 → 505.27 → 386.27 ps track half the period steps.
  With 995.82 ps of borrowable latch time (0 used), it never limits frequency.
- **The actual fmax limiter** (`period_min = 966.00 ps → 1035.20 MHz`): the
  multiplier feedback path `ex_stage_i.mult_i.mulh_CS[0] →
  id_stage_i.alu_operand_a_ex_o[20]`, threading **six FAx1 full adders plus a
  half adder** in series (the iterative multiplier's carry/accumulation chain).
  Arrival 951.24 ps + 14.75 ps setup = 965.99 ps — matching the reported
  minimum period to the last digit.

## Verification (why "zero violations" is trusted)

- Every standard-cell Liberty file reports `time_unit : "1ps"`; the only 1ns
  libraries in the platform are unused fakeram SRAM models (this design has no
  memories).
- `report_units` from OpenSTA itself: `time 1ps`; `report_clock_properties`:
  `clk_i 2000.00`.
- **Negative control:** a deliberately impossible 4 GHz (250 ps) clock fails
  with `wns max −638.94` — exactly 250 − 888.94. Same engine, same libraries:
  fails at 250 ps, passes at 1429 ps, crossover at the reported minimum period.

## Repository Structure

````
cv32e40p_submission_v2/
├── README.md                    ← this file
├── docker_terminal_full.log     ← complete raw run log (1506 lines)
│
├── rtl/                         ← synthesised RTL (27 SystemVerilog files)
│   ├── include/                 ← package files (cv32e40p_pkg.sv, …)
│   └── cv32e40p_*.sv            ← core modules (FF register file variant)
│
├── flow_config/                 ← ORFS design configuration
│   ├── config_{500,600,700}.mk          ← Makefile configs (slang frontend)
│   └── constraint_{500,600,700}.sdc     ← clock constraints, in picoseconds
│
├── synthesis_500/               ← per-frequency deliverables
│   ├── 1_2_yosys.v              ← gate-level netlist (Yosys + ABC mapped)
│   ├── 1_2_yosys.log            ← synthesis log
│   ├── synth_stat.txt           ← cell-count and area statistics
│   ├── 2_floorplan_final.rpt    ← OpenROAD STA timing report
│   └── config.json              ← OpenLane-style configuration wrapper
├── synthesis_600/               ← same structure
└── synthesis_700/               ← same structure
````

## Toolchain

| Component | Version / setting |
|---|---|
| Container | `openroad/orfs:latest` (e4e71714221b) |
| Synthesis | Yosys 0.64+post, **slang** SystemVerilog frontend |
| Technology mapping | ABC (speed script) |
| STA / floorplan | OpenROAD (OpenSTA), `time 1ps` |
| Cell library | `asap7sc7p5t_AO` 7.5-track, RVT, FF-corner NLDM |
| Constraints | `MAX_FANOUT = 8`, `DIE_AREA 0 0 200 200`, `CORE_AREA 2 2 198 198` |

## How to Reproduce

````bash
# 1. Clone this repo next to an OpenROAD-flow-scripts checkout
git clone https://github.com/azrafinam/cv32e40p_submission_v2.git
cd cv32e40p_submission_v2

# 2. Stage RTL and configs into ORFS
mkdir -p ../OpenROAD-flow-scripts/flow/designs/src/cv32e40p
mkdir -p ../OpenROAD-flow-scripts/flow/designs/asap7/cv32e40p
cp -r rtl/*        ../OpenROAD-flow-scripts/flow/designs/src/cv32e40p/
cp flow_config/*   ../OpenROAD-flow-scripts/flow/designs/asap7/cv32e40p/

# 3. Run the sweep (floorplan pulls synthesis automatically)
cd ../OpenROAD-flow-scripts
docker run --rm \
  -v $(pwd)/flow:/OpenROAD-flow-scripts/flow \
  openroad/orfs bash -c "
    cd /OpenROAD-flow-scripts/flow
    for FREQ in 500 600 700; do
      make DESIGN_CONFIG=./designs/asap7/cv32e40p/config_\${FREQ}.mk clean_synth
      make DESIGN_CONFIG=./designs/asap7/cv32e40p/config_\${FREQ}.mk clean_floorplan
      make DESIGN_CONFIG=./designs/asap7/cv32e40p/config_\${FREQ}.mk floorplan
    done"

# 4. Verify the periods reached the tools in picoseconds
grep -i 'Setting clock period' flow/logs/asap7/cv32e40p_*/base/1_2_yosys.log
#   expected: 2000 / 1667 / 1429

# 5. Extract the headline metrics
for FREQ in 500 600 700; do
  echo "=== ${FREQ} MHz ==="
  grep -E "cells$|Chip area|sequential" flow/reports/asap7/cv32e40p_${FREQ}/base/synth_stat.txt
  grep -E "tns max|wns max|worst slack|period_min" flow/reports/asap7/cv32e40p_${FREQ}/base/2_floorplan_final.rpt
done
````

The exact terminal input/output of the original runs — including the units
debugging, the OpenSTA audit, and the 4 GHz negative control — is preserved in
`docker_terminal_full.log`.

## RTL Attribution

RTL sources are derived from the
[OpenHW Group cv32e40p](https://github.com/openhwgroup/cv32e40p) project
(Solderpad Hardware License v2.0). The flip-flop register-file variant
(`cv32e40p_register_file_ff.sv`) is used, as appropriate for standard-cell
synthesis; the latch variant is excluded.
````
````
