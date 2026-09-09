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
| FR-08 | Supervisory Tier | The system shall transpile a trained and pruned Decision Tree model (scikit-learn) into a standalone, portable C99 header containing static decision lookup matrices. | Automated build check verifying valid C99 syntax, compilation, and equivalent inference outputs. |
| FR-09 | Supervisory Tier | The system shall provide an interactive web dashboard (Dash/Plotly) displaying live telemetry charts, confusion matrices, and model performance metrics. | Browser-based user interface validation test. |
| FR-10 | Validation Testbench | The system shall include an automated traffic playback engine capable of streaming benchmark network flows (Bot-IoT [2] and Edge-IIoTset [3]) to the hardware device under test. | CI pipeline execution measuring classification accuracy against ground truth. |

---

## 2.3 Non-Functional Requirements (NFR) & Quality Attributes

Non-functional requirements define the operational qualities, performance bounds, and safety attributes of the system.

### 2.3.1 Principle of Computational & Energy Transparency
In mission-critical embedded control systems, a security monitor must adhere to the principle of transparent execution: its operational presence must never perturb the deterministic schedule or the thermal/energy budget of the primary industrial process:

Δt_IDS << T_loop   and   E_IDS << E_core

Where Δt_IDS represents the worst-case inspection latency, T_loop denotes the period of the primary sensing and actuation loop, E_IDS is the energy consumed per packet inspection, and E_core is the operational power budget of the primary application.

* NFR-01: Deterministic Real-Time Latency (Computational Transparency)
  * Worst-Case Execution Time (WCET): The packet classification routine shall exhibit an asymptotic time complexity strictly bounded by O(depth), where depth represents the maximum depth of the decision tree.
  * Latency Ceiling: On an ARM Cortex-M4 microcontroller running at 168 MHz, the total inspection time per packet shall not exceed 50 microseconds (Δt_IDS <= 50 µs), ensuring that execution jitter introduced into the primary control cycle remains below 5%.
* NFR-02: Ultra-Low-Power Operation (Energy Transparency)
  * Event-Driven Execution: The edge runtime library shall operate strictly under an interrupt-driven model, executing zero active polling loops (while spinlocks) during idle intervals.
  * Sleep Mode Preservation: The edge library shall permit the microcontroller core to remain in deep sleep (Sleep/Stop modes) until awakened by a peripheral receive interrupt (ISR).
  * CPU Budget: The complete feature extraction and tree traversal routine shall execute in fewer than 500 clock cycles per frame, confining the energy overhead (E_IDS) to less than 1% of the node's total operational power budget.
* NFR-03: Zero Dynamic Memory Allocation (Heapless Architecture)
  * Heap Prohibition: The edge library shall execute zero calls to dynamic memory managers (malloc, calloc, realloc, free), eliminating runtime non-determinism and catastrophic heap fragmentation.
  * Static Footprint: The entire edge runtime shall consume less than 16 KB of non-volatile Flash memory (program code and static decision tables) and less than 4 KB of Static RAM (SRAM) for internal ring buffers.
  * Coding Standards: The C99 codebase shall comply with safety-critical embedded coding standards (MISRA C:2012 guidelines).
* NFR-04: Intrinsic Explainability (Explainable AI - XAI)
  * Every classification decision emitted by the edge engine shall output the internal Rule Identifier (Rule ID) and the specific feature index responsible for the leaf traversal. Black-box inference models (e.g., deep neural networks) are strictly excluded from the edge runtime.
* NFR-05: Diagnostic Channel Integrity & Security
  * The out-of-band telemetry frames dispatched over serial links shall incorporate frame integrity validation (CRC32 checksum) to detect and reject packet corruption or injection attempts on the supervisory bus.
* NFR-06: Software Engineering & Code Quality Standards
  * The Python supervisory ecosystem shall enforce strict static type checking via PEP 484 and PEP 526 annotations validated through mypy, docstring documentation following PEP 257, and automated formatting compliance with black and flake8.

---

## 2.4 Domain Constraints & Regulatory Frameworks

