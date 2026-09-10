---
title: Design
has_children: false
nav_order: 4
---

# 3. Architectural & Detailed Design

## 3.1 Architectural Style & Component Decoupling

The design phase formalizes the Solution Domain—the structural, behavioral, and infrastructural patterns chosen to satisfy the operational constraints established during requirements engineering—while maintaining deliberate independence from low-level implementation libraries [1].

MicroShield is structurally partitioned across two fundamentally distinct operational environments: a hard real-time, resource-constrained bare-metal microcontroller unit (Edge Tier) and an asynchronous supervisory telemetry workstation (Supervisory MLOps Tier). A monolithic architectural paradigm is incapable of addressing both sets of constraints simultaneously.

### 3.1.1 Evaluation of Candidate Architectural Styles

To establish an optimal architectural pattern, four classical software paradigms were systematically evaluated against the system's operational constraints:

| Architectural Style | Theoretical Foundation & Mechanics | Suitability for Edge Runtime | Rejection Rationale & Technical Failure Modes |
| :--- | :--- | :--- | :--- |
| Strictly Layered | Components are organized hierarchically; requests traverse contiguous abstraction layers sequentially (Hardware -> Driver -> Protocol -> Security -> Application). | Rejected | Introduces deep function-call invocation chains. On an ARM Cortex-M4, every intermediate layer traversal incurs stack pointer adjustments, register spills (push/pop of R0-R3, R12, LR), and pipeline stalls, violating the strict sub-50 µs latency budget (NFR-01). |
| Shared Dataspace (Blackboard / Tuple Space) | Decoupled components communicate exclusively by writing, reading, and consuming structured tuples from a centralized, concurrent memory pool (e.g., Linda model) [1]. | Rejected | Requires mutual exclusion primitives (mutexes, spinlocks, or semaphores). On a bare-metal microcontroller without an RTOS, shared-memory locking introduces severe priority inversion risks (as demonstrated historically in the Mars Pathfinder incident) and non-deterministic execution times, while violating the zero-dynamic-allocation mandate (NFR-03). |
| Microkernel (Plug-in Architecture) | A minimal core executes basic hardware mediation while extended capabilities are loaded dynamically as modular shared libraries or drivers. | Rejected | Relies on dynamic linking, relocation tables, and virtual memory page mapping, mechanisms unavailable on ARM Cortex-M microcontrollers lacking a Memory Management Unit (MMU). |
| Hexagonal (Ports & Adapters) + Event-Driven Pipeline | The algorithmic domain logic is isolated at the center of an abstraction boundary (Hexagon) communicating via abstract interfaces (Ports); concrete hardware peripherals interact via boundary drivers (Adapters). | Adopted | The pure C99 detection logic remains entirely decoupled from target microcontroller registers. Enables automated desktop host testing via POSIX mocks and native compilation on bare-metal hardware without modifying a single line of classification code. |

### 3.1.2 The Edge Hexagonal Architecture (Ports & Adapters)

At the Edge Runtime Tier, MicroShield implements Cockburn's Hexagonal Architecture pattern to enforce complete decoupling between safety-critical detection logic and underlying silicon hardware. 

The algorithmic core resides inside an isolated logical boundary, communicating with external hardware peripherals strictly through abstract contractual interfaces (Ports) implemented by concrete device drivers (Adapters):

- Hexagonal Domain Core: Contains strictly deterministic, portable C99 algorithmic logic. It performs windowed statistical feature extraction and traverses pre-compiled decision tree matrices. The core possesses zero awareness of hardware registers, STMicroelectronics Hardware Abstraction Layer (HAL) definitions, or low-level timer peripherals.
- Frame Ingress Port: A formal interface through which incoming data-link frames are injected into the detection pipeline via static memory pointers.
- Gatekeeping Port: An outbound contractual interface that signals the core industrial firmware application whether an intercepted frame is benign (forwarded immediately) or malicious (quarantined and dropped).
- Telemetry Egress Port: An outbound interface that offloads anomaly descriptors to diagnostic serial buffers without blocking the primary execution thread.

### 3.1.3 Supervisory Event-Driven Pipeline Architecture

