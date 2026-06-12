# CV32E40P — ASAP7 Frequency Sweep Synthesis

Automated RTL-to-GDSII frequency-sweep synthesis for the **CV32E40P** RISC-V core
using the [OpenROAD-flow-scripts (ORFS)](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)
tool chain and the **ASAP7 7nm FinFET Predictive PDK**.

Three clock frequencies are benchmarked: **500 MHz**, **600 MHz**, and **700 MHz**.

---

## Results at a Glance

| Target | Period | WNS | TNS | Fmax | Area |
|--------|--------|-----|-----|------|------|
| 500 MHz | 2000 ps | +671.77 ps ✅ | 0.00 ps | 1035 MHz | 3146.87 µm² |
| 600 MHz | 1667 ps | +505.27 ps ✅ | 0.00 ps | 1035 MHz | 3146.87 µm² |
| 700 MHz | 1429 ps | +386.27 ps ✅ | 0.00 ps | 1035 MHz | 3146.87 µm² |

> **TNS = 0** means timing is fully met — Total Negative Slack only accumulates
> when endpoints are *violating* (WNS < 0). See `REPORT.md` Section 3 for details.

---

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

---

## Prerequisites

| Tool | Version used | Install |
|------|-------------|---------|
| Docker | any recent | `sudo apt install docker.io` |
| `openroad/orfs` image | `latest` (43d097) | `docker pull openroad/orfs` |
| git-lfs | any | `sudo apt install git-lfs && git lfs install` |

---

## How to Reproduce

Full step-by-step instructions with exact terminal input/output are in **`REPORT.md` — Section 6**.

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

---

## Key Design Notes

- **ASAP7 time unit is 1 ps** — SDC clock periods must be in picoseconds.
  `2.000` would mean 2 ps (500 GHz), not 2 ns (500 MHz).
- **HDL frontend**: `slang` (full SystemVerilog support, required for cv32e40p).
- **Cell library**: `asap7sc7p5t_AO` RVT, Best-Case (FF) corner.
- **Fmax = 1035 MHz** is the physical speed limit of the cell mapping.
  All three target frequencies are comfortably within this limit.

---

## RTL Attribution

RTL source files are derived from the
[OpenHW Group cv32e40p](https://github.com/openhwgroup/cv32e40p) project
(Solderpad Hardware Licence v2.0).
