---
title: Development
has_children: false
nav_order: 5
---

# 4. Implementation & Development

## 4.1 Repository Scaffolding & Polyglot Build Automation

The implementation phase translates the architectural patterns established in the design specification into concrete, verifiable software artifacts. To maintain rigorous separation of concerns while coordinating heterogeneous runtimes, the project repository is partitioned into three autonomous development domains: bare-metal C99 edge firmware, a supervisory Python MLOps suite, and a containerized adversary playback harness.

### 4.1.1 Artifact Directory Layout

The physical directory tree of the software repository isolates runtime dependencies, test harnesses, and build scripts into decoupled subsystems:

<pre style="line-height: 1.15; font-size: 0.9em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #f8f9fa; padding: 14px 18px; border-radius: 6px; border: 1px solid #e2e8f0; overflow-x: auto;">
artifact/
├── Makefile
├── edge/
│   ├── include/
│   ├── src/
│   ├── model/
│   ├── tests/
│   └── Makefile
├── supervisor/
│   ├── pyproject.toml
│   ├── README.md
│   ├── dashield/
│   │   ├── domain/
│   │   ├── transport/
│   │   ├── drift/
│   │   ├── transpiler/
│   │   └── ui/
│   └── tests/
└── simulation/
    ├── whispers/
    └── docker/
</pre>

### 4.1.2 Concrete Software Implementation Pipeline

The execution flow bridging real-time packet interception on the microcontroller to supervisory MLOps analytics is realized through a unidirectional data pipeline. Each compilation translation unit and Python module occupies an explicit stage along this processing path (click image to expand to full resolution):

[![MicroShield Concrete Software Implementation Pipeline](../../pictures/impl_pipeline.png)](../../pictures/impl_pipeline.png)

1. **Ingress & Feature Calculation (`microshield_features.c`):** Inbound network frames residing in DMA receive buffers are processed via zero-copy pointer access, extracting the 16-byte `FeatureVector_t` in bounded execution time (&le; 18 &mu;s).
2. **Deterministic Inference (`microshield_engine.c`):** The extracted vector traverses constant lookup matrices defined in `transpiled_model.h`, returning a ternary verdict in bounded time (&le; 12 &mu;s).
3. **Framing & Serial Egress (`microshield_cobs.c`):** Frames classified as `ATTACK` or `AMBIGUOUS` trigger the construction of a 32-byte `TelemetryFrame_t`, protected by an IEEE 802.3 CRC32 checksum and encoded using Consistent Overhead Byte Stuffing (COBS).
4. **Supervisory Parsing (`dashield.transport`):** The supervisory daemon reconstructs incoming frames across the physical serial link, drops corrupted packets, and yields strongly typed domain transfer objects.
5. **Drift Surveillance & Retraining (`dashield.drift` & `dashield.transpiler`):** Validated records feed the rolling ambiguity tracker. When the ratio exceeds 5%, the AST transpiler re-fits the decision tree and emits an updated `transpiled_model.h` for firmware deployment.
6. **Reactive Presentation (`dashield.ui`):** The web console visualizes incoming anomalies and presents borderline cases to human operators for triage.

### 4.1.3 Unified Polyglot Orchestration via Root Makefile

Managing a heterogeneous software repository spanning two completely distinct toolchains—GNU Compiler Collection (GCC) for bare-metal C99 and Poetry for Python 3.11+—introduces workflow friction if commands are not centralized. 

The root `Makefile` establishes a declarative, cross-platform interface exposing standard development targets:

| Make Target | Governed Domain | Underlying Toolchain Action | Target Acceptance Gate |
| :--- | :--- | :--- | :--- |
| `make test-c` | Edge Runtime (C99) | Invokes `edge/Makefile` targeting native GCC with strict flags. | Zero runtime test failures, zero memory leaks. |
| `make test-py` | Supervisory Tier (Python) | Executes `poetry run pytest -v tests/` across all unit suites. | 100% test pass rate for all domain invariants. |
| `make typecheck` | Supervisory Tier (Python) | Runs `poetry run mypy --strict dashield/ tests/`. | Zero type errors, complete PEP 484/526 coverage. |
| `make lint` | Supervisory Tier (Python) | Runs `flake8` and `black --check` across Python modules. | Conformity with PEP 8 styling conventions. |
| `make test` | Full Monorepo | Sequentially runs `test-c`, `test-py`, and `typecheck`. | Universal regression test pass before Git commit. |
| `make clean` | Full Monorepo | Strips compiled `.o` binaries, `__pycache__`, and test cache dirs. | Repository cleaned to pristine source state. |

### 4.1.4 Quality Assurance & Static Verification Gates

