# EVMMv2: Elastic Vertical Memory Management v2

EVMMv2 is a vertical memory management framework for Kubernetes that dynamically adjusts container memory resources without requiring container restarts. 
It is designed to improve memory utilization efficiency and reduce OOM events in oversubscribed cluster environments.

This repository provides a **reference implementation of EVMMv2 corresponding to a research paper currently under review**.

---

## Overview

EVMMv2 operates as a control component that monitors container memory behavior and applies memory scaling decisions at runtime. 
Unlike conventional approaches, EVMMv2 focuses on **automated vertical memory management** while maintaining application continuity.

The design targets containerized environments where memory oversubscription is common and application restarts are undesirable.

---

## Environment

The implementation has been evaluated under the following environment:

- Kubernetes with cgroup v2 enabled
- CRI-O container runtime
- Linux kernel with swap support
- One container per pod configuration

> **Note:** Docker is not required.

---

## Documentation

- **System Setup**  
  Detailed instructions for preparing the experimental environment are provided in  
  [`docs/SETUP.md`](docs/SETUP.md).

- **Usage**  
  An overview of how EVMMv2 operates and how it is used in experiments is described in  
  [`docs/USAGE.md`](docs/USAGE.md).

---

## Patent Notice

The underlying techniques of EVMMv2 are subject to a **pending patent application**. 
This repository provides an open-source reference implementation for research and evaluation purposes only. 
The availability of the source code **does not grant any patent rights**.

---

## License

This project is licensed under the **MIT License**. 
See the `LICENSE` file for details.
