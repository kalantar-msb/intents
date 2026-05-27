# BLIS Project Intents

## Summary

~130 intents organized into 12 categories:

1. **Foundational** (6) — why BLIS exists
2. **Invariant** (18) — unconditional system guarantees
3. **Fidelity** (9) — behavioral accuracy vs real systems
4. **Feature** (40+) — capabilities grouped by subsystem
5. **Architecture** (13) — structural properties
6. **Extensibility** (10) — how the system should grow
7. **Optimization** (8) — "better is better" goals without hard SLAs
8. **Safety & Correctness** (12) — defensive properties preventing bug classes
9. **Process** (18) — development methodology, quality gates, experiment rigor, code hygiene
10. **Platform Parity** (6) — which external systems to model faithfully
11. **Validation** (6) — how correctness is established
12. **Usability** (7) — user experience

Sources: the 13 invariants + PD invariants, 23 antipattern rules, engineering principles, design guidelines (DES foundations, module architecture, extension framework), experiment standards, PR workflow/convergence protocol, recent issues (#1364 tune, #1348 EP/DP, #1336 disaggregation signals, #1321 mid-decode preemption), and the code's feature surface (commands, modules, interfaces, scoring framework).

The taxonomy is organized for navigation, not purity — overlaps between categories are acknowledged at the bottom.

---

An **intent** is a statement about what this system should do, be, or guarantee. Intents are not implementation details — they describe desired properties at varying levels of abstraction. They evolve: new intents emerge, existing ones sharpen, some become obsolete.

This document attempts an exhaustive enumeration, organized by kind.

---

## 1. FOUNDATIONAL INTENTS (raison d'être)

These define what BLIS *is* and why it exists.

| ID | Intent |
|----|--------|
| F-1 | BLIS is a discrete-event simulator for LLM inference serving systems — it predicts system behavior without requiring real GPUs. |
| F-2 | BLIS enables capacity planning: given a model, hardware, and workload, determine sustainable throughput and latency characteristics. |
| F-3 | BLIS enables policy optimization research: compare routing, admission, scheduling, and batch formation strategies under controlled conditions. |
| F-4 | BLIS enables performance prediction across model/GPU/TP configurations — cross-model generalization without per-model calibration. |
| F-5 | BLIS models an extensible distributed inference platform — not any single system. llm-d, vLLM, SGLang, Mooncake, and LMCache are all target systems whose behaviors should be expressible through module composition. |
| F-6 | BLIS is CPU-only and deterministic — it runs anywhere without special hardware and produces reproducible results. |

---

## 2. INVARIANT INTENTS (must hold in all circumstances)

These are unconditional guarantees the system must maintain regardless of configuration, workload, or policy selection.

### 2.1 Conservation & Lifecycle

| ID | Intent |
|----|--------|
| INV-1 | Every injected request is accounted for at simulation end: completed + queued + running + dropped + timed_out + routing_rejected + gateway_queued + gateway_shed + gateway_rejected + gateway_evicted + gateway_expired + encode_routing_rejected = injected. No silent loss. |
| INV-2 | Requests follow a strict lifecycle: queued → running → completed. No request may be in a state it hasn't transitioned into. |
| INV-4 | KV cache blocks are conserved: `allocated + free = total` at all times. No blocks are created or destroyed during simulation. |
| INV-11 | Every session reaches exactly one terminal state (completed, cancelled, horizon-interrupted, budget-exhausted). No session is silently abandoned. |
| INV-12 | After batch formation Phase 1, every non-preempted running request in decode phase has `NumNewTokens > 0`. No request silently skipped. |

### 2.2 Temporal & Causal

| ID | Intent |
|----|--------|
| INV-3 | The simulation clock never decreases. |
| INV-5 | Causality holds: `arrival ≤ enqueue ≤ schedule ≤ completion` for every request. |
| INV-8 | The simulator is work-conserving: if requests are waiting and the instance is not at capacity, work must proceed. No idling with pending work. |
| INV-10 | For closed-loop sessions: `round[N+1].arrival ≥ round[N].completion + think_time`. Follow-ups never arrive before their predecessor completes. |

### 2.3 Determinism & Reproducibility

| ID | Intent |
|----|--------|
| INV-6 | Same seed produces byte-identical stdout across runs. Non-deterministic information (wall-clock timing, diagnostics) goes to stderr only. |
| INV-13 | A trace exported by `blis run` and replayed by `blis replay` with identical flags produces identical per-request metrics. The pipeline is lossless and round-trip-safe. |

### 2.4 Information Boundary

| ID | Intent |
|----|--------|
| INV-9 | The control plane (admission, routing, scheduling, priority) must not access `Request.OutputTokens` — only the execution engine may know actual output length. Control decisions use `MaxOutputLen` (client budget) or input-only signals. No oracle knowledge leakage. |
| INV-7 | Routing signals have explicit freshness tiers: some are synchronous (always current), others are periodic (bounded staleness). The system must model and respect this distinction. |

### 2.5 Disaggregation-Specific

| ID | Intent |
|----|--------|
| INV-PD-1 | A decode request cannot begin until its KV transfer is complete. |
| INV-PD-2 | Prefill sub-requests route only to the prefill pool; decode only to the decode pool. Pool boundaries are never violated. |
| INV-PD-3 | KV transfers are conserved: `initiated = completed` at simulation end. |
| INV-PD-4 | Disaggregated phase causality: arrival ≤ prefill_enqueue ≤ prefill_complete ≤ transfer_start ≤ transfer_complete ≤ decode_enqueue ≤ completion. |
| INV-PD-5 | Pool membership is fixed at construction and never changes during a simulation run. |

---

## 3. FIDELITY INTENTS (behavioral accuracy)

These describe how accurately BLIS should model real systems.

| ID | Intent |
|----|--------|
| FID-1 | Latency predictions should be within calibration-grade accuracy of real servers for the modeled hardware/model/TP configurations. |
| FID-2 | The simulator should reproduce the qualitative behavior of vLLM's continuous batching scheduler (request interleaving, chunked prefill, preemption under memory pressure). |
| FID-3 | The routing layer should reproduce llm-d's weighted scorer framework with the same signal semantics and normalization. |
| FID-4 | KV cache behavior should match vLLM's block manager: prefix caching, eviction under pressure, tiered offloading. |
| FID-5 | Flow control (gateway queue, saturation detection, dispatch) should match GIE/llm-d's admission control semantics. |
| FID-6 | Signal staleness in routing snapshots should reflect real-world timing: some signals are synchronous (router-local), others periodic (reported from instances with bounded delay). |
| FID-7 | Workload distributions (arrival processes, token lengths, session structures) should match empirically observed patterns from production inference workloads. |
| FID-8 | MoE architecture modeling should account for expert parallelism, sparse activation, and per-layer overhead patterns. |
| FID-9 | The observe/replay/calibrate pipeline should enable quantitative validation against real servers — not just qualitative similarity. |

---

## 4. FEATURE INTENTS (capabilities the system should have)

### 4.1 Simulation Modes

| ID | Intent |
|----|--------|
| FEAT-1 | `blis run` — drive a simulation from a workload specification, producing full metrics and optional trace export. |
| FEAT-2 | `blis replay` — replay a recorded trace through the DES engine, reproducing timing from the trace or computing new timing. |
| FEAT-3 | `blis observe` — dispatch workloads to real servers, recording actual latencies into TraceV2 format for calibration. |
| FEAT-4 | `blis calibrate` — compare simulated vs observed latencies with statistical rigor (MAPE, Pearson R, quality grades). |
| FEAT-5 | `blis convert` — interoperate with external workload formats (ServeGen, inference-perf, presets). |
| FEAT-6 | `blis tune` — automatically find the maximum sustainable request rate without persistent saturation (binary search with saturation detection). |

### 4.2 Cluster Architecture

| ID | Intent |
|----|--------|
| FEAT-10 | Model multi-instance clusters with shared-clock orchestration and per-instance isolation. |
| FEAT-11 | Support prefill/decode disaggregation with dedicated instance pools and KV transfer modeling. |
| FEAT-12 | Support heterogeneous hardware within a cluster (different GPUs for prefill vs decode pools). |
| FEAT-13 | Support dynamic scaling (autoscaler policies that add/remove instances based on load signals). |
| FEAT-14 | Support flow control with gateway queue, saturation-gated dispatch, and configurable dispatch ordering (FIFO, priority, SLO-deadline, EDF). |

### 4.3 Request Pipeline

| ID | Intent |
|----|--------|
| FEAT-20 | Pluggable admission control policies (always-admit, token-bucket, tier-shed, saturation-based). |
| FEAT-21 | Pluggable routing policies — both simple (round-robin, least-loaded) and composable weighted scoring. |
| FEAT-22 | Composable scorer framework: combine N scoring dimensions with configurable weights for weighted routing. |
| FEAT-23 | Pluggable per-instance scheduling (FCFS, priority-FCFS, SJF). |
| FEAT-24 | Batch formation with configurable limits (max running requests, max scheduled tokens, chunked prefill). |
| FEAT-25 | Priority preemption: evict lower-priority requests under memory pressure to admit higher-priority work. |
| FEAT-26 | Request TTL: time-out queued requests that exceed their deadline. |
| FEAT-27 | SLO-aware processing with multi-tier priority (critical, standard, batch, sheddable, background). |

### 4.4 Latency Modeling

| ID | Intent |
|----|--------|
| FEAT-30 | Trained-physics latency model: roofline basis functions + learned correction coefficients for cross-model generalization. |
| FEAT-31 | Pure roofline model: analytical FLOPs/bandwidth estimation when learned coefficients are unavailable. |
| FEAT-32 | Automatic model configuration from HuggingFace (architecture detection, parameter count, quantization). |
| FEAT-33 | MoE-aware latency modeling (expert parallelism, sparse activation patterns, interleaved vs uniform expert layers). |
| FEAT-34 | Quantized model support: auto-detect weight precision from multiple sources and adjust bandwidth calculations. |

### 4.5 KV Cache

| ID | Intent |
|----|--------|
| FEAT-40 | Block-based KV cache with LRU eviction and prefix caching (hierarchical block hashing). |
| FEAT-41 | Tiered KV cache (GPU + CPU) with configurable offload threshold and transfer bandwidth. |
| FEAT-42 | Precise prefix scoring: query actual per-instance KV cache state for routing decisions. |
| FEAT-43 | Auto-derivation of KV block capacity from model architecture and GPU memory. |

### 4.6 Workload Generation

| ID | Intent |
|----|--------|
| FEAT-50 | Support multiple arrival processes (Poisson, Gamma, Weibull, constant). |
| FEAT-51 | Support multiple token-length distributions (Gaussian, exponential, Pareto-lognormal, empirical). |
| FEAT-52 | Multi-cohort workloads with per-cohort rate, SLO class, model, and prefix group. |
| FEAT-53 | Temporal dynamics: diurnal patterns, spikes, drains for time-varying load. |
| FEAT-54 | Session modeling: multi-round conversations with closed-loop or open-loop arrival. |
| FEAT-55 | Prefix groups: shared prompt caching affinity across requests. |
| FEAT-56 | Multimodal workloads: text + image + audio + video token mixing. |

### 4.7 Observability & Metrics

| ID | Intent |
|----|--------|
| FEAT-60 | Per-request metrics: TTFT, TPOT, E2E, ITL (inter-token latency). |
| FEAT-61 | System-level metrics: throughput (tokens/sec, requests/sec), utilization, queue depths. |
| FEAT-62 | Per-SLO-class metric breakdown for multi-tenant analysis. |
| FEAT-63 | Decision tracing: record admission, routing, and scoring decisions for post-hoc analysis. |
| FEAT-64 | Counterfactual regret: compute what alternative routing decisions would have yielded. |
| FEAT-65 | Fitness evaluation: combine multiple metrics into a single optimization score. |
| FEAT-66 | Post-hoc saturation detection: classify completed runs as stable/backlogged/overloaded. |
| FEAT-67 | TraceV2 format: lossless capture of all per-request data for replay and calibration. |

---

## 5. ARCHITECTURE INTENTS (structural properties)

These describe how the system should be organized, independent of specific features.

| ID | Intent |
|----|--------|
| ARCH-1 | Two-layer architecture: domain-agnostic simulation kernel (event queue, clock, RNG, statistics) separated from domain-specific modules (router, scheduler, KV cache, latency model). |
| ARCH-2 | Module boundaries defined by behavioral contracts (observes, controls, owns, invariants, events, extension friction) — not by implementation details. |
| ARCH-3 | Single-method interfaces where possible. Interfaces must accommodate multiple implementations. |
| ARCH-4 | Unidirectional dependency flow: `cmd/ → sim/cluster/ → sim/`. No reverse dependencies. |
| ARCH-5 | `sim/` is a library — it never terminates the process, never calls `os.Exit` or `logrus.Fatalf`. Only `cmd/` may terminate. |
| ARCH-6 | Configuration grouped by module, each independently validatable. Factory functions accept the narrowest sub-config. |
| ARCH-7 | Canonical constructors: struct literals in exactly one place. No scattered construction sites. |
| ARCH-8 | Output channel separation: stdout for deterministic results, stderr for diagnostics. |
| ARCH-9 | Partitioned RNG per subsystem to isolate randomness streams and enable common-random-numbers experiments. |
| ARCH-10 | Low extension friction: adding a new policy variant should touch ≤3 files. Adding a new config parameter should touch ≤2 files. |
| ARCH-11 | Each module's state and statistics are separated. No method both mutates state and computes metrics. |
| ARCH-12 | Events are minimal and atomic — one event = one state change. Events classified as exogenous (workload-driven) or endogenous (state-driven). |
| ARCH-13 | Deterministic event ordering via `(timestamp, priority, seqID)` three-key scheme. Ties are fully resolved — no ambiguity. |

---

## 6. EXTENSIBILITY INTENTS (how the system should grow)

| ID | Intent |
|----|--------|
| EXT-1 | New routing algorithms can be added without modifying existing routing code (policy template pattern). |
| EXT-2 | New scoring dimensions can be added to weighted routing by implementing a scorer function and registering it (2 touch points). |
| EXT-3 | New admission policies can be added via the same pattern. |
| EXT-4 | New scheduling strategies can be added per-instance without cluster-level changes. |
| EXT-5 | New KV cache tiers can be composed by wrapping existing tiers. |
| EXT-6 | New latency model backends can be registered via init-based factory. |
| EXT-7 | New workload formats can be supported via converters to the canonical v2 spec. |
| EXT-8 | New trace record types can be added without modifying existing recording logic. |
| EXT-9 | New per-request metrics can be added (current friction acknowledged: ~6 files; target: ~3). |
| EXT-10 | The system can model new real-world systems (beyond vLLM/llm-d) by composing existing modules with new policy variants. |

---

## 7. OPTIMIZATION INTENTS (desired properties without hard SLAs)

These are "better is better" goals — there's no pass/fail threshold but improvement is always desirable.

| ID | Intent |
|----|--------|
| OPT-1 | Simulation speed: running a typical workload (1000 requests, single instance) should complete in seconds, not minutes. Wall-clock time should not dominate iteration cycles. |
| OPT-2 | Test suite performance: `go test ./...` should complete in under 60 seconds. No individual test should take >5 seconds without `testing.Short()` gating. |
| OPT-3 | Latency model accuracy should improve over time through better coefficients and architecture-aware modeling — without requiring per-model calibration data. |
| OPT-4 | Configuration ergonomics: sensible defaults should produce reasonable results without requiring deep understanding of all parameters. |
| OPT-5 | Cross-model generalization: a single set of trained coefficients should work across model families, sizes, and quantization levels. |
| OPT-6 | Low-friction experimentation: changing one policy or parameter should not require touching unrelated code. |
| OPT-7 | Memory efficiency: the simulator should handle large workloads (100K+ requests) without excessive memory consumption. |
| OPT-8 | Parameter-free algorithms are preferred over threshold-tuned ones. Structural/geometric properties that hold universally are better than magic numbers. |

---

## 8. SAFETY & CORRECTNESS INTENTS

These describe defensive properties that prevent classes of bugs.

| ID | Intent |
|----|--------|
| SAFE-1 | No silent data loss: every error path must return an error, panic with context, or increment a counter. No silent `continue` that drops requests. (R1) |
| SAFE-2 | No division by zero in runtime computation: any division where denominator derives from runtime state must guard against zero. (R11) |
| SAFE-3 | No livelock: retry loops where exit depends on unavailable resources must have circuit breakers (max iterations, progress assertions). (R19) |
| SAFE-4 | No non-determinism from map iteration: any map iteration feeding a running sum or output ordering must sort keys first. (R2) |
| SAFE-5 | No partial state on allocation failure: resource allocation uses check-then-act or rollback — never leaves partially-mutated state. (R5) |
| SAFE-6 | No stale construction sites: before adding a struct field, all construction sites must be found and updated. (R4) |
| SAFE-7 | Strict YAML parsing: typos in field names must cause parse errors, not silent acceptance. (R10) |
| SAFE-8 | No exported mutable maps: validation lookup maps exposed only through accessor functions. (R8) |
| SAFE-9 | No premature interface lock-in: interfaces must work for ≥2 implementations before being frozen. (R13) |
| SAFE-10 | Parallel code paths must apply identical transformations when producing the same output type. (R23) |
| SAFE-11 | Pre-check must be at least as permissive as actual operation — over-conservative pre-checks cause false rejections. (R22) |
| SAFE-12 | Degenerate inputs (empty sets, single-instance concentration, all-zero distributions) must be handled explicitly by detectors and scorers. (R20) |

---

## 9. PROCESS INTENTS (how development should happen)

### 9.1 Development Methodology

| ID | Intent |
|----|--------|
| PROC-1 | BDD/TDD: behavioral contracts (GIVEN/WHEN/THEN) are written before tests, tests before code. |
| PROC-2 | Tests verify laws (conservation, causality, monotonicity) not just values. Golden tests must have companion invariant tests. |
| PROC-3 | Tests must survive refactoring: if the implementation changes but behavior is preserved, tests should still pass. No structural testing. |
| PROC-4 | Design docs before implementation for features that introduce module boundaries or modify architecture. |
| PROC-5 | Four-tier document hierarchy: design guidelines → design docs → macro plans → micro plans. Each tier has a specific abstraction level. |
| PROC-6 | The abstraction rule: design docs describe *what*; macro plans describe *what to build in what order*; micro plans describe *how*. Go code belongs only in micro plans. |

### 9.2 Quality Gates

| ID | Intent |
|----|--------|
| PROC-10 | Convergence protocol for reviews: parallel perspectives, classify findings (CRITICAL/IMPORTANT/SUGGESTION), iterate until zero CRITICAL and zero IMPORTANT. |
| PROC-11 | Three review gates per hypothesis experiment: design review → code review → findings review. |
| PROC-12 | PR workflow: worktree → plan → convergence review → implement (TDD) → code convergence review → self-audit → commit. |
| PROC-13 | Pre-commit self-audit: 10 dimensions of critical thinking (logic bugs, determinism, consistency, construction sites, error paths, etc.). |
| PROC-14 | Agent trust boundaries: read operations trusted; code edits verify-after; self-assessments never-trusted. |

### 9.3 Experiment Rigor

| ID | Intent |
|----|--------|
| PROC-20 | Experiments use VV&UQ framing: verification (code correctness), validation (model fidelity), uncertainty quantification (confidence intervals). |
| PROC-21 | Causal claims require root-cause verification: cite file:line, first-principles calculation, control experiments, and devil's advocate. |
| PROC-22 | Experiments vary exactly one dimension; hold everything else constant. |
| PROC-23 | Findings must be classified and audited against standards to identify new rules, invariants, or principles. |
| PROC-24 | Every experiment must be fully reproducible via a single `run.sh` script — no manual steps. |

### 9.4 Code Hygiene

| ID | Intent |
|----|--------|
| PROC-30 | CI runs on all PRs: build, lint (golangci-lint), full test suite. |
| PROC-31 | Golden datasets regenerated when output format, metrics, or defaults change. |
| PROC-32 | Stale references (to completed PRs, removed features) are actively resolved. |
| PROC-33 | Documentation has a single source of truth — other locations reference the canonical source with explicit headers. |
| PROC-34 | 23 antipattern rules (R1-R23), each traced to a real bug, are checked during code review. |

---

## 10. PLATFORM PARITY INTENTS (external system alignment)

These describe which external systems BLIS should be able to faithfully model.

| ID | Intent |
|----|--------|
| PAR-1 | vLLM v1 scheduler parity: continuous batching, chunked prefill, preemption (FCFS/LIFO), priority scheduling. |
| PAR-2 | llm-d routing parity: weighted scorer framework, precise-prefix-cache scoring, queue-depth/kv-utilization signals, configurable weights. |
| PAR-3 | GIE/gateway-api-inference-extension parity: gateway queue, saturation-gated dispatch, SLO-deadline ordering, EDF ordering. |
| PAR-4 | llm-d disaggregation parity: prefill/decode pool split, KV transfer modeling, pool-aware routing. |
| PAR-5 | vLLM mid-decode preemption: interrupt running requests when KV memory exhausted, full recompute on resume. |
| PAR-6 | Expert parallelism (EP/DP) as used by vLLM for MoE models (DeepSeek-V3 etc): All-to-All communication, expert sharding strategies. |

---

## 11. VALIDATION INTENTS (how correctness is established)

| ID | Intent |
|----|--------|
| VAL-1 | The observe/replay/calibrate pipeline provides a closed loop for validating simulator accuracy against real servers. |
| VAL-2 | Calibration uses statistical metrics (MAPE, Pearson R) with quality grades — not just "close enough" eyeballing. |
| VAL-3 | Invariant tests verify conservation, causality, and determinism — they're not optional companions to golden tests. |
| VAL-4 | Hypothesis experiments use formal statistical tests (KS, Mann-Whitney U) with effect sizes and confidence intervals. |
| VAL-5 | The saturation detection framework provides automated classification of simulation runs without manual inspection. |
| VAL-6 | Cross-policy comparisons use dominance analysis or Pareto frontier identification, not single-metric ranking. |

---

## 12. USABILITY INTENTS (user experience)

| ID | Intent |
|----|--------|
| USE-1 | A user with a model name and workload description can run a useful simulation without deep knowledge of inference internals. |
| USE-2 | HuggingFace model configs are auto-fetched and cached — users don't need to manually specify architecture parameters. |
| USE-3 | Named workload presets (chatbot, summarization, contentgen, multidoc) provide one-word access to realistic workload patterns. |
| USE-4 | CLI flag defaults represent production-reasonable configurations (llm-d parity where applicable). |
| USE-5 | Error messages identify what went wrong and what to do about it — not just "invalid input." |
| USE-6 | Trace formats are interoperable: observe→replay→calibrate forms a complete pipeline without manual format conversion. |
| USE-7 | The `blis tune` command automates the most common capacity planning question: "what rate can this config sustain?" |

---

## Taxonomy Notes

The categories above are neither exhaustive nor perfectly orthogonal. Observed overlaps:

- **Invariants** and **Safety** overlap: an invariant violation IS a safety failure.
- **Fidelity** and **Platform Parity** overlap: parity with a specific system IS a fidelity claim.
- **Architecture** and **Extensibility** overlap: good architecture enables extensibility.
- **Process** and **Validation** overlap: process defines how validation is done.

The taxonomy serves navigation, not classification. An intent may belong in multiple categories — it should live where it's most likely to be looked for.

---

## Evolution

This list reflects the project as of 2026-05-22. Expected changes:

- New **platform parity** intents as BLIS tracks SGLang, Mooncake, LMCache more closely
- New **feature** intents as the `blis tune` command matures and policy search (Nous) lands
- Sharpening of **optimization** intents as performance baselines are established
- New **invariants** as disaggregation and expert parallelism introduce new conservation laws
- Possible new **categories** (e.g., "Governance" for multi-tenant isolation, "Composability" for policy combination semantics)
