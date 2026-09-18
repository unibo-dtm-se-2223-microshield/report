---
title: Validation
has_children: false
nav_order: 6
---

# 5. Validation

## 5.1 Testing Strategy & Quality Assurance Gates

The validation phase establishes empirical verification that MicroShield satisfies its hard real-time, deterministic memory, and classification invariants across both target operational domains: bare-metal C99 firmware executing on the ARM Cortex-M4 microcontroller and asynchronous Python MLOps telemetry running on the supervisory host.

### 5.1.1 Test-Driven Development (TDD) Methodology

Software implementation adhered strictly to a multi-tiered Test-Driven Development (TDD) lifecycle:

1. **Interface Contract Formalization:** Struct layouts, memory alignment boundaries, bit-level wire protocols, and constant decision tree lookup matrices were defined as contractual header interfaces (`microshield.h`, `transpiled_model.h`) prior to implementing operational code.
2. **Defensive Test Suite Synthesis:** Unit and integration test harnesses were authored to probe numerical stability, timer rollover vulnerabilities, MTU boundary clipping, CRC32 polynomial correctness, and null-pointer dereferencing guards.
3. **Run-to-Completion Implementation:** Firmware translation units and supervisory modules were authored with zero dynamic allocation to fulfill test assertions within deterministic cycle budgets.
4. **Static Analysis & Compiler Quality Gates:** Every compilation unit is subjected to strict static gates before merging into the main branch.

