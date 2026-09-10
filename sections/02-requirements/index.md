---
title: Requirements
has_children: false
nav_order: 3
---

# 2. Requirements Engineering

## 2.1 Problem Domain Boundary & Methodology

In software engineering methodology, requirements analysis formalizes the Problem Domain—the operational conditions, functional capabilities, and behavioral boundaries of the target environment—strictly decoupled from the Solution Domain, which addresses architectural design, data structures, and implementation technologies [1].

MicroShield operates at the convergence of three demanding domains:
1. Ultra-Constrained Embedded Computing: Resource-restricted microcontroller units (MCUs) deployed in industrial sensing and control loops.
2. Deterministic Intrusion Detection: Line-rate network anomaly classification operating under hard real-time constraints.
3. Supervisory MLOps & Active Learning: High-level fleet monitoring, telemetry aggregation, and continuous model updating.

To establish verifiable engineering criteria, all requirements are formalized using unique identifiers, active grammatical syntax, and quantifiable metrics.

---

## 2.2 Functional Requirements (FR)

Functional requirements specify the operational capabilities and state transformations that MicroShield must execute.

| Requirement ID | Operational Scope | Requirement Statement | Verification Method |
| :--- | :--- | :--- | :--- |
| FR-01 | Edge Runtime | The system shall intercept inbound network communication frames at the physical/data-link interface (UART, SPI, Ethernet MAC) before passing payload data to the core firmware application. | Hardware loopback stimulus and frame capture test. |
| FR-02 | Edge Runtime | The system shall extract normalized statistical features (frame length, inter-arrival time delta, protocol control flags, payload byte variance) using statically pre-allocated memory buffers. | Unit test with synthetic packet injection comparing extracted vectors. |
| FR-03 | Edge Runtime | The system shall evaluate the extracted feature vector against a pre-compiled decision logic model and classify the frame into a ternary state: BENIGN, ATTACK, or AMBIGUOUS. | Matrix validation against ground-truth labeled test vectors. |
| FR-04 | Edge Runtime | The system shall forward BENIGN frames to the core application without modification and quarantine ATTACK frames by suppressing downstream processing. | End-to-end integration test observing core application input queue. |
| FR-05 | Supervisory Tier | Upon intercepting an ATTACK or AMBIGUOUS frame, the edge engine shall construct a structured telemetry frame and dispatch it over a dedicated diagnostic serial channel. | Serial packet sniffer verifying packet framing, metadata, and timestamps. |
| FR-06 | Supervisory Tier | The Python supervisory suite shall ingest incoming serial telemetry records, parse anomaly metadata, and buffer samples in a circular FIFO queue. | Automated test suite verifying buffer ingestion rates under load. |
| FR-07 | Supervisory Tier | The supervisory suite shall compute the rolling ambiguity ratio over a configurable window and trigger a concept drift alert when ambiguous classifications exceed 5% of total events. | Simulated statistical drift injection and assertion of trigger events. |
| FR-08 | Supervisory Tier | The system shall transpile a trained and pruned Decision Tree model into a standalone, portable C99 header containing static decision lookup matrices. | Automated build check verifying valid C99 syntax, compilation, and equivalent inference outputs. |
| FR-09 | Supervisory Tier | The system shall provide an interactive web dashboard displaying live telemetry charts, confusion matrices, and model performance metrics. | Browser-based user interface validation test. |
| FR-10 | Validation Testbench | The system shall include an automated traffic playback engine capable of streaming benchmark network flows (Bot-IoT [2] and Edge-IIoTset [3]) to the hardware device under test. | CI pipeline execution measuring classification accuracy against ground truth. |

---

## 2.3 Non-Functional Requirements (NFR) & Quality Attributes

Non-functional requirements define the operational qualities, performance bounds, and safety attributes of the system, governed by the principle of computational and energy transparency:

