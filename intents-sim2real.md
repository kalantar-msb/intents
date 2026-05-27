# Intents

Statements about what this system should do, be, or guarantee. Living document.

## Summary

73 intents across 12 categories:

| Category | Count | What it captures |
|----------|-------|------------------|
| Mission | 3 | Why the system exists |
| Invariants | 10 | Must hold unconditionally; violations are bugs |
| Architecture | 7 | Structural design properties |
| Features | 11 | Capabilities the system provides |
| Resilience | 5 | Graceful failure handling |
| Observability | 5 | How it communicates state |
| Process | 6 | Contributor workflow requirements |
| Ergonomics | 6 | Operator/developer experience |
| Safety | 5 | Security and correctness guarantees |
| Optimizations | 6 | Better-is-better properties without hard SLAs |
| Consistency | 4 | Cross-component coherence |
| Extensibility | 5 | How the system accommodates growth |

Sources: issues (violated invariants, desired features), code (architectural properties, validation rules), CLAUDE.md (process requirements), skills (capability intents).

---

## Mission

1. **Transfer simulation-discovered algorithms to production.** The system exists to bridge the gap between BLIS (discrete-event simulation) and llm-d (production inference scheduler). An algorithm that works in simulation should be deployable to production without manual reimplementation.

2. **Make transfer repeatable and auditable.** Any transfer run should be reproducible from its inputs (transfer.yaml, algorithm source, component repo state). The workspace trail documents what was done, why, and with what inputs.

3. **Automate mechanics; preserve human judgment.** Mechanical steps (build, deploy, collect, compare) should be fully automated. Decisions requiring judgment (config correctness, algorithm semantics, deploy/no-deploy) should gate on human approval.

---

## Invariants

Properties that must hold unconditionally. Violations are bugs.

4. **Workspace is always regenerable.** Every file under `workspace/` can be reproduced by re-running the stage that created it. No workspace file is hand-maintained.

5. **Fix upstream, never in workspace.** When a workspace artifact is wrong, the fix goes in the generating code (pipeline script or skill), not the artifact. A workspace-only fix will be silently lost.

6. **Single source of truth per piece of state.** Progress lives in the ConfigMap. Phase completion lives in `.state.json`. Manifest lives in `transfer.yaml`. No state has two authoritative homes.

7. **Schema-first contracts.** All inter-stage communication uses explicit, validated schemas. No implicit behavior; no untyped bags.

8. **Phases are idempotent.** Re-running a completed phase (without `--force`) is a no-op. The state machine skips it. This makes re-runs safe and cheap.

9. **Package names are globally unique and constrained.** All baseline and algorithm names must be lowercase alphanumeric, 1-20 characters, no hyphens or underscores. Collision across baseline+algorithm namespace is rejected at manifest load time.

10. **Each subcommand owns exactly one responsibility.** `wipe` = file cleanup. `reset` = status change. `run` = dispatch. `collect` = pull results. No subcommand has side effects belonging to another.

11. **Atomic state writes.** All persistent state (`.state.json`, `run_metadata.json`, ConfigMap) is written atomically (tempfile + rename, or single kubectl apply). Partial writes cannot leave corrupt state.

12. **Per-pair cost derivation.** GPU capacity gating uses the resolved scenario for each specific (workload, package) pair to determine cost. Uniform cost assumptions from a single baseline are incorrect for heterogeneous workloads.

13. **Exit code semantics are stable.** 0 = success or planned checkpoint. Non-zero = failure. Scripts and callers depend on this contract.

---

## Architecture

Structural properties of the system's design.

14. **Linear pipeline with checkpoint.** The pipeline is `setup → prepare → [translate skill] → deploy`. Stages are sequential; each depends on the prior stage's output.

15. **Bundle + overlay assembly model.** Baselines are version-controlled bundles in the experiment repo. Treatments are constructed by deep-merging baseline + diffs + skill-generated overlays. Assembly is deterministic given inputs.

16. **Three-tier list merge.** YAML list merging follows: replace (non-dict items), merge-by-key (all-dict lists with common key), positional merge (all-dict lists without key). This is the system's answer to "what does deep merge mean for arrays."

17. **Multi-slot parallel orchestration.** Workload pairs are dispatched across multiple namespace slots concurrently. The orchestrator manages a pool, not a queue.

18. **State machine recovery.** `prepare.py` is a 6-phase state machine. Interruption at any point allows resumption from the last completed phase without loss of prior work.

