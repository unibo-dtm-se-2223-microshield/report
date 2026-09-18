---
title: Validation
has_children: false
nav_order: 6
---

# 5. Validation

## 5.1 Testing Strategy & Requirements Traceability

The validation phase provides empirical evidence that MicroShield satisfies its operational requirements across its two execution environments: the bare-metal C99 edge runtime executing on the ARM Cortex-M4 microcontroller and the asynchronous Python MLOps supervisor hosted on the workstation.

### 5.1.1 Multi-Tier Verification Lifecycle

While individual translation units and Python classes underwent functional unit verification during implementation (as detailed in Chapter 4), system validation follows an empirical, multi-tiered pipeline progressing from virtual simulation to physical in-silico benchmarking:

<pre style="line-height: 1.15; font-size: 0.9em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #f8f9fa; padding: 14px 18px; border-radius: 6px; border: 1px solid #e2e8f0; overflow-x: auto;">
    ┌─────────────────────────┐        ┌─────────────────────────┐        ┌─────────────────────────┐
    │ 1. Modular Quality Gate │        │ 2. Software-in-the-Loop │        │ 3. Hardware-in-the-Loop │
    │    (Regression Suite)   │ ───►   │    (SIL Integration)    │ ───►   │    (Physical Silicon)   │
    │  make test (35/35 pass) │        │  Raw Frames -> C -> Py  │        │  DWT Benchmarks, MatMul │
    └─────────────────────────┘        └─────────────────────────┘        └─────────────────────────┘
</pre>

### 5.1.2 Traceability Matrix to System Requirements

Every automated verification target, desktop integration harness, and silicon benchmark traces directly to the formal engineering requirements established in the project specification:

| Requirement ID | Operational Constraint | Validation Target | Acceptance Target / Margin | Verification Harness |
| :--- | :--- | :--- | :--- | :--- |
| **R-EDGE-01** | Bounded Real-Time Latency | Inference & Traversal | WCET &le; 50.0 &mu;s (&ge; 20% margin) | In-Silicon DWT Cyccnt |
| **R-EDGE-02** | Zero Dynamic Memory | Static RAM Footprint | 0 bytes dynamic heap; 0 bytes System SRAM | GCC Linker Map Audit |
| **R-EDGE-03** | Non-Blocking Egress | DMA Buffer Integrity | Zero CPU wait states during alert dispatch | DWT Timing Profiling |
| **R-TRANS-01** | Wire Transport Integrity | CRC32 Verification | Exact IEEE 802.3 bit residue matching | Cross-Language SIL Stream |
| **R-TRANS-02** | Frame Delineation | COBS Byte Stuffing | Framing delimiter 0x00 guaranteed unique | Cross-Language SIL Stream |
| **R-ML-01** | Structural Depth Ceiling | AST Tree Depth | Depth &le; 6 comparisons (&le; 64 leaves) | Transpiler AST Validator |
| **R-ML-02** | Header Transpilation | Code Generation | Static const Flash matrices in .rodata | Native GCC Compilation |
| **R-DRIFT-01** | Drift Surveillance | Rolling Ambiguity | Window W=100, drift asserted at &tau; &gt; 5.0% | Sliding-Window Monitor |
| **R-UI-01** | Operator Visibility | Reactive UI Console | 1 Hz decoupled state polling; live retrain | Containerized Dashboard |
| **R-PWR-01** | Ultra-Low Power Budget | Core Sleep Duty Cycle | Active duty cycle &le; 1.0% (&ge; 99% sleep) | Analytical DWT Model |

---

## 5.2 Software-in-the-Loop (SIL) System Integration

Following component-level verification in Chapter 4, the entire software vertical slice was validated in an integrated Software-in-the-Loop (SIL) environment on the Linux host platform before flashing firmware to target hardware.

### 5.2.1 Unified Quality Gate Status

The complete automated regression suite serves as the initial deployment gate. Executing the centralized top-level targets asserts that all modular contracts hold simultaneously across both toolchains:

<pre style="line-height: 1.25; font-size: 0.85em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #1e293b; padding: 14px 18px; border-radius: 6px; border: 1px solid #334155; overflow-x: auto; color: #f8fafc;">
<span style="color: #94a3b8;">wearemassive@wearemassive:~/microshield/artifact$</span> <span style="color: #38bdf8;">make test</span>
cd edge &amp;&amp; make test
--- Running MicroShield Edge C99 Unit Test Suites ---
[PASSED] test_features (Normalization, Timing Delta, Rollover, Variance)
[PASSED] test_engine (Tree Traversal, Null Pointer Safety, XAI Mapping)
[PASSED] test_cobs (IEEE 802.3 CRC32, Framing Roundtrip, Telemetry Serialization)
cd supervisor &amp;&amp; poetry run pytest -v tests/ &amp;&amp; poetry run mypy --strict dashield/ tests/
============================== 23 passed in 1.14s ==============================
Success: no issues found in 19 source files
--- ALL 35 UNIT AND CONTRACT TESTS PASSED (100% SUCCESS RATE) ---
</pre>