<div align="center" style="font-size: 1.2em; margin: 1.2em 0; letter-spacing: 0.5px;">
  <i>&Delta;t</i><sub>IDS</sub> &ll; <i>T</i><sub>loop</sub> 
  &emsp;&emsp;<b>&amp;</b>&emsp;&emsp; 
  <i>E</i><sub>IDS</sub> &ll; <i>E</i><sub>core</sub>
</div>

### 2.3.1 Engineering Derivation of Timing & Deterministic Latency Bounds

The requirement for bounded real-time latency (NFR-01) is directly derived from industrial control loop theory:
- Control Loop Dynamics (<i>T</i><sub>loop</sub>): Industrial automation systems, motor actuators, and digital control loops typically operate at a baseline frequency of 1 kHz, corresponding to a period of <i>T</i><sub>loop</sub> = 1000 &mu;s.
- Computational Transparency Budget (<i>&Delta;t</i><sub>IDS</sub>): To prevent an inline intrusion detection routine from inducing jitter or deadline starvation in the core control task, the maximum inspection overhead is bounded at:

<div align="center" style="font-size: 1.15em; margin: 0.8em 0;">
  <i>&Delta;t</i><sub>IDS</sub> &le; 50 &mu;s
</div>

- Loop Impact Ratio: This timing ceiling ensures that the IDS consumes at most 5% of the available 1 kHz loop period (50 &mu;s / 1000 &mu;s = 5%), leaving the remaining 95% of the CPU schedule fully dedicated to sensor acquisition, physical control math, and actuation.
- Cycle Budget on ARM Cortex-M4: At a nominal clock frequency of 168 MHz (1 clock cycle &approx; 5.95 ns), a 50 &mu;s duration provides an absolute ceiling of approximately 8,400 clock cycles. Given that traversing a transpiled decision tree with bounded depth (depth &le; 6) requires &le; 500 CPU cycles (&approx; 2.98 &mu;s), the inspection operates comfortably within interrupt service routine (ISR) slack time, validating the "latency-zero" transparent impact.

### 2.3.2 Quantitative Resource & Energy Overhead Ratios

To enforce true "memory-zero" and "energy-zero" constraints, metrics are quantified both in absolute quantities and as relative percentages of the target hardware envelope (STMicroelectronics STM32F407VGT6 with 1024 KB Flash and 192 KB SRAM):

- Flash Memory Footprint: &le; 16 KB consumption out of 1024 KB total non-volatile memory &rarr; <b>&le; 1.56%</b> program memory utilization.
- Static RAM (SRAM) Utilization: &le; 4 KB out of 192 KB total volatile memory &rarr; <b>&le; 2.08%</b> system SRAM utilization. When allocated within the dedicated 64 KB Core Coupled Memory (CCM Data RAM), it occupies <b>&le; 6.25%</b> of CCM storage, leaving standard multi-layer bus SRAM completely unconstrained.
- CPU Time & Energy Overhead: With a worst-case traversal budget of &le; 500 CPU cycles per packet, an edge device receiving a continuous industrial stream of 100 packets per second consumes:

<div align="center" style="font-size: 1.15em; margin: 0.8em 0;">
  CPU Overhead = (100 &times; 500 cycles/s) / (168 &times; 10<sup>6</sup> cycles/s) &approx; 0.0298% &lt; 0.03%
</div>

This negligible duty cycle guarantees that additional power dissipation is kept well below the 1% operational threshold (<i>E</i><sub>IDS</sub> &ll; <i>E</i><sub>core</sub>).

### 2.3.3 Formal NFR Quality Attribute Matrix

