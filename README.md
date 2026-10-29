# Hi! I'm Wilberth Gutiérrez Montero 😄
#### I am an **Electronics Engineer** specializing in **RTL Design**, **SystemVerilog/UVM Functional Verification**, and **SoC Architecture**. My focus is on designing efficient hardware modules and building robust, scalable verification environments to ensure bug-free silicon and high-reliability digital systems.

📍 **Location:** Costa Rica | 📧 **Email:** wilberthgutierrez020502@gmail.com | 💼 **LinkedIn:** www.linkedin.com/in/wilberth-gutierrezm |

<br>

## 🛠️ Technical Skills

| Category | Technologies, Methodologies & Tools |
| :--- | :--- |
| **VLSI, RTL & Architecture** | SystemVerilog, RTL Design, Digital Logic, Computer Architecture, Digital Algorithms (ALU, Arbitration), FSM, Pipelining, RISC-V, Microarchitecture, Clock Domain Crossing (Synchronizers, Gray-coding), FPGA Prototyping, Physical Implementation (Place & Route), Static Timing Analysis (STA), Timing Closure, RTL Linting, HAS/MAS Specification |
| **Functional Verification** | UVM (Agents, Sequencers, Drivers, Monitors, Scoreboards, Coverage Collectors, TLM / Analysis Ports, `uvm_config_db`), SystemVerilog Assertions (SVA), Constrained-Random Stimulus, Functional & Code Coverage, Coverage Closure, Test & Verification Planning, Regression Debug, Reference-Model / ISS Co-simulation |
| **Protocols & Interfaces** | AXI, UART, SPI, I2C |
| **EDA & Simulation Tools** | Synopsys VCS, Synopsys Verdi (Interactive Waveform Debugger), AMD Vivado, QuestaSim, SpyGlass RTL Linting, Synopsys Fusion Compiler |
| **Languages & Scripting** | Python, C/C++, Shell Scripting, MATLAB, Linux, Git |

<br>

# 💻 RTL Design and UVM Verification Projects

#### 🤖 I developed these projects from scratch (excluding university research projects); however, all the code was reviewed using AI to obtain feedback (excluding university research projects), as well as to expedite debugging and the generation of documentation. All changes and corrections made to the code based on AI suggestions (Claude Opus 5.5) are documented via comments that include the tag [AI Feedback]. 🤖

#### 🔒 This portfolio does not include projects related to past or current employment due to confidentiality. 🔒

## 💻 RTL Design Projects

### 🔹 1. [RTL/SystemVerilog] [RISC-V RV32I-Zicsr Multicycle Processor Standardization (Siwa MCU Core)]

- **Overview:**
  Took the CPU of Siwa, Costa Rica's first 32-bit microcontroller (implantable biomedical use), from a low-power custom core to a **fully standard RV32I + Zicsr, Machine-mode** processor. The result runs unmodified `riscv-arch-test` programs and the stock GNU toolchain, and it can be reused as an independent IP core. The trade-off, measured with logic synthesis on TSMC 65nm: **+10.6% area, +47% power**, in exchange for a robust timing profile (**+11.67 ns setup slack with no time borrowing**, versus 0.00 ns held only by latch borrowing in the original).

