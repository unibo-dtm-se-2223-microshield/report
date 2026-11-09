---
title: Design
has_children: false
nav_order: 4
---

# Chapter 3: Architectural, Detailed & Infrastructure Design

---

## 3.1 Architectural Style, Nominal- vs. Effective-Bound Translation & Component Topology

Designing a cyber-physical Intrusion Detection System (IDS) that spans bare-metal/RTOS microcontrollers and enterprise Security Operations Center (SOC) analytics requires resolving an inherent tension between **market value propositions** during an industrial digital transformation process and **hard real-time embedded systems engineering**.

### 3.1.1 The Dichotomy Between Nominal Market Claims and Effective Worst-Case Engineering Bounds

When industrial organizations undergo a digital transformation of legacy Operational Technology (OT) infrastructures, decision-makers demand security solutions that introduce **nominally "Zero"** operational friction: *Zero Latency* (no delay on physical actuation loops), *Zero Memory* (deployable on existing microcontrollers without hardware expansion), and *Ultra-Low-Power* (negligible energy drain). While these nominal terms serve as powerful market value propositions to emphasize non-intrusiveness, physical computing systems cannot operate in zero time, zero space, or zero energy.

In rigorous hard real-time engineering, this dichotomy between **nominal** and **effective (real)** metrics is resolved by defining "Zero" as a **strict, mathematically and empirically guaranteed Worst-Case Bound**—the ceiling that the software guarantees never to exceed under any operational or adversarial load. Table 3.1 formalizes the mapping between the nominal market value proposition and the effective worst-case engineering bounds of `libmicroshield` (whose empirical verification under stress is detailed in **Appendix A**).

| Nominal Value Proposition (Market / Executive Emphasis) | Strategic Meaning in Digital Transformation | Effective Engineering Reality (Hard Real-Time Worst-Case Bound) | Guaranteed Worst-Case Ceiling & Verification Protocol |
| :--- | :--- | :--- | :--- |
| **"Zero Latency"** *(Sub-Frame Deterministic Execution)* | Inline packet inspection never delays safety-critical control tasks or causes RTOS scheduler starvation. | **Bounded Worst-Case Execution Time (`WCET`) & Near-Zero Jitter:** Feature extraction executes in `O(1)` and tree traversal executes in `O(tree_depth)` (`D <= 8`). Effectively, worst-case execution requires a tiny fraction of a single byte's transmission window on a standard `115.2 kbps` fieldbus (`~86.8 us`), with execution jitter bounded below `20 CPU clock cycles`. | **`WCET <= 1,500 clock cycles`** per frame (`<= 8.9 us` on **STM32F407RE** at `168 MHz`; `<= 18.7 us` at `80 MHz`), with **Jitter `< 20 cycles`**, empirically validated both at idle and under heavy CPU contention (**see Appendix A**). |
| **"Zero Memory"** *(Zero Dynamic Heap & Micro-Static Ceiling)* | The library requires zero RAM upgrades on legacy MCUs and is immune to memory leaks over multi-year uptimes. | **Literal `0 Bytes` Dynamic Heap (`malloc`-free) & Bounded Static Footprint:** While static memory cannot be zero, dynamic heap allocation is literally `0 Bytes` (`0 calls` to `malloc`/`free`), eliminating heap fragmentation. All state resides in fixed compile-time `.bss` structures and naturally aligned `12-byte` feature vectors. | **Dynamic Heap: `0 Bytes`** (verified via ELF symbol table scan). **Static RAM Ceiling (`.bss`/`.data`): `<= 128 Bytes`** per interface context (`ms_context_t`). **Worst-Case Stack Ceiling: `<= 96 Bytes`**. **Total Flash Ceiling (`.text` + `.rodata`): `<= 4.0 KB`**. |
| **"Ultra-Low-Power"** *(Sub-Microjoule Inspection & Interrupt-Gated Sleep)* | Security inspection does not exhaust power budgets or induce thermal throttling in sealed field enclosures. | **Worst-Case Energy-per-Frame Ceiling & `WFI` Sleep Maximization:** Active energy per frame is bounded by `E_max = V_DD * I_run * WCET`. By minimizing active cycle counts and disabling receive interrupts (`RXNE IE = 0`) during Level-3 flood lockdowns, the MCU maximizes residency in low-power *Wait-For-Interrupt* (`WFI`) Sleep Mode. | **Worst-Case Active Energy `< 1.0 uJ` per frame** on STM32F407RE (`3.3V * 35mA * 8.5us = 0.98 uJ`). Preserves **`>= 99.1%` MCU Sleep Mode residency** (`< 2.5 mA` vs. `35 mA` active) under nominal loads, and **`100%` sleep residency** during flood lockdowns. |

*Table 3.1: Resolution of the dichotomy between nominal market value propositions and effective hard real-time worst-case engineering bounds.*

---

### 3.1.2 Comparative Evaluation of Architectural Styles

No single monolithic architectural style can simultaneously satisfy microsecond worst-case bounds on a microcontroller and interactive, data-intensive MLOps governance on a fleet supervision server. Table 3.2 evaluates canonical software architectural patterns against MicroShield's dual-tier requirements (`FR-01..12`, `NFR-01..08`).