### 5.2.2 End-to-End Ingestion, Transpilation & Telemetry Streaming

To validate system-level interoperability without hardware peripherals, a POSIX IPC pipeline was established connecting the native C99 edge engine directly to the Python supervisory daemon:

1. **Synthetic Adversarial Playback:** A raw frame generator injects representative industrial traffic: nominal Modbus frames interspersed with volumetric line floods and high-entropy random payloads.
2. **Native Edge Processing:** The compiled edge engine (`libmicroshield.a`) ingests raw byte streams, calculates normalized features, traverses the static decision tree, and serializes alerts via COBS into standard output.
3. **Supervisory Framing & Drift Surveillance:** The Python ingestion worker decodes the binary stream via standard input, confirms CRC32 integrity, and populates the rolling ambiguity window.
4. **Reactive Operator Presentation:** The Plotly Dash console visualizes incoming feature vectors and ambiguous cases. When injected ambiguity exceeds 5.0%, the interface transitions dynamically to `DRIFT DETECTED`, enabling the operator to trigger model re-fitting.

---

## 5.3 Hardware-in-the-Loop (HIL) Metrology & Heavy Workload Benchmarking

Physical execution metrics were gathered directly on the target silicon using the STMicroelectronics NUCLEO-F407RE development platform.

### 5.3.1 Physical Hardware Testbed Setup & Breadboard Interconnect

The physical testbed connects the STM32 microcontroller to external status indicators and operator controls without permanent modifications, utilizing contiguous pins on the Arduino CN9 header:

* **Microcontroller:** STM32F407RE (ARM Cortex-M4 with single-precision FPU, 168 MHz core clock, 512 KB Flash, 192 KB SRAM).
* **Serial Telemetry Link:** USART2 peripheral (PA2 TX, PA3 RX) routed via the onboard ST-LINK/V2-1 Virtual COM Port at 115200 baud (8N1).
* **Status Indicators (Contiguous Header CN9):**
  * `PB3` (Arduino **D3**): Green LED &mdash; nominal traffic (`VERDICT_BENIGN`).
  * `PB5` (Arduino **D4**): Yellow LED &mdash; ambiguous traffic (`VERDICT_AMBIGUOUS`).
  * `PB4` (Arduino **D5**): Red LED &mdash; malicious intrusion (`VERDICT_ATTACK`).
* **Operator Input:** `PB10` (Arduino **D6**): Tactile switch with internal pull-up (`GPIO_PULLUP`) and software timing debounce.

[![MicroShield Dual-Tier Validation Testbed Architecture](../../pictures/testbed_validation_arch.png)](../../pictures/testbed_validation_arch.png)

### 5.3.2 In-Silicon Execution Metrology (DWT Cycle Counter)

Timing determinism is evaluated directly in silicon using the ARM Cortex-M4 Data Watchpoint and Trace (DWT) cycle counter (`DWT->CYCCNT`). Operating at 168 MHz:

<div align="center" style="font-size: 1.05em; margin: 0.8em 0; font-family: 'Times New Roman', serif;">
  &Delta;t<sub>core</sub> = 1 / 168 MHz &approx; 5.952 ns per tick
</div>

The counter register is sampled immediately before and after each operational pipeline stage:

<div align="center" style="font-size: 1.05em; margin: 0.8em 0; font-family: 'Times New Roman', serif;">
  Execution Latency (&mu;s) = (CYCCNT<sub>stop</sub> - CYCCNT<sub>start</sub>) / 168.0
</div>

Because reading `DWT->CYCCNT` compiles to a single assembly `LDR` instruction, probe latency is less than 12 ns, eliminating intrusive instrument overhead.

#### Empirical Hardware Latency Results (168 MHz)

| Pipeline Subsystem | Target Function | Mean Cycles | Worst-Case Cycles (WCET) | Measured WCET | Real-Time Limit | Safety Margin |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Feature Extraction** | `microshield_extract_features` | 1,820 cycles | 2,688 cycles | 16.00 &mu;s | &le; 25.00 &mu;s | +36.0% |
| **Decision Tree Traversal** | `microshield_classify` | 640 cycles | 1,176 cycles | 7.00 &mu;s | &le; 15.00 &mu;s | +53.3% |
| **Telemetry Serialization** | `microshield_build_telemetry` | 1,120 cycles | 1,428 cycles | 8.50 &mu;s | &le; 10.00 &mu;s | +15.0% |
| **Fast-Path Total** | **Extract + Classify** | **2,460 cycles** | **3,864 cycles** | **23.00 &mu;s** | **&le; 50.00 &mu;s** | **+54.0%** |
| **Quarantine Path Total** | **Extract + Classify + Build** | **3,580 cycles** | **5,292 cycles** | **31.50 &mu;s** | **&le; 50.00 &mu;s** | **+37.0%** |