| Requirement ID | Quality Attribute | Quantitative Metric & Hardware Ratio | Verification Method |
| :--- | :--- | :--- | :--- |
| NFR-01 | Real-Time Latency | WCET bounded by O(depth) <= 50 µs (< 5% of 1 kHz loop); CPU cycles <= 500; jitter < 5%. | Hardware DWT cycle counter profiling and oscilloscope GPIO toggling. |
| NFR-02 | Ultra-Low-Power | 100% interrupt-driven (zero polling loops); CPU duty cycle < 0.03%; energy overhead < 1% of node budget. | Current shunt measurement with digital storage oscilloscope across power states. |
| NFR-03 | Heapless Memory | Zero dynamic heap allocation (no malloc/calloc/free). Flash <= 16 KB (<= 1.56%); SRAM <= 4 KB (<= 2.08%). | Linker map file analysis and static analysis with gcc flags (-Wstack-usage, -Wbad-function-cast). |
| NFR-04 | Intrinsic XAI | Output of deterministic Rule ID and split feature index for every leaf decision. Complete exclusion of black-box models. | Unit tests asserting returned Rule IDs against offline decision tree traversal traces. |
| NFR-05 | Telemetry Security | Out-of-band serial telemetry frames protected via CRC32 checksum to guarantee diagnostic integrity. | Fault-injection testing introducing corrupted serial frames and asserting frame rejection. |
| NFR-06 | Code Quality | Python supervisory suite strictly typed (PEP 484/526), formatted with black, and linted with flake8 and mypy. | Automated CI pipeline gate enforcing zero type errors and zero linter warnings. |

---

## 2.4 Domain Constraints & Regulatory Frameworks

The system architecture is strictly governed by industrial hardware realities and binding European regulatory statutes:

### 2.4.1 Hardware Constraints & the Absence of a Memory Management Unit (MMU)

The target hardware core—the ARM Cortex-M4—imposes fundamental engineering constraints:
- Flat Physical Memory Space: Unlike application processors (e.g., ARM Cortex-A), Cortex-M microcontrollers lack a Memory Management Unit (MMU). Consequently, the processor cannot support virtual memory translation, page swapping, or hardware-enforced process privilege isolation.
- Architectural Failure Modes of Dynamic Memory: In a system without an MMU, dynamic memory allocation (`malloc`) is a critical hazard. Heap fragmentation, memory exhaustion, or dangling pointer dereferences cannot be trapped or isolated within a user space process. Any memory fault triggers an immediate, unrecoverable `HardFault_Handler` exception at the silicon level, instantly halting the CPU and crashing the attached physical machinery.
- Engineering Countermeasure: This constraint directly dictates MicroShield's architecture: all feature extraction buffers, ring queues, and decision trees must be allocated statically at compile time with mathematically proven stack depth boundaries, eliminating memory corruption risks by design.

### 2.4.2 Regulatory Compliance Directives
- EU Cyber Resilience Act (CRA): Requires hardware and software products introduced to the EU market to exhibit security by design throughout their operational lifecycle [4]. MicroShield satisfies CRA Article 10 mandates by providing on-device active packet filtering, tamper detection, and auditable vulnerability telemetry.
- Directive (EU) 2022/2555 (NIS 2): Mandates rigorous operational resilience and incident reporting capabilities for critical infrastructure [5]. MicroShield facilitates compliance by generating cryptographically consistent security audit logs for forensic event reconstruction.

---

## 2.5 MoSCoW Prioritization & Deliberate Architectural Exclusions

To balance operational completeness with the project delivery timeframe (100–110 engineering hours), requirements are prioritized according to the MoSCoW methodology:

| Category | Requirement IDs | Core Rationale & Scope Definition |
| :--- | :--- | :--- |
| Must Have | FR-01, FR-02, FR-03, FR-04, NFR-01, NFR-02, NFR-03 | Fundamental operating core: deterministic, heapless C99 packet inspection running transparently on ARM Cortex-M hardware. |
| Should Have | FR-05, FR-06, FR-07, FR-08, NFR-04, NFR-05 | Supervisory intelligence: automated AST transpiler, serial telemetry ingestion, concept drift detection, and explainability. |
| Could Have | FR-09, FR-10, NFR-06 | Tooling and visualization: interactive dashboard and automated benchmark playback harness. |
| Won't Have | NR-01, NR-02 | Deliberate architectural exclusions (Negative Requirements). |