- **Key Contributions & Architecture:**
  - **Latches → flip-flops:** Replaced a register file built from 1,024 hand-instantiated latches and tri-state gates with a parameterized `always_ff` array (x0 hardwired to 0). This removed the latch time-borrowing dependency and made timing predictable.
  - **Standard CSR subsystem without a 4096-entry bank:** Implemented 12-bit CSR addressing with a combinational `csr_decode` that maps standard addresses to a 5-bit local index. This gives 21 CSRs (`mstatus`, `mtvec`, `mie`, `mip`, `mepc`, `mcause`, `mtval`, `mscratch`, `misa`, `mcycle[h]`, `minstret[h]`, plus legacy custom ones) at a fraction of the area. The three non-standard `mcauseA/B/C` registers became a single standard `mcause`.
  - **Spec-accurate trap handling:** Split the single catch-all address error into distinct exception codes. Added real `ECALL`/`EBREAK` FSM states (before, they fell through as illegal instructions and only looked correct). Fixed `MRET` to restore MIE←MPIE and MPP. Populated `mtval`, including for external interrupts.
  - **Timer moved into the CSR block:** `mtime`/`mtimecmp` now live next to the compare logic, so the timer starts counting independently of boot and the interrupt path is shorter.
  - **IP-core boundary:** Pulled the 65-bit data bus out of the CPU into the SoC top, and widened internal addressing from 25 to 32 bits to remove silent address truncation without touching the external bus.
  - **Single source of truth:** Created `definitions_pkg` holding the CSR maps, memory map, exception/interrupt encodings, and sizing parameters, so an architectural change touches one file.
  - **Six RTL bugs root-caused and fixed** (all found by UVM, none by manual validation):
    - **JAL:** `write_reg_fl` and `s_reg_w` were registered one cycle apart, so rd received the immediate instead of PC+4. Moved the write to the earlier FSM state.
    - **LBU/LHU:** the address offset was zero-extended. It must always be sign-extended, while zero-extension belongs to the loaded data in the MBC.
    - **CSRRW:** the combinational CSR output reflected the new value before the old one was captured. Added a shadow register to restore atomic swap.
    - **JALR with rs1 = rd:** the rd write corrupted the jump-target source. Added a target-holding register.
    - **mtval = X on misaligned load/store:** a floating port plus a wrong cause code. Reconnected the port and added `LD_MISALIGN`/`ST_MISALIGN` states.
    - **FENCE:** it hit the decoder default and raised an illegal-instruction trap. Now decoded as a NOP, which is valid for an in-order single-hart core.
  - **Lint and implementation quality:** SpyGlass went from **56 errors / 213 warnings to 0 / 0**. Real latch inferences were fixed, and the intentional FSM latches were kept and waived because removing them broke timing. Hold violations that remain after synthesis are expected before place-and-route.


- 📂 **[Access Unavailable: confidentiality of the University Research]**

### 🔹 2. [RTL/SystemVerilog] [NetSwitch-4: MAC-to-Core Ethernet Switching Subsystem (5 chained IP blocks)] [In Progress]
 
- **Overview:**
  A mini Ethernet switch datapath built as **five chained, independently specified IP blocks**, each owning one hard problem of switch design: clock domain crossing, register access, line-rate packet parsing, buffered switching, and egress traffic management. Every block ships with a HAS/MAS, synthesizable RTL with SVA, and quantitative evidence (Fmax, area, MTBF, throughput) instead of "it simulates". The reference system moves 64-bit AXI-Stream frames between a recovered MAC clock (156.25 MHz, ±100 ppm), a 250 MHz core, and a 100 MHz control domain.
- **Key Contributions & Architecture:**
  - **1. CDCKit — CDC library and asynchronous FIFO** *(RTL complete, detailed below):* Safe transfer of levels, resets, events, words, and streams across unrelated clocks.
  - **2. Statistics CSR block (AXI4-Lite + APB):** Slave that decouples AW and W channels and responds only when it holds both. Register types cover RW, hardware RO, W1C interrupts, and clear-on-read without losing a same-cycle event. 64-bit counters are read atomically through a high-half latch, and cross in from their own clock domain through the CDCKit handshake. *Planned.*
  - **3. Ethernet RX parser with parallel CRC-32 (AXI-Stream, 64-bit):** Three-block FSM with registered Moore outputs, VLAN/PCP extraction without latches, and runt detection. The FCS check compares a lookahead parallel CRC against the chained baseline, with the **Fmax/area trade-off measured, not asserted**. Skid buffers break the `tready` path. *Planned.*
  - **4. 4×4 fabric with VOQ, linked-list shared buffer, and iSLIP:** Cell-based shared memory (free list, next-pointer RAM, per-VOQ head/tail/count), iterative matching arbiter, and one-hot crossbar. Switchable to an input-FIFO mode to **reproduce the classic head-of-line blocking throughput gap** against VOQ+iSLIP. *Planned.*
  - **5. QoS egress port:** Eight-class strict-priority + deficit round-robin scheduler counting bytes, fixed-point token-bucket shaper, full Priority Flow Control (hysteresis generation, per-priority pause timers, watchdog), credit interface to the fabric, and a UPF-defined power domain sequenced by an always-on FSM. *Planned.*
  - 
