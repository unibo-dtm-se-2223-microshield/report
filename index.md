---
layout: page
title: Home
nav_order: 1
description: "MicroShield Project Overview"
---

# MicroShield

### A Deterministic Ultra-Low-Power Embedded C Intrusion Detection System

---

## Abstract

The relentless expansion of the Internet of Things (IoT) and Industrial Internet of Things (IIoT) into critical cyber-physical domains has dramatically widened the threat surface of connected edge systems. Resource-constrained microcontrollers (MCUs) deployed in supervisory control, telecommunications, and field sensing environments are frequently exposed to networked attacks such as volumetric Denial of Service (DoS/DDoS), scanning probes, protocol spoofing, and malicious payload injections. Traditional host-based intrusion prevention systems and runtime monitors are strictly unviable for ultra-constrained platforms due to severe memory limitations, lack of Memory Management Units (MMUs), non-deterministic execution times, and dynamic memory allocation risks (`malloc`).

**MicroShield** resolves this structural vulnerability by introducing a hardware-agnostic intrusion detection library specifically engineered for bare-metal and RTOS-driven embedded targets (such as ARM Cortex-M microcontrollers). The core edge engine consists of an ultra-low-power, zero-dependency, C99-compliant packet inspection framework operating with strictly deterministic worst-case execution time bounds (`O(depth)`) and a fully static memory footprint, eliminating heap fragmentation risks. Network anomaly detection is governed by compact, transpiled Decision Trees that evaluate physical protocol headers and payload metrics at line rate, providing intrinsic Explainable Artificial Intelligence (XAI) for every detection event.

To validate and orchestrate the embedded library across its operational lifecycle, MicroShield is complemented by an auxiliary Python-based MLOps pipeline and monitoring suite. This companion layer ingests high-resolution industrial network telemetry (derived from benchmark corpora including Bot-IoT and Edge-IIoTset), handles offline model training and transpilation, coordinates automated cross-platform verification via CI/CD containerization pipelines, and exposes an interactive monitoring dashboard. By enforcing proactive cyber resilience at the most vulnerable layers of connected field devices, MicroShield directly complies with emerging regulatory frameworks including the EU Cyber Resilience Act (CRA) and Directive NIS 2, delivering measurable enterprise risk reduction.

---

## Academic Framework

This project is developed as the final examination for the following academic courses:

* **Software Engineering** (Prof. Giovanni Ciatto)  
  *Department of Computer Science and Engineering (DISI)*  
  *Alma Mater Studiorum – Università di Bologna (Cesena Campus)*
* **Software per le Telecomunicazioni** (Prof. Alessandro Guidotti)  
  *Department of Electrical, Electronic and Information Engineering “Guglielmo Marconi” (DEI)*  
  *Alma Mater Studiorum – Università di Bologna (Bologna Campus)*

---

## Disclaimer

During the preparation of this work, the author utilized automated language model tools as interactive thought partners, code formatters, and drafting aids for exploratory technical synthesis. Following generation, the author rigorously verified, independently adapted, and empirically validated all architectural designs, algorithms, source code implementations, and analytical conclusions, assuming sole and full responsibility for the contents of the final report and software artifacts.