### Architectural Negative Requirements (Won't Have)

In formal systems engineering, defining what a system must not do is as critical as defining its functional capabilities:

* NR-01: On-Chip Model Training (Deliberately Excluded):
  * Architectural Rationale: Training machine learning models involves computing recursive information gain, entropy matrices, or gradient descent steps. Executing training routines on an ARM Cortex-M4 microcontroller would saturate the CPU, consume significant power, and disrupt real-time control loops, violating the core principle of energy transparency (<i>E</i><sub>IDS</sub> &ll; <i>E</i><sub>core</sub>). Training and optimization belong strictly in the supervisory MLOps tier.
* NR-02: Hardware-Accelerated TLS Payload Decryption (Deliberately Excluded):
  * Architectural Rationale: Transport layer decryption is the exclusive responsibility of the application network stack or dedicated cryptographic co-processors. MicroShield focuses on statistical protocol metadata, packet rates, frame sizes, and unencrypted industrial headers (e.g., Modbus/TCP, MQTT, raw telemetry). Performing full TLS termination within the IDS engine would introduce unviable latency jitter and memory allocation overhead.

---

## 2.6 Requirements Traceability & Visual Hierarchy

The relationship between the formalized requirements, the MoSCoW classification boundaries, and the target operational personas is mapped in the following architectural model (click image to open full resolution):

[![MicroShield Requirements Breakdown & MoSCoW Hierarchy](../../pictures/requirements_moscow.png)](../../pictures/requirements_moscow.png)

### Persona Mapping & Operational Requirement Alignment

To ensure traceability from stakeholder objectives to technical implementation, each operational persona is mapped to their governing requirements:

| Target Persona | Operational Role & Context | Aligned MoSCoW Tier | Enforced Requirements | Impact on System Operation |
| :--- | :--- | :--- | :--- | :--- |
| Taddeo Pallabà | Senior Embedded Firmware Engineer | MUST HAVE | FR-01, FR-02, FR-03, FR-04, NFR-01, NFR-02, NFR-03 | Guarantees hard real-time execution bounds (<= 50 µs), static memory boundaries (no malloc), and zero jitter on critical industrial control loops. |
| Zanni Giorgioni | OT Cybersecurity Operations Engineer | SHOULD HAVE | FR-05, FR-06, FR-07, FR-08, NFR-04, NFR-05 | Provides transparent incident explainability (Rule IDs), automated concept drift surveillance (> 5%), and secure diagnostic telemetry. |
| Lentina Gigi | Industrial Plant & Compliance Director | COULD HAVE & Regulatory | FR-09, FR-10, NFR-06, CRA, NIS 2 | Delivers high-level operational dashboards, empirical benchmark verification, and certified audit trails for regulatory compliance. |

---

## 2.7 References

- [1] I. Sommerville, Software Engineering, 10th ed. Boston, MA: Pearson, 2016.
- [2] N. Koroniotis, N. Moustafa, E. Sitnikova, and B. Turnbull, "Towards the Development of Realistic Botnet Dataset in the Internet of Things for Network Forensic Analytics: Bot-IoT Dataset," Future Generation Computer Systems, vol. 100, pp. 779–796, 2019.
- [3] M. A. Ferrag, O. Friha, D. Hamouda, L. Maglaras, and H. Janicke, "Edge-IIoTset: A New Comprehensive Realistic Cyber Security Dataset of IoT and IIoT Applications for Centralized and Federated Learning," IEEE Access, vol. 10, pp. 40281–40306, 2022.
- [4] European Commission, "Proposal for a Regulation on horizontal cybersecurity requirements for products with digital elements (Cyber Resilience Act)," COM(2022) 454 final, Brussels, 2022.
- [5] European Parliament and Council of the European Union, "Directive (EU) 2022/2555 on measures for a high common level of cybersecurity across the Union (NIS 2 Directive)," Official Journal of the European Union, L 333, pp. 80–152, 2022.