| Architectural Style | Core Mechanism | Suitability for Tier 1 (`libmicroshield` C99 Edge) | Suitability for Tier 2 (**`Dashield`** Python Host) | MicroShield Adoption Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **Pure Layered Architecture** | Strict hierarchical call stack (Presentation -> Business -> Persistence -> Hardware). | **Partially Suitable:** Clean separation between HAL, Feature Extractor, and Tree Evaluator, but deep call stacks increase function-call overhead if not inlined. | **Insufficient Alone:** Traditional layering tightly couples UI rendering to database schemas, hindering pluggable transport adapters (Physical UART vs. Docker TCP Socket). | **Adopted with Flat Inlining on Tier 1; Replaced by Hexagonal on Tier 2.** |
| **Object-Based / OOP Architecture** | Encapsulation of state and behavior into polymorphic classes with dynamic dispatch (`vtables`). | **Rejected on Edge:** Dynamic polymorphism (`vtables` / function pointer indirection) defeats compiler branch prediction and violates strict WCET analyzability on MCUs. | **Ideal on Host:** Python 3 OOP provides clean encapsulation for MLOps pipelines, Shadow Auditing ensembles, and stateful multi-tab view controllers. | **Strictly Rejected on Tier 1 (C99); Adopted as Primary Paradigm on Tier 2 (Python).** |
| **Event-Driven / Reactive Pipeline** | Asynchronous or interrupt-triggered processing of discrete data frames through deterministic stages. | **Ideal on Edge:** Matches physical packet arrival interrupts (ISRs) and deterministic Finite-State Machine (FSM) transitions (`O(1)` -> `O(depth)` -> `FSM`). | **Ideal for Telemetry Ingestion:** Decouples incoming 24-byte UART frames from downstream consumers (Alert Store, Shadow Auditor, Drift Monitor) via an Event Bus. | **Adopted across Both Tiers (Synchronous Pipeline on Edge; Observer Event Bus on Host).** |
| **Shared Dataspace (Blackboard)** | Multiple independent workers read/write concurrently to a central shared memory repository. | **Rejected on Edge:** Concurrent shared-memory locking introduces mutex contention, priority inversion, and unbounded blocking times in RTOS tasks. | **Partially Suitable:** Useful only at the persistence boundary (`SQLite WAL` shared between collector daemon and web console). | **Adopted Narrowly for Host Persistence (`SQLite WAL` Repository).** |
| **Hexagonal (Ports & Adapters)** | Domain logic sits at the center, isolated from external I/O (serial links, databases, UI) via explicit Ports and pluggable Adapters. | **Over-Engineered for C99:** Excessive abstraction layers bloat Flash footprint, though the C99 API naturally acts as a pure, hardware-agnostic port. | **Ideal on Host:** Allows **`Dashield`** to switch seamlessly between a real **STM32F407RE** board (`/dev/ttyACM0`) and $N$ virtualized Docker C99 nodes (`TCP Socket`) without changing a single line of domain code. | **Adopted as Master Architecture for Tier 2 (**`Dashield`**).** |

*Table 3.2: Analytical evaluation and justification of architectural styles across the Edge and Host tiers.*

---

### 3.1.3 Master Architectural Topology: Asymmetric Dual-Tier Hybrid Design

Based on the trade-off analysis in Table 3.2, MicroShield adopts an **Asymmetric Dual-Tier Hybrid Architecture**, illustrated in Figure 3.1:

1. **Tier 1 — Core Embedded C99 Library (`libmicroshield`):** Engineered as an **Event-Driven, Zero-Allocation Synchronous Pipeline**. Every incoming frame triggers a deterministic four-stage cascade:
   * `ms_extract_features()`: Constant-time `O(1)` update of circular sliding registers and Shannon entropy calculation (evaluated empirically against hardware floating-point in **Appendix A**).
   * `ms_evaluate_tree()`: Bounded `O(tree_depth)` traversal of a static array of decision nodes (`ms_tree_node_t[]`), outputting both the threat class and the glass-box `matched_rule_id`, while flagging boundary-adjacent benign frames (`is_boundary_sample = 1`).
   * `ms_update_fsm()`: A 5-state **Leaky Bucket + Binary Exponential Backoff Finite-State Machine** that escalates protection from single-frame drops (`SUSPICIOUS`) to temporary throttling (`THROTTLED_L1`), exponential quarantine (`QUARANTINED_L2`), and full hardware receive-interrupt lockdown (`LOCKDOWN_L3`).
   * `ms_pack_telemetry_frame()`: A constant-time binary codec that serializes anomaly alerts (`100%`) and stratified benign shadow samples (`1:K` + boundary frames) into compact `24-byte` frames protected by a `CRC-16-CCITT` checksum.
