# Neuromorphic Adaptive‑Precision RISC‑V Processor
*A from‑scratch RV32 core with a neuromorphic controller for dynamic precision and “neuronal state” management.*

> **TL;DR**: This project implements a complete single‑cycle RISC‑V processor (control, datapath, ALU, IMEM/DMEM) plus a **Neuromorphic Adaptive Dynamic Precision Scaling (ADPS)** controller. The ADPS block observes instruction/operand activity and selectively switches ALU precision modes (1×64, 2×32, 4×16 slices) with hysteresis & refractory logic to reduce energy with minimal performance impact.

---

## Table of Contents
- [Features](#features)
- [Architecture Overview](#architecture-overview)
- [Repository Layout](#repository-layout)
- [Quick Start](#quick-start)
  - [Prerequisites](#prerequisites)
  - [Build & Run — Simulation](#build--run--simulation)
  - [Build & Run — FPGA (Zybo Z7-10)](#build--run--fpga-zybo-z7-10)
- [Firmware & Loading Flow](#firmware--loading-flow)
- [Configuration](#configuration)
  - [ADPS CSRs](#adps-csrs)
  - [Operating Modes](#operating-modes)
  - [Tuning Parameters](#tuning-parameters)
- [ADPS Controller — Algorithm](#adps-controller--algorithm)
- [Benchmarks & Methodology](#benchmarks--methodology)
- [Results (fill with your data)](#results-fill-with-your-data)
- [Known Limitations](#known-limitations)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)
- [Citation](#citation)
- [Acknowledgements](#acknowledgements)

---

## Features
- **From‑scratch RV32 single‑cycle core**: custom Control, Datapath, Register File, ALU, IMEM/DMEM, and top‑level integration.
- **Neuromorphic ADPS controller**: activity‑aware precision switching with **hysteresis** and **refractory** safeguards to avoid oscillations.
- **Sliced ALU**: supports **1×64**, **2×32**, or **4×16** slice execution for selective precision/parallelism (exact slicing is configurable in RTL).
- **Software‑visible CSRs** to enable/disable ADPS, select modes, set thresholds, read counters/telemetry.
- **Directed testbenches + example programs** (e.g., memory search‑and‑replace, matmul kernels).
- **FPGA target: Zybo Z7‑10** (Xilinx Zynq‑7000), with constraints and a batch Vivado flow.

> Replace or keep the slice widths above to match your RTL (e.g., 32‑only, 16‑only, or 64‑bit internal datapath for ALU).

---

## Architecture Overview
High‑level block diagram (add your figure in `docs/figs/architecture.svg`):

```
             +-------------------+
             |      IMEM         |
             |  (read-only)      |
             +---------+---------+
                       |
                 +-----v-----+
                 |  Control  |<-----------------------------+
                 +-----+-----+                              |
                       |                                    |
+-----------+     +----v----+      +-----------------+      |
| Register  |<--->| Datapath|<---->|  Sliced  ALU    |<-----+-- ADPS Controller
|   File    |     +----+----+      | (1x64/2x32/4x16)|      |   (hysteresis &
+-----------+          |            +-----------------+      |    refractory)
                       |                     ^               |
                 +-----v-----+               |               |
                 |   DMEM    |<--------------+---------------+
                 | (read/    |          CSR bus & telemetry
                 |  write)   |
                 +-----------+
```

- **Control**: instruction decode, branch/pc logic, ALU/RegFile control.
- **Datapath**: operand select muxes, immediate gen, write‑back paths.
- **Sliced ALU**: width/parallelism selectable; integrates with ADPS muxing.
- **ADPS Controller**: monitors activity, updates precision mode using CSR policies.
- **IMEM/DMEM**: initialized via `$readmemh` (sim) or included in bitstream (FPGA).

---

## Repository Layout
```
.
├── rtl/
│   ├── core/
│   │   ├── control/
│   │   ├── datapath/
│   │   ├── regfile.v
│   │   └── alu/
│   │       ├── alu_top.v
│   │       └── slices/
│   ├── adps/
│   │   ├── adps_ctrl.v
│   │   ├── adps_fsm.v
│   │   └── adps_csrs.v
│   ├── mem/
│   │   ├── imem.v
│   │   └── dmem.v
│   └── top/
│       └── top_zybo.v
├── sim/
│   ├── tb/
│   │   └── tb_top.v
│   ├── mem/
│   │   └── imem.hex
│   ├── waves.gtkw
│   └── Makefile
├── fw/
│   ├── tests/
│   │   ├── search_replace.S
│   │   └── matmul.c
│   ├── linker.ld
│   └── Makefile
├── fpga/
│   └── zybo_z7/
│       ├── build.tcl
│       └── zybo_z7.xdc
├── scripts/
│   ├── build_sim.sh
│   ├── format.py
│   └── run_vivado.tcl
├── docs/
│   └── figs/
│       ├── architecture.svg
│       └── adps_fsm.svg
└── README.md
```

> Adjust the tree to mirror your repo. Keep RTL and tests cleanly separated; add `lint/` (verilator) or `ci/` if you use them.

---

## Quick Start

### Prerequisites
- **RISC‑V GCC** toolchain (e.g., `riscv64-unknown-elf-gcc`, `binutils`, `newlib`)
- **Icarus Verilog** (11+) & **GTKWave**
- **Vivado** 2024.1+ (for FPGA build)
- Python 3.9+ (optional scripts)

### Build & Run — Simulation
**Option A: Makefile (recommended)**
```bash
# from repo root
make -C fw            # builds firmware to fw/out/firmware.hex
make -C sim           # compiles RTL + tb, runs sim, dumps build/waves.vcd
gtkwave build/waves.vcd
```

**Option B: Manual (Icarus Verilog)**
```bash
# 1) Build firmware (creates fw/out/firmware.hex)
riscv64-unknown-elf-gcc -Os -march=rv32i -mabi=ilp32   -T fw/linker.ld -nostdlib -Wl,--gc-sections   fw/tests/search_replace.S -o fw/out/firmware.elf
riscv64-unknown-elf-objcopy -O verilog fw/out/firmware.elf fw/out/firmware.hex

# 2) Compile RTL + Testbench
mkdir -p build
iverilog -g2012 -o build/sim.vvp -I rtl -I sim   $(find rtl -name "*.v") sim/tb/tb_top.v

# 3) Run and inspect
vvp build/sim.vvp
gtkwave build/waves.vcd
```

### Build & Run — FPGA (Zybo Z7-10)
```bash
# Batch bitstream build
vivado -mode batch -source fpga/zybo_z7/build.tcl

# Program via hardware server (Vivado GUI or hw_server + openocd, per your setup)
# Memory init: IMEM/DMEM are synthesized with $readmemh from fw/out/firmware.hex
# Ensure build.tcl copies fw/out/firmware.hex into the synthesis directory.
```

> Put your board constraints in `fpga/zybo_z7/zybo_z7.xdc` (clock pins, LEDs, buttons, UART).

---

## Firmware & Loading Flow
- **IMEM init**: `$readmemh("fw/out/firmware.hex", imem.mem)` in `imem.v` (sim). Vivado `read_mem` / pre‑init for FPGA.
- **DMEM layout (example)**:
  - `0x1000_0000`: `N` (array length)
  - `0x1000_0004`: `TARGET` (word to find)
  - `0x1000_0008`: `REPLACEMENT`
  - `0x1000_0100`: `ARRAY[N]`
  - `0x1000_0FFC`: `COUNT` (number of replacements)
- **Search‑and‑replace** program scans `ARRAY`, replaces matches with `REPLACEMENT`, increments `COUNT`. Useful directed test to validate load/store/branch/ALU paths.

---

## Configuration

### ADPS CSRs
> Suggested map — update to match your RTL.
| CSR Name           | Addr   | R/W | Description |
|--------------------|--------|-----|-------------|
| `CSR_ADPS_CTRL`    | 0x7C0  | R/W | Bit0: enable; Bits[3:2]: mode (00=1x64, 01=2x32, 10=4x16, 11=auto) |
| `CSR_ADPS_THRESH_HI` | 0x7C1| R/W | Activity high threshold |
| `CSR_ADPS_THRESH_LO` | 0x7C2| R/W | Activity low threshold |
| `CSR_ADPS_HYST`    | 0x7C3  | R/W | Hysteresis window (ops) |
| `CSR_ADPS_REFRACT` | 0x7C4  | R/W | Refractory (cycles before another switch) |
| `CSR_ADPS_STATUS`  | 0x7C5  |  R  | Mode, recent activity, last switch stamp |
| `CSR_ADPS_CNTS0`   | 0x7C6  |  R  | Mode dwell counters / switch counts |
| `CSR_ADPS_CNTS1`   | 0x7C7  |  R  | Stall/underflow/overflow counters |

### Operating Modes
- **Fixed**: force **1×64**, **2×32**, or **4×16** (for A/B energy vs perf experiments).
- **Auto**: ADPS selects mode based on recent activity windows and thresholds.

### Tuning Parameters
- **Hysteresis**: minimum dwell in current mode to prevent flapping.
- **Refractory**: cooldown after a switch before considering another change.
- **Hi/Lo thresholds**: define “busy” vs “idle/fine‑grained” criteria.

---

## ADPS Controller — Algorithm
Pseudo‑code (reference your RTL FSM implementation):
```text
on every cycle:
  activity  = instr_class != NOP ? 1 : 0      # or use operand toggles / ALU opcodes
  win_sum   = decay(win_sum) + activity       # e.g., EMA or sliding window
  if (now - last_switch) < refractory: continue

  if mode == WIDE and win_sum < THRESH_LO:
      switch_to(NARROW); last_switch = now
  elif mode == NARROW and win_sum > THRESH_HI:
      switch_to(WIDE);   last_switch = now

  # Optional mid mode: 2×32 between 1×64 and 4×16
  # Apply hysteresis: require dwell >= HYST before any switch
```
> Replace with your exact counters (e.g., EMA alpha, window length) and the FSM graph in `docs/figs/adps_fsm.svg`.

---

## Benchmarks & Methodology
- **Workloads**: mix of integer kernels (search‑replace, sort), MAC/ML micro‑kernels (dot, matmul), and control‑heavy loops.
- **Perf metric**: cycles, IPC, total runtime; report both **mean** and **median**.
- **Energy metric**: FPGA board rails via INA/XADC; same clock/voltage; exclude programming time.
- **Protocol**:
  1. Warm‑up run (discard).
  2. 10 trials per mode (fixed 1×64 / 2×32 / 4×16 / auto).
  3. Report geo‑mean energy and perf; include 95% CI.
- **Fairness**: identical binaries; ADPS thresholds fixed across runs unless swept in a separate experiment.

---

## Results (fill with your data)
Example table — **replace with your actual numbers**.

| Mode     | Energy (J) | ΔEnergy vs 1×64 | Time (ms) | ΔTime | Notes |
|----------|------------|------------------|-----------|-------|-------|
| 1×64     | 1.00       | —                | 10.0      | —     | Baseline |
| 2×32     | 0.92       | −8%              | 10.1      | +1%   | |
| 4×16     | 0.85       | −15%             | 10.2      | +2%   | |
| Auto     | 0.84       | −16%             | 10.1      | +1%   | ADPS |

> If you measured ~**12–17% energy savings** with **<1–2% perf overhead**, summarize that here and add per‑kernel plots in `docs/figs/`.

---

## Known Limitations
- Single‑cycle micro‑architecture (throughput limited at higher clocks).  
- ADPS currently targets ALU; memory & branch units are fixed‑precision.  
- No caches/MMU; memory is BRAM‑backed IMEM/DMEM for simplicity.

---

## Roadmap
- [ ] Pipeline the core (2–5 stages) and extend ADPS to multi‑stage ALU.
- [ ] Add **per‑op** precision hints via custom CSR or instruction prefix.
- [ ] Integrate **verilator** CI (lint + unit tests).
- [ ] Add on‑board UART printf & LEDs for demo telemetry.
- [ ] Formal checks for CSRs & ADPS FSM (SymbiYosys).

---

## Contributing
PRs welcome! Please:
1. Run `make -C sim` (tests must pass).
2. Include waveform or counter evidence for fixes/features.
3. Update docs/figs if you change interfaces or CSRs.

---

## License
Choose one and add a `LICENSE` file (MIT/BSD‑3‑Clause/Apache‑2.0).

---

## Acknowledgements
- RISC‑V community and open‑source RTL projects for inspiration.
- Digilent Zybo Z7‑10 reference designs and documentation.
- Contributors and reviewers who provided feedback on the ADPS design.