To satisfy the safety and reliability standards required in industrial environments, the build pipeline integrates mandatory static verification mechanisms across both programming languages.

For the bare-metal edge tier, strict ISO/IEC 9899:1999 compliance (`-std=c99`) guarantees cross-compiler portability between local desktop GCC and target ARM embedded toolchains without reliance on proprietary GNU extensions. The build system enforces a zero-warning policy by activating standard and extended compiler diagnostics (`-Wall`, `-Wextra`) while promoting every warning to a fatal build-terminating error (`-Werror`). In addition, syntactic and semantic interface validation is executed directly on header contracts without generating intermediate object code (`-fsyntax-only`), enabling fast verification during automated integration pipelines.

In the supervisory tier, code quality is governed by static type theory rather than dynamic type inference. Leveraging modern Python type specifications (PEP 484, PEP 526), the MLOps pipeline enforces strict static typing via `mypy` configured in full strict mode. This configuration systematically prohibits dynamically typed functions, untyped decorators, and ambiguous return values, ensuring that domain entities and value objects remain strictly typed, immutable, and provably correct before execution.

---

## 4.2 Edge Firmware Architecture, Memory Layout & Application Hooking

The edge runtime is designed as a self-contained, statically linkable C99 library (`libmicroshield.a`) that embeds seamlessly into real-time industrial firmware without requiring an underlying Real-Time Operating System (RTOS).

### 4.2.1 Modular Decomposition & Function Call Interactions

Rather than implementing a monolithic firmware file, the edge detection engine is structured into three discrete translation units governed by the public contractual header `microshield.h`. This separation guarantees single responsibility and isolates hardware peripherals from classification logic:

[![MicroShield Edge Modular Function & Header Interaction Graph](../../pictures/edge_interactions.png)](../../pictures/edge_interactions.png)

The interaction sequence operates entirely within the hardware interrupt context:
1. **Public Contract Ingress (`microshield.h`):** The application firmware (`main.c`) includes exclusively `microshield.h`. When a new packet arrives at the physical MAC interface, the hardware triggers `HAL_Network_RxCallback()`, passing a direct pointer to the contiguous DMA memory buffer.
2. **Feature Extraction Call (`microshield_features.c`):** The callback invokes `microshield_extract_features()`. The module calculates the 16-byte `FeatureVector_t` in bounded execution time (&le; 18 &mu;s) using the hardware DWT cycle counter for microsecond-level timing delta acquisition.
3. **Deterministic Traversal Call (`microshield_engine.c`):** The feature vector is passed to `microshield_classify()`. The engine traverses the static binary decision tree defined in `transpiled_model.h` in <i>O</i>(depth) time (&le; 12 &mu;s), resolving the ternary verdict (`BENIGN`, `ATTACK`, `AMBIGUOUS`), the leaf `rule_id`, and the dominant `split_feature`.
4. **Boundary Gatekeeping & Signaling:**
   - **Nominal Traffic (`BENIGN`):** The callback immediately forwards the packet pointer to the industrial process queue (e.g., Modbus engine) and sets the on-board green LED to steady state. Zero buffering delay is imposed on nominal physical control tasks.
   - **Malicious or Ambiguous Traffic (`ATTACK` / `AMBIGUOUS`):** The packet pointer is quarantined, the red LED is triggered, and `microshield_build_telemetry()` is invoked in `microshield_cobs.c`.
5. **Telemetry Assembly & DMA Egress (`microshield_cobs.c`):** An alert frame is formatted, stamped with an IEEE 802.3 CRC32 checksum, COBS-encoded, and dispatched via non-blocking USART DMA (`HAL_UART_Transmit_DMA()`). Primary CPU execution returns immediately to the core process loop without waiting for the physical serial baud clock.

### 4.2.2 Silicon Placement, Compilation Pipeline & CCM RAM Allocation

On resource-restricted microcontrollers, software architecture directly dictates physical silicon utilization. The compilation toolchain and memory layout for the STM32F407RE microcontroller are formalized below (click image to expand to full resolution):

[![MicroShield Embedded Toolchain Compilation & Silicon Memory Allocation](../../pictures/edge_compilation_memory.png)](../../pictures/edge_compilation_memory.png)

The GCC toolchain (`arm-none-eabi-gcc`) compiles each translation unit into relocatable ELF object files (`.o`), which are subsequently positioned across physical memory segments by the linker script (`STM32F407RETx_FLASH.ld`):