#### 🔸 Deep dive: Project 1 — CDCKit (RTL) [In Progress]
 
- **Overview:**
  A parameterizable library of five clock-domain-crossing primitives (level, reset, pulse, multi-bit word, and streaming FIFO), each matched to the signal class it can cross safely. The design goal was to make **every crossing correct by construction, so that no simulation luck is needed**: the structure itself rules out the failure.

- **Key Contributions & Architecture:**
  - **Asynchronous FIFO (Cummings-style):** Gray pointers are **registered before crossing**, so the synchronizer never sees combinational glitches from the binary-to-Gray conversion, and a metastable sample can only resolve to the old or the new pointer. Full/empty are computed from the *next* pointer for single-cycle response and are **conservative by construction**: a stale remote pointer can only hide space or data, never invent it, so overflow and underflow are structurally impossible.
  - **First-word fall-through on distributed RAM:** Asynchronous read puts the head word on the output with zero read latency, trading BRAM efficiency for a simpler consumer. An optional gate blocks the memory-to-output path when empty.
  - **Reset crossing:** Async-assert / sync-deassert synchronizers per domain, with a scan bypass. Independent resets between FIFO sides are left as a documented reset-domain-crossing hazard, not silently ignored.
  - **Pulse crossing:** Toggle-based, because a one-cycle pulse can be missed entirely by a slower clock. An arming window masks the false edge the destination sees when its synchronizer converges after reset.
  - **Multi-bit word crossing:** Four-phase handshake where **only request and acknowledge are synchronized, never the data**. A hold register keeps the word stable for the whole transaction, trading throughput for minimal area and any data width.
  - **Synthesis-aware RTL:** Synchronizer flops are protected from retiming and shift-register packing, and the Gray pointer registers from **merging with their binary twins** (identical MSB). That merge is silent and breaks the one-bit-change guarantee the whole FIFO relies on.
  - **Shared package: A single package holds the binary-to-Gray and Gray-to-binary conversions, so both FIFO pointer managers reuse one verified implementation and a width or encoding change touches one file.
  - **Embedded SVA:** Bound assertions for single-bit pointer change, no push when full, no pop when empty, handshake ordering, and minimum pulse spacing, kept out of the design source.

#### 🔸 Deep dive: Project 2 — Statistics CSR block (AXI4-Lite + APB) (RTL) [Planned]

#### 🔸 Deep dive: Project 3 — Ethernet RX parser with parallel CRC-32 (AXI-Stream, 64-bit) (RTL) [Planned]

#### 🔸 Deep dive: Project 4 —  4×4 fabric with VOQ, linked-list shared buffer, and iSLIP (RTL) [Planned]

#### 🔸 Deep dive: Project 5 — QoS egress port (RTL) [Planned]

<br>


## 💻 UVM Verification Projects

### 🔹 1. [UVM/SystemVerilog] [Verification Environment for a RISC-V Core with Spike Golden-Reference Co-Verification]

- **Overview:**
  Built a UVM 1.2 environment that verifies the standardized core by **comparing memory signatures against the Spike ISS** instead of writing a custom reference model, so correctness follows the official ISA and not my own interpretation. The regression runs **60 tests** (52 `riscv-arch-test` + 8 directed), **all passing**. It exposed **six RTL bugs** that manual waveform and register-dump review had missed. Coverage closed at **100% functional**, **100% FSM**, and **>91% line/branch/toggle/condition**, with every exclusion justified.

