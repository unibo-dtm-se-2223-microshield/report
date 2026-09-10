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