1. **Flash Program Memory (`.text` Section @ 0x08000000):** All compiled machine instructions for `microshield_features.o`, `microshield_engine.o`, and `microshield_cobs.o` occupy less than 16 KB of Flash (&le; 3.13% of total 512 KB Flash storage).
2. **Flash Read-Only Memory (`.rodata` Section @ 0x08000000):** The transpiled decision tree thresholds and feature index matrices (`transpiled_model.h`) are declared `static const`. The compiler places them directly into Flash memory. **Their volatile RAM consumption is exactly zero bytes**, preserving all volatile memory for runtime processing.
3. **Core Coupled Memory (`.ccmram` Section @ 0x10000000):** The volatile workspace—including temporary feature extraction structs, telemetry transmission buffers, and monotonic sequence counters—is explicitly mapped to the 64 KB Core Coupled Memory (CCM Data RAM) using GCC attributes (`__attribute__((section(".ccmram")))`). Because CCM RAM is wired directly to the Cortex-M4 D-bus, memory access executes with zero wait-states and causes zero bus contention with DMA transfers on the multi-layer AHB matrix.
4. **Preservation of System SRAM (@ 0x20000000):** The primary 112 KB + 16 KB SRAM pool remains completely unconstrained, ensuring that customer industrial control tasks, RTOS stacks, and communication buffers operate without memory starvation.

### 4.2.3 Application Hooking in Industrial Control Loops

To integrate MicroShield into existing industrial firmware (e.g., generated via STM32CubeMX or STM32CubeIDE), the developer adds minimal, non-intrusive integration hooks into `main.c`:

<pre style="line-height: 1.25; font-size: 0.88em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #f8f9fa; padding: 16px 20px; border-radius: 6px; border: 1px solid #e2e8f0; overflow-x: auto; color: #1e293b;"><code><span style="color: #64748b;">#include</span> <span style="color: #0d9488;">"main.h"</span>
<span style="color: #64748b;">#include</span> <span style="color: #0d9488;">"microshield.h"</span> <span style="color: #94a3b8;">/* Public API contract */</span>

<span style="color: #0284c7;">extern</span> UART_HandleTypeDef huart2;

<span style="color: #0284c7;">int</span> <span style="color: #4338ca; font-weight: bold;">main</span>(<span style="color: #0284c7;">void</span>) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    <span style="color: #94a3b8;">/* 1. Initialize MicroShield internal state (Node ID: 101) */</span>
    microshield_init(101);

    <span style="color: #0284c7;">while</span> (1) {
        <span style="color: #94a3b8;">/* Primary industrial control and actuator loop (1 kHz) */</span>
    }
}

<span style="color: #94a3b8;">/**
 * @brief Ethernet/Fieldbus physical layer receive callback.
 * Executes inline within the interrupt context (Total WCET &lt;= 50 &mu;s).
 */</span>
<span style="color: #0284c7;">void</span> <span style="color: #4338ca; font-weight: bold;">HAL_Network_RxCallback</span>(<span style="color: #0284c7;">uint8_t</span> *frame_buffer, <span style="color: #0284c7;">uint16_t</span> length) {
    microshield_features_t features;
    <span style="color: #0284c7;">uint16_t</span> rule_id;
    <span style="color: #0284c7;">uint8_t</span> split_feature;
    <span style="color: #0284c7;">uint32_t</span> now_us = DWT-&gt;CYCCNT / 168; <span style="color: #94a3b8;">// Microsecond DWT hardware timestamp</span>

    <span style="color: #94a3b8;">/* 2. Zero-copy feature extraction */</span>
    microshield_extract_features(frame_buffer, length, now_us, &amp;features);

    <span style="color: #94a3b8;">/* 3. Deterministic decision tree inference */</span>
    microshield_verdict_t verdict = microshield_classify(&amp;features, &amp;rule_id, &amp;split_feature);

    <span style="color: #0284c7;">if</span> (verdict == VERDICT_BENIGN) {
        <span style="color: #94a3b8;">/* Fast-Path: pass frame pointer to core application queue */</span>
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET); <span style="color: #94a3b8;">// Green LED on</span>
        process_industrial_payload(frame_buffer, length);
    } 
    <span style="color: #0284c7;">else</span> {
        <span style="color: #94a3b8;">/* Anomaly Gatekeeping: suppress payload and alert supervisor */</span>
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_SET); <span style="color: #94a3b8;">// Red LED on</span>
        
        <span style="color: #0284c7;">static</span> microshield_telemetry_t telemetry;
        <span style="color: #0284c7;">uint16_t</span> bytes_to_send = microshield_build_telemetry(verdict, rule_id, split_feature, &amp;features, &amp;telemetry);
        
        <span style="color: #94a3b8;">/* Non-blocking DMA transfer: CPU returns to control loop immediately */</span>
        HAL_UART_Transmit_DMA(&amp;huart2, (<span style="color: #0284c7;">uint8_t</span> *)&amp;telemetry, bytes_to_send);
    }
}</code></pre>

