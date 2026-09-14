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
