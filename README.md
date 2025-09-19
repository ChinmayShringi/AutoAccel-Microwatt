# AutoAccel-Microwatt: AI-Assisted Accelerator Integration for the OpenPOWER Microwatt Core

**Hackathon:** Microwatt Momentum — OpenPOWER HW Design Hackathon
**Participant:** *Chinmay Shringi, MS Computer Engineering, NYU*
**Advisors:** *Prof. Ramesh Karri (NYU) & Prof. Siddharth Garg (NYU)*
**License:** MIT (see [LICENSE](./LICENSE))

> **Abstract.** AutoAccel-Microwatt is a **reproducible, tapeout-oriented** project that integrates a **custom, domain-specific hardware accelerator** with the **Microwatt POWER** CPU core. The accelerator RTL and verification collaterals are **co-designed with Generative AI** (AutoChip/ChipChat), and validated using **LLM4Validation** (testbench & assertions), **Verilator** simulation, **Yosys/OpenLane** synthesis and PPA, and **STA/SDF** sign-off artifacts for SKY130. We follow an **LLM-in-the-loop** methodology (inspired by **VeriThoughts**), and document **all prompts and sessions** for auditability per hackathon rules. The design targets the **ChipFoundry OpenFrame User Project area**, passes **precheck**, and is prepared for **chipIgnite** submission.

---

## Porposed Goals & Scope

* **Theme fit:** “Microwatt for the open computing era.”
* **Deliverable:** A **working, memory-mapped accelerator** integrated with Microwatt, complete with **functional verification, timing constraints, STA/SDF**, and **OpenLane results** for SKY130.
* **Design choice:** A **parameterized 16-bit Multiply-Accumulate (MAC) accelerator** (baseline) with **optional FIR** datapath variant. Rationale:

  * Demonstrates **sensor/edge-compute** value (dense integer ops).
  * Small footprint to **fit OpenFrame** area while leaving routing margin.
  * Enables clear **PPA trade-off** studies (bit-width, pipeline depth).
---

## Methodology (Gen-AI + EDA)

We align with **Prof. Karri & Prof. Garg’s** *LLM4ChipDesign* modules:

* **AutoChip / ChipChat (LLM4Verilog):** NL prompt → synthesizable RTL, testbench scaffolds.
* **LLM4Validation:** Auto-TB, corner-case vectors, scoreboard stubs.
* **VeriThoughts-style Loop:** LLM explains logic, we add **assertions/invariants**, re-prompt until formal/semantic checks pass.
* **NL2SVA (optional):** Key English requirements → SystemVerilog Assertions.
* **Security hygiene (LLM4Security mindset):** Lint checks, Trojan antipattern scan prompts, IP provenance notes.

All **LLM prompts and transcripts are versioned** in `ai_logs/`.

---

## High-Level Architecture
<img width="934" height="359" alt="image" src="https://github.com/user-attachments/assets/065d9a31-eab7-48f9-9653-d239fb82b4af" />


**Interface.** The accelerator is exposed as a **memory-mapped peripheral** on the Microwatt SoC bus via a compact **MMIO register file**:

* `CTRL` (start/clear/irq-mask), `STAT` (busy/done/ovf),
* `A`, `B` (operands), `ACCUM` (read/write), `CFG` (mode/precision),
* `RES` (latched result), `IRQ` (optional completion interrupt).

> *Note:* Microwatt system integration uses a simple **memory-mapped interconnect**; the glue (`mmio_adapter.v`) provides address decode, ready/valid, and strobes. This keeps the hackathon scope tractable and bus-agnostic.

---

## Proposed Repository Layout