---

## 4.3 End-to-End Diagnostic Transport: Framing (COBS) & Algebraic Integrity (CRC32)

Establishing a robust communication link between edge microcontrollers and supervisory workstations requires solving two transport hazards: packet boundary desynchronization across continuous byte streams and data corruption induced by industrial electromagnetic interference (EMI).

### 4.3.1 Transport-Agnostic Zero-Trust Rationale

In industrial deployments, diagnostic telemetry traverses heterogeneous physical channels:
- Short PCB inter-chip traces (e.g., UART links connecting the STM32 to local Wi-Fi or BLE transceivers).
- Differential long-span factory floor fieldbuses (e.g., isolated RS-485 networks extending hundreds of meters).
- Packetized IP backhauls (Ethernet bridges, VPN tunnels, or cellular gateways).

In accordance with Saltzer's classical End-to-End Argument [8], intermediate network layers cannot be trusted to guarantee end-to-end payload integrity. High-voltage switching transients (IEC 61000-4-4 Electrical Fast Transients) and radio fading can flip bits at any stage along the transit pipeline. 

MicroShield establishes a **Transport-Agnostic Zero-Trust Boundary**: diagnostic records are cryptographically and algebraically sealed directly in STM32 silicon and validated exclusively upon domain deserialization within the Python supervisory runtime. The underlying physical carrier remains completely transparent to the telemetry contract.

### 4.3.2 Algebraic Integrity via IEEE 802.3 CRC32

To detect in-transit corruption without resorting to heavy cryptographic signatures, the telemetry frame incorporates an IEEE 802.3 standard 32-bit Cyclic Redundancy Check (CRC32) [9].

#### Polynomial Division in Galois Field GF(2)
The payload byte array is treated as a single binary polynomial <i>M</i>(<i>x</i>) in the Galois Field GF(2), where addition and subtraction correspond to the bitwise XOR operation (&oplus;). The checksum is defined as the remainder <i>R</i>(<i>x</i>) of the polynomial division against the standard generator polynomial <i>G</i>(<i>x</i>):

<div align="center" style="font-size: 1.15em; margin: 1em 0; font-family: 'Times New Roman', serif;">
  [<i>M</i>(<i>x</i>) &sdot; <i>x</i><sup>32</sup>] / <i>G</i>(<i>x</i>) = <i>Q</i>(<i>x</i>) &oplus; [<i>R</i>(<i>x</i>) / <i>G</i>(<i>x</i>)]
</div>

where <i>G</i>(<i>x</i>) is the reversed representation constant <code>0xEDB88320</code>:

<div align="center" style="font-size: 1.05em; margin: 0.8em 0; font-family: 'Times New Roman', serif;">
  <i>G</i>(<i>x</i>) = <i>x</i><sup>32</sup> + <i>x</i><sup>26</sup> + <i>x</i><sup>23</sup> + <i>x</i><sup>22</sup> + <i>x</i><sup>16</sup> + <i>x</i><sup>12</sup> + <i>x</i><sup>11</sup> + <i>x</i><sup>10</sup> + <i>x</i><sup>8</sup> + <i>x</i><sup>7</sup> + <i>x</i><sup>5</sup> + <i>x</i><sup>4</sup> + <i>x</i><sup>2</sup> + <i>x</i> + 1
</div>

#### Precalculated Flash Lookup Table Optimization
Iterative bit-by-bit software division requires 8 branch iterations per byte (224 conditional branches across the 28-byte payload), causing instruction pipeline stalls on ARM Cortex-M4 cores.

MicroShield precalculates the 256-entry polynomial table (<code>CRC32_TABLE</code>), mapped statically into Flash memory (<code>.rodata</code>):
- **Flash Memory Footprint:** 256 &times; 4 bytes = 1024 bytes (&le; 0.20% of 512 KB Flash).
- **Volatile RAM Footprint:** **Exactly 0 bytes**.
- **Computational Latency:** 28 single-cycle table lookups and XOR operations executing in less than 150 clock cycles (&approx; 0.89 &mu;s @ 168 MHz), fully satisfying the real-time budget.

### 4.3.3 Consistent Overhead Byte Stuffing (COBS) Protocol

Asynchronous serial interfaces lack intrinsic packet boundaries. To allow the receiver to detect frame start and termination unambiguously, a null byte (<code>0x00</code>) is designated as the universal packet delimiter.

