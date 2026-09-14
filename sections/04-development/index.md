---
title: Development
has_children: false
nav_order: 5
---

# 4. Implementation & Development

## 4.1 Repository Scaffolding & Polyglot Build Automation

The implementation phase translates architectural abstractions into deterministic, verifiable software artifacts. To isolate hardware dependencies while coordinating heterogeneous execution environments, the project monorepo is partitioned into three decoupled functional domains: a bare-metal C99 runtime, a supervisory Python MLOps suite, and a containerized adversary traffic playback harness.

### 4.1.1 Artifact Directory Layout

The physical directory tree isolates compilation units, static models, and testing harnesses:

<pre><code>artifact/
├── Makefile                  # Root polyglot orchestration harness (POSIX make)
├── edge/                     # Deterministic C99 bare-metal runtime
│   ├── include/              # Public hexagonal contracts and data structures
│   ├── src/                  # Zero-heap algorithmic implementations
│   ├── model/                # Transpiled C99 decision tree lookup matrices
│   ├── tests/                # Host and cross-target verification test suites
│   └── Makefile              # Standalone edge compilation harness
├── supervisor/               # Supervisory MLOps tier (DaShield)
│   ├── pyproject.toml        # Poetry workspace and typecheck configuration
│   ├── dashield/             # Supervisory management package
│   │   ├── domain/           # Immutable domain types and value objects
│   │   ├── transport/        # COBS framing and CRC32 verification engine
│   │   ├── drift/            # Sliding-window statistical drift estimators
│   │   ├── transpiler/       # AST-based scikit-learn model compiler
│   │   └── ui/               # Reactive telemetry visualizer
│   └── tests/                # Automated pytest invariant test suite
└── simulation/               # Validation testbed and adversary tooling
    ├── whispers/             # Bot-IoT and Edge-IIoTset packet injection engine
    └── docker/               # Containerized testbed definitions</code></pre>

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

To satisfy the dependability requirements of safety-critical embedded systems and formal software engineering methodology, the build pipeline enforces automated static verification across both compilation environments.

At the edge tier, the C99 build system adheres strictly to the ISO/IEC 9899:1999 standard (`-std=c99`), deliberately disabling non-standard compiler extensions to ensure that source units compile identically under ARM GCC, Keil MDK, and IAR Embedded Workbench. Software dependability is enforced through an uncompromising zero-warning policy: standard and extended diagnostic checks (`-Wall`, `-Wextra`) are configured to treat any warning as an immediate compilation failure (`-Werror`). This setup eliminates implicit type coercions, unused parameters, and unaligned memory offsets before code generation. During architectural scaffolding, public header contracts are formally validated through dry syntax checking (`-fsyntax-only`), verifying types and macro expansions without producing intermediate binary artifacts.

Concurrently, the supervisory Python tier prevents dynamic runtime faults by enforcing strict static type analysis through Poetry and `mypy` under full strict mode (`strict = true`). Dynamic duck-typing is eliminated at compile-time by prohibiting untyped function signatures (`disallow_untyped_defs = true`) and trapping implicit unconstrained return values (`warn_return_any = true`). At the architectural level, domain consistency is safeguarded by modeling core transfer entities as immutable frozen data structures (`@dataclass(frozen=True)`) and mapping categorical verdicts to explicit integer enumerations (`IntEnum`), with invariant preservation verified by automated regression test suites.