2. **Tier 2 — `Dashield` Fleet Supervision & MLOps Suite:** Engineered as a **Unified Pure-Python Hexagonal (Ports & Adapters) & Observer OOP Architecture**. Unifying the entire host stack—from low-level serial ingestion and Scikit-Learn MLOps pipelines to the stateful multi-tab presentation layer—within a single object-oriented language eliminates cross-language Inter-Process Communication (IPC) serialization overhead and guarantees type-model consistency across the domain boundary. At its core sits the `DashieldOrchestratorFacade`, which coordinates the `TelemetryEventBus`, the `ShadowAuditEnsembleService` (a 100-tree Random Forest that hunts silent False Negatives), the `ConceptDriftAndGaugeService`, and the `C99TreeTranspilerFactory`.

[![MicroShield Master Component and Hexagonal Architecture](../../puml/des-01-architecture-components.svg)](../../puml/des-01-architecture-components.svg)
*(Click on the diagram to open the full-resolution vector graphic in a new tab)*  
*Figure 3.1: Master UML Component Diagram illustrating the Event-Driven C99 pipeline on Tier 1 (`libmicroshield`) and the Hexagonal Ports-and-Adapters OOP architecture of **`Dashield`** on Tier 2.*

---

### 3.1.4 Intrinsic Glass-Box Interpretability, Hybrid Drift Theory & Top-Down UX Design

Two foundational theoretical principles govern the human-computer interaction design of **`Dashield`**:

1. **Intrinsic (Ante-Hoc) Glass-Box Interpretability vs. Post-Hoc XAI (Rudin, 2019) [1]:**
   In contemporary machine learning literature, a sharp epistemological distinction exists between *Post-Hoc Explainable AI (XAI)*—where secondary approximation algorithms such as SHAP or LIME attempt to reconstruct why an opaque black-box neural network made a decision—and *Intrinsically Interpretable Glass-Box Models* (such as CART Decision Trees), which are self-explanatory by mathematical construction. As demonstrated by Rudin (2019) [1], post-hoc approximations are computationally expensive and frequently unfaithful to the underlying model mechanics. MicroShield deliberately rejects black-box classifiers and post-hoc XAI overhead: because `libmicroshield` evaluates a shallow boolean Decision Tree directly over physical telecom metrics, every classification is **natively and deterministically transparent (*ante-hoc interpretable*)**, exposing the exact conjunction of physical threshold splits traversed (`matched_rule_id`).
