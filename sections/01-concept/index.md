---
title: Concept
has_children: false
nav_order: 2
---

# 1. Concept

## 1.1 Problem Statement & Industrial Motivation

The accelerated digital transformation of modern industrial environments—conceptualized under Industry 4.0 and Cyber-Physical Systems (CPS)—relies on pervasive connectivity among physical operational technologies (OT), distributed field sensing nodes, and central supervisory computing platforms [1], [2]. In manufacturing plants, energy distribution grids, and smart infrastructure, low-power microcontrollers (MCUs) serve as the fundamental execution layer for data acquisition, physical actuation, and real-time fieldbus networking [2]. However, this widespread connectivity has dismantled the traditional security model based on the physical isolation of air-gapped networks [1].

Industrial IoT (IIoT) edge nodes are continuously exposed to network-level cyber threats, ranging from volumetric Denial of Service (DoS/DDoS) floods to stealthy network probing, ARP/DNS cache poisoning, and unauthorized payload injection [1], [2]. While high-end enterprise servers and cloud infrastructure defend boundaries using multi-layered intrusion detection systems (IDS) and deep packet inspection (DPI) engines, field-level sensing and control devices remain unprotected [2].

Microcontroller platforms, such as the ARM Cortex-M architecture, operate under strict computational and hardware constraints:
- **Stringent Memory Boundaries:** Static RAM (SRAM) is typically constrained to tens or hundreds of kilobytes (e.g., 192 KB on the STM32F407 family), alongside limited non-volatile Flash storage (e.g., 1024 KB).
- **Absence of Virtual Memory Architecture:** These microcontrollers lack a hardware Memory Management Unit (MMU), executing firmware bare-metal or on top of lightweight real-time kernels such as FreeRTOS.
- **Prohibition of Dynamic Heap Allocation:** Mission-critical and safety-regulated firmware standards (e.g., MISRA C) strictly deprecate dynamic heap management (`malloc`, `free`) to prevent runtime non-determinism, memory exhaustion, and catastrophic heap fragmentation.
- **Ultra-Low-Power and Deterministic Latency Constraints:** Edge sensing devices often run on energy-harvesting or battery-backed profiles requiring aggressive sleep modes. Defensive monitoring routines must be computationally lightweight and execute in bounded, deterministic time (O(depth)) without introducing operational jitter or depleting thermal and power budgets.

Consequently, standard intrusion detection agents cannot run on bare-metal or RTOS-level microcontrollers. When adversaries target industrial field networks, edge nodes experience resource starvation, firmware lockups, or remote hijacking, jeopardizing human safety and operational continuity [1], [2].

---

## 1.2 Digital Transformation, Economic Rationale & Regulatory Compliance

Deploying edge-native cybersecurity is an economic imperative and a mandatory requirement of the European digital transformation roadmap.

### 1.2.1 Regulatory Compliance: European CRA and NIS 2 Directives
Regulatory frameworks are enforcing mandatory security-by-design standards across the European single market:
- **EU Cyber Resilience Act (CRA):** Mandates that all hardware and software products with digital connectivity implement security by design and by default across their entire lifecycle. Manufacturers are legally obligated to prevent unauthorized access, mitigate vulnerabilities, and maintain auditable vulnerability management processes [3].
- **Directive (EU) 2022/2555 (NIS 2):** Enforces rigorous baseline cyber hygiene, supply-chain verification, and mandatory incident notification protocols for operators of essential and important entities. Non-compliance exposes organizations to administrative penalties of up to 10 million EUR or 2% of global annual turnover [4].

Deploying field microcontrollers without embedded defense mechanisms introduces immediate legal, operational, and commercial exposure for equipment vendors and industrial facility operators.

### 1.2.2 Quantifiable Economic Impact & Downtime Mitigation
In modern automated manufacturing, unplanned industrial downtime costs between $10,000 and $250,000 per hour depending on plant throughput. A single compromised field sensor emitting malformed traffic on an industrial fieldbus (such as Modbus/TCP) can cascade into an emergency shutdown of a programmable logic controller (PLC) [2]. Furthermore, industrial insurers increasingly require documented technical security controls before issuing cyber insurance policies. Incorporating deterministic packet inspection at the MCU level lowers underwriting risk profiles, leading directly to reduced insurance premiums.

---

## 1.3 State of the Art & Positioning Analysis

To establish the contribution of MicroShield, its architectural boundary is contrasted against existing hardware and software security solutions:

| Architectural Tier | Reference Paradigm | Operational Target | Scope & Inherent Limitations | MicroShield Positioning |
| :--- | :--- | :--- | :--- | :--- |
| **Silicon / Hardware** | **OpenTitan** (LowRISC / Google) | Dedicated ASIC / FPGA (Hardware Root of Trust) | Focuses on cryptographic device identity, hardware attestation, and secure boot. It does not monitor, parse, or filter operational network traffic. | Complementary. MicroShield operates as a lightweight software inspection engine independent of dedicated silicon RoT hardware. |
| **Operating System** | **Exein Core** | Embedded Linux (Kernel / eBPF) | Intercepts system calls and network sockets via kernel eBPF probes. Requires an MMU, multi-core gigahertz processors, and megabytes of RAM. | Differentiating. MicroShield addresses the operational space below Linux, targeting bare-metal and RTOS platforms where eBPF cannot execute. |
| **Network Gateway** | **Snort / Suricata / Zeek** | Enterprise Servers / Edge Gateways [2] | Deep packet inspection over complex protocols. Requires gigabytes of system memory, complex runtime engines, and high computational power [1], [2]. | Differentiating. MicroShield brings line-rate, bounded packet validation directly inside the endpoint microcontroller boundary before payload ingestion. |

MicroShield bridges the operational gap between hardware-level Root-of-Trust primitives and high-level Linux/gateway security software, providing an autonomous, deterministic intrusion detection system inside ultra-constrained field nodes.

---

## 1.4 Product Categorization & System Architecture

MicroShield is architected as a **heterogeneous multi-platform software system** composed of two complementary subsystems:

1. **The Edge Runtime Tier (MicroShield C-Engine):**
   - **Form Factor:** Static, portable C99 software library optimized for ultra-low-power embedded targets.
   - **Target Environment:** Embedded microcontrollers (validated on the STMicroelectronics STM32 Nucleo-F407RE board featuring an ARM Cortex-M4 core running at 168 MHz).
   - **Operational Mode:** Fast Path inline packet inspection for communication peripherals (UART, SPI, Ethernet).
   - **Algorithmic Engine:** Transpiled Decision Tree classifier. The classification graph is compiled into nested conditional statements and static lookup tables, guaranteeing a bounded execution time of O(depth) and zero dynamic heap allocation.
   - **Explainability (XAI):** Rather than returning an opaque classification score, the engine outputs the exact rule identifier and feature threshold that triggered the anomaly, providing immediate explainability for every detection event.

2. **The Supervisory Management Tier (MicroShield Fleet Orchestrator):**
   - **Form Factor:** Python 3 orchestration suite and interactive web dashboard.
   - **Target Environment:** Development workstations, Docker containers, or industrial edge compute servers.
   - **Operational Mode:** Slow Path fleet management, telemetry ingestion, Active Learning, and automated retraining.
   - **Tooling Stack:** Managed and packaged declaratively via `Poetry`. Employs `scikit-learn` for supervised classification, `NumPy` for mathematical processing, and `Dash` / `Plotly` for interactive web visualization.

### Conceptual Architecture & System Data Flow

The operational interaction between the deterministic edge engine and the supervisory management tier is depicted in the following architectural model:

[![MicroShield High-Level Conceptual Architecture](../../pictures/conceptual_architecture.png)](../../pictures/conceptual_architecture.png)

The edge engine inspects inbound frames on the fast path. If a frame is classified as benign, it is passed immediately to the core firmware application without latency penalties. When an anomalous or ambiguous packet is intercepted, the edge engine isolates the threat, enforces rate-limiting policies, and dispatches a compact telemetry record over a dedicated serial channel to the Python fleet orchestrator. The supervisory tier processes telemetry, detects concept drift, incorporates human-in-the-loop validation, and drives the automated retraining pipeline.

---

## 1.5 Target Personas & Use Case Collection

To ensure the architectural decisions address real-world industrial and operational needs, three target personas have been formalized:

### 1.5.1 Persona 1: Zanni Giorgioni (OT Security Engineer)
- **Profile:** Operational Technology (OT) cybersecurity analyst supervising telemetry and network health across an industrial manufacturing complex.
- **Key Goals:**
  - Real-time auditability and forensic traceability of all security events affecting field equipment.
  - Complete elimination of black-box models; requires Explainable AI (XAI) to verify the exact packet attributes that triggered an alarm.
  - Rapid identification of distributed attack patterns (e.g., Modbus scanning or synchronized denial-of-service floods) [2].