However, arbitrary binary structures (<code>float</code> values, integer timestamps, and CRC checksums) naturally contain raw <code>0x00</code> bytes, which would cause premature frame truncation. MicroShield implements Consistent Overhead Byte Stuffing (COBS) [10]:
1. **Zero-Byte Elimination:** The payload is partitioned into sub-blocks delimited by zeros. Each zero byte is replaced with an offset pointer indicating the distance to the next zero.
2. **Minimal Bounded Overhead:** For frames under 254 bytes, COBS adds exactly one prefix overhead byte and one trailing delimiter byte.
3. **Delimiter Uniqueness:** The byte <code>0x00</code> is mathematically guaranteed never to appear within the encoded body, turning every <code>0x00</code> on the wire into an unambiguous end-of-frame signal.

### 4.3.4 Wire Protocol & Cross-Language Pipeline

The physical bit-level packing layout and the symmetric transformation pipeline bridging C99 and Python are modeled below (click image to expand to full resolution):

[![MicroShield Transport-Agnostic Wire Protocol & Pipeline](../../pictures/transport_wire_protocol.png)](../../pictures/transport_wire_protocol.png)

1. **Edge Construction (`microshield_cobs.c`):** The engine writes telemetry metadata into the 32-byte `microshield_telemetry_t` struct, calculates the CRC32 across the first 28 bytes, appends the checksum into the trailing 4 bytes, and applies `microshield_cobs_encode()`.
2. **Asynchronous Dispatch:** The framed byte stream (maximum 34 bytes) is handed off to the non-blocking USART2 DMA peripheral.
3. **Supervisory Parsing (`dashield.transport.framing`):** The Python daemon splits incoming streams by `0x00`, unmasks bytes via `cobs.decode()`, computes `zlib.crc32()` over the payload, and instantiates an immutable `TelemetryRecord`.

### 4.3.5 Verification Evidence & Cross-Language Consistency

Verification of the transport vertical slice was executed across both runtime environments using deterministic synthetic vectors.

#### Dual-Tier Unit Test Results

| Test ID | Test Target / Environment | Input Vector | Expected Output | Verification Status |
| :--- | :--- | :--- | :--- | :--- |
| `UT-C-01` | `edge/tests/test_cobs.c` (GCC C99) | ASCII string `"123456789"` | CRC32 `0xCBF43926` | **PASSED** (Matches IEEE 802.3) |
| `UT-C-02` | `edge/tests/test_cobs.c` (GCC C99) | 8-byte payload with 3 embedded null bytes | 10-byte COBS stream, 100% bit recovery | **PASSED** (Lossless Roundtrip) |
| `UT-C-03` | `edge/tests/test_cobs.c` (GCC C99) | Synthetic `microshield_telemetry_t` frame | CRC32 `0xD8C5B3C9` matches struct field | **PASSED** (Valid Field Integrity) |
| `UT-PY-01`| `tests/test_framing.py` (Python 3.11+) | Binary payload matching `UT-C-03` | Reconstructed `TelemetryRecord` (`ATTACK`, Node 101) | **PASSED** (Cross-Language Match) |
| `UT-PY-02`| `tests/test_framing.py` (Python 3.11+) | Injected 1-bit corruption in CRC field | Rejection with `FramingError("CRC32 mismatch")` | **PASSED** (Tamper Detection) |
| `UT-PY-03`| `tests/test_framing.py` (Python 3.11+) | Truncated 4-byte malformed frame | Rejection with `FramingError("Unexpected length")` | **PASSED** (Truncation Guard) |

#### Verification Execution Logs

The native C99 unit test harness execution log confirms arithmetic conformity on Linux desktop:

<pre style="line-height: 1.25; font-size: 0.85em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #1e293b; padding: 14px 18px; border-radius: 6px; border: 1px solid #334155; overflow-x: auto; color: #f8fafc;">
<span style="color: #94a3b8;">wearemassive@wearemassive:~/microshield/artifact$</span> <span style="color: #38bdf8;">gcc -std=c99 -Wall -Wextra -Werror -Iedge/include edge/src/microshield_cobs.c edge/tests/test_cobs.c -o edge/tests/test_cobs_bin &amp;&amp; ./edge/tests/test_cobs_bin</span>
--- Running MicroShield Edge C99 Framing &amp; Integrity Tests ---
[TEST] CRC32('123456789'): 0xCBF43926 (Expected: 0xCBF43926)
[TEST] COBS Roundtrip: 8 raw bytes -> 10 encoded bytes -> matched
[TEST] TelemetryFrame CRC32 Verified: 0xD8C5B3C9 (Sequence: 0)
--- ALL C99 FRAMING &amp; INTEGRITY TESTS PASSED SUCCESSFULLY ---
</pre>

