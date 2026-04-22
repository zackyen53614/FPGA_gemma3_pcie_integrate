# vhk158_llm_accelerator

FPGA-based LLM inference accelerator on the **Xilinx Versal HBM VHK158** (`xcvh1582-vsva3697-2MP-e-S`). The system connects a host CPU to the FPGA over PCIe, runs the LLM compute pipeline in the programmable logic using HBM for weight storage, and exposes a software API for inference on the host.

## System Overview

```
┌─────────────────────────────────────────────────────────┐
│  Host Machine                                           │
│  ┌──────────────┐    ┌──────────────────────────────┐   │
│  │ host_software│───>│ host_firmware (PCIe / XDMA)  │   │
│  └──────────────┘    └──────────────┬───────────────┘   │
└─────────────────────────────────────│───────────────────┘
                                      │ PCIe
┌─────────────────────────────────────▼───────────────────┐
│  VHK158 (Versal HBM)                                    │
│  ┌──────────────────────┐   ┌─────────────────────────┐ │
│  │ accelerator_hardware │   │ accelerator_firmware    │ │
│  │ (PL: RTL + HBM + IP) │   │ (APU/RPU embedded code) │ │
│  └──────────────────────┘   └─────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## Repository Structure

| Directory | Contents |
|-----------|----------|
| [`accelerator_hardware/`](accelerator_hardware/) | Vivado project — RTL, block designs, constraints, custom IP cores |
| [`accelerator_firmware/`](accelerator_firmware/) | Firmware for Versal embedded processors (APU/RPU) |
| [`host_firmware/`](host_firmware/) | Host-side PCIe driver / XDMA layer |
| [`host_software/`](host_software/) | Host application — model loading, inference, result handling |

## Target Device

| Field | Value |
|-------|-------|
| Part | `xcvh1582-vsva3697-2MP-e-S` |
| Board | VHK158 (Versal HBM) |
| Tool | Vivado 2024.2 |

---

## accelerator_hardware

Vivado project containing the full programmable logic design.

### Structure

```
accelerator_hardware/
  vhk158_llm_accelerator.xpr            — Open this in Vivado
  vhk158_llm_accelerator.srcs/
    constrs_1/                           — XDC timing and pin constraints
    sim_1/                               — Simulation testbenches
    sources_1/
      bd/                                — Block designs
      imports/                           — RTL source files
  vhk158_llm_accelerator.ip_user_files/ — IP simulation stubs
  scripts/                               — Vivado batch TCL (synth/impl/pdi/lint)
  Makefile
```

### Block Designs

| BD | Purpose |
|----|---------|
| `cpm_pcie` | CPM PCIe host interface with AXI NoC and HBM |
| `bram_group` | On-chip BRAM array |
| `sram_sp` / `sram_tp` | Single-port / true-dual-port SRAM wrappers |
| `fifo_1/2/3` | AXI-stream FIFOs |

### Custom Floating-Point IP Cores

| IP | Operation |
|----|-----------|
| `FPGA_FP32_ADDER` | FP32 addition |
| `FPGA_FP32_MULTIPLIER` | FP32 multiplication |
| `FPGA_FP32_DIVIDER` | FP32 division |
| `FPGA_FP32_RECIP` | FP32 reciprocal |
| `FPGA_FP32_INVSQRT` | FP32 inverse square root |
| `FPGA_FP32_EXPONENT` | FP32 exponent (e^x) |
| `FPGA_BF16_ADDER` | BF16 addition |
| `FPGA_BF16_MULTIPLIER` | BF16 multiplication |
| `FPGA_BF16_SUBTRACTOR` | BF16 subtraction |
| `FPGA_BF32_COMP` | BF32 comparator |
| `FPGA_FP16TO32` | FP16 → FP32 conversion |

### Build

Run all targets from inside `accelerator_hardware/`. Requires `vivado` on `PATH`.

```bash
cd accelerator_hardware
make synth        # synthesise RTL
make impl         # place and route
make pdi          # generate Versal device image
make sim-all      # run all IP simulations (xsim)
make sim IP=FPGA_FP32_ADDER SIM=questa   # single IP, specific simulator
make lint         # Vivado syntax check
make clean        # remove logs and run outputs
```

---

## CI (GitHub Actions)

Defined in [.github/workflows/ci.yml](.github/workflows/ci.yml). Each job gates the next:

```
push / PR  →  Lint (Verilator)  →  IP Simulation  →  Synthesis  →  Impl + PDI
                  (ubuntu)          (self-hosted)    (main only)    (tags only)
```

| Job | Runner | Description |
|-----|--------|-------------|
| **Lint** | `ubuntu-latest` | Verilator syntax check — no Vivado licence needed |
| **IP Simulation** | self-hosted `vivado` | `make sim-all SIM=xsim` for all custom IP cores |
| **Synthesis** | self-hosted `vivado` | `make synth` on every push to `main` |
| **Impl + PDI** | self-hosted `vivado` | `make pdi` on version tags (`v*`); uploads `.pdi` as release artifact |

The self-hosted runner must have Vivado 2024.2 installed and be labelled `self-hosted, vivado`.

---

## Getting Started

```bash
# 1. Clone
git clone https://github.com/Frank-Tsai-23026407/vhk158_llm_accelerator.git
cd vhk158_llm_accelerator

# 2. Open hardware project in Vivado
cd accelerator_hardware
vivado vhk158_llm_accelerator.xpr
# Vivado will re-generate IP outputs on first open

# 3. Build
make synth
make impl
make pdi
```
