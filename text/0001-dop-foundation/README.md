# RFC: Dora Operating Platform (dop) Foundation & Command Matrix

- **RFC Name:** `dop_foundation_and_command_matrix`
- **Start Date:** 2026-04-15
- **Author(s):** Leon
- **Status:** Proposed
- **Translations:** [简体中文](./README.zh.md)
- **RFC PR:** [dorafun-io/rfcs#0001](https://github.com/dorafun-io/rfcs/pull/2)
- **Tracking Issue:** [dorafun-io/dop#1](https://github.com/dorafun-io/dop/issues/1)

---

## 1. Summary

> *(Provide a concise, high-level abstract of the proposal.)*

This proposal establishes the architectural foundation and global command matrix for the **Dora Operating Platform (`dop`)**. As the core engine connecting the LoopFun open-source community and DoraOne's commercial delivery, `dop` aims to provide an "Apple-style" developer experience (DevEx). By enforcing a strict sandboxing environment (`DOP_HOME`) and a standardized set of 26 core commands, `dop` eliminates engineering friction caused by environment fragmentation, cross-language orchestration, and edge deployment complexities in Embodied AI development.

## 2. Motivation

> *(Why are we proposing this change?)*

- **Pain Points:** Current robotics toolchains (e.g., ROS 2) are often heavyweight and lack modern package management. Standard tools like `cargo` or `uv` cannot natively solve the scheduling of cross-language (Rust/Python/C++) zero-copy shared memory buses.
- **Expected ROI:** Achieve zero-pollution isolation between the development environment and the host system. Centralize the full lifecycle into a single entry point. Establish standard access contracts for the DoraFun platform.
- **Cost of Inaction:** Community developers will continue to struggle with fragmented scripts, preventing the formation of a unified technical standard.

## 3. Detailed Design

### 3.1 Core Philosophy

1. **Convention over Configuration**: Reuse `dataflow.yaml`, `Cargo.toml`, and `pyproject.toml` as native contracts.
2. **Fail-Fast & Graceful Degradation**: The system must possess built-in disaster recovery for edge physical scenarios.

### 3.2 Sandboxing Engine

All underlying I/O must follow these routing logics for absolute isolation:

- The engine prioritizes the `$DOP_HOME` environment variable, defaulting to `~/.dop/` if unset.
- It maintains four core internal regions: `bin/`, `cache/`, `models/` (with symlink support), and `logs/`.

### 3.3 Command Matrix (The 26 Commands)

This RFC validates the legality of 6 arrays containing 26 core command signatures:

- **Scaffold**: `init`, `new`
- **Cloud**: `login`, `logout`, `config`, `search`, `info`
- **Lifecycle**: `start`, `run`, `build`, `test`, `stop`
- **Ecosystem**: `add`, `remove`, `update`, `publish`
- **Embodied**: `record`, `replay`, `model`, `deploy`
- **Ops**: `status`, `logs`, `doctor`, `shell`, `clean`, `self-update`

## 4. Trade-offs & Technical Debt (Drawbacks)

- **Cognitive Load**: Introducing 26 commands may be overwhelming. Mitigation: Converge high-frequency usage into `new`, `run`, and `add`.
- **Disk Usage**: Absolute isolation may lead to significant disk usage. Mitigation: Provide `dop clean --all` and use symlinks for model weights.

## 5. Prior Art & Alternatives

- **Alternative A (Docker-based)**: Rejected due to high overhead in mounting physical hardware interfaces (CAN bus, USB cameras) on macOS/Windows.
- **Alternative B (Wrapper Scripts)**: Rejected as it fails to achieve true cross-language unified scheduling.

## 6. Unresolved Questions

- [ ] Precise implementation of `dop record` with MCAP format and TF Tree temporal alignment.
- [ ] Cross-compilation specifications for `dop deploy` on heterogeneous hardware (e.g., Huawei Ascend CANN).
- [ ] AST parsing anti-shake strategy for `dop publish` synchronization.