The companion Python supervisory test suite execution log confirms symmetric validation via `pytest` and `mypy --strict`:

<pre style="line-height: 1.25; font-size: 0.85em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #1e293b; padding: 14px 18px; border-radius: 6px; border: 1px solid #334155; overflow-x: auto; color: #f8fafc;">
<span style="color: #94a3b8;">wearemassive@wearemassive:~/microshield/artifact/supervisor$</span> <span style="color: #38bdf8;">poetry run pytest -v tests/test_framing.py &amp;&amp; poetry run mypy --strict dashield/ tests/</span>
============================= test session starts ==============================
collected 3 items

tests/test_framing.py::test_decode_valid_frame_matching_c_test PASSED    [ 33%]
tests/test_framing.py::test_corrupted_crc_rejection PASSED               [ 66%]
tests/test_framing.py::test_truncated_frame_rejection PASSED             [100%]

============================== 3 passed in 0.03s ===============================
Success: no issues found in 11 source files
</pre>

---

## 4.4 Zero-Copy Feature Extraction & Deterministic Inference Engine

The fast path of the edge intrusion detection system bridges raw network buffer reception to instant packet gatekeeping. This requires two coordinated components: a statistical feature extraction pipeline and a deterministic binary decision tree engine.

### 4.4.1 Zero-Copy Feature Calculation Pipeline

Rather than copying incoming Ethernet frames into temporary scratchpad buffers, `microshield_features.c` operates via direct read-only pointer dereferencing on physical DMA memory. The module constructs the 16-byte `microshield_features_t` structure through four specialized mathematical algorithms:

1. **Normalized Frame Length ($f_0$):** Network frame sizes are saturated to the maximum Ethernet transmission unit (MTU = 1500 bytes) and mapped linearly to the interval $[0.0, 1.0]$. Jumbo frames are capped defensively to prevent metric divergence.
2. **Hardware Rollover-Safe Inter-Arrival Delta ($f_1$):** Inter-packet arrival timing is acquired from the ARM Cortex-M4 Data Watchpoint and Trace (DWT) cycle counter running at 168 MHz. To guard against hardware timer overflow (occurring every &approx; 71.58 minutes on 32-bit registers), delta calculation relies on unsigned integer modular subtraction in $\mathbb{Z}_{2^{32}}$:
   $$\Delta t = t_{\text{curr}} - t_{\text{prev}} \pmod{2^{32}}$$
   This formulation guarantees accurate microsecond intervals across timer wrap-around events without requiring conditional branch overhead.
3. **Defensive Protocol Flag Extraction ($f_2$):** Byte offsets corresponding to TCP control flags (offset 47) or data-link EtherType fields (offset 12) are checked against actual buffer boundaries. Runt packets (length &lt; 14 bytes) default safely to zero, eliminating buffer over-read vulnerabilities.
4. **Numerically Stable Two-Pass Payload Variance ($f_3$):** Single-pass variance estimators based on $\sum x_i^2 - (\sum x_i)^2 / N$ suffer from severe catastrophic cancellation when executed on single-precision IEEE 754 floating-point hardware, frequently resulting in negative variances due to round-off error. MicroShield adopts an industrial two-pass algorithm:
   - **Pass 1:** Accumulates a 32-bit unsigned integer sum of all payload bytes ($\sum x_i \le 1500 \times 255 = 382500$), deriving the exact sample mean $\mu$.
   - **Pass 2:** Accumulates squared deviations $(x_i - \mu)^2$ strictly as positive quantities, guaranteeing $\sigma^2 \ge 0$ with optimal floating-point mantissa precision.

### 4.4.2 MISRA-Compliant Deterministic Decision Tree Traversal

The inference engine implemented in `microshield_engine.c` executes deterministic classification by traversing the static matrices defined in `transpiled_model.h`.

#### Elimination of Recursion (MISRA C:2012 Rule 17.2)
In safety-critical embedded systems, function recursion introduces non-deterministic stack memory growth, posing severe risk of stack overflow and unrecoverable hardware `HardFault` exceptions [11]. In strict compliance with MISRA C:2012 Rule 17.2, `microshield_classify()` implements an iterative traversal loop driven by pre-compiled child index lookup tables:

<pre style="line-height: 1.25; font-size: 0.88em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #f8f9fa; padding: 14px 18px; border-radius: 6px; border: 1px solid #e2e8f0; overflow-x: auto; color: #1e293b;"><code><span style="color: #0284c7;">while</span> ((current_node &gt;= 0) &amp;&amp; 
       (current_node &lt; (<span style="color: #0284c7;">int16_t</span>)MICROSHIELD_TREE_NODE_COUNT) &amp;&amp; 
       (TREE_CHILD_LEFT[current_node] != -1)) {

    <span style="color: #94a3b8;">/* Bounded depth guard: fail-safe against infinite iteration */</span>
    <span style="color: #0284c7;">if</span> (depth &gt;= MICROSHIELD_TREE_MAX_DEPTH) {
        *out_rule_id = 0U;
        *out_split_feature = last_split_feature;
        <span style="color: #0284c7;">return</span> VERDICT_AMBIGUOUS;
    }

    <span style="color: #0284c7;">uint8_t</span> feat_idx = TREE_FEATURE[current_node];
    <span style="color: #0284c7;">float</span> val = extract_feature_by_index(features, feat_idx);
    <span style="color: #0284c7;">float</span> threshold = TREE_THRESHOLD[current_node];

    last_split_feature = feat_idx;
    current_node = (val &lt;= threshold) ? TREE_CHILD_LEFT[current_node] : TREE_CHILD_RIGHT[current_node];
    depth++;
}</code></pre>

- **Bounded Execution Time:** Traversal depth is clamped to `MICROSHIELD_TREE_MAX_DEPTH = 6`, guaranteeing an execution bound of &le; 6 branch iterations (&le; 12 &mu;s @ 168 MHz).
- **Fail-Safe Mechanism:** If tree depth exceeds the bound due to corrupted lookup arrays, traversal halts immediately and returns `VERDICT_AMBIGUOUS`, triggering quarantine action.
- **Intrinsic Attribution:** Terminal leaves (marked by child index `-1`) emit the immutable `rule_id` and the primary `split_feature`, providing immediate XAI explainability without additional latency.

### 4.4.3 Verification Evidence & Unit Test Results

The feature extraction and inference engines were verified through dedicated native C99 test suites on Linux desktop under `-std=c99 -Wall -Wextra -Werror`.

#### Feature Extraction & Inference Test Matrix

| Test ID | Test Target | Test Case Description | Stimulus Vector | Expected Classification / Metric | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `UT-FEAT-01` | `test_features.c` | MTU Length Normalization | Buffer lengths: 60B, 1500B, 2000B | $f_0 \in \{0.04, 1.00, 1.00\}$ (Capped) | **PASSED** |
| `UT-FEAT-02` | `test_features.c` | 32-bit Timer Rollover | $t_{\text{prev}} = \text{0xFFFFFFF0}$, $t_{\text{curr}} = \text{0x00000010}$ | $\Delta t = 32.0\ \mu\text{s}$ (Modular Exact) | **PASSED** |
| `UT-FEAT-03` | `test_features.c` | Protocol Flag Boundary Guard | Offset 47 TCP SYN (0x02) vs. 10B Runt | $f_2 = 0.0078$ (SYN) / $f_2 = 0.0$ (Runt) | **PASSED** |
| `UT-FEAT-04` | `test_features.c` | Two-Pass Variance Precision | Constant buffer vs. $[0, 100, 0, 100]$ | $\sigma^2 = 0.00$ vs. $\sigma^2 = 2500.00$ | **PASSED** |
| `UT-ENG-01`  | `test_engine.c`   | Nominal Industrial Traffic | $\Delta t = 120.0$, $\sigma^2 = 30.0$ | `VERDICT_BENIGN`, Rule ID 1 | **PASSED** |
| `UT-ENG-02`  | `test_engine.c`   | Volumetric Flood Attack | $\Delta t = 20.0$, $L_{\text{norm}} = 0.85$ | `VERDICT_ATTACK`, Rule ID 14 | **PASSED** |
| `UT-ENG-03`  | `test_engine.c`   | High-Entropy Fuzzing Scan | $\Delta t = 20.0$, $\sigma^2 = 180.0$ | `VERDICT_ATTACK`, Rule ID 22 | **PASSED** |
| `UT-ENG-04`  | `test_engine.c`   | Ambiguous Drift Candidate | $\Delta t = 120.0$, $\sigma^2 = 75.0$ | `VERDICT_AMBIGUOUS`, Rule ID 4 | **PASSED** |
| `UT-ENG-05`  | `test_engine.c`   | Null Pointer Defensive Guard | `features = NULL` | `VERDICT_AMBIGUOUS` (Fail-Safe) | **PASSED** |

#### Verification Execution Logs

Execution logs from the feature extractor test harness demonstrate numerical precision across all conditions:

<pre style="line-height: 1.25; font-size: 0.85em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #1e293b; padding: 14px 18px; border-radius: 6px; border: 1px solid #334155; overflow-x: auto; color: #f8fafc;">
<span style="color: #94a3b8;">wearemassive@wearemassive:~/microshield/artifact$</span> <span style="color: #38bdf8;">gcc -std=c99 -Wall -Wextra -Werror -Iedge/include edge/src/microshield_features.c edge/tests/test_features.c -o edge/tests/test_features_bin -lm &amp;&amp; ./edge/tests/test_features_bin</span>
--- Running MicroShield Edge C99 Feature Extractor Tests ---
[TEST] Length Normalization: PASSED (60B, 1500B, 2000B)
[TEST] Timing Delta &amp; Rollover Protection: PASSED (Boot default, nominal, wrap-around)
[TEST] Protocol Flags Extraction: PASSED (TCP SYN detected, runt frame guarded)
[TEST] Two-Pass Variance Accuracy: PASSED (Constant=0.0, Known-dist=2500.0)
--- ALL C99 FEATURE EXTRACTOR TESTS PASSED SUCCESSFULLY ---
</pre>

Execution logs from the inference engine test harness confirm bounded $O(\text{depth})$ path resolution:

<pre style="line-height: 1.25; font-size: 0.85em; font-family: ui-monospace, SFMono-Regular, 'Liberation Mono', Menlo, Consolas, monospace; background-color: #1e293b; padding: 14px 18px; border-radius: 6px; border: 1px solid #334155; overflow-x: auto; color: #f8fafc;">
<span style="color: #94a3b8;">wearemassive@wearemassive:~/microshield/artifact$</span> <span style="color: #38bdf8;">gcc -std=c99 -Wall -Wextra -Werror -Iedge/include -Iedge/model edge/src/microshield_engine.c edge/tests/test_engine.c -o edge/tests/test_engine_bin &amp;&amp; ./edge/tests/test_engine_bin</span>
--- Running MicroShield Edge C99 Inference Engine Tests ---
[TEST] Nominal: verdict=0, rule_id=1, split_feat=3
[TEST] Volumetric Flood: verdict=1, rule_id=14, split_feat=0
[TEST] Fuzzing Attack: verdict=1, rule_id=22, split_feat=3
[TEST] Ambiguous Drift: verdict=2, rule_id=4, split_feat=3
[TEST] Null Pointer Safety: verdict=2
--- ALL C99 INFERENCE ENGINE TESTS PASSED SUCCESSFULLY ---
</pre>

---

## 4.5 References

- [1] I. Sommerville, *Software Engineering*, 10th ed. Boston, MA: Pearson, 2016.
- [2] A. Cockburn, "Hexagonal Architecture: Ports and Adapters," *Alistair Cockburn Humans and Technology*, 2005.
- [3] E. Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software*. Boston, MA: Addison-Wesley, 2004.
- [4] European Commission, "Proposal for a Regulation on horizontal cybersecurity requirements for products with digital elements (Cyber Resilience Act)," COM(2022) 454 final, Brussels, 2022.
- [5] European Parliament and Council of the European Union, "Directive (EU) 2022/2555 on measures for a high common level of cybersecurity across the Union (NIS 2 Directive)," *Official Journal of the European Union*, L 333, pp. 80–152, 2022.
- [6] N. Koroniotis, N. Moustafa, E. Sitnikova, and B. Turnbull, "Towards the Development of Realistic Botnet Dataset in the Internet of Things for Network Forensic Analytics: Bot-IoT Dataset," *Future Generation Computer Systems*, vol. 100, pp. 779–796, 2019.
- [7] M. A. Ferrag, O. Friha, D. Hamouda, L. Maglaras, and H. Janicke, "Edge-IIoTset: A New Comprehensive Realistic Cyber Security Dataset of IoT and IIoT Applications for Centralized and Federated Learning," *IEEE Access*, vol. 10, pp. 40281–40306, 2022.
- [8] J. H. Saltzer, D. P. Reed, and D. D. Clark, "End-to-End Arguments in System Design," *ACM Transactions on Computer Systems (TOCS)*, vol. 2, no. 4, pp. 277–288, 1984.
- [9] IEEE Standards Association, "IEEE Standard for Ethernet," *IEEE Std 802.3-2022*, pp. 1–7025, 2022.
- [10] S. Cheshire and M. Baker, "Consistent Overhead Byte Stuffing," *IEEE/ACM Transactions on Networking*, vol. 7, no. 2, pp. 159–172, 1999.
- [11] MISRA, *MISRA C:2012 - Guidelines for the use of the C language in critical systems*, 3rd ed. Nuneaton, Warwickshire, UK: MIRA Ltd, 2013.