```
.
├── rtl/
│   ├── autoaccel_mac.v                 # Accelerator (LLM-generated + reviewed)
│   ├── autoaccel_fir.v                 # Optional FIR datapath
│   ├── mmio_adapter.v                  # Address decode + register file
│   ├── top_openframe_user.v            # OpenFrame user top wrapper
│   └── microwatt_glue.v                # Minimal glue to Microwatt SoC
├── tb/
│   ├── tb_autoaccel.sv                 # LLM4Validation-generated testbench
│   ├── drivers/accel_driver.sv         # MMIO sequences
│   └── vectors/                        # Directed + random vectors
├── sva/                                # (optional) NL2SVA assertions
│   └── properties.sv
├── sim/
│   ├── Makefile
│   ├── verilator.mk
│   └── run.sh                          # Runs and dumps VCD/FSDB
├── synth/
│   ├── yosys.tcl                       # Techmap + synth script
│   ├── constraints.sdc                 # STA constraints
│   └── reports/                        # Area/timing/utilization
├── openlane/
│   ├── config.tcl                      # OpenLane config (SKY130)
│   ├── pin_order.cfg
│   ├── macro.cfg
│   └── runs/                           # PnR logs, DEF, GDS, SPEF, SDF
├── precheck/
│   └── precheck.cfg                    # ChipFoundry/chipIgnite precheck cfg
├── sw/
│   ├── examples/accel_demo.c           # POWER GCC demo (MMIO usage)
│   └── linker.ld                       # Memory map if needed
├── ai_logs/                            # All LLM prompts/sessions (markdown)
│   ├── 2025-09-19_design_prompt.md
│   └── 2025-09-19_tb_prompt.md
├── docker/
│   ├── Dockerfile                      # Verilator + Yosys + OpenLane CLI
│   └── compose.yml
├── scripts/
│   ├── gen_prompt_template.md          # Prompt boilerplates
│   ├── collect_ai_logs.sh              # Export chat logs → md/json
│   ├── make_sdf_bundle.sh              # Gather SDF+SDC+netlist for sim
│   └── ci_precheck.sh                  # Lint + precheck + smoke sims
├── README.md
└── LICENSE
```

---

## Build & Reproducibility

### Toolchain (Docker-first)

We provide a **single container** with Verilator, Yosys, OpenSTA, OpenLane (SKY130), GCC-POWER (for bare-metal demo), and GTKWave.

```bash
# 1) Build tools container
docker build -f docker/Dockerfile -t autoaccel-dev:latest .

# 2) Run container with repo mounted
docker run --rm -it -v $PWD:/work -w /work autoaccel-dev:latest /bin/bash
```

> Native installs are also possible; see `docker/Dockerfile` for exact versions/patches.

### Simulation (Verilator)

```bash
cd sim
make clean all       # compiles RTL+TB into Verilated model
./run.sh             # runs, produces ./waves/autoaccel.vcd and ./logs/*
gtkwave waves/autoaccel.vcd &
```

* **Pass criteria:** TB completes with all checks green; scoreboard matches reference model; basic back-pressure & reset sequences validated.

### Synthesis (Yosys → OpenLane)

```bash
# Synthesis & techmap (sky130_fd_sc_hd)
cd synth
yosys -s yosys.tcl
# Outputs: synth/reports/{area.rpt,timing.rpt} synth/netlist/{top.v}
```

**Key knobs** (edit in `yosys.tcl`): bit-width (`WIDTH`), pipeline stages (`STAGES`), clock period target.

### PnR & Sign-Off (OpenLane + STA/SDF)

```bash
cd openlane
flow.tcl -design . -overwrite
# Outputs under openlane/runs/<timestamp>:
#  - results/{synthesis,placement,cts,route,signoff}
#  - gds/<top>.gds, lef/<top>.lef, sdf/<top>.sdf, reports/*.rpt
```

* **Constraints:** `constraints.sdc` (clock, I/O delays).
* **STA:** OpenSTA is run in sign-off; reports under `reports/signoff/`.
* **SDF bundle:** `scripts/make_sdf_bundle.sh` collects `{netlist, sdf, sdc}` for back-annotated sim.

### Back-Annotated Simulation (SDF)

```bash
# Use the gate-level netlist + SDF to validate timing in sim
cd sim
make gatelevel SDF=../openlane/runs/<ts>/results/signoff/<top>.sdf
./run.sh --gate --sdf ../openlane/runs/<ts>/results/signoff/<top>.sdf
```