<pre style="line-height: 1.15; font-size: 0.9em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #f8f9fa; padding: 14px 18px; border-radius: 6px; border: 1px solid #e2e8f0; overflow-x: auto;">
       CONTRACT DEFINITION               FAILING TEST HARNESS                MINIMAL C99 / PYTHON               STATIC VERIFICATION
    ┌────────────────────────┐        ┌────────────────────────┐          ┌────────────────────────┐         ┌────────────────────────┐
    │  microshield.h         │        │  test_features.c       │          │  microshield_*.c       │         │  -Wall -Wextra -Werror │
    │  transpiled_model.h    │  ───►  │  test_cobs.c           │   ───►   │  dashield/*.py         │  ───►   │  mypy --strict         │
    │  wire_protocol (COBS)  │        │  pytest suites         │          │  (0 dynamic alloc)     │         │  MISRA C:2012 Rules    │
    └────────────────────────┘        └────────────────────────┘          └────────────────────────┘         └────────────────────────┘
</pre>

### 5.1.2 Dual-Tier Verification Toolchain

Verification operates across two independent continuous-integration toolchains:

* **Edge Runtime Tier (C99 Bare-Metal):** Compiled using native GCC under strict ISO C99 (`-std=c99`). The compiler enforces a zero-warning gate by activating `-Wall` and `-Wextra`, elevating every warning to an immediate fatal error (`-Werror`). Dynamic memory routines (`malloc`, `free`, `realloc`) are strictly prohibited in the source tree, guaranteeing zero heap utilization.
* **Supervisory Tier (Python 3.11+):** Managed via Poetry and tested via `pytest`. The domain model systematically eliminates dynamic type uncertainties through `mypy` configured in full strict mode (PEP 484, PEP 526). Untyped functions, implicit optional values, and dynamic decorators are rejected at compile time.

### 5.1.3 Requirements Traceability Matrix

Every automated and empirical test case maps directly to the formal engineering requirements:

| Requirement ID | Operational Constraint | Verification Target | Target Metric / Acceptance Gate | Governing Test Harness |
| :--- | :--- | :--- | :--- | :--- |
| **R-EDGE-01** | Bounded Real-Time Latency | Inference & Traversal | Worst-Case Execution Time (WCET) &le; 50.0 &mu;s | `test_engine.c` &amp; In-Silicon DWT |
| **R-EDGE-02** | Zero Heap Allocation | Static SRAM Budget | Zero bytes dynamic heap; Flash-resident `.rodata` | Linker Script Audit &amp; GCC Map |
| **R-EDGE-03** | Non-Blocking Serialization | DMA Buffer Safety | Zero CPU wait states during serial egress | `test_cobs.c` &amp; DMA ISR Profiling |
| **R-TRANS-01** | End-to-End Data Integrity | CRC32 Algebraic Check | Exact IEEE 802.3 residue verification | `test_cobs.c` &amp; `test_framing.py` |
| **R-TRANS-02** | Serial Frame Delineation | COBS Byte Stuffing | Unambiguous framing; delimiter byte `0x00` unique | `test_cobs.c` &amp; `test_framing.py` |
| **R-ML-01** | Model Complexity Bound | AST Tree Depth | Depth &le; 6 comparisons (&le; 64 leaf nodes) | `test_trainer.py` &amp; `test_transpiler.py` |
| **R-ML-02** | Automated Code Emission | Transpiler Fidelity | MISRA-compliant `.h` with identical predictions | `test_transpiler.py` &amp; Native GCC |
| **R-DRIFT-01** | Concept Drift Detection | Ambiguity Monitoring | Sliding FIFO window (W=100), alarm at &tau; &gt; 5.0% | `test_drift.py` |
| **R-UI-01** | Operator Visibility &amp; Triage | Reactive MLOps Console | 1 Hz decoupled polling; live Retrain trigger | `test_ui.py` &amp; Standalone Dash |

---

## 5.2 Automated Testing & Software-in-the-Loop (SIL) Verification

The first validation stage executes entirely within a desktop Software-in-the-Loop (SIL) environment, isolating algorithmic correctness from physical target silicon.

### 5.2.1 Edge Runtime Unit Verification (C99)

Unit tests for the C99 edge engine are compiled natively on Linux x86_64, asserting arithmetic precision, array boundary safety, and error containment.

#### Feature Extraction Engine (`test_features.c`)
1. **Length Saturation:** Verifies that frames ranging from small runts (60 bytes) up to standard MTU (1500 bytes) and oversized jumbo frames (2000 bytes) map deterministically to f0 &isin; [0.0, 1.0] with hard saturation at 1.0.
2. **Timer Rollover Invariance:** Probes the modular arithmetic of the inter-arrival calculation across 32-bit integer overflow events. Unsigned subtraction in modulo 2^32 guarantees an exact 32.0 &mu;s delta without conditional branching.
3. **Defensive Runt Guards:** Evaluates frames smaller than minimum data-link headers (&lt; 14 bytes). Protocol flags safely default to zero, preventing out-of-bounds pointer offsets.
4. **Two-Pass Byte Variance:** Confirms numerical stability on uniform payloads (&sigma;^2 = 0.00) versus known high-dispersion arrays (&sigma;^2 = 2500.00), validating that floating-point mantissa precision remains positive and bounded.

#### Deterministic Decision Engine (`test_engine.c`)
1. **Bounded Iterative Traversal:** Verifies O(depth) path resolution across nominal Modbus packets (`VERDICT_BENIGN`), volumetric line floods (`VERDICT_ATTACK`), and high-entropy payload fuzzing (`VERDICT_ATTACK`).
2. **Fail-Safe Null Pointer Handling:** Injects `NULL` pointers for the feature struct; the engine defaults safely to `VERDICT_AMBIGUOUS` with zero segmentation faults.
3. **Symbolic XAI Attribution:** Asserts that terminal leaves return an immutable 16-bit `rule_id` and the dominant `split_feature` index alongside the ternary verdict.

#### Framing & Checksum Engine (`test_cobs.c`)
1. **IEEE 802.3 CRC32 Conformance:** Evaluates the standard 9-byte ASCII vector `"123456789"`, asserting exact bit-level agreement with standard polynomial residue `0xCBF43926`.
2. **COBS Delimiter Elimination:** Encodes binary buffers containing multiple embedded `0x00` null bytes. Verifies that the encoded byte stream is free of null characters, guaranteeing unambiguous boundary detection.

<pre style="line-height: 1.25; font-size: 0.85em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #1e293b; padding: 14px 18px; border-radius: 6px; border: 1px solid #334155; overflow-x: auto; color: #f8fafc;">
<span style="color: #94a3b8;">wearemassive@wearemassive:~/microshield/artifact$</span> <span style="color: #38bdf8;">make test-c</span>
cd edge &amp;&amp; make test
make[1]: Entering directory '/home/wearemassive/microshield/artifact/edge'
gcc -std=c99 -Wall -Wextra -Werror -Iinclude -Imodel -c src/microshield_features.c -o build/microshield_features.o
gcc -std=c99 -Wall -Wextra -Werror -Iinclude -Imodel -c src/microshield_engine.c -o build/microshield_engine.o
gcc -std=c99 -Wall -Wextra -Werror -Iinclude -Imodel -c src/microshield_cobs.c -o build/microshield_cobs.o
gcc -std=c99 -Wall -Wextra -Werror -Iinclude -Imodel src/microshield_features.c tests/test_features.c -o tests/test_features_bin -lm
gcc -std=c99 -Wall -Wextra -Werror -Iinclude -Imodel src/microshield_engine.c tests/test_engine.c -o tests/test_engine_bin
gcc -std=c99 -Wall -Wextra -Werror -Iinclude -Imodel src/microshield_cobs.c tests/test_cobs.c -o tests/test_cobs_bin
./tests/test_features_bin
--- Running MicroShield Edge C99 Feature Extractor Tests ---
[TEST] Length Normalization: PASSED (60B, 1500B, 2000B)
[TEST] Timing Delta &amp; Rollover Protection: PASSED (Boot default, nominal, wrap-around)
[TEST] Protocol Flags Extraction: PASSED (TCP SYN detected, runt frame guarded)
[TEST] Two-Pass Variance Accuracy: PASSED (Constant=0.0, Known-dist=2500.0)
--- ALL C99 FEATURE EXTRACTOR TESTS PASSED SUCCESSFULLY ---
./tests/test_engine_bin
--- Running MicroShield Edge C99 Inference Engine Tests ---
[TEST] Nominal: verdict=0, rule_id=1, split_feat=3
[TEST] Volumetric Flood: verdict=1, rule_id=14, split_feat=0
[TEST] Fuzzing Attack: verdict=1, rule_id=22, split_feat=3
[TEST] Ambiguous Drift: verdict=2, rule_id=4, split_feat=3
[TEST] Null Pointer Safety: verdict=2
--- ALL C99 INFERENCE ENGINE TESTS PASSED SUCCESSFULLY ---
./tests/test_cobs_bin
--- Running MicroShield Edge C99 Framing &amp; Integrity Tests ---
[TEST] CRC32('123456789'): 0xCBF43926 (Expected: 0xCBF43926)
[TEST] COBS Roundtrip: 8 raw bytes -&gt; 10 encoded bytes -&gt; matched
[TEST] TelemetryFrame CRC32 Verified: 0xD8C5B3C9 (Sequence: 0)
--- ALL C99 FRAMING &amp; INTEGRITY TESTS PASSED SUCCESSFULLY ---
make[1]: Leaving directory '/home/wearemassive/microshield/artifact/edge'
</pre>

### 5.2.2 Supervisory Tier Unit Verification (Python)

Supervisory verification confirms type correctness, statistical drift detection logic, AST code generation, and UI presentation components under Python 3.12:

<pre style="line-height: 1.25; font-size: 0.85em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #1e293b; padding: 14px 18px; border-radius: 6px; border: 1px solid #334155; overflow-x: auto; color: #f8fafc;">
<span style="color: #94a3b8;">wearemassive@wearemassive:~/microshield/artifact$</span> <span style="color: #38bdf8;">make test-py &amp;&amp; make typecheck</span>
cd supervisor &amp;&amp; poetry run pytest -v tests/
============================= test session starts ==============================
collected 23 items

tests/test_drift.py::test_nominal_traffic_no_drift PASSED                 [  4%]
tests/test_drift.py::test_warmup_guard_prevents_premature_alarm PASSED    [  8%]
tests/test_drift.py::test_concept_drift_alarm_trigger PASSED              [ 13%]
tests/test_drift.py::test_fifo_sliding_window_recovery PASSED             [ 17%]
tests/test_drift.py::test_detector_reset PASSED                           [ 21%]
tests/test_drift.py::test_invalid_parameters_raise_value_error PASSED     [ 26%]
tests/test_framing.py::test_decode_valid_frame_matching_c_test PASSED    [ 30%]
tests/test_framing.py::test_corrupted_crc_rejection PASSED               [ 34%]
tests/test_framing.py::test_truncated_frame_rejection PASSED             [ 39%]
tests/test_trainer.py::test_dataset_generation_dimensions PASSED         [ 43%]
tests/test_trainer.py::test_tree_depth_hardware_constraint PASSED        [ 47%]
tests/test_trainer.py::test_tree_classification_performance PASSED       [ 52%]
tests/test_transpiler.py::test_tree_depth_exceeded_guard PASSED          [ 56%]
tests/test_transpiler.py::test_c_header_generation_syntax PASSED          [ 60%]
tests/test_transpiler.py::test_rule_dictionary_xai_generation PASSED      [ 65%]
tests/test_transpiler.py::test_artifacts_disk_emission PASSED             [ 69%]
tests/test_types.py::test_verdict_enum_values PASSED                      [ 73%]
tests/test_types.py::test_telemetry_record_immutability PASSED           [ 78%]
tests/test_types.py::test_telemetry_record_serialization PASSED           [ 82%]
tests/test_ui.py::test_dashboard_instantiation PASSED                    [ 86%]
tests/test_ui.py::test_layout_contains_critical_components PASSED        [ 91%]
tests/test_ui.py::test_callback_registration PASSED                      [ 95%]
tests/test_ui.py::test_metric_state_reaction_with_drift PASSED           [100%]

============================== 23 passed in 1.14s ==============================
cd supervisor &amp;&amp; poetry run mypy --strict dashield/ tests/
Success: no issues found in 19 source files
</pre>

### 5.2.3 Consolidated Unit Test Pass Matrix

| Execution Domain | Test Suite Target | Primary Verified Invariant | Executed | Passed | Failed | Pass Rate |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Edge C99** | `test_features.c` | MTU normalization, timer overflow invariance, runt safety | 4 | 4 | 0 | 100.0% |
| **Edge C99** | `test_engine.c` | O(depth) traversal, null pointer safety, XAI output | 5 | 5 | 0 | 100.0% |
| **Edge C99** | `test_cobs.c` | IEEE 802.3 CRC32, COBS byte stuffing, telemetry packing | 3 | 3 | 0 | 100.0% |
| **Supervisory Py** | `test_framing.py` | Cross-language binary deserialization, CRC bit-flip trap | 3 | 3 | 0 | 100.0% |
| **Supervisory Py** | `test_trainer.py` | Benchmark corpus generation, depth &le; 6 enforcement | 3 | 3 | 0 | 100.0% |
| **Supervisory Py** | `test_transpiler.py` | C99 `.rodata` header synthesis, XAI rule dictionary | 4 | 4 | 0 | 100.0% |
| **Supervisory Py** | `test_drift.py` | FIFO sliding window, warm-up barrier, self-healing | 6 | 6 | 0 | 100.0% |
| **Supervisory Py** | `test_types.py` | Dataclass immutability, enum mappings, serialization | 3 | 3 | 0 | 100.0% |
| **Supervisory Py** | `test_ui.py` | Dash DOM hierarchy, reactive drift callbacks, gauges | 4 | 4 | 0 | 100.0% |
| **Total Pipeline** | **All Modules** | **Static analysis &amp; runtime dynamic test coverage** | **35** | **35** | **0** | **100.0%** |

---

## 5.3 Hardware-in-the-Loop (HIL) Metrology & Heavy Workload Benchmarking

Empirical silicon benchmarking was executed on the STMicroelectronics NUCLEO-F407RE development platform to quantify physical timing, silicon memory footprint, and CPU workload interference under heavy operational stress.

### 5.3.1 Physical Hardware Testbed Setup & Breadboard Interconnect

* **Target Microcontroller:** STM32F407RE (ARM Cortex-M4 with single-precision hardware FPU, 168 MHz core clock, 512 KB Flash, 192 KB SRAM).
* **Diagnostic Serial Interface:** USART2 routed over ST-LINK/V2-1 Virtual COM Port at 115200 baud (8N1).
* **Visual Status Indicators (Contiguous Header CN9):**
  * `PB3` (Arduino **D3**): Green LED indicating nominal traffic (`VERDICT_BENIGN`).
  * `PB5` (Arduino **D4**): Yellow LED indicating ambiguous traffic (`VERDICT_AMBIGUOUS`).
  * `PB4` (Arduino **D5**): Red LED indicating malicious intrusion (`VERDICT_ATTACK`).
* **Operator Input:** `PB10` (Arduino **D6**): External tactile pushbutton configured with internal pull-up (`GPIO_PULLUP`) and software timing debounce.

[![MicroShield Dual-Tier Validation Testbed Architecture](../../pictures/testbed_validation_arch.png)](../../pictures/testbed_validation_arch.png)

### 5.3.2 In-Silicon Execution Metrology (DWT Cycle Counter)

Execution latencies are measured using the ARM Cortex-M4 Data Watchpoint and Trace (DWT) cycle counter (`DWT->CYCCNT`). Operating at 168 MHz:

<div align="center" style="font-size: 1.05em; margin: 0.8em 0; font-family: 'Times New Roman', serif;">
  &Delta;t<sub>core</sub> = 1 / 168 MHz &approx; 5.952 ns per cycle
</div>

The counter register is sampled directly before and after the target operational section:

<div align="center" style="font-size: 1.05em; margin: 0.8em 0; font-family: 'Times New Roman', serif;">
  Execution Latency (&mu;s) = (CYCCNT<sub>stop</sub> - CYCCNT<sub>start</sub>) / 168.0
</div>

Sampling executes via a single assembly `LDR` instruction, introducing less than 12 ns probe overhead.

#### Empirical Hardware Profiling Results (168 MHz)

| Pipeline Subsystem | Target Translation Unit | Average Cycles | Worst-Case Cycles (WCET) | Measured WCET | Real-Time Budget | Timing Margin |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Feature Extraction** | `microshield_features.c` | 1,820 cycles | 2,688 cycles | 16.00 &mu;s | &le; 25.00 &mu;s | +36.0% |
| **Decision Tree Traversal** | `microshield_engine.c` | 640 cycles | 1,176 cycles | 7.00 &mu;s | &le; 15.00 &mu;s | +53.3% |
| **Telemetry &amp; COBS Frame** | `microshield_cobs.c` | 1,120 cycles | 1,428 cycles | 8.50 &mu;s | &le; 10.00 &mu;s | +15.0% |
| **Fast-Path Total** | **Extraction + Inference** | **2,460 cycles** | **3,864 cycles** | **23.00 &mu;s** | **&le; 50.00 &mu;s** | **+54.0%** |
| **Quarantine Path Total** | **Extract + Infer + Encode** | **3,580 cycles** | **5,292 cycles** | **31.50 &mu;s** | **&le; 50.00 &mu;s** | **+37.0%** |

The measured Worst-Case Execution Time across the complete inline quarantine sequence is **31.50 &mu;s**, operating well within the 50.00 &mu;s ceiling with a **37.0% deterministic safety margin**.

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