19. **Experiment repo is the unit of configuration.** Each experiment carries its own `transfer.yaml`, baselines, algorithms, workloads, and component submodule. The framework repo provides the pipeline; the experiment repo provides the inputs.

20. **Skill boundary at Phase 3.** The translate skill is invoked between prepare phases 3 and 4. It reads `skill_input.json` and produces `translation_output.json` + overlay configs. This is the seam where AI-generated code enters the pipeline.

---

## Features

Capabilities the system must provide.

21. **Bootstrap from BLIS output.** Given a BLIS-generated experiment folder (algorithms/, workloads/, config), the system can derive all pipeline-required artifacts (transfer.yaml, baselines/, submodule) with human approval at each gate.

22. **AI-assisted translation.** The translate skill generates production Go plugin code from simulation algorithm source, using a 3-agent team (expert, writer, reviewer) with multi-round review.

23. **Baseline and treatment comparison.** Every experiment produces paired runs: baseline (unmodified scheduler) vs treatment (with evolved algorithm). Results are collected and compared per-workload.

24. **Multi-workload benchmarking.** A single transfer run can exercise multiple workloads, each independently scheduled and collected.

25. **Image build with staleness detection.** EPP images are only rebuilt when the source hash changes. Registry checks avoid redundant builds.

26. **Run management.** Multiple transfer runs coexist in the workspace. `run.py list/inspect/switch` navigates between them. `switch` syncs generated files into the component submodule.

27. **Remote orchestration.** The orchestrator can run as a Kubernetes Job (`--remote`), not just locally. Input data is staged via ConfigMap.

28. **Health monitoring with tiered remediation.** Pod failures are classified into tiers: auto-remediate (delete pod), suggest (human action), or diagnose (API-assisted root cause analysis).

29. **Results collection from cluster.** `deploy collect` pulls trace data from PVCs across all namespace slots, producing per-workload result directories.

30. **Interactive analysis.** The analyze skill produces latency comparison tables, distribution charts, and HTML reports from collected results.

31. **Parity validation.** The check skill validates that the production deployment matches the simulation's assumptions (workload parity, config parity, signal mapping, policy equivalence).

---

## Resilience

Properties related to graceful handling of failure and scarcity.

32. **Exponential backoff on GPU scarcity.** When free GPUs drop below minimum pair cost, the orchestrator backs off exponentially rather than spinning. It recovers immediately when capacity returns.

33. **Recoverable vs non-recoverable failure classification.** Pod pending reasons are classified. Resource scarcity is waited out; config errors (bad affinity, missing PVC) are surfaced immediately.

34. **Reclaim-based escalation.** Repeated reclaim events (3+ in 600s window) trigger backoff escalation even without a direct scarcity signal.

35. **Graceful degradation on API failure.** When kubectl is unreachable or a ConfigMap is not found, the system should report the specific failure mode rather than returning empty state and proceeding silently.

36. **Timeout protection on external calls.** Long-running subprocess calls (kubectl, image push) should have timeouts so a hung process doesn't block the orchestrator indefinitely.

---

## Observability

How the system communicates its state to operators.

37. **State-transition logging.** The orchestrator logs capacity, dispatch decisions, and pair state only when they change — not on every poll iteration. Noise reduction is a design goal.

38. **Per-pair progress reporting.** `deploy status` and `deploy collect` report progress at the (workload, package) pair level, not just aggregates.

39. **Structured progress printing.** Entry points use `step(n, total, title)` patterns with color for TTY terminals. Progress is scannable.

40. **Health report persistence.** Monitor findings are written to `health_report.md` with timestamps. Prior-session findings are preserved across restarts.

41. **Run inspection.** `run.py inspect` shows phase completion, generated files, deploy stages, and review rounds for any run — a full audit trail.

---

## Process

Practices and workflows that must be followed by contributors.

42. **CI must pass before merge.** Lint (ruff, pyflakes F-codes) and tests (pytest) run on every PR. Failures block merge.

43. **Issue before implementation.** Changes are motivated by filed GitHub issues. The issue captures the problem; the PR captures the solution.

44. **Branch + PR workflow.** No direct pushes to main. All changes go through a branch and pull request.

45. **Pipeline README stays current.** Any change to CLI flags, phase behavior, artifact schema, or subcommands must be reflected in `pipeline/README.md` in the same PR.

46. **Test coverage for pipeline logic.** Pipeline library modules have unit tests. New modules and behavioral changes require corresponding test updates.

47. **CI config tracks new paths.** When a new module, test file, or skill is added, `.github/workflows/test.yml` must be updated to include it.