2. **Hybrid Concept Drift Detection & The Retraining Readiness Index (Gama et al., 2004; Siddiqi, 2006) [2, 3]:**
   To determine objectively when the static tree deployed across the microcontroller fleet has become obsolete and requires retraining, **`Dashield`** implements a **Hybrid Drift Detection Policy** that synthesizes two complementary paradigms from statistical learning theory:
   * *Supervised Error-Rate Monitoring (Gama et al., 2004 [2]):* Tracks confirmed classification failures—specifically operator-flagged False Positives (`N_FP`) and Shadow-Audited False Negatives (`N_FN`).
   * *Unsupervised Covariate Shift Monitoring (Siddiqi, 2006 [3]):* Tracks shifts in the underlying input feature distribution `P(X)` via the **Population Stability Index (`PSI`)**, where literature establishes `PSI >= 0.10` as moderate shift and `PSI >= 0.20..0.25` as the critical action threshold requiring model reconstruction.
   The `ConceptDriftAndGaugeService` combines these three normalized error and shift ratios into a composite heuristic indicator—the **Retraining Readiness Index (`R_index`)** bounded in `[0%, 100%]`:
   ```
   R_index = min(100%,  w_FP * (N_FP / Tau_FP) + w_FN * (N_FN / Tau_FN) + w_PSI * (PSI_max / Tau_PSI))
   ```
   where `Tau_FP = 25`, `Tau_FN = 10` (assigning higher sensitivity to silent evasions unmasked by Shadow Auditing), and `Tau_PSI = 0.20` (Siddiqi's critical shift boundary), with default convex weights `w_FP = 0.25`, `w_FN = 0.45`, `w_PSI = 0.30` (such that any single metric exceeding its critical cap or a combined degradation drives `R_index -> 100%`, triggering a `RED: RETRAIN REQUIRED` state).

[![Dashield 5-Tab UX Architecture and Retraining Gauge](../../puml/des-03-dashield-ux-tabs.svg)](../../puml/des-03-dashield-ux-tabs.svg)
*(Click on the diagram to open the full-resolution vector graphic in a new tab)*  
*Figure 3.2: Top-down Human-Computer Interaction (HCI) topology of the **`Dashield`** 5-Tab console, mapping stakeholder personas (Taddeo, Lentina, Zanni) to the Hybrid Drift Readiness Gauge and functional views.*

Table 3.3 details the responsibilities and interactive controls of each OOP View Tab within **`Dashield`**.

| Dashield View Component | OOP Controller Class | Target Persona | Real-Time Visualizations & Telemetry Widgets | Interactive Control Actions & System Side-Effects |
| :--- | :--- | :--- | :--- | :--- |
| **Global Header** *(Hybrid Retraining Readiness Gauge)* | `ReadinessGaugeHeaderView` | Lentina Gigi, Zanni Giorgioni | Progress bar (`0%..100%`), color-coded health badge (`GREEN`/`AMBER`/`RED`), live counters of supervised errors (`Verified FPs`, `Unmasked FNs`) and unsupervised shift (`Peak PSI`). | Clicking **`[Jump to Retraining Pipeline]`** navigates directly to Tab 4 with pre-loaded feedback buffers when `R_index >= 80%`. |
| **Tab 1: Telecom Link & Edge FSM Monitor** | `TelecomAndFsmTabView` | Taddeo Pallabà, Zanni Giorgioni | Live time-series of `iat_delta_us`, `pkt_rate_win`, and `payload_entropy`; UART bandwidth utilization meter (`<= 2.0%` of `115.2 kbps`); 5-state FSM badge per node; **Worst-Case Cycle Profiler (Ref. Appendix A)**. | **`[Manual Quarantine Reset]`**: dispatches an 8-byte `CMD_RESET_FSM` command frame over UART/Socket to restore a locked-down node (`LOCKDOWN_L3 -> NORMAL`). |
| **Tab 2: Intrinsic Glass-Box Rule Inspector** | `GlassBoxRuleInspectorTabView` | Lentina Gigi | Filterable incident table showing timestamp, node ID, verdict, and `matched_rule_id`; interactive **Ante-Hoc Boolean Path Visualizer** highlighting the exact C99 threshold splits traversed. | **`[Flag as False Positive (FP)]`**: marks alert as `VERIFIED_FP`, appends vector with label `BENIGN` to retraining store, increments `Readiness Gauge`, and unlocks node FSM. |
| **Tab 3: Asymmetric Shadow Audit & FN Hunter** | `ShadowAuditFnTabView` | Lentina Gigi | Live stream of stratified benign samples (`1:K` + boundary frames where `|val - thr| <= epsilon`); side-by-side comparison between Edge C99 Tree verdict (`BENIGN`) and Host 100-Tree Random Forest verdict. | Automatic highlighting of **`UNMASKED_FN`** discrepancies; **`[Confirm FN & Queue for Retrain]`** feeds evasive vectors into the training corpus and advances the `Readiness Gauge`. |
| **Tab 4: Concept Drift & Live C99 Transpiler** | `DriftAndTranspilerTabView` | Lentina Gigi, Taddeo Pallabà | Per-feature **PSI** and **Kolmogorov-Smirnov (K-S)** drift bar charts against baseline **Bot-IoT / Edge-IIoTset** distributions; confusion matrix, Macro F1-Score, and live syntax-highlighted view of the generated `ms_tree_node_t[]` C99 array. | **`[Retrain & Transpile C99 Tree]`**: runs CART training (`D <= 8`), validates F1 `>= 0.92`, generates `ms_tree_model.h`, and enables **`[Broadcast New Tree to Fleet]`** via UART/Socket. |
| **Tab 5: Red-Team Injector & CRA/NIS 2 Dossier** | `RedTeamAndComplianceTabView` | Zanni Giorgioni, Taddeo Pallabà | Scenario injection status monitor; live EU CRA static memory budget table (`.text`, `.rodata`, `.bss`, stack); NIS 2 incident timeline preview. | **`[Inject Traffic Scenario]`** (Benign, DoS Flood, Scan Probe, High-Entropy Exploit, Stealth Evasion); **`[Toggle Synthetic CPU Stress]`**; **`[Export NIS 2 / CRA Dossier]`**. |

*Table 3.3: Detailed specification of the **`Dashield`** object-oriented multi-tab console and interactive governance controls.*

---

## 3.2 Distributed Infrastructure, Container Topology & Service Discovery

Because MicroShield operates across distributed field nodes and a central supervision server—and must guarantee deterministic reproducibility across both physical testbeds and virtualized clusters—its infrastructure is designed with **Dual-Mode Deployment Symmetry** (Physical Hardware-in-the-Loop vs. Containerized Multi-Node Cluster).

[![MicroShield UML Deployment and Container Topology](../../puml/des-02-deployment-topology.svg)](../../puml/des-02-deployment-topology.svg)
*(Click on the diagram to open the full-resolution vector graphic in a new tab)*  
*Figure 3.3: UML Deployment Diagram illustrating the dual-mode infrastructure: physical **STM32F407RE** Hardware-in-the-Loop via UART (`/dev/ttyACM0`) alongside the containerized $N$-node Docker Compose topology (`172.28.0.0/16`).*

### 3.2.1 Infrastructural Components & Multiplicity

To fulfill the distributed systems design criteria, Table 3.4 inventories every infrastructural component, its multiplicity (`1` vs. `N`), its runtime execution environment, and its exact architectural role.

| Infrastructural Component | Multiplicity | Execution Technology | Network Location & Isolation | Core Infrastructural Responsibility |
| :--- | :--- | :--- | :--- | :--- |
| **Physical Edge MCU Node (`stm32-edge-hw`)** | `0 .. M` *(Optional HIL)* | **STM32F407RE** (ARM Cortex-M4F @ `168 MHz`, `192 KB` SRAM, `512 KB` Flash) running bare-metal/RTOS C99 firmware. | Field OT cabinet; connected to Host via physical USB-UART bridge (`/dev/ttyACM0` at `115,200 bps`, `8N1`). | Executes `libmicroshield.a` inline on physical frames, runs the hardware `DWT->CYCCNT` cycle profiler (**see Appendix A**), and emits 24-byte `CRC-16` telemetry frames. |
| **Virtualized Edge C99 Daemon Containers (`edge-node-01 .. 0N`)** | `N` *(Default `N = 3` in Docker Compose)* | Docker Container (`debian-slim` + `gcc` C99 runtime) executing compiled binary `ms_edge_node_daemon`. | Isolated Docker Bridge Subnet (`172.28.0.0/16`); no external internet exposure. | Executes the **exact same ANSI C99 `libmicroshield` source code** compiled natively, runs a background POSIX thread generating synthetic CPU contention, processes injected network traffic, and streams 24-byte binary frames over virtual serial sockets. |
| **Async Telemetry Collector & Bridge Worker (`dashield-collector`)** | `1` *(Singleton Daemon)* | Python `3.11+` background worker managed by **Poetry** inside the `dashield-orchestrator` container. | Bound to `0.0.0.0:9999` (TCP Socket bridge for Docker nodes) and `/dev/ttyACM0` (for physical STM32). | Multiplexes incoming 24-byte binary streams from all `N + M` edge nodes, verifies `CRC-16-CCITT` checksums, triggers the `ShadowAuditEnsembleService` on benign samples, and writes validated records to persistent storage. |
| **Transactional Audit & Telemetry Database (`dashield_audit.db`)** | `1` *(Embedded Shared Store)* | **SQLite 3** configured in **Write-Ahead Logging (`WAL`)** mode + curated Parquet/CSV benchmark partitions. | Mounted on persistent Docker volume `/data/dashield_storage` inside `dashield-orchestrator`. | Provides ACID-compliant, zero-configuration concurrent persistence (`1` concurrent writer from `dashield-collector`, multiple concurrent readers from `dashield-web`) for incidents, shadow discrepancies, and `FP`/`FN` feedback vectors. |
| **Supervision Web Server & MLOps Engine (`dashield-web`)** | `1` *(Singleton Server)* | **Streamlit** pure-Python OOP application managed via **Poetry** (`poetry run streamlit run`). | Exposed on host loopback/LAN port `http://localhost:8501` via Docker port forwarding. | Serves the 5-Tab interactive console to human operators, computes live PSI/K-S drift metrics, executes Scikit-Learn CART retraining, transpiles C99 node arrays, and dispatches control commands back to edge nodes. |

*Table 3.4: Inventory, multiplicity, and network distribution of infrastructural components across the MicroShield ecosystem.*

---

### 3.2.2 Network Distribution, Naming Conventions & Deterministic Service Discovery

In a distributed cyber-physical system, components must discover and authenticate communication endpoints deterministically without relying on fragile manual IP configuration:

1. **Containerized Cluster Naming & DNS Service Discovery:**
   Within the `docker-compose.yml` orchestration manifest, all containers attach to a dedicated user-defined bridge network named `microshield_ot_net` (`172.28.0.0/16`). Docker's embedded DNS server (`127.0.0.11`) provides automatic name resolution:
   * The central supervision container is registered under the canonical hostname **`dashield-orchestrator`**.
   * Each edge C99 container is provisioned with a deterministic node identifier (`MS_NODE_ID = 1..N`) and hostname (`edge-node-01`, `edge-node-02`, `edge-node-03`) via environment variables. Upon startup, `ms_edge_node_daemon` resolves `dashield-orchestrator:9999` via standard POSIX `getaddrinfo()` and establishes a persistent full-duplex byte-stream connection that emulates a physical UART wire (`PySerial` `socket://` protocol handler).
2. **Physical Hardware-in-the-Loop (`STM32F407RE`) Auto-Detection:**
   When the physical **STM32F407RE** board is connected via USB ST-Link Virtual COM Port, the Linux udev subsystem maps it to `/dev/ttyACM0` (or `/dev/ttyUSB0`). The `TelemetryTransportPort` in **`Dashield`** implements an automatic discovery probe: if `/dev/ttyACM0` is present and responds with the MicroShield `0xAA55` sync handshake, it attaches a `PhysicalUartTransport` worker alongside the virtualized Docker nodes, allowing physical and simulated nodes to be monitored side-by-side on Tab 1.

---

## 3.3 Wire-Speed Telemetry Protocol, 24-Byte Binary Framing & Cross-Tier Handshake

In distributed cyber-physical systems engineering, defining the cross-tier boundary contract is a fundamental software design obligation. Rather than adopting heavyweight, serialized data interchange formats (such as JSON or Protobuf) that would mandate dynamic heap allocations and non-deterministic parsing routines on bare-metal microcontrollers, MicroShield defines a bespoke, **naturally aligned 24-byte binary telemetry protocol**.

This protocol serves as the formal boundary contract between the low-level ANSI C99 edge inspection engine (`libmicroshield`) and the auxiliary Python OOP supervision suite (`Dashield`).

[![MicroShield 24-Byte Binary Frame Layout](../../puml/des-04-binary-frame-layout.svg)](../../puml/des-04-binary-frame-layout.svg)  
*(Click on the diagram to open the full-resolution vector graphic in a new tab)*  
*Figure 3.4: Memory-aligned binary layout of the MicroShield 24-byte telemetry frame, illustrating 32-bit boundary alignment, fixed-width fields, and CRC-16-CCITT integrity block.*

### 3.3.1 Binary Frame Architecture & Memory Alignment Invariants

The binary frame is structured to adhere to strict hardware execution constraints and object serialization contracts:
1. **Natural 32-Bit Word Alignment:**  
   The total frame length is exactly $24\text{ Bytes}$ ($192\text{ bits}$), which is an exact multiple of 4 and 8 bytes. All internal 32-bit fields (`seq_ts`, `feat_iat`) and 16-bit fields (`sync_word`, `rule_id`, `feat_len`, `aux_meta`, `crc16`) reside on naturally aligned byte boundaries. This layout permits single-cycle, unaligned-fault-free load/store memory operations (`LDR`/`STR`) on ARM Cortex-M4 architectures and standard memory packing across POSIX C compilers (`#pragma pack(push, 1)`).
2. **Zero-Allocation Stack Serialization:**  
   The frame is serialized and deserialized directly within fixed-size stack buffers (`uint8_t buffer[24]`), completely eliminating dynamic heap allocations (`malloc`/`free`) and preventing memory fragmentation.
3. **Payload Structure:**
   * **`sync_word` (`0x00..0x01` - uint16):** Two-byte magic sequence (`0xAA55`) used for frame demarcation, boundary recovery, and deserializer synchronization.
   * **`msg_type` (`0x02` - uint8):** Distinguishes operational message types (`0x01 = ANOMALY_ALERT`, `0x02 = SHADOW_SAMPLE`, `0x10 = CMD_RESET_FSM`).
   * **`node_id` (`0x03` - uint8):** Logical source/target edge node identifier (`0x01 .. 0xFF`).
   * **`seq_ts` (`0x04..0x07` - uint32):** Monotonically increasing sequence number or hardware millisecond timestamp for ordering and replay-attack detection.
   * **`verdict` (`0x08` - uint8):** Classification decision (`0 = BENIGN`, `1 = DOS_FLOOD`, `2 = SCAN_PROBE`, `3 = INJECTION`).
   * **`fsm_state` (`0x09` - uint8):** Current mitigation FSM operational state (`NORMAL`, `SUSPICIOUS`, `THROTTLED_L1`, `QUARANTINED_L2`, `LOCKDOWN_L3`).
   * **`rule_id` (`0x0A..0x0B` - uint16):** Ante-hoc Explainable AI (XAI) identifier specifying the exact decision tree leaf path traversed.
   * **`feat_iat`, `feat_len`, `feat_ent`, `feat_rate` (`0x0C..0x13`):** Compact representation of extracted frame telemetry for out-of-band auditability and drift calculation.
   * **`aux_meta` (`0x14..0x15` - uint16):** Sub-threshold decision distance ($\epsilon$) or command argument.
   * **`crc16` (`0x16..0x17` - uint16):** ITU-T CRC-16-CCITT checksum (polynomial `0x1021`, initial value `0xFFFF`) computed over bytes `0x00` through `0x15`.

---

### 3.3.2 Physical Transport Agnosticism & Plane Decoupling

A critical architectural property of MicroShield is the strict decoupling between the **Operational Data Plane** and the **Supervisory Telemetry/Control Plane**:

* **Operational Field Data Plane:**  
  The industrial communication bus (e.g., Modbus RTU over RS-485, CAN bus, or Ethernet) transports primary operational payload packets directly between physical plant sensors, refrigeration controllers, and actuators. `libmicroshield` inspects these frames inline with strictly zero-copy memory overhead.
* **Out-of-Band Supervisory Plane:**  
  The 24-byte binary telemetry frames travel over a dedicated secondary channel. While physical bench testing implements this channel over a USB-CDC Virtual COM Port (UART at $115,200\text{ bps}$, 8N1), the transport interface in both C99 and Python is engineered to be **hardware- and medium-agnostic**. The byte stream can be mapped transparently across:
  * Cellular modems (4G LTE-M / NB-IoT / 5G) via AT command tunnels.
  * Long-range industrial radio links (LoRaWAN, Zigbee, WirelessHART).
  * Standard TCP/IP sockets for containerized virtual testbeds (`socket://dashield-orchestrator:9999`).

[![Cross-Tier Handshake and Streaming Sequence](../../puml/des-05-uart-handshake-sequence.svg)](../../puml/des-05-uart-handshake-sequence.svg)  
*(Click on the diagram to open the full-resolution vector graphic in a new tab)*  
*Figure 3.5: UML Sequence Diagram documenting cross-tier channel synchronization, asymmetric dual-path streaming (Alerts vs. Shadow Audit samples), and downlink operator quarantine override.*

---

### 3.3.3 Communication Protocol Dynamics & Handshake Sequence

The protocol lifecycle, documented in Figure 3.5, enforces a robust state machine across both endpoints:
1. **Channel Discovery & Sync Handshake:**  
   The Python supervision daemon (`dashield-collector`) probes the transport endpoint. The edge node acknowledges synchronization by emitting its 24-byte descriptor frame with `sync_word = 0xAA55`. The Python ingestion service verifies the CRC-16 checksum; if valid, the node is registered into the active fleet state as `ONLINE`.
2. **Asymmetric Ingress Streaming:**  
   * **Incident Alerts (`msg_type = 0x01`):** Every detected anomaly triggers instantaneous serialization of a 24-byte alert frame. Ingestion workers parse the frame, log the incident into the transactional SQLite WAL store, and notify the web UI via asynchronous event callbacks.
   * **Stratified Shadow Auditing (`msg_type = 0x02`):** Benign frames are sampled at a configurable ratio (1:K, with priority given to boundary-adjacent vectors where $|feature - split| < \epsilon$). These samples are routed to the Python `ShadowAuditEnsembleService` to evaluate potential False Negatives against a high-capacity reference model without saturating the channel.
3. **Downlink Override Command Dispatch:**  
   When an operator overrides a False Positive via the web console, `dashield-collector` dispatches an authenticated 24-byte command frame (`msg_type = 0x10`, `CMD_RESET_FSM`). The edge firmware validates the frame checksum, executes the state transition from `LOCKDOWN_L3` or `QUARANTINED_L2` back to `NORMAL`, re-enables the hardware receiver interrupt, and transmits a confirmation acknowledgment.

---

## 3.4 Embedded C99 Edge Architecture & Micro-Engine Design (`libmicroshield`)

The primary engineering artifact deployed on resource-constrained field nodes is `libmicroshield.a`, a freestanding, zero-dependency ANSI C99 static library. Engineered strictly under MISRA-C safety guidelines and the **Zero-Malloc Invariant** (`IR-01`, `IR-02`, `NFR-02`), the library provides deterministic real-time intrusion detection and autonomous local mitigation without utilizing dynamic heap allocation (`0 Bytes` `malloc`/`free`).

[![Deterministic C99 Inspection Pipeline](../../puml/des-07-c99-pipeline-flow.svg)](../../puml/des-07-c99-pipeline-flow.svg)  
*(Click on the diagram to open the full-resolution vector graphic in a new tab)*  
*Figure 3.6: Activity flow of the ANSI C99 edge engine, illustrating zero-copy frame ingestion, O(1) Q8 fixed-point feature extraction via Flash LUT, deterministic decision tree traversal, and FSM mitigation dispatch.*

### 3.4.1 Zero-Copy Ingestion & Static State Memory Layout

Traditional network intrusion detection systems buffer multi-packet streams into dynamic heap queues, incurring memory fragmentation and non-deterministic allocation latency. On an ARM Cortex-M4F microcontroller operating bare-metal or under an RTOS, memory must be allocated statically at compile time.

`libmicroshield` implements a zero-copy frame ingestion model:
1. **Pass-by-Reference Interface:**  
   The primary entry point `ms_ingest_frame(ms_context_t *ctx, const uint8_t *payload, uint16_t len, uint32_t ts_us)` receives a raw pointer to the network frame in the hardware receive buffer. The payload is inspected in-place without duplicating bytes into intermediate SRAM buffers.
2. **Context Memory Footprint (`<= 128 Bytes`):**  
   Per-interface state is strictly encapsulated within a statically allocated structure `ms_context_t`. It contains only fixed-width sliding accumulators:
   * Previous arrival timestamp (`uint32_t prev_ts_us`) for inter-arrival delta tracking.
   * Rolling window packet counter (`uint8_t win_pkt_count`) and jitter accumulator (`uint16_t win_jitter_accum`).
   * Connection cardinality bitmask (`uint32_t active_conn_mask`).
   * Leaky Bucket penalty level and exponential backoff cooldown timestamps.
   The total static SRAM footprint per protected interface is bounded to **96 Bytes**, well within the 128-byte ceiling mandated by `NFR-02`.

---

### 3.4.2 Constant-Time Feature Extraction & Flash LUT Fixed-Point Entropy

To guarantee hard real-time execution bounds (`WCET <= 1,500 CPU clock cycles`, `NFR-01`), feature extraction must complete in strictly constant time ($O(1)$) independently of packet contents:

1. **Eight-Dimensional Telemetry Vector:**  
   For every frame, the engine evaluates eight physical and statistical dimensions packed into a 12-byte naturally aligned layout:
   * `feat_iat_delta_us` (32-bit integer): Inter-arrival time difference ($\Delta t = t_i - t_{i-1}$) in microseconds.
   * `feat_payload_len` (16-bit integer): Frame byte length.
   * `feat_proto_id` (8-bit integer): Transport protocol identifier (e.g., Modbus/TCP, CAN).
   * `feat_transport_flags` (8-bit integer): Bitwise flags (SYN, ACK, RST, URG).
   * `feat_pkt_rate` (8-bit integer): Sliding-window packet frequency.
   * `feat_jitter` (8-bit integer): Arrival variance relative to expected industrial polling intervals.
   * `feat_conn_fan_in` (8-bit integer): Source address cardinality.
   * `feat_payload_entropy_q8` (8-bit unsigned fixed-point integer): Normalized Shannon byte entropy.

2. **Flash Look-Up Table (LUT) for Shannon Entropy in Q8 Arithmetic:**  
   Standard Shannon entropy requires computing transcendental logarithms:
   $$H(X) = -\sum_{i=1}^{n} p(x_i) \log_2 p(x_i)$$
   Calling floating-point `log2()` or math libraries on microcontrollers introduces non-deterministic execution jitter and code bloat. `libmicroshield` replaces runtime logarithmic calculations with a precomputed 256-entry Look-Up Table stored in Flash (`const uint8_t ms_entropy_lut[256]`):
   * Each entry stores the pre-scaled fixed-point product $p \cdot \log_2(p)$ in $Q8$ format ($0$ to $255$ representing $[0.0, 1.0]$).
   * Byte frequencies are sampled across the ingress payload in a single linear pass ($O(\text{payload\_len})$).
   * Total entropy is accumulated using 16-bit integer addition and right-shifted back to $Q8$, executing in $< 350$ clock cycles with zero floating-point operations.


### 3.4.3 Transpiled Compact Decision Tree Runtime & Deterministic Inference

Once the 8-feature vector is assembled, it is evaluated by the inference engine against an immutable node array stored directly in Flash memory (const ms_tree_node_t ms_model_nodes[]).

#### Epistemological & Engineering Justification for the Decision Tree
In edge-tier cyber-physical systems engineering, deploying opaque black-box models (such as deep artificial neural networks or recurrent architectures) introduces severe operational hazards:
1. **Mathematical Determinism (O(tree_depth)):** A Decision Tree of maximum depth D guarantees that any classification completes in at most D conditional integer comparisons. With D <= 8, traversal requires at most 8 branch evaluations, ensuring mathematically bounded cycle consumption regardless of adversarial input.
2. **Intrinsic Ante-Hoc Explainability (Rudin, 2019):** Every decision maps to an exact boolean rule path composed of physical threshold splits. When an edge node quarantines an interface, operators can inspect the exact clause triggered (matched_rule_id), eliminating post-hoc approximation uncertainty.
3. **Hardware Efficiency on Cortex-M4F:** Unlike neural networks requiring thousands of multiply-accumulate (MAC) operations, weight quantization scratchpads, and matrix tensor arenas, a transpiled tree executes entirely within standard CPU registers without floating-point emulation overhead.

#### Node Array Layout & Branch Traversal
The node data structure is packed into a compact 8-byte layout containing the 8-bit feature selector (feature_index), the leaf flag (is_leaf), the 16-bit glass-box rule identifier (rule_id), the 32-bit fixed-point comparison split threshold (threshold), and the 16-bit indices of the left and right child nodes (left_child, right_child).

Traversal begins at root index 0. At each internal node, the evaluated feature is compared against threshold: if strictly less, execution jumps to left_child; otherwise, it proceeds to right_child. Upon encountering is_leaf == 1, traversal terminates, returning the classification verdict and the unique rule_id.

---

### 3.4.4 Multi-Stage Containment FSM (Leaky Bucket & Exponential Backoff)

To prevent brittle binary tripwires—such as the operational paralysis seen in the November 2026 civil aviation alert-storm incidents—mitigation logic is decoupled from raw instantaneous verdicts. The engine executes a 5-state Finite-State Machine combining a Leaky Bucket filter with Binary Exponential Backoff.

[![Multi-Stage Mitigation FSM](../../puml/des-06-edge-mitigation-fsm.svg)](../../puml/des-06-edge-mitigation-fsm.svg)  
*(Click on the diagram to open the full-resolution vector graphic in a new tab)*  
*Figure 3.7: Deterministic 5-state edge mitigation FSM, illustrating proportional escalation from transient anomaly drops to hardware interrupt isolation.*

1. **NORMAL (Operational Pass-Through):** Ingress frames pass unrestricted (ALLOW). Stratified benign telemetry samples (1:K) are queued for out-of-band shadow auditing.
2. **SUSPICIOUS (Transient Anomaly Filtering):** Isolated anomalies drop the offending packet (DROP) and increment the Leaky Bucket penalty counter by +1. If no further violations occur before the leak interval (T_leak) elapses, the penalty decrements back to zero, returning the interface to NORMAL.
3. **THROTTLED_L1 (Burst Rate-Limiting):** If consecutive violations exceed threshold L1, the interface enters micro-throttling (MICRO_BLOCK), applying a 50% duty cycle on incoming frames for T_hold = 200 ms.
4. **QUARANTINED_L2 (Subsystem Isolation):** Persistent anomalies exceeding threshold L2 trigger full algorithmic isolation (EXP_BLOCK), dropping all ingress frames for T_hold = 2,000 ms.
5. **LOCKDOWN_L3 (Hardware Receiver Perimeter Severance):** Sustained volumetric floods or lateral scanning sweeps (the BBT SE scenario) escalate to hardware lockdown (HW_LOCKDOWN). The engine explicitly disables the peripheral receive interrupt (e.g., clearing the RXNEIE bit in the STM32 USART Control Register), physically severing the compromised interface from the processor bus until an authenticated downlink command (CMD_RESET_FSM) is received from the supervision console.