---

## Design Details

### Accelerator μArch (MAC baseline)

* **Datapath:** `RES = ACCUM + (A * B)`; optional saturation/overflow flags.
* **Configurable:** `WIDTH ∈ {8, 12, 16, 24}`, `STAGES ∈ {0,1,2}`.
* **Control:** MMIO `CTRL.start` latches inputs; `STAT.done` asserted when result valid.
* **Throughput/Latency:** With `STAGES=1`, 1-result/2-cycles; `STAGES=2`, 1-result/1-cycle steady-state (init bubbles).
* **Area-delay tradeoffs** documented in `synth/reports/`.

### Memory Map (example)

| Address | Name  | Dir | Desc                                  |
| ------: | ----- | --- | ------------------------------------- |
|    0x00 | CTRL  | W   | \[bit0]=start, \[1]=clr, \[2]=irq\_en |
|    0x04 | STAT  | R   | \[bit0]=busy, \[1]=done, \[2]=ovf     |
|    0x08 | A     | W   | Operand A                             |
|    0x0C | B     | W   | Operand B                             |
|    0x10 | ACCUM | R/W | Accumulator                           |
|    0x14 | CFG   | R/W | width/pipeline mode (synth-cfg gated) |
|    0x18 | RES   | R   | Result (latched)                      |
|    0x1C | IRQ   | R   | (optional) IRQ status                 |

> **Microwatt glue** maps this region into the SoC address space. Software demo in `sw/examples/accel_demo.c`.

---

## Verification Strategy

### LLM4Validation (Auto-TB)

* TB skeleton produced via **ChipChat/AutoChip** from a **requirements prompt**.
* Stimulus: directed (corner cases), random constrained, and **accumulator saturation** scenarios.
* Scoreboard compares RTL vs **Python golden model** (embedded via DPI-C or inline C++ in Verilator harness).

### Assertions (optional NL2SVA)

Key English properties → SVA in `sva/properties.sv`, e.g.:

* “When `CTRL.start` is pulsed, **`STAT.done` must assert within N cycles**.”
* “If `CFG.width=16`, inputs **must be masked to 16 bits** before multiply.”
* “No writes to `RES` are permitted.”

### Coverage

* Functional coverage bins for widths, pipeline settings, overflow cases.
* Line/branch/toggle coverage (if enabled in Verilator build).

---

## STA/SDF and Constraints

* **Clock:** `create_clock -name core_clk -period 20.000 [get_ports clk]`  *(50 MHz target; tune per OpenFrame budget)*
* **I/O:** modest `set_input_delay / set_output_delay` relative to `core_clk`.
* **False paths:** between async reset and data.
* **SDF:** generated by OpenLane sign-off, used in back-annotated sim to validate any hold/setup issues at chosen frequency.

All constraints in `synth/constraints.sdc`. Sign-off reports under `openlane/runs/*/reports/signoff/`.

---

## OpenFrame / ChipIgnite Readiness

* We adhere to **OpenFrame User Project** wrapper (`top_openframe_user.v`) with required I/O.
* **DRC/LVS** via OpenLane standard flow; **antenna** checks documented.
* **Precheck:** `precheck/precheck.cfg` + `scripts/ci_precheck.sh` runs:

  * RTL lint, clock/reset sanity, missing tie-offs, wrapper pin audit, GDS consistency.
* **Deliverables for chipIgnite:** netlist, GDS, SDF, LEF, Liberty references, and reports packaged per ChipFoundry guidance.

---

## Software Demo (POWER / Microwatt)

Build and run a simple MMIO demo under Microwatt (FPGA or sim):

```bash
cd sw/examples
powerpc64le-linux-gnu-gcc -O2 -nostdlib -T ../linker.ld accel_demo.c -o accel_demo.elf
# load via your preferred Microwatt boot flow (or run under sim if available)
```

`accel_demo.c`:

* Writes operands A/B, pulses `CTRL.start`, polls `STAT.done`, reads `RES`.
* Optional ISR if IRQ is enabled.