On the host workstation, the supervisory tier is structured as an asynchronous, event-driven data processing pipeline. Incoming diagnostic frames arriving over the serial telemetry bus generate discrete domain events that propagate through four decoupled stages:
- Serial Ingestion Daemon: Manages non-blocking character reading from the operating system virtual COM port (VCP), reconstructs framed packets, and enqueues parsed payloads into an in-memory ring buffer.
- Concept Drift Analyzer: Subscribes to the ingestion queue and computes statistical ambiguity distributions across sliding temporal windows using a pluggable Strategy pattern.
- AST Decision Tree Transpiler: A model compiler that consumes scikit-learn tree graphs, evaluates leaf node distributions, and emits optimized, MISRA-compliant C99 header files.
- Interactive Operations UI: A reactive web service built with Dash and Plotly that streams network flow analytics, confusion matrices, and audit logs to operations personnel.

### 3.1.4 Component Decomposition & Architectural Responsibilities

The functional boundaries and interaction points between the edge and supervisory tiers are illustrated in the component model below (click image to expand to full resolution):

[![MicroShield Dual-Tier Component & Hexagonal Architecture](../../pictures/design_components.png)](../../pictures/design_components.png)

### Architectural Responsibilities by Component:

| Component Identifier | Hosting Tier | Primary Engineering Responsibility | Interface Contract & Coupling |
| :--- | :--- | :--- | :--- |
| Statistical Feature Extractor | Edge Runtime (C99) | Calculates normalized length, inter-arrival time delta, protocol control flags, and payload byte variance across a statically allocated sliding history. | Coupled only to static frame buffers; zero library dependencies. |
| Transpiled Decision Logic | Edge Runtime (C99) | Evaluates the extracted feature vector via static lookup arrays; returns ternary state (BENIGN, ATTACK, AMBIGUOUS) in bounded O(depth) time. | Strictly constant-time traversal; consumes transpiled_model.h. |
| COBS Framing Adapter | Edge Runtime (C99) | Encodes diagnostic alert structures using Consistent Overhead Byte Stuffing and computes CRC32 checksums for physical serial transmission. | Operates over dedicated DMA-backed circular transmission buffers. |
| Serial Ingestion Daemon | Supervisory (Python) | Deserializes incoming byte streams, verifies CRC32 integrity, and populates immutable AnomalyRecord instances. | Non-blocking OS serial interface; producer for the telemetry queue. |
| Concept Drift Analyzer | Supervisory (Python) | Evaluates rolling ambiguity ratios; triggers a drift alert when ambiguous classifications exceed the 5% threshold. | Implements AbstractDriftDetector Strategy interface. |
| AST Model Transpiler | Supervisory (Python) | Generates static C99 decision lookup arrays and rule boundaries from trained Python estimators. | Consumes DecisionTreeClassifier; outputs valid C header syntax. |
| Interactive Dashboard | Supervisory (Python) | Visualizes telemetry streams, alerts, and system health in a multi-page web browser interface. | Consumes SQLite event repository; isolated from serial ingestion thread. |

---

## 3.2 Distributed Infrastructure & Communication Topologies

MicroShield operates across a physically distributed, heterogeneous topology connecting real-time industrial edge nodes with supervisory workstations.

### 3.2.1 High-Level Central-Star Diagnostic Topology

At the highest level of abstraction, the system establishes a Central-Star Diagnostic Architecture. While industrial field devices participate in diverse operational topologies (such as peer-to-peer wireless meshes among mobile robots, linear multidrop RS-485 factory buses, or redundant industrial Ethernet rings), each MicroShield-enabled node maintains a dedicated, out-of-band diagnostic link routed directly to the supervisory management station.

This decoupled star topology ensures that diagnostic telemetry traffic generated during severe cyberattack incidents cannot saturate, degrade, or introduce communication jitter into the operational control network.

### 3.2.2 Execution Environments & Hardware Allocation

The physical allocation of computational tasks is partitioned across two target environments:

1. Edge Node Platform (STMicroelectronics STM32F407VGT6):
   - Core Architecture: ARM Cortex-M4 32-bit RISC core with Hardware Floating Point Unit (Single-Precision FPU).
   - Clock Frequency: 168 MHz (delivering up to 210 DMIPS / 1.25 DMIPS/MHz).
   - Memory Mapping: 1024 KB on-chip non-volatile Flash memory for instructions and constant lookup tables; 192 KB contiguous Static RAM (112 KB System SRAM, 16 KB Auxiliary SRAM, 64 KB Core Coupled Memory - CCM Data RAM).
   - Hardware Isolation: CCM RAM is utilized specifically for the IDS feature calculation workspace, completely bypassing the shared multi-layer AHB bus matrix to eliminate contention with CPU instruction fetching and direct memory access (DMA) transfers.