---

## Ergonomics

Properties affecting operator and developer experience.

48. **Sensible defaults.** `--experiment-root` defaults to current directory. Manifest location resolves from conventional paths. Flags have useful defaults.

49. **Resumable by default.** Re-running any pipeline script without arguments picks up where it left off. Explicit `--force` required to regenerate.

50. **Color output with TTY detection.** Color codes are emitted only when stdout is a terminal. Piped/redirected output is clean.

51. **Filter normalization.** Workload and package filters should accept both hyphens and underscores; strip common prefixes (e.g., `wl-`).

52. **Clear error messages over tracebacks.** User-facing errors should be a sentence, not a Python stack trace. Tracebacks indicate a bug in error handling.

53. **Confirmation before destructive operations.** `switch` confirms when the submodule is dirty. `collect` warns above a size threshold. Irreversible operations gate on explicit consent.

---

## Safety

Security and correctness guarantees.

54. **No silent failures.** A failing subprocess, missing file, or unreachable service must produce a visible error — never empty output interpreted as success.

55. **ConfigMap "not found" distinct from "connection error."** The progress store must distinguish "no data yet" from "cannot reach cluster." The former is normal; the latter is actionable.

56. **Basename collision detection.** When syncing generated files to the component submodule, colliding basenames (different source paths, same filename) must be detected and rejected before any writes occur.

57. **No credential leakage.** Secrets (HF tokens, GitHub tokens, registry credentials) flow through Kubernetes secrets and environment variables, never through workspace files or logs.

58. **Subprocess isolation.** All external calls (kubectl, git, docker) are wrapped with explicit error handling and output capture. Raw shell expansion is avoided.

---

## Optimizations

Desired properties without hard SLAs. Better is better.

59. **Minimize wasted GPU-hours.** Capacity-aware scheduling, backoff, and early failure detection all serve to avoid submitting work that will pend indefinitely or fail immediately.

60. **Avoid redundant image builds.** Source hash tracking and registry existence checks mean images are only built when something changed.

61. **Parallel collection across slots.** I/O-bound collection from multiple namespaces runs concurrently (ThreadPoolExecutor), not sequentially.

62. **Context caching by content hash.** The context document is cached by SHA-256 of its inputs. Unchanged inputs skip reassembly.

63. **Skip completed phases.** The state machine avoids re-running expensive phases (context assembly, translation, PipelineRun generation) when their outputs already exist.

64. **Minimize orchestrator log volume.** State-transition gating reduces log output to what's actionable, making real signals visible in long-running deployments.

---

## Consistency

How the system maintains coherence across its parts.

65. **Overlay name alignment.** When merging overlays into scenarios, the overlay's scenario name must match the base's name to prevent list-merge duplication.

66. **ConfigMap is authoritative over local files.** All subcommands read progress from the ConfigMap. Local files are caches or backups, never the source of truth.

67. **Run isolation.** Runs within the same workspace do not interfere. Each run has its own `.state.json`, `run_metadata.json`, and generated artifacts.

68. **Cross-run ConfigMap isolation (desired).** The progress ConfigMap should include the run name to prevent stale entries from prior runs causing confusion in a reused namespace.

---

## Extensibility

How the system accommodates growth.

69. **Manifest versioning.** The manifest carries a `version` field. Only v3 is accepted today; future versions can introduce new semantics without breaking old tooling.

70. **Pipeline definition is overridable.** `--pipeline-yaml` allows custom Tekton Pipeline definitions without modifying the framework.

71. **Workloads are pluggable.** Any conforming YAML in `workloads/` can be included in a transfer run. The system doesn't hardcode workload semantics.

72. **Baseline-only mode.** Algorithms are optional. A manifest with only baselines and workloads can be used for benchmarking without any treatment arm — useful for establishing baseline measurements.

73. **Skill-based translation is replaceable.** The translate step is decoupled via `skill_input.json` / `translation_output.json` files. An alternative translation mechanism (manual, different AI, deterministic template) could substitute without pipeline changes.

---

## Open Questions

Intents that are suspected but not yet confirmed or precisely stated.

- Should the orchestrator be fully declarative (desired state vs current state reconciliation) or remain imperative (step-by-step execution)?
- What is the SLA for result collection latency after a PipelineRun completes?
- Should the system support rollback (reverting a deployed algorithm to baseline) as a first-class operation?
- Is there an intent around reproducibility of AI-generated translations (same inputs → same plugin code)?
- Should experiment repos be fully self-contained (no framework repo dependency at runtime)?
