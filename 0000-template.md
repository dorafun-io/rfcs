# RFC: [Insert RFC Title Here, e.g., dop-record-mcap-format]

- **RFC Name:** `[Short semantic identifier, e.g., dop_record_mcap]`
- **Start Date:** `YYYY-MM-DD`
- **Author(s):** `[Your Name / GitHub Handle]`
- **RFC PR:** [dorafun-io/rfcs#0000](https://github.com/dorafun-io/rfcs/pull/0000) *(Fill in after opening the PR)*
- **Tracking Issue:** [dorafun-io/dop#0000](https://github.com/dorafun-io/dop/issues/0000) *(Fill in after RFC is merged)*

---

## 1. Summary
*(Provide a concise, high-level abstract of the proposal. If you were pitching this architectural change to a new contributor in the LoopFun ecosystem, how would you explain it in a single paragraph?)*

## 2. Motivation
*(Why are we proposing this change?)*
- **Pain Points:** What specific bottlenecks, architectural limitations, or friction points exist in the current ecosystem (`dop` CLI, DoraFun Platform, or edge deployments)?
- **Expected ROI (Return on Investment):** How does this enhance Developer Ergonomics (DevEx), runtime performance, or facilitate commercial deployment for DoraOne?
- **Cost of Inaction:** What technical debt or ecosystem fragmentation will accumulate if we maintain the status quo?

## 3. Detailed Design
*(This is the technical core of the RFC. Write it with the rigor expected in a Principal Engineer's Code Review.)*
- **Architectural Topology:** Which subsystems are modified? Does this introduce new foundational dependencies (e.g., external Rust crates, system-level libraries)?
- **CLI & UX Signatures:** For modifications to `dop`, explicitly define the command signatures, flags, environment variable injections (e.g., `$DOP_HOME`), and expected terminal output semantics.
- **Data Flow & IPC:** Describe changes to the DORA zero-copy shared memory bus, network payload schemas, or serialization standards (e.g., Arrow/MCAP structures).
- **Fault Tolerance & Edge Cases:** Detail the defensive programming mechanisms. How does the system degrade gracefully under extreme physical constraints (e.g., sudden power loss, OOM, I/O saturation, network partitions)?

## 4. Trade-offs & Technical Debt (Drawbacks)
*(Every architectural decision carries a cost. Be brutally honest about the negative externalities.)*
- **Performance Overhead:** Does this introduce latency in the data plane, CPU cycle waste, or increase the memory footprint (RSS)?
- **Cognitive Load:** Does this steepen the learning curve for beginners entering the LoopFun open-source community?
- **Backward Compatibility & Portability:** Does this violate existing `dataflow.yaml` contracts? Will it compile seamlessly across heterogeneous hardware targets (e.g., x86_64, ARM, Huawei Ascend NPU)?
- **Justification:** Why are these trade-offs acceptable for the broader ecosystem?

## 5. Prior Art & Alternatives
*(What other paradigms or solutions were evaluated before arriving at this proposal?)*
- **Alternative A:** Description of the approach and the rationale for rejection (e.g., unacceptable I/O bottleneck, ecosystem lock-in).
- **Alternative B:** Description and rejection rationale.
- Why does the proposed design represent the optimal equilibrium for the `dorafun-io` infrastructure?

## 6. Unresolved Questions
*(What implementation details or edge cases remain ambiguous and require community consensus during the Final Comment Period (FCP)?)*
- [ ] Open Issue 1...
- [ ] Open Issue 2...