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