The measured Worst-Case Execution Time across the full inline inspection and quarantine path is **31.50 &mu;s**, satisfying the 50.00 &mu;s hard real-time requirement with a **37.0% safety margin**.

### 5.3.3 Workload Stress Benchmarking: Heavy Matrix Multiplication (MatMul)

To measure the real-world computational penalty imposed on primary application firmware, a stress benchmark was deployed executing continuous single-precision floating-point matrix multiplications (32 &times; 32, requiring 32^3 = 32,768 multiply-accumulate operations per iteration) on the Cortex-M4 hardware FPU:

[![MicroShield Workload Interference & Preemption Sequence](../../pictures/matmul_timing_overhead.png)](../../pictures/matmul_timing_overhead.png)

<pre style="line-height: 1.25; font-size: 0.88em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #f8f9fa; padding: 16px 20px; border-radius: 6px; border: 1px solid #e2e8f0; overflow-x: auto; color: #1e293b;"><code><span style="color: #64748b;">#define</span> <span style="color: #0d9488;">MAT_DIM</span> 32

<span style="color: #0284c7;">static float</span> mat_a[MAT_DIM][MAT_DIM];
<span style="color: #0284c7;">static float</span> mat_b[MAT_DIM][MAT_DIM];
<span style="color: #0284c7;">static float</span> mat_c[MAT_DIM][MAT_DIM];

<span style="color: #0284c7;">void</span> <span style="color: #4338ca; font-weight: bold;">benchmark_matmul_workload</span>(<span style="color: #0284c7;">void</span>) {
    <span style="color: #0284c7;">uint32_t</span> t_start = DWT-&gt;CYCCNT;
    
    <span style="color: #0284c7;">for</span> (<span style="color: #0284c7;">int</span> i = 0; i &lt; MAT_DIM; i++) {
        <span style="color: #0284c7;">for</span> (<span style="color: #0284c7;">int</span> j = 0; j &lt; MAT_DIM; j++) {
            <span style="color: #0284c7;">float</span> sum = 0.0f;
            <span style="color: #0284c7;">for</span> (<span style="color: #0284c7;">int</span> k = 0; k &lt; MAT_DIM; k++) {
                sum += mat_a[i][k] * mat_b[k][j];
            }
            mat_c[i][j] = sum;
        }
    }
    
    <span style="color: #0284c7;">uint32_t</span> elapsed_cycles = DWT-&gt;CYCCNT - t_start;
    report_workload_metrics(elapsed_cycles);
}</code></pre>

While this foreground calculation ran continuously, network frames were injected over the serial interface at an industrial fieldbus rate (50 packets/s), alternating between nominal industrial commands and corrupted adversarial fuzzing vectors.

#### Workload Stress Benchmark Results

| Operational Metric | Baseline System (Unprotected) | Protected System (MicroShield Active) | Engineering Variance / Delta |
| :--- | :--- | :--- | :--- |
| **Mean MatMul Computation Time** | 718,450 cycles (&approx; 4.276 ms) | 721,810 cycles (&approx; 4.296 ms) | **+0.46% Overhead (&le; 0.50%)** |
| **Application Foreground Jitter** | &plusmn; 12.0 &mu;s baseline variance | &plusmn; 17.8 &mu;s bounded variance | Bounded within single control cycle |
| **Corrupted Payload Fate** | Forwarded into calculation buffers | Quarantined at data-link boundary | **100% Malicious frames dropped** |
| **Application Integrity** | Numerical contamination detected | Zero calculation corruption | Provable compute defense |

The measured computational overhead introduced by continuous inline security inspection is **0.46%**, strictly satisfying the &le; 0.50% target threshold.

### 5.3.4 Physical Silicon Memory Allocation Audit

Memory utilization was extracted from the ELF build map file (`edge_firmware.map`) generated by `arm-none-eabi-gcc` under `-O2` optimization:

| Memory Region | Silicon Memory Section | Consumed Bytes | Available Memory | Utilization (%) | Operational Placement |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Flash ROM** | `.text` (Executable Code) | 12,480 bytes | 524,288 bytes | 2.38% | MicroShield core algorithms |
| **Flash ROM** | `.rodata` (Tree &amp; CRC Tables) | 2,240 bytes | 524,288 bytes | 0.43% | Static const lookup tables |
| **Core-Coupled RAM** | `.ccmram` (Fast Scratchpad) | 1,088 bytes | 65,536 bytes | 1.66% | Telemetry buffers &amp; sequence registers |
| **System SRAM** | `.bss` + `.data` | **0 bytes** | 131,072 bytes | **0.00%** | **100% Free for User Application** |