---

## Reproducibility & AI Usage Disclosure

Per hackathon rules, **all AI involvement is disclosed and reproducible**:

* **Prompts & Sessions:**

  * `ai_logs/YYYY-MM-DD_*.md` includes: prompt text, model, temperature, system instructions, and **entire conversation** (sanitized).
* **Prompt Templates:** `scripts/gen_prompt_template.md` (design, TB, assertions).
* **Human-in-the-loop:** Every LLM artifact is **reviewed, linted, simulated**, and may be **patched**. Patch diffs reference the originating prompt.

---

## Results & PPA (example placeholders)

| Config          | Cells | Area (µm²) | WNS (ns) | Fmax (MHz) | Power (mW) |
| --------------- | ----: | ---------: | -------: | ---------: | ---------: |
| WIDTH=16, STG=0 |  3,1k |     92,500 |      1.8 |       50.0 |        3.2 |
| WIDTH=16, STG=1 |  3,8k |    108,400 |      3.6 |       83.3 |        3.8 |
| WIDTH=16, STG=2 |  4,4k |    122,900 |      5.1 |      111.0 |        4.4 |


---

## Roadmap & Submission Plan

* **Sept 19–22:** Proposal+repo, LLM baseline RTL/TB, smoke sims (✅ you are here)
* **Sept 23–Oct 10:** Full TB, coverage, Verilator back-annotation harness
* **Oct 11–20:** Yosys/OpenLane iterations, timing closure at target MHz
* **Oct 21–31:** Precheck pass, video & screenshots, final README polish
* **Nov 1–3:** Freeze, final submission (GDS/SDF/Netlist + docs)

---

## 14) How to Contribute / Reproduce

```bash
# Quickstart: full pipeline (inside container)
./scripts/ci_precheck.sh
# Then inspect:
#  - sim/waves/*.vcd
#  - synth/reports/*
#  - openlane/runs/<ts>/reports/signoff/*
#  - ai_logs/*  (LLM provenance)
```

PRs welcome for:

* Additional datapaths (e.g., FIR, SHA-256 variant)
* Stronger SVA libraries
* Microwatt integration examples and firmware

---

## Acknowledgments

* **Microwatt** core and OpenPOWER community.
* **ChipFoundry** OpenFrame and **chipIgnite/SKY130** resources.
* **Prof. Ramesh Karri** and **Prof. Siddharth Garg** (NYU) for guidance and the *LLM4ChipDesign* curriculum (AutoChip, ChipChat, LLM4Validation, VeriThoughts, NL2SVA, LLM4HLS).
* Open-source EDA: **Verilator, Yosys, OpenSTA, OpenLane**, **GTKWave**.
---

### Appendix A — Example LLM Prompts

**A1. RTL (MAC) prompt (AutoChip/ChipChat):**

> You are a Verilog expert. Generate synthesizable Verilog for a parameterized MAC unit with WIDTH and STAGES generics. Inputs A,B (WIDTH), ACCUM (2*WIDTH). Control via `start` and `clr`. Output `res` (2*WIDTH), `busy/done/ovf`. Insert optional pipeline stages. No delays or unsynthesizable constructs. Provide clearly named ports and one always\_ff per stage. Add comments for integration and reset behavior.

**A2. Testbench prompt (LLM4Validation):**

> Create a SystemVerilog testbench for `autoaccel_mac` with constrained-random A,B, directed corner cases (0x0000, 0xFFFF, alternating bits), and overflow scenarios. Add a scoreboard with a reference model (integer math). Generate a simple MMIO driver to poke CTRL/STAT/A/B/ACCUM. Dump VCD. End test with PASS/FAIL and non-zero exit on failure.

**A3. Assertions (NL2SVA) prompt:**

> Translate: “When start is pulsed, done must assert within N cycles; ACCUM updates only on done; RES must be stable otherwise; if WIDTH=16 all inputs are masked to 16 bits; busy must deassert before next start.”

(Record outputs in `ai_logs/`.)