- **Interaction with MicroShield:**
  - Connects to the centralized Python web dashboard via desktop browser.
  - Inspects live node telemetry, anomalous packet signatures, and alert distributions.
  - Confirms zero-day attack classifications and approves automated retraining triggers.

### 1.5.2 Persona 2: Taddeo Pallabà (Embedded Firmware Engineer)
- **Profile:** Senior low-level C developer responsible for programming sensor nodes, motor actuators, and gateway boards running on ARM Cortex-M hardware.
- **Key Goals:**
  - Zero computational overhead that could disrupt core sensing routines or violate strict real-time deadlines.
  - Absolute elimination of dynamic heap allocation (`no malloc`) to ensure total protection against memory fragmentation and runtime crashes.
  - Modular, drop-in C library API that integrates cleanly into existing firmware codebases without external dependencies.
- **Interaction with MicroShield:**
  - Links the static MicroShield C library into the target application during firmware compilation.
  - Connects physical peripheral receive buffers (UART/SPI/Ethernet) to the MicroShield ingress handler.
  - Consumes generated C header files emitted by the Python transpiler to update decision rules.

### 1.5.3 Persona 3: Lentina Gigi (Plant & Operations Manager)
- **Profile:** Executive manufacturing plant director responsible for overall equipment effectiveness (OEE), operational continuity, compliance, and plant safety.
- **Key Goals:**
  - Maximum production uptime; elimination of false-positive alarms causing unwarranted machinery shutdowns.
  - Full regulatory compliance with EU standards (Cyber Resilience Act, NIS 2).
  - Containment of recurring enterprise cybersecurity risk insurance premiums through auditable technical controls.
- **Interaction with MicroShield:**
  - Reviews high-level dashboard metrics (uptime ratios, threat mitigation rates, node availability).
  - Exports compliance audit logs generated by MicroShield for formal regulatory inspections.

---

## 1.6 Architectural Scope & Lifecycle Boundaries

To ensure robust and focused delivery within the allocated academic scope (100–110 engineering hours), explicit project boundaries are established:

- **In Scope:**
  - Extraction and normalization of deterministic statistical network features from industrial benchmark corpora (Bot-IoT [1] and Edge-IIoTset [2]).
  - Supervised training, hyperparameter optimization, and pruning of Decision Tree classifiers in Python using `scikit-learn`.
  - Automatic transpilation of trained tree models into pure, deterministic C99 source headers with static lookup tables.
  - Integration of the C engine into an ARM Cortex-M microcontroller (STM32 Nucleo-F407RE) with cycle-accurate timing and memory profiling.
  - Bidirectional telemetry and quarantine management via virtualized serial (UART) channels.
  - Full DevOps lifecycle automation: Conventional Commits, Semantic Versioning, GitHub Actions CI/CD matrix builds, and Docker containerized test environments.
  - Interactive multi-tab web dashboard implemented via Python Dash/Plotly fulfilling telecommunications examination criteria.

- **Out of Scope:**
  - On-chip model retraining: Training routines (gradient descent, recursive tree splitting) require substantial floating-point matrix operations and extensive memory footprints that are incompatible with microcontrollers. Retraining remains strictly confined to the Python supervisory host.
  - Cryptographic payload decryption: MicroShield evaluates unencrypted transport and application headers (e.g., Modbus/TCP, MQTT, raw telemetry frames) or statistical features of payload bytes (entropy, length, byte variance); it does not perform hardware-accelerated TLS termination on the MCU [2].

---

## 1.7 References

- **[1]** N. Koroniotis, N. Moustafa, E. Sitnikova, and B. Turnbull, "Towards the Development of Realistic Botnet Dataset in the Internet of Things for Network Forensic Analytics: Bot-IoT Dataset," *Future Generation Computer Systems*, vol. 100, pp. 779–796, 2019.
- **[2]** M. A. Ferrag, O. Friha, D. Hamouda, L. Maglaras, and H. Janicke, "Edge-IIoTset: A New Comprehensive Realistic Cyber Security Dataset of IoT and IIoT Applications for Centralized and Federated Learning," *IEEE Access*, vol. 10, pp. 40281–40306, 2022.
- **[3]** European Commission, "Proposal for a Regulation on horizontal cybersecurity requirements for products with digital elements (Cyber Resilience Act)," COM(2022) 454 final, Brussels, 2022.
- **[4]** European Parliament and Council of the European Union, "Directive (EU) 2022/2555 on measures for a high common level of cybersecurity across the Union (NIS 2 Directive)," *Official Journal of the European Union*, L 333, pp. 80–152, 2022.