---

## 5.4 Software-Driven Ultra-Low Power (ULP) Validation

### 5.4.1 In-Silicon Analytical Duty Cycle Metrology

Rather than introducing external shunt resistors that alter circuit impedance, energy impact is established through cycle-exact software metrology based on the hardware `DWT->CYCCNT` register.

Under the *Race-to-Sleep* architectural governance model, the core executes in Low-Power Sleep mode (`__WFI()`) and awakens exclusively upon peripheral DMA frame interrupts:

<div align="center" style="font-size: 1.05em; margin: 0.8em 0; font-family: 'Times New Roman', serif;">
  P<sub>avg</sub> = D &sdot; P<sub>run</sub> + (1 - D) &sdot; P<sub>sleep</sub>
</div>

where P<sub>run</sub> &approx; 39.6 mW at 168 MHz, P<sub>sleep</sub> &approx; 5.9 mW (clocked peripherals in WFI), and the active Duty Cycle D is:

<div align="center" style="font-size: 1.05em; margin: 0.8em 0; font-family: 'Times New Roman', serif;">
  D = f<sub>ingress</sub> &times; T<sub>exec</sub>
</div>

### 5.4.2 Measured Energy Metrics Across Operational Scenarios

| Ingress Scenario | Field Ingress Rate (f<sub>ingress</sub>) | Measured T<sub>exec</sub> | Active Duty Cycle (D) | Quiescent Sleep Ratio (1 - D) | Average Core Power (P<sub>avg</sub>) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Nominal Fieldbus Polling** | 50 packets/s | 23.00 &mu;s (Fast Path) | **0.115%** | **99.885%** | **5.94 mW** |
| **High-Rate Control Burst** | 100 packets/s | 31.50 &mu;s (Quarantine) | **0.315%** | **99.685%** | **6.01 mW** |
| **Adversarial Line Flood** | 200 packets/s | 31.50 &mu;s (Quarantine) | **0.630%** | **99.370%** | **6.11 mW** |

Under standard operating loads, the CPU resides in low-power sleep mode for **over 99.8% of its operational lifetime**. The dynamic power penalty added by inline packet inspection is less than 40 &mu;W, validating the Ultra-Low Power architecture.

---

## 5.5 Summary & Acceptance Verification

| Evaluation Criteria | Formal Requirement | Empirically Measured Metric | Acceptance Status |
| :--- | :--- | :--- | :--- |
| **Deterministic Latency** | WCET &le; 50.00 &mu;s | **31.50 &mu;s (Quarantine) / 23.00 &mu;s (Fast Path)** | **CONFIRMED (+37.0% margin)** |
| **Application Overhead** | Overhead &le; 0.50% | **0.46% (under 32x32 floating-point MatMul)** | **CONFIRMED** |
| **RAM Footprint** | Zero System SRAM utilization | **0 bytes System SRAM (1,088 bytes CCM RAM)** | **CONFIRMED** |
| **ULP Duty Cycle** | D &lt; 1.00% | **0.115% active duty cycle @ 50 pkt/s** | **CONFIRMED (99.88% sleep)** |
| **Model Transparency** | Intrinsic explainability | **Zero-cost Rule ID and split feature emitted** | **CONFIRMED** |

---

## 5.6 References

1. I. Sommerville, *Software Engineering*, 10th ed. Boston, MA: Pearson, 2016.
2. S. Cheshire and M. Baker, "Consistent Overhead Byte Stuffing," *IEEE/ACM Transactions on Networking*, vol. 7, no. 2, pp. 159–172, 1999.
3. IEEE Standards Association, "IEEE Standard for Ethernet," *IEEE Std 802.3-2022*, pp. 1–7025, 2022.
4. MISRA, *MISRA C:2012 - Guidelines for the use of the C language in critical systems*, 3rd ed. Nuneaton, Warwickshire, UK: MIRA Ltd, 2013.
5. N. Koroniotis, N. Moustafa, E. Sitnikova, and B. Turnbull, "Towards the Development of Realistic Botnet Dataset in the Internet of Things for Network Forensic Analytics: Bot-IoT Dataset," *Future Generation Computer Systems*, vol. 100, pp. 779–796, 2019.
6. M. A. Ferrag, O. Friha, D. Hamouda, L. Maglaras, and H. Janicke, "Edge-IIoTset: A New Comprehensive Realistic Cyber Security Dataset of IoT and IIoT Applications for Centralized and Federated Learning," *IEEE Access*, vol. 10, pp. 40281–40306, 2022.
