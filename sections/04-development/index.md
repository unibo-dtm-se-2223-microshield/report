---
title: Development
has_children: false
nav_order: 5
---

# 4. Implementation & Development

## 4.1 Repository Scaffolding & Polyglot Build Automation

The implementation phase translates the architectural patterns established in the design specification into concrete, verifiable software artifacts. To maintain rigorous separation of concerns while coordinating heterogeneous runtimes, the project repository is partitioned into three autonomous development domains: bare-metal C99 edge firmware, a supervisory Python MLOps suite, and a containerized adversary playback harness.

### 4.1.1 Artifact Subsystem Layout

The physical directory tree of the software repository isolates runtime dependencies, test harnesses, and build scripts into decoupled subsystems, organized as follows:

| Directory / File Path | Architectural Subsystem | Engineering Responsibility & Technical Scope |
| :--- | :--- | :--- |
| <code>edge/include/</code> | Edge Core Runtime | Public C99 contracts, fixed-width integer types (<code>stdint.h</code>), and hexagonal ports. |
| <code>edge/src/</code> | Edge Core Runtime | Zero-heap algorithmic logic (feature calculation, decision tree traversal, COBS encoding). |
| <code>edge/model/</code> | Edge Core Runtime | Transpiled static decision tree lookup matrices (<code>transpiled_model.h</code>). |
| <code>edge/tests/</code> | Edge Testbench | Deterministic unit tests with cycle and memory assertions compiled via native GCC. |
| <code>supervisor/dashield/</code> | Supervisory MLOps Tier | Core Python package orchestrating serial ingestion, drift detection, and operator UI. |
| <code>supervisor/tests/</code> | Supervisory Testbench | Pytest suite validating value object immutability, data types, and drift thresholds. |
| <code>simulation/whispers/</code> | Adversary Simulation | Network traffic playback engine streaming Bot-IoT and Edge-IIoTset benchmark captures. |
| <code>simulation/docker/</code> | Virtual Infrastructure | Container specifications and multi-container orchestration for the testbed. |
| <code>Makefile</code> | Polyglot Root Orchestrator | Declarative build automation bridging the native C99 and Poetry toolchains. |

### 4.1.2 Concrete Software Implementation Pipeline

The execution flow bridging real-time packet interception on the microcontroller to supervisory MLOps analytics is realized through a unidirectional data pipeline. Each compilation translation unit and Python module occupies an explicit stage along this processing path (click image to expand to full resolution):

[![MicroShield Concrete Software Implementation Pipeline](../../pictures/impl_pipeline.png)](../../pictures/impl_pipeline.png)

1. **Ingress & Feature Calculation (<code>microshield_features.c</code>):** Inbound network frames residing in DMA receive buffers are processed via zero-copy pointer access, extracting the 16-byte <code>FeatureVector_t</code> in bounded execution time (&le; 18 &mu;s).
2. **Deterministic Inference (<code>microshield_engine.c</code>):** The extracted vector traverses constant lookup matrices defined in <code>transpiled_model.h</code>, returning a ternary verdict in bounded time (&le; 12 &mu;s).
3. **Framing & Serial Egress (<code>microshield_cobs.c</code>):** Frames classified as <code>ATTACK</code> or <code>AMBIGUOUS</code> trigger the construction of a 32-byte <code>TelemetryFrame_t</code>, protected by an IEEE 802.3 CRC32 checksum and encoded using Consistent Overhead Byte Stuffing (COBS).
4. **Supervisory Parsing (<code>dashield.transport</code>):** The supervisory daemon reconstructs incoming frames across the physical serial link, drops corrupted packets, and yields strongly typed domain transfer objects.
5. **Drift Surveillance & Retraining (<code>dashield.drift</code> & <code>dashield.transpiler</code>):** Validated records feed the rolling ambiguity tracker. When the ratio exceeds 5%, the AST transpiler re-fits the decision tree and emits an updated <code>transpiled_model.h</code> for firmware deployment.
6. **Reactive Presentation (<code>dashield.ui</code>):** The web console visualizes incoming anomalies and presents borderline cases to human operators for triage.

### 4.1.3 Unified Polyglot Orchestration via Root Makefile

Managing a heterogeneous software repository spanning two completely distinct toolchains—the GNU Compiler Collection (GCC) for bare-metal C99 and Poetry for Python 3.11+—introduces operational complexity if workflows are fragmented. The root Makefile establishes a declarative, cross-platform interface exposing standard development targets:

| Make Target | Governed Subsystem | Underlying Toolchain Invocation | Target Acceptance Gate |
| :--- | :--- | :--- | :--- |
| <code>make test-c</code> | Edge Runtime (C99) | Invokes edge Makefile targeting native GCC with strict flags. | Zero test failures, zero memory leaks. |
| <code>make test-py</code> | Supervisory Tier (Python) | Executes <code>poetry run pytest -v tests/</code> across all unit suites. | 100% test pass rate for all domain invariants. |
| <code>make typecheck</code> | Supervisory Tier (Python) | Runs <code>poetry run mypy --strict dashield/ tests/</code>. | Zero type errors, full PEP 484/526 coverage. |
| <code>make lint</code> | Supervisory Tier (Python) | Runs <code>flake8</code> and <code>black --check</code> across Python modules. | Conformity with PEP 8 styling conventions. |
| <code>make test</code> | Full Monorepo | Sequentially executes <code>test-c</code>, <code>test-py</code>, and <code>typecheck</code>. | Universal regression pass before commit. |
| <code>make clean</code> | Full Monorepo | Strips compiled binaries, cached bytecodes, and profiling traces. | Repository reset to pristine source state. |

### 4.1.4 Quality Assurance & Static Verification Gates

In high-reliability and safety-critical embedded systems, runtime failures frequently trace back to undefined behaviors, implicit type promotions, or non-deterministic data structures. To eliminate these failure modes by design, MicroShield enforces uncompromising static verification gates across both runtimes.

For the bare-metal C99 tier, the build harness enforces strict compliance with the ISO/IEC 9899:1999 standard (<code>-std=c99</code>), deliberately prohibiting compiler-specific extensions to guarantee seamless portability across ARM GCC, Keil, and IAR toolchains. All standard and extended diagnostic warnings are activated (<code>-Wall</code>, <code>-Wextra</code>) to trap issues such as uninitialized variables, integer sign mismatches, and unhandled branches. Under the zero-warning policy, every detected warning is elevated to a fatal compilation error (<code>-Werror</code>), halting the build pipeline immediately. Furthermore, during interface design and continuous integration checks, contractual integrity is validated through pure lexical and grammatical parsing (<code>-fsyntax-only</code>), verifying header definitions without generating transient machine binaries.

On the supervisory Python tier, the dynamic nature of the language is constrained to adhere to strict object-oriented paradigms. Static type analysis is enforced through Mypy in strict mode (<code>--strict</code>), mandating explicit type signatures on all functions and attributes while eliminating implicit untyped fallbacks. Domain entities are structured as immutable Value Objects via frozen dataclasses, ensuring thread-safe data transfer between asynchronous serial ingestion routines, drift detection workers, and reactive dashboard rendering threads.