2. Supervisory Station Platform (Industrial Workstation / Server):
   - Processor Architecture: x86-64 multi-core processor running modern Linux distributions.
   - Execution Environment: CPython 3.11+ runtime leveraging native vectorized math libraries (NumPy, SciPy) and parallel dashboard request handling.

### 3.2.3 Comparative Evaluation of Edge Diagnostic Physical Interfaces

The choice of physical diagnostic interface directly determines transmission latency, cabling complexity, and hardware electrical resilience:

| Physical Interface | Bandwidth & Wire Complexity | Master/Slave Protocol Dependency | Industrial Resilience & Noise Immunity | Selection Decision & Justification |
| :--- | :--- | :--- | :--- | :--- |
| I2C (Inter-Integrated Circuit) | 100 to 400 kbit/s; 2 wires (SDA, SCL). | Strict Master/Slave; microcontroller cannot initiate asynchronous alerts autonomously without dedicated interrupt lines. | Vulnerable to capacitive bus loading and electromagnetic interference (EMI) beyond 1 meter. | Rejected: Incompatible with long-distance diagnostic routing and asynchronous edge-initiated alert pushes. |
| SPI (Serial Peripheral Interface) | Up to 42 Mbit/s; 4 wires (MOSI, MISO, SCK, CS per node). | Synchronous Master/Slave; requires dedicated Chip Select routing for multi-drop topologies. | High speed but poor noise immunity over cabling lengths exceeding 0.5 meters; pin-intensive. | Rejected: Pin count scaling is prohibitive; synchronous master-polling violates event-driven edge alert requirements. |
| UART / USART (Universal Asynchronous Receiver-Transmitter) | Configurable (115.2 kbit/s to 10.5 Mbit/s); 2 wires (TX, RX). | Asynchronous Full-Duplex; edge node initiates transmissions autonomously via internal interrupt or DMA. | Highly versatile; translatable directly to USB Virtual COM Ports (VCP), differential RS-485 for factory spans, or wireless bridges (LoRa, BLE, cellular). | Adopted: Minimal pin overhead, native asynchronous alert capability, zero master-polling latency, and ubiquitous host workstation compatibility. |

### 3.2.4 Open Systems Interconnection (OSI) Stack Placement

To ensure universal compatibility across diverse industrial applications, MicroShield positions its packet inspection boundary strictly at OSI Layer 2 (Data-Link Layer), interposing directly between physical layer transceivers and host network stacks:

| OSI Layer | Protocol Examples in Scope | MicroShield Operational Role & Boundary |
| :--- | :--- | :--- |
| Layer 7 - Application | Modbus/TCP, MQTT, CoAP, DNP3, CIP | Consumed by core application firmware; unparsed by MicroShield to avoid deep payload inspection overhead. |
| Layer 4 - Transport | TCP, UDP | Header parsing bypassed; transport behavior observed indirectly through statistical flow properties. |
| Layer 3 - Network | IPv4, IPv6, 6LoWPAN | IP header lengths and address variation evaluated via lightweight statistical extraction. |
| Layer 2 - Data-Link | Ethernet MAC, CAN Frame, SLIP/COBS | INLINE BUMP-IN-THE-WIRE INSPECTION BOUNDARY: Real-time feature calculation and ternary decision tree classification. |
| Layer 1 - Physical | 10/100 Ethernet PHY (RMII), RS-485 | Hardware signal acquisition via dedicated peripheral interrupt service routines (ISRs). |

- Bump-in-the-Wire Rationale: By intercepting inbound frames directly from the physical layer controller (PHY/MAC) before parsing by higher-level software network stacks (e.g., LwIP), MicroShield protects the node against low-level resource exhaustion attacks, packet malformation exploits, and transport layer denial-of-service floods.
- Protocol Agnosticism: Operating at Layer 2 enables the statistical feature extractor to derive universal behavioral metrics (frame length distributions, burstiness, inter-arrival intervals) regardless of whether the encapsulated payload is unencrypted Modbus, MQTT, or an encrypted TLS tunnel.

### 3.2.5 Physical Framing & Data Integrity Protocol (COBS + CRC32)

Serial diagnostic links are asynchronous byte streams lacking native frame delimiters. To guarantee unambiguous frame boundary reconstruction without risking data corruption, MicroShield implements Consistent Overhead Byte Stuffing (COBS) paired with a trailing Cyclic Redundancy Check (CRC32):