The system architecture is strictly governed by industrial hardware realities and binding European regulatory statutes:

### 2.4.1 Hardware Architectural Constraints
- Processor Architecture: Target platform is the ARM Cortex-M4 core (specifically STMicroelectronics STM32F407VGT6 on the Nucleo-F407RE development board) operating at 168 MHz with 192 KB of SRAM and 1024 KB of Flash.
- Absence of Memory Management Unit (MMU): The hardware core does not support virtual memory translation or process isolation paging. Memory protection relies entirely on static compilation boundaries and optional hardware Memory Protection Unit (MPU) configurations.

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
| Could Have | FR-09, FR-10, NFR-06 | Tooling and visualization: interactive Dash/Plotly dashboard and automated benchmark playback harness. |
| Won't Have | NR-01, NR-02 | Deliberate architectural exclusions (Negative Requirements). |

### Architectural Negative Requirements (Won't Have)

In formal systems engineering, defining what a system must not do is as critical as defining its functional capabilities:

* NR-01: On-Chip Model Training (Deliberately Excluded):
  * Architectural Rationale: Training machine learning models involves computing recursive information gain, entropy matrices, or gradient descent steps. Executing training routines on an ARM Cortex-M4 microcontroller would saturate the CPU, consume significant power, and disrupt real-time control loops, violating the core principle of energy transparency (E_IDS << E_core). Training and optimization belong strictly in the supervisory MLOps tier.
* NR-02: Hardware-Accelerated TLS Payload Decryption (Deliberately Excluded):
  * Architectural Rationale: Transport layer decryption is the exclusive responsibility of the application network stack or dedicated cryptographic co-processors. MicroShield focuses on statistical protocol metadata, packet rates, frame sizes, and unencrypted industrial headers (e.g., Modbus/TCP, MQTT, raw telemetry). Performing full TLS termination within the IDS engine would introduce unviable latency jitter and memory allocation overhead.

---

## 2.6 Requirements Traceability & Visual Hierarchy

The relationship between the formalized requirements, the MoSCoW classification boundaries, and the target operational personas is mapped in the following architectural model:

![MicroShield Requirements Breakdown & MoSCoW Hierarchy](../../pictures/requirements_moscow.png)

### Persona Alignment
- Taddeo Pallabà (Firmware Engineer): Governed by the Must Have tier. Enforces static memory boundaries (no malloc), predictable cycle budgets (<= 500 cycles), and hard real-time latency ceilings (<= 50 µs) to safeguard core application stability.
- Zanni Giorgioni (OT Security Engineer): Governed by the Should Have tier. Manages concept drift detection, reviews explainable decision trees (Rule IDs), and supervises automated retraining cycles.
- Lentina Gigi (Plant Manager): Governed by the Could Have and regulatory compliance tiers. Consumes fleet health dashboards, monitors uptime preservation metrics, and verifies audit trails for CRA/NIS 2 regulatory compliance.

---

## 2.7 References

- [1] I. Sommerville, Software Engineering, 10th ed. Boston, MA: Pearson, 2016.
- [2] N. Koroniotis, N. Moustafa, E. Sitnikova, and B. Turnbull, "Towards the Development of Realistic Botnet Dataset in the Internet of Things for Network Forensic Analytics: Bot-IoT Dataset," Future Generation Computer Systems, vol. 100, pp. 779–796, 2019.
- [3] M. A. Ferrag, O. Friha, D. Hamouda, L. Maglaras, and H. Janicke, "Edge-IIoTset: A New Comprehensive Realistic Cyber Security Dataset of IoT and IIoT Applications for Centralized and Federated Learning," IEEE Access, vol. 10, pp. 40281–40306, 2022.
- [4] European Commission, "Proposal for a Regulation on horizontal cybersecurity requirements for products with digital elements (Cyber Resilience Act)," COM(2022) 454 final, Brussels, 2022.
- [5] European Parliament and Council of the European Union, "Directive (EU) 2022/2555 on measures for a high common level of cybersecurity across the Union (NIS 2 Directive)," Official Journal of the European Union, L 333, pp. 80–152, 2022.