- **Key Contributions & Architecture:**
  - **Signature-based checking:** Spike and the DUT run the same binary. The monitor captures stores inside the `begin_signature`/`end_signature` window, and the scoreboard diffs `DUT-uvm.signature` against `DUT-spike.signature`. This is the same method used by official RISC-V conformance suites.
  - **Passive + active agents:** A passive `core_agent` observes the memory bus and feeds the scoreboard and coverage collector through one analysis port. An active `int_agent` injects MEIP/MAIP as precisely timed pulses.
  - **Self-terminating tests:** The scoreboard watches the instruction bus for the halt-loop opcode (`0xFF9FF06F`), waits 3 cycles for the last store to land, drops the objection, and compares in `check_phase`. It also reports per-test **CPI** from `mcycle`/`minstret`.
  - **Handling what Spike cannot model:** Asynchronous interrupts, access faults, and the boot sequence depend on multicycle timing, so I wrote 8 directed tests. The UVM driver emulates the SPI-side boot protocol, and expected results were derived from the privileged spec and not from the DUT.
  - **Meaningful functional coverage:** Opcode groups, data-value ranges, and interrupt crosses, including **interrupt arriving during an active store**, a nasty corner case for a multicycle core.
  - **Coverage closure with discipline:** Merged per-test VCS databases with `urg`. Exclusions are limited to structurally unreachable or irrelevant logic (boot FSM, peripheral outputs, constant CSR fields), and the reasons are documented.
  - **Verification plan:** 13 features, **28 corner cases** (e.g., silent overflow, `x0` aliasing, PC wrap at `0xFFFFFFFC`, WPRI bits, simultaneous MAIP+MEIP), measurable exit criteria, and a rule of zero unexplained scoreboard mismatches.
  - **Automation:** The regression compiles once, loops over a CSV of tests and signature ranges, decides PASS/FAIL from `UVM_ERROR`/`[FAIL]` markers, and produces consolidated HTML coverage.

- 📂 **[Access Unavailable: confidentiality of the University Research]**


---

### 🔹 2. [UVM/SystemVerilog] [Verification Environment for a UART Communication Protocol)

- **Overview:**
  Built a UVM 1.2 environment for a UART with parallel FIFO interfaces, checking both TX and RX paths end to end under random delays and toggle-focused patterns. Verification exposed **DUT bugs caused by hardcoded signal widths**. I **parameterized the DUT** and revalidated it across **multiple baud rates and FIFO widths** by changing a single `DATA_FIFO` define.

- **Key Contributions & Architecture:**
  - **Two independent agents:** The TX agent drives the parallel side and its monitor decodes the serial output. The RX agent serializes frames into the DUT and its monitor observes the received parallel word.
  - **Cycle-accurate protocol modeling:** The driver emulates the `pndng_T`/`pop_T` handshake. The RX driver generates 11-bit frames (start, 8 data, parity, stop) at 16 samples per bit. The monitors sample at mid-bit for stability. A local copy of the baud clock was needed in the testbench because the DUT generates it too late to synchronize against.
  - **Width-agnostic scoreboard:** Two `uvm_analysis_imp` ports assemble bytes into full `DATA_FIFO`-wide words and compare them in both directions. It also checks start/parity/stop bits and exports CSV reports (delay, sent, expected).
  - **Root-causing hardcoded widths:** Scoreboard mismatches that appeared only at some FIFO sizes pointed to hardcoded widths in the DUT. I replaced them with parameters (data width, FIFO depth, clock divider, baud) and propagated them consistently through interfaces, drivers, monitors, and sequences.
  - **Targeted stimulus:** Constrained-random delay ranges plus a dedicated toggle sequence (`0x00/0xFF/0x55/0xAA`) to hit toggle coverage. Behavior is configurable at runtime with plusargs (`+num_ns`, `+num_t`, `+t_delay`, `+data_TX/RX`).
  - **Coverage and assertions:** `uvm_subscriber` collectors for both interfaces, SVA checks for X/Z on pushed FIFO data and TX idle state, and a final reset toggle to close a reset-coverage hole. VCS collects line/toggle/cond/FSM/branch/assert coverage.

- 📂 **[Access Unavailable: confidentiality of the University Research]**

---