1. Delimiter Independence: A null byte (0x00) is reserved exclusively as the end-of-frame packet delimiter.
2. Deterministic Overhead: COBS encodes payload bytes such that null bytes appearing within the actual telemetry record are eliminated through pointer substitution, guaranteeing an overhead bounded by at most 1 byte per 254 bytes of payload.
3. Transmission Validation: Every telemetry frame concludes with an IEEE 802.3 compliant 32-bit CRC. The supervisory ingestion daemon validates the checksum prior to allocating domain entity objects, dropping corrupted frames caused by industrial line noise.

### 3.2.6 Deployment Topography & Regulatory Security Boundaries

The physical deployment topology, bus routing, and regulatory security perimeters mandated by the EU Cyber Resilience Act (CRA) and NIS 2 Directive are modeled in the deployment diagram below (click image to expand to full resolution):

[![MicroShield Distributed Deployment & Physical Infrastructure](../../pictures/design_deployment.png)](../../pictures/design_deployment.png)

### 3.2.7 Architectural Scalability: Transitioning to Horizontally Scaled Infrastructure

While the experimental validation testbench executes the supervisory suite on a single workstation, the underlying architecture is deliberately decoupled to support enterprise-grade horizontal scaling across industrial plants:
- Broker-Mediated Fleet Ingestion: The point-to-point serial link between field nodes and the supervisory station can be replaced transparently by industrial IoT edge gateways running MQTT, Kafka, or Zenoh brokers. The Python SerialIngestionDaemon conforms to an abstract transport interface (AbstractTelemetryTransport), allowing substitution with a DistributedBrokerTransport without modifying downstream drift analysis or retraining pipelines.
- Stateless Analytics Workers: The ConceptDriftAnalyzer maintains state only across configurable sliding temporal windows. In a multi-plant deployment, multiple worker processes can be orchestrated across container clusters (e.g., Kubernetes) to process partition-sharded telemetry from thousands of field nodes concurrently.
- Centralized Model Registry: Re-transpiled C99 decision headers can be committed directly to a GitOps repository or firmware over-the-air (FOTA) distribution server, enabling automated canary updates across the entire device fleet.

---

## 3.3 Domain-Driven Design (DDD) Modelling

To prevent domain logic corruption and manage cognitive complexity across the edge-to-cloud spectrum, MicroShield strictly applies the principles of Domain-Driven Design (DDD) [1]. The Problem Domain is segregated into three autonomous **Bounded Contexts**, each characterized by a unified Ubiquitous Language, explicit boundary contracts, and tailored data models.

### 3.3.1 Bounded Context Formalization & Ubiquitous Language

The system segregates operational responsibilities across three autonomous contexts:

| Bounded Context | Governing Persona | Operational Boundary & Purpose | Ubiquitous Vocabulary |
| :--- | :--- | :--- | :--- |
| **Edge Detection Context** | Taddeo Pallabà (Firmware Engineer) | Hard real-time packet interception, feature normalization, and deterministic ternary inference executed bare-metal on the microcontroller[cite: 1]. | RawFrame, FeatureVector, ClassificationVerdict, RuleDescriptor, RingBuffer, StaticLookupMatrix. |
| **Supervisory MLOps Context** | Zanni Giorgioni (OT Security Engineer) | Telemetry stream ingestion, sliding-window statistical drift tracking, model retraining triggers, and automated C99 code generation[cite: 1]. | AnomalyRecord, AmbiguityRatio, ConceptDrift, RollingWindow, TranspilationPipeline, ASTNode. |
| **Compliance & Audit Context** | Lentina Gigi (Plant Manager) | Forensic logging, incident reconstruction, uptime preservation monitoring, and automated generation of CRA/NIS 2 regulatory compliance evidence[cite: 1]. | AuditTrail, ForensicEvidence, NonRepudiationHash, IncidentReport, CRAArticle10Compliance. |

### 3.3.2 Domain Concepts & Structural Classification

Within each bounded context, structural concepts are classified strictly into Value Objects, Entities, Aggregate Roots, Domain Events, and Domain Services[cite: 1]:

#### 1. Edge Detection Context (Bare-Metal C99)
- **Value Objects (Immutable by definition):**
  - `RawFrame`: A contiguous byte array representing an intercepted data-link packet, encapsulated with a hardware microsecond timestamp and length descriptor. Lacks identity; two identical frames are computationally interchangeable.
  - `FeatureVector`: A 128-bit statically allocated struct containing four normalized single-precision floating-point metrics (`norm_length`, `delta_time_us`, `protocol_flags`, `byte_variance`).
  - `ClassificationVerdict`: An immutable ternary enumeration (`BENIGN = 0`, `ATTACK = 1`, `AMBIGUOUS = 2`).
  - `RuleDescriptor`: A lightweight compound identifier exporting the active decision tree leaf ID and the specific feature index responsible for the split.
