---
title: Home
layout: home
has_children: false
nav_order: 1
---

# MicroShield
### Deterministic Ultra-Low-Power Embedded C Intrusion Detection System & Python MLOps Fleet Orchestrator

---

## Abstract

The rapid convergence of the Internet of Things (IoT) and Industrial Internet of Things (IIoT) into safety-critical domains has expanded the attack surface of connected edge devices. Microcontroller units (MCUs) deployed in industrial automation, telecommunications, and robotics are routinely subjected to network-level threats, including volumetric Denial of Service (DoS/DDoS), scanning probes, protocol spoofing, and malicious payload injections. Conventional intrusion detection system (IDS) paradigms engineered for general-purpose operating systems depend on virtual memory, dynamic memory allocation (`malloc`), and variable-latency pattern matching—architectural mechanisms that are fundamentally incompatible with bare-metal microcontrollers lacking a Memory Management Unit (MMU) and operating under strict microsecond control loops.

**MicroShield** addresses this structural vulnerability by introducing a dual-tier, hardware-agnostic intrusion detection and prevention architecture designed specifically for bare-metal and RTOS-driven embedded targets (such as the ARM Cortex-M family). The runtime edge tier consists of an ultra-low-power, zero-dependency, C99-compliant packet inspection engine. By evaluating incoming data-link frames through statically transpiled Decision Trees, MicroShield delivers strictly bounded, deterministic execution times ($O(\text{depth}) \le 50\ \mu\text{s}$) with zero heap allocation. Every classification event generates intrinsic Explainable Artificial Intelligence (XAI) metadata, providing an explicit Rule ID and split threshold without the computational burden of post-hoc explanation models.

The edge runtime is supported by a centralized Python-based MLOps supervisory tier. This fleet management service ingests out-of-band serial telemetry, continuously monitors statistical concept drift across sliding temporal windows, triggers automated decision tree retraining upon distribution shifts, and transpiles updated estimators directly into portable C99 header files. An interactive operations dashboard provides real-time fleet health, confusion matrices, and tamper-evident incident logging aligned with the mandates of the EU Cyber Resilience Act (CRA Article 10) and the NIS 2 Directive.

---

## Academic Framework

This project is developed as the comprehensive software engineering report and experimental artifact for the degree requirements at:

* **Alma Mater Studiorum – Università di Bologna (Cesena Campus)**  
  *Department of Computer Science and Engineering (DISI)*  
  *Degree Course in Computer Science and Engineering*