- **Aggregate Root:**
  - `DetectionEngine`: The operational gatekeeper. It encapsulates the static circular reception buffers, orchestrates the atomic state transition of intercepted frames (quarantined vs. forwarded), and exposes strict public API boundaries preventing external corruption of internal lookup matrices[cite: 1].
- **Domain Events:**
  - `FrameIntercepted`: Published when a complete frame is loaded into the DMA reception buffer.
  - `FrameQuarantined`: Emitted when an anomaly is verified, triggering immediate payload suppression.
  - `TelemetryEmitted`: Generated when an attack or ambiguous classification is queued for serial transmission.

#### 2. Supervisory MLOps Context (Python 3.11+)
- **Entities (Identity-driven):**
  - `AnomalyRecord`: Represents an intercepted anomalous event. Even if two records share identical feature vectors, they maintain distinct operational identities defined by an immutable UUID, a high-resolution UTC timestamp, and the physical `NodeID`.
- **Aggregate Root:**
  - `FleetDriftTracker`: Governs the sliding temporal history of received classifications. It maintains consistency across rolling ambiguity evaluations and enforces the state transitions of the retraining trigger[cite: 1].
- **Domain Services:**
  - `ASTTranspilerService`: A stateless domain service that consumes trained scikit-learn tree estimators, validates depth constraints against the worst-case execution time (WCET) budget, and emits portable C99 header files[cite: 1].
  - `DriftEvaluationService`: Executes rolling statistical hypothesis testing to detect concept drift without altering the telemetry storage state[cite: 1].
- **Domain Events:**
  - `ConceptDriftDetected`: Published when ambiguous decisions exceed the 5% threshold over the evaluation window.
  - `ModelRetrained`: Emitted following offline supervised estimator convergence.
  - `TranspilationCompleted`: Signifies that a new `transpiled_model.h` has been validated and staged for firmware deployment.

#### 3. Compliance & Audit Context (Regulatory)
- **Entities & Repositories:**
  - `SecurityIncident`: An immutable forensic audit entity capturing incident telemetry paired with a cryptographic SHA-256 integrity hash.
  - `IForensicEvidenceRepository`: An abstract persistence repository providing append-only guarantees for statutory compliance under the EU Cyber Resilience Act (CRA) Article 10 and NIS 2 Directive[cite: 1].
- **Domain Events:**
  - `CRAIncidentReported`: Triggered when high-severity malicious anomalies require formal European vulnerability registry notifications.

### 3.3.3 Context Map & Contractual Relationships

The relationships, boundary governance, and integration patterns between the bounded contexts are formally modeled in the Context Map below (click image to expand to full resolution)[cite: 1]:

[![MicroShield Domain-Driven Design Context Map](../../pictures/design_ddd_context_map.png)](../../pictures/design_ddd_context_map.png)

- **Upstream/Downstream & Customer/Supplier (Edge -> MLOps):** The *Edge Detection Context* acts as an **Upstream (U)** supplier delivering raw telemetry frames to the **Downstream (D)** *Supervisory MLOps Context*. The supervisory suite functions as a customer, establishing processing SLAs without possessing authority to alter edge timing contracts.
- **Published Language (PL):** Integration between the bare-metal C99 runtime and the Python supervisory tier is mediated exclusively through a Published Language: the **COBS + CRC32 Serial Telemetry Protocol**. Neither context exposes internal memory pointers or runtime objects across the physical serial boundary.
- **Audit Downstream Ingestion:** The *Compliance & Audit Context* consumes validated alerts from both upstream contexts, enforcing non-repudiation and append-only constraints on incident logs.

---

## 3.4 Structural & Object-Oriented Modelling

To satisfy academic software engineering criteria and ensure code maintainability, the system establishes a clean separation between high-level object-oriented design on the supervisory workstation and deterministic object-based data layouts on the embedded microcontroller[cite: 1].

### 3.4.1 Supervisory Tier Object-Oriented Architecture (Python 3.11+)

The supervisory ecosystem is structured around pure object-oriented design patterns, leveraging Python's `abc` module for abstract base classes, frozen dataclasses for domain value objects, and strict static type typing (PEP 484, PEP 526)[cite: 1]:

- **Strategy Pattern (Algorithmic Drift Detection):**
  The interface `AbstractDriftDetector` defines the abstract contract `update(record: TelemetryRecord) -> None` and `is_drift_detected() -> bool`[cite: 1]. The concrete class `RollingAmbiguityDetector` implements the baseline sliding-window ratio. This pattern permits seamless algorithmic substitution (e.g., swapping to ADWIN, Page-Hinkley, or Kolmogorov-Smirnov statistical tests) without modifying the ingestion orchestrator or the dashboard presentation layer.
- **Observer Pattern (Reactive Event Dispatch):**
  The `FleetDriftTracker` aggregate root maintains an internal subscriber registry[cite: 1]. When an incoming telemetry frame triggers a `ConceptDriftDetected` event, all registered observers (the Dash live visualizer, the SQLite forensic logger, and the automated transpilation pipeline) are notified asynchronously without tight coupling.
- **Immutable Domain Transfer Objects:**
  Incoming serial payloads are deserialized directly into immutable `TelemetryRecord` dataclasses (`frozen=True`). This guarantees thread safety across concurrent dashboard rendering and drift calculation routines, preventing subtle state corruption bugs.
- **Visitor Pattern (AST Code Transpiler):**
  The `DecisionTreeTranspiler` traverses the underlying binary decision tree structure (`tree_.children_left`, `tree_.children_right`, `tree_.threshold`) emitted by scikit-learn. It converts the hierarchical estimator graph into constant C99 array initializers without coupling model validation logic to C syntax generation[cite: 1].

### 3.4.2 Edge Runtime Object-Based Architecture (C99 Bare-Metal)

On the STM32F407 microcontroller, object-oriented concepts are realized using **Object-Based C99** idioms optimized for single-cycle execution and deterministic memory alignment[cite: 1]:
- **Encapsulation via Header Modularity:** Private internal variables (ring buffer pointers, threshold tables) are marked `static` within the implementation translation unit (`microshield_engine.c`). External callers interact exclusively through opaque function signatures defined in `microshield_engine.h`.
- **Elimination of Virtual Tables (vptrs):** Dynamic dispatch (function pointers inside structs) is deliberately excluded from the real-time fast path. In ARM Cortex-M4 architectures, indirect function calls through vtables disrupt instruction prefetching, introduce branch misprediction latency, and consume additional SRAM for pointer tables. All classification calls are resolved at link-time as direct relative branches (`BL` instructions).
- **Structure Padding & Cache Alignment:** Memory structs (`RawFrame_t`, `FeatureVector_t`, `TelemetryFrame_t`) are explicitly designed with natural 32-bit alignment (4-byte boundaries). This eliminates compiler-induced structure padding overhead and prevents unaligned memory access penalties on the ARM Cortex-M bus interface.

### 3.4.3 Unified Structural Class Model

The complete structural organization, interface contracts, and relationships bridging the Python supervisory hierarchy and the C99 memory layout are detailed in the class diagram below (click image to expand to full resolution)[cite: 1]:

[![MicroShield Structural Class & Data Type Model](../../pictures/design_class_diagram.png)](../../pictures/design_class_diagram.png)

### 3.4.4 Distributed Mapping of Domain Concepts

The mapping between logical domain concepts, software data types, and their physical execution environments is formalized in the structural cross-reference table below[cite: 1]:

| Domain Concept | Software Data Type / Construct | Runtime Hosting Environment | Concurrency & Access Model |
| :--- | :--- | :--- | :--- |
| **RawFrame** | `RawFrame_t` (C99 `struct`) | Edge SRAM (CCM Data RAM) | Single-producer (DMA ISR), single-consumer (Engine). Zero-copy access via pointers. |
| **FeatureVector** | `FeatureVector_t` (C99 `struct`) | Edge Registers / Stack Frame | Allocated strictly on stack; lifetime confined to single-packet inspection cycle. |
| **TelemetryFrame** | `TelemetryFrame_t` (Packed C99) | Edge TX Ring Buffer (SRAM) | Written by IDS engine on anomaly; read by UART DMA controller. |
| **TelemetryRecord** | `TelemetryRecord` (Python `@dataclass`) | Supervisory Heap (CPython) | Immutable; shared concurrently across ingestion, drift monitor, and dashboard threads. |
| **DriftDetector** | `AbstractDriftDetector` (Python `ABC`) | Supervisory Workstation Process | Evaluated synchronously on packet arrival within the ingestion worker thread. |
| **IncidentAuditLog** | `SecurityIncident` (Python Class / SQL) | Persistent Storage (SQLite/Parquet) | Append-only write transactions protected via file locks; concurrent read by UI. |
