# Cross-Project Intent Analysis

## What this is

An analysis of the intents documents from three projects developed by
the same team:

- **Nous** (50 intents) — AI-driven hypothesis experimentation framework
- **BLIS** (~130 intents) — LLM inference serving simulator
- **sim2real** (73 intents) — Pipeline for transferring algorithms from simulation to production

The goal: identify common underlying principles that appear across all
three projects, understand where context-specific expression obscures
shared values, and discover where a general statement might replace
multiple specific ones.

---

## Summary of Findings

**18 common principles** emerge across the three projects. These fall
into three tiers:

| Tier | Meaning | Count |
|------|---------|-------|
| Universal | Present in all three, clearly the same underlying value | 12 |
| Strong | Present in two strongly, implied in the third | 4 |
| Emergent | The pattern is visible but expressed at different levels of maturity | 2 |

The most over-specified principles (where generalization would help)
are: atomic writes, schema validation, isolation mechanisms, and
failure visibility. Each project re-derives these from scratch in
domain-specific language when a shared statement would clarify that
they're the same commitment.

---

## Universal Principles (all three projects)

### 1. NO SILENT LOSS

| Project | How it's expressed |
|---------|-------------------|
| Nous | PROC-7: "All failures logged, not swallowed. Retry events go to retry_log.jsonl. Failed iterations go to ledger.json. Nothing disappears silently." |
| BLIS | INV-1: "Every injected request is accounted for at simulation end... No silent loss." SAFE-1: "No silent data loss: every error path must return an error, panic with context, or increment a counter." |
| sim2real | SAFE-54: "No silent failures. A failing subprocess, missing file, or unreachable service must produce a visible error — never empty output interpreted as success." |

**General statement:** *The system never discards information, swallows errors, or proceeds past a failure without an explicit record. Every input is accounted for; every failure is surfaced.*

**Why it's over-specified:** Each project phrases this as though it's a domain-specific insight ("requests," "retry events," "subprocess output") when it's really a universal engineering discipline. The general principle is about information conservation at system boundaries.

---

### 2. CRASH-SAFE STATE PERSISTENCE

| Project | How it's expressed |
|---------|-------------------|
| Nous | INV-2: "state.json is written via temp-file + fsync + rename. A crash at any point during write leaves the previous valid state intact." |
| BLIS | (Implied by INV-6 determinism; the simulator doesn't persist across runs but uses stdout/stderr separation for the same reason) |
| sim2real | INV-11: "All persistent state (.state.json, run_metadata.json, ConfigMap) is written atomically (tempfile + rename, or single kubectl apply). Partial writes cannot leave corrupt state." |

**General statement:** *Persistent state transitions atomically from one valid state to another. No intermediate or partial state is ever observable by the system or its operators.*

**Why it's over-specified:** Both Nous and sim2real describe the *mechanism* (temp + rename) rather than the *property* (atomic transitions). The mechanism is an implementation detail that happens to be the same because it's the correct one — but the intent is about the guarantee, not the technique.

---

### 3. DETERMINISM AND REPRODUCIBILITY

| Project | How it's expressed |
|---------|-------------------|
| Nous | ARCH-5: "experiment_plan.yaml records exact commands. nous replay re-runs them mechanically. Patches are saved as git diffs. The full experiment is reconstructable from artifacts alone." |
| BLIS | INV-6: "Same seed produces byte-identical stdout across runs." INV-13: "A trace exported and replayed produces identical per-request metrics." |
| sim2real | INV-4: "Workspace is always regenerable. Every file under workspace/ can be reproduced by re-running the stage that created it." |

**General statement:** *Given the same inputs, the system produces the same outputs. Any past result can be reconstructed from its recorded inputs without additional context.*

**The spectrum:** BLIS is the most extreme (byte-identical). Nous is deterministic at the experiment level (same plan → same execution). sim2real is deterministic at the pipeline level (same inputs → same workspace). The strictness varies by domain but the commitment is the same: **you can always go back**.

---

### 4. SCHEMA-GOVERNED CONTRACTS

| Project | How it's expressed |
|---------|-------------------|
| Nous | INV-4: "Every artifact exchanged between components is validated against a JSON Schema (Draft 2020-12)." PROC-3: "New artifacts start as a schema. The schema is the contract." |
| BLIS | SAFE-7: "Strict YAML parsing: typos in field names must cause parse errors, not silent acceptance." ARCH-6: "Configuration grouped by module, each independently validatable." |
| sim2real | INV-7: "Schema-first contracts. All inter-stage communication uses explicit, validated schemas. No implicit behavior; no untyped bags." |

**General statement:** *Data exchanged between components has a formal, machine-verifiable contract. Violations are caught at the boundary, not discovered downstream. The schema is designed first; the implementation conforms to it.*

**Why it's over-specified:** The phrase "schema-first" appears in both Nous and sim2real, and BLIS enforces the same property through strict parsing. Each project could simply say "contracts are explicit and validated" — the choice of JSON Schema vs strict YAML parsing vs interface types is context-dependent detail.

---

### 5. HUMAN JUDGMENT AT DECISION POINTS

| Project | How it's expressed |
|---------|-------------------|
| Nous | INV-5: "Human gates cannot be bypassed." SAFE-5: "The human sees both the design and the findings before the system advances." |
| BLIS | PROC-10: "Convergence protocol for reviews: parallel perspectives, classify findings, iterate until zero CRITICAL." PROC-11: "Three review gates per hypothesis experiment." |
| sim2real | Mission-3: "Automate mechanics; preserve human judgment. Decisions requiring judgment gate on human approval." |

**General statement:** *The system automates mechanical work but requires explicit human approval at points where judgment, domain expertise, or risk assessment is needed. Automation never silently makes decisions that belong to humans.*

**Interesting divergence:** Nous makes this an invariant (hard gate that cannot be bypassed). sim2real makes it a mission statement. BLIS makes it a process requirement. The enforcement level differs, but the value is identical. sim2real's phrasing ("automate mechanics; preserve human judgment") is the most general and could serve all three.

---

### 6. ISOLATION / CONTAINMENT

| Project | How it's expressed |
|---------|-------------------|
| Nous | INV-9: "Experiments run in isolated git worktrees." SAFE-1: "The executor never modifies the main working tree." |
| BLIS | ARCH-9: "Partitioned RNG per subsystem to isolate randomness streams." INV-9: "The control plane must not access Request.OutputTokens — only the execution engine may know actual output length." |
| sim2real | CONS-67: "Run isolation. Runs within same workspace don't interfere." ARCH-17: "Multi-slot parallel orchestration (namespace isolation)." |

**General statement:** *Activities that should be independent are physically separated so they cannot interfere. The system enforces boundaries rather than relying on discipline.*

**The forms of isolation:** git worktrees (Nous), RNG streams and information boundaries (BLIS), Kubernetes namespaces (sim2real). These are all the same principle — **compartmentalization** — expressed through the available mechanisms of each domain. The general statement is about the *purpose* (non-interference), not the *mechanism* (worktrees, namespaces, RNG partitions).

---

### 7. CHECKPOINT AND RESUME

| Project | How it's expressed |
|---------|-------------------|
| Nous | FEAT-5: "The campaign can be killed at any point and restarted from the last committed state." |
| BLIS | (Implicit — the simulator runs to completion, but INV-6 guarantees that any interrupted run can be restarted with identical results) |
| sim2real | ARCH-18: "State machine recovery. Interruption at any point allows resumption from last completed phase without loss of prior work." |

**General statement:** *Long-running processes survive interruption. The system records progress at well-defined checkpoints and resumes from the last one without requiring a full restart or losing completed work.*

---

### 8. PLUGGABLE IMPLEMENTATIONS

| Project | How it's expressed |
|---------|-------------------|
| Nous | ARCH-2: "Multiple dispatch backends (Stub, CLI, Inline) behind a common interface. New backends can be added without touching control flow." |
| BLIS | EXT-1 through EXT-10: "New routing/admission/scheduling algorithms can be added without modifying existing code." ARCH-3: "Single-method interfaces where possible." |
| sim2real | EXT-73: "Skill-based translation is replaceable. An alternative mechanism could substitute without pipeline changes." ARCH-20: "Skill boundary at Phase 3" (file-based seam). |

**General statement:** *The system separates policy from mechanism. Implementations are swappable behind stable interfaces without modifying orchestration or control flow.*

**The spectrum of pluggability:** BLIS achieves it through Go interfaces. Nous through Python class inheritance. sim2real through file-based boundaries (skill_input.json / translation_output.json). The most general form (sim2real's file boundary) imposes the weakest coupling — any process that reads/writes the right files can participate.

---

### 9. SEPARATION OF ORCHESTRATION AND INTELLIGENCE

| Project | How it's expressed |
|---------|-------------------|
| Nous | ARCH-1: "The orchestrator is a dumb pipe — it routes, validates, and checkpoints. All reasoning lives in agent prompts." INV-1: "The orchestrator never calls an LLM." |
| BLIS | ARCH-1: "Two-layer architecture: domain-agnostic simulation kernel separated from domain-specific modules." ARCH-4: "Unidirectional dependency flow." |
| sim2real | ARCH-14: "Linear pipeline with checkpoint." INV-10: "Each subcommand owns exactly one responsibility." Mission-3: "Automate mechanics; preserve human judgment." |

**General statement:** *Control flow is separated from domain logic. The orchestration layer is deterministic, auditable, and testable in isolation. Domain intelligence (AI reasoning, simulation physics, algorithm translation) is invoked through well-defined dispatch points, never entangled with the control machinery.*

**Why this matters:** All three projects are systems where "the interesting work" happens somewhere else (in LLM agents, in physics models, in translation skills). The orchestrator's job is to make that work reliable, resumable, and observable — not to do it.

---

### 10. SELF-CONTAINED ARTIFACTS

| Project | How it's expressed |
|---------|-------------------|
| Nous | ARCH-3: "A campaign directory contains everything needed to understand what happened: config, state, ledger, principles, per-iteration artifacts, metrics." |
| BLIS | FEAT-67: "TraceV2 format: lossless capture of all per-request data for replay and calibration." ARCH-5: Reproducibility. |
| sim2real | ARCH-19: "Experiment repo is the unit of configuration. Each experiment carries its own transfer.yaml, baselines, algorithms, workloads." |

**General statement:** *The unit of work is self-describing. Everything needed to understand, reproduce, or audit a result lives together — no external context required.*

---

### 11. OBSERVABILITY AND AUDIT TRAIL

| Project | How it's expressed |
|---------|-------------------|
| Nous | FEAT-15: LLM cost tracking. PROC-7: Failure logging. ARCH-3: Complete artifact trail. |
| BLIS | FEAT-60-67: Per-request metrics, decision tracing, counterfactual regret. |
| sim2real | OBS-37-41: State-transition logging, per-pair progress, health reports, run inspection. |

**General statement:** *The system records what happened, when, why, and at what cost. Operators can reconstruct the decision history without re-running. Observability is designed in, not bolted on.*

---

### 12. RESILIENT UNDER FAILURE

| Project | How it's expressed |
|---------|-------------------|
| Nous | OPT-5: "Survive transient failures via exponential backoff and retry. Failed iterations are recorded and skipped, not fatal." |
| BLIS | SAFE-3: "No livelock: retry loops must have circuit breakers." |
| sim2real | RES-32-36: "Exponential backoff on GPU scarcity. Recoverable vs non-recoverable failure classification. Timeout protection." |

**General statement:** *Transient failures are retried with backoff. Permanent failures are surfaced immediately. The system distinguishes between the two and never hangs indefinitely on either. Long-running operations survive their environment being temporarily hostile.*

---

## Strong Principles (two projects + implied in third)

### 13. IDEMPOTENT OPERATIONS

| Project | Strength |
|---------|----------|
| Nous | Explicit: INV-8 (principle merge), FEAT-5 (checkpoint/resume implies idempotent phase transitions) |
| sim2real | Explicit: INV-8 "Phases are idempotent. Re-running a completed phase is a no-op." |
| BLIS | Implied: Determinism (INV-6) guarantees that re-running produces same output, which is a form of idempotency |

**General statement:** *Re-executing a completed operation is safe and produces no additional effect. This makes recovery trivial: just run again.*

---

### 14. PLAN BEFORE IMPLEMENT

| Project | Strength |
|---------|----------|
| Nous | Explicit: PROC-1 (analyze → plan → review → gate → implement) |
| BLIS | Explicit: PROC-4 "Design docs before implementation." PROC-5: Four-tier document hierarchy. |
| sim2real | Weaker: PROC-43 "Issue before implementation" (issue as lightweight plan) |

**General statement:** *Think before doing. The plan is a separate artifact from the implementation. Plans are reviewed before code is written.*

---

### 15. SENSIBLE DEFAULTS / MINIMAL CONFIGURATION

| Project | Strength |
|---------|----------|
| Nous | Explicit: FEAT-17 (defaults.yaml), campaign can be 3 fields |
| BLIS | Explicit: OPT-4 "Sensible defaults should produce reasonable results without requiring deep understanding." |
| sim2real | Explicit: ERG-48 "Sensible defaults. --experiment-root defaults to current directory." |

**General statement:** *The system works out of the box with minimal configuration. Complexity is opt-in. The common case requires only the essential inputs.*

---

### 16. TESTABLE WITHOUT EXPENSIVE EXTERNALS

| Project | Strength |
|---------|----------|
| Nous | Explicit: PROC-2 "Fully testable without LLMs via StubDispatcher." |
| BLIS | Strong: OPT-2 "go test ./... in under 60 seconds." The simulator IS the test infrastructure. |
| sim2real | Moderate: PROC-46 "Test coverage for pipeline logic." (Tests exist but the cluster dependency is harder to stub.) |

**General statement:** *The core logic is testable without invoking expensive or slow external services. Stub/mock boundaries exist at the points where real infrastructure would be needed.*

---

## Emergent Principles (visible but at different maturity)

### 17. KNOWLEDGE COMPOUNDS OVER TIME

| Project | Maturity |
|---------|----------|
| Nous | Core feature: FEAT-4 "Principles from iteration N constrain iteration N+1. The system gets smarter over time." |
| BLIS | Process-level: Design docs, antipattern rules (R1-R23), and the 13 invariants are accumulated institutional knowledge — but not automated. |
| sim2real | Nascent: OBS-40 "Health report persistence. Prior-session findings are preserved across restarts." |

**General statement:** *The system accumulates knowledge from its operation and uses it to improve future behavior. Past mistakes become constraints; past successes become defaults.*

This is most explicitly automated in Nous (principle extraction), partially manual in BLIS (design guidelines and rules written by humans after bugs), and barely present in sim2real (health reports persist but don't feed back). The trajectory suggests all three will eventually want automated knowledge accumulation.

---

### 18. SINGLE SOURCE OF TRUTH

| Project | Maturity |
|---------|----------|
| sim2real | Explicit: INV-6 "Single source of truth per piece of state. No state has two authoritative homes." CONS-66: "ConfigMap is authoritative over local files." |
| Nous | Implicit: state.json is authoritative for phase, principles.json for knowledge, ledger.json for history. But not explicitly stated as an intent. |
| BLIS | Implicit: PROC-33 "Documentation has a single source of truth." Architecture enforces it (one construction site per struct). |

**General statement:** *Every piece of state has exactly one authoritative location. Other representations are caches, summaries, or views — never independent sources. When they conflict, the canonical source wins.*

---

## Where Generalization Would Help

The following intents are expressed with domain-specific detail that
obscures the shared principle. A team-level "meta-intent" could serve
as the anchor, with each project inheriting it:

| General Principle | Nous says | BLIS says | sim2real says |
|-------------------|-----------|-----------|---------------|
| Atomic state transitions | "temp + fsync + rename for state.json" | "stdout for deterministic, stderr for diagnostics" | "tempfile + rename for .state.json; single kubectl apply for ConfigMap" |
| No silent loss | "retry_log.jsonl, ledger.json record failures" | "every request accounted for; every error path returns an error" | "no silent failures; visible errors from subprocesses" |
| Isolation | "git worktrees" | "partitioned RNG; information boundaries" | "namespace slots; run isolation" |
| Pluggable backends | "LLMDispatcher subclasses" | "single-method interfaces; factory registration" | "file-based skill boundary" |
| Self-contained artifacts | ".nous/<run_id>/ has everything" | "TraceV2 is lossless" | "experiment repo is the unit of configuration" |

In each case, the project-specific language is necessary for
implementors, but the *reason* is identical across all three. A
shared document of team engineering principles could state these once,
and each project's intents document could reference the principle and
add domain-specific enforcement details.

---

## Principles Found in Only One Project (potential gaps)

These appear in only one project but seem like they *should* be universal:

| Principle | Where it appears | Why it might apply elsewhere |
|-----------|-----------------|------------------------------|
| Falsifiability of claims | Nous EPIST-1 | BLIS experiments and sim2real parity validation both make claims — are they falsifiable? |
| No oracle knowledge leakage | BLIS INV-9 | Nous agents shouldn't leak future knowledge; sim2real translation shouldn't use production-only signals |
| Fix upstream, never in artifact | sim2real INV-5 | Nous shouldn't patch generated artifacts; BLIS shouldn't hand-edit golden files |
| Degenerate input handling | BLIS SAFE-12 | Nous should handle empty principle stores; sim2real should handle single-workload manifests |
| Per-pair cost derivation | sim2real INV-12 | Nous could reason about per-phase costs; BLIS could do per-request cost attribution |
| Causal ordering guarantees | BLIS INV-5 | Nous has implicit phase ordering; sim2real has pipeline ordering — but neither states causality as a hard invariant |

---

## Taxonomy Observations

The three projects use overlapping but non-identical category names:

| Concept | Nous calls it | BLIS calls it | sim2real calls it |
|---------|--------------|---------------|-------------------|
| Must-hold guarantees | Invariant | Invariant | Invariant |
| How it's built | Architecture | Architecture | Architecture |
| What it does | Feature | Feature | Feature |
| How to develop | Process | Process | Process |
| Security/harm-prevention | Safety | Safety & Correctness | Safety |
| Better-is-better | Optimization | Optimization | Optimization |
| Growth accommodation | (in Architecture) | Extensibility | Extensibility |
| User experience | (not present) | Usability | Ergonomics |
| Scientific rigor | Epistemology | (in Process: experiment rigor) | (not present) |
| Future desired state | Aspirational | (in Evolution notes) | Open Questions |

The team has converged on a common vocabulary for 6 of the categories
(Invariant, Architecture, Feature, Process, Safety, Optimization).
Three diverge meaningfully:

1. **Extensibility** — BLIS and sim2real have it; Nous folds it into
   Architecture. Given that all three value pluggability, it probably
   deserves its own section everywhere.

2. **Epistemology** — Unique to Nous but applicable to BLIS (experiment
   rigor is epistemological) and sim2real (parity validation makes
   empirical claims). BLIS has this under "Process: Experiment Rigor"
   but doesn't elevate it to a top-level concern.

3. **Usability/Ergonomics** — BLIS and sim2real have it; Nous doesn't.
   Nous has usability concerns (the CLI, sensible defaults, gate
   summaries) but doesn't collect them as a category.

---

## Proposed Team-Level Meta-Intents

If the team were to maintain a single shared intents document that
each project inherits from, these 12 statements would be candidates:

1. **No silent loss.** The system never discards information or
   swallows errors without an explicit record.

2. **Atomic state transitions.** Persistent state moves from one
   valid state to another with no observable intermediate.

3. **Reproducibility.** Given the same inputs, the same outputs can
   be reconstructed. Past results are always recoverable.

4. **Schema-governed contracts.** Data crossing component boundaries
   has a formal, machine-verifiable contract.

5. **Human judgment preserved.** Automation handles mechanics;
   decisions requiring understanding gate on human approval.

6. **Isolation by construction.** Activities that should be
   independent are physically separated so they cannot interfere.

7. **Checkpoint and resume.** Long-running processes survive
   interruption and resume from the last checkpoint without loss.

8. **Pluggable implementations.** Policy is separated from mechanism.
   Implementations are swappable without touching orchestration.

9. **Orchestration separated from intelligence.** Control flow is
   deterministic, auditable, and testable in isolation from domain
   logic.

10. **Self-contained artifacts.** The unit of work carries everything
    needed to understand, reproduce, or audit it.

11. **Resilient under failure.** Transient failures are retried;
    permanent failures are surfaced. The system never hangs.

12. **Knowledge compounds.** The system accumulates learning from its
    operation and feeds it back to improve future behavior.

---

## Notes

- The convergence in vocabulary (6/8 shared category names) suggests
  the team has an implicit shared engineering culture that hasn't been
  written down as such.
- The strongest universals (#1-#6) are all defensive properties — they
  describe what the system must NOT do (lose data, corrupt state, skip
  validation, bypass humans, interfere across boundaries, lose work).
  The team's engineering instincts are most aligned on *avoiding harm*.
- The weaker universals (#7-#12) are constructive properties — they
  describe what the system SHOULD do. These show more variation because
  they're shaped by each project's specific domain needs.
- Epistemology as a category is Nous-specific in naming but the
  concerns (falsifiability, independence of measurement, causal vs
  correlational claims) apply to any project that makes empirical
  claims about system behavior — which BLIS certainly does.

---

## Proposed: Implicit Intents That Should Be Explicit

The three projects express many intents derived from hard-won
experience (bugs, campaign failures, data loss events). But there are
categories of best practice that the team almost certainly *follows*
without having stated them. Making these explicit serves two purposes:
(1) it prevents regression when new contributors join, and (2) it
surfaces gaps where the practice may not actually be followed
consistently.

Each proposal below includes a justification — why this is considered
a best practice, and evidence (where visible) that the team already
implicitly values it.

---

### A. Security

#### IMP-SEC-1: Least privilege for automated agents

*Every automated process should have only the permissions it needs
to complete its task, and no more. Permissions should be scoped by
time (revoked after use) and target (limited to specific paths,
namespaces, or resources).*

**Justification:** The principle of least privilege (NIST SP 800-53
AC-6) is the foundational security control. Over-privileged processes
create blast radius: a compromised or misbehaving agent can damage
systems beyond its intended scope.

**Evidence of implicit value:** Nous #135 (per-campaign permission
policy) and #128 (plan enforcer) both move toward this. sim2real
uses namespace isolation. But none of the three explicitly states the
general principle — they jump to specific mechanisms.

**Gap risk:** Nous currently uses `--dangerously-skip-permissions`
which is the *opposite* of this principle. BLIS runs locally so the
risk is lower, but any future multi-user or cloud deployment would
need this.

---

#### IMP-SEC-2: No secrets in version-controlled artifacts

*Credentials, API keys, tokens, and other secrets must never appear
in files that are committed to version control, written to log files,
or included in error messages. Secrets flow through environment
variables or dedicated secret stores.*

**Justification:** Secrets in repos are the #1 source of credential
leakage (GitHub's secret scanning blocks ~100 leaked keys per day).
Once committed, secrets persist in git history even after deletion.

**Evidence of implicit value:** Nous SAFE-7 states this for campaign
artifacts. sim2real SAFE-57 mentions it for workspace files. But
neither states it as a universal constraint. The .gitignore files
exclude `.env` but there's no validation that artifacts don't contain
secrets.

**Gap risk:** LLM prompts could inadvertently include secrets from
environment context. Agent-generated code might hardcode credentials
found during exploration.

---

#### IMP-SEC-3: Untrusted input is validated at system boundaries

*Any data entering the system from an external source (user input,
LLM output, API responses, file system reads, environment variables)
is validated before use. Validation happens at the boundary, not
deep inside the system.*

**Justification:** OWASP's input validation principle. Boundary
validation prevents injection attacks, type confusion, and
assumption violations from propagating. Validating once at entry
is cheaper and more reliable than validating at every use site.

**Evidence of implicit value:** All three projects validate schemas
at component boundaries. But LLM output — despite being untrusted
by nature — gets schema validation (structural) without semantic
validation (is this command safe to execute? does this path escape
the sandbox?).

**Gap risk:** Nous executes LLM-generated shell commands in
experiment worktrees. A malicious or confused LLM response could
generate commands that escape the worktree (e.g., absolute paths,
symlink traversal). sim2real's translate skill generates Go code
from LLM output — structural compilation catches syntax errors
but not semantic security issues.

---

#### IMP-SEC-4: Defense in depth for AI-generated actions

*When an AI agent proposes an action (shell command, code change,
file write, network request), multiple independent checks should
verify the action is within bounds. No single mechanism should be
the sole defense.*

**Justification:** LLMs are stochastic systems that can produce
unexpected output. Unlike traditional programs where behavior is
bounded by code paths, LLM outputs have unbounded variance. A
single validation layer (e.g., schema check) may pass structurally
valid but semantically dangerous output. Defense in depth (schema +
path check + permission scope + human gate) provides layered
protection.

**Evidence of implicit value:** Nous has schema validation + human
gates + worktree isolation (three layers). But in `--auto-approve`
mode (overnight runs), the human gate is removed, leaving only
schema + isolation. sim2real has translation review + compilation +
human approval (three layers for generated code).

**Gap risk:** The layers are not explicitly designed as a defense
stack. Removing one (e.g., for speed) may not trigger awareness
that the remaining layers are now the sole protection.

---

### B. Implementation

#### IMP-IMPL-1: All external calls have timeouts

*Every call to an external service (subprocess, HTTP API, database,
file system on network mounts) must have an explicit timeout. No
external call may block indefinitely.*

**Justification:** Unbounded waits are the primary cause of hung
processes in distributed systems (Nygard, *Release It!*). A hung
subprocess or unresponsive API consumes resources (threads, file
descriptors, memory) without producing value. Timeouts convert
unbounded waits into bounded failures that the retry/recovery
machinery can handle.

**Evidence of implicit value:** Nous has `--timeout` for claude -p
calls. sim2real RES-36 mentions timeout protection. BLIS doesn't
make external calls (it's a simulator) so this is N/A. But it's
not stated as a universal principle.

**Gap risk:** Not all subprocess calls in Nous have timeouts (e.g.,
`nous validate` calls within agent sessions). sim2real's kubectl
calls may not all have timeouts. New code added without this
principle risks introducing the same class of hang.

---

#### IMP-IMPL-2: Resource cleanup is guaranteed, not best-effort

*Resources acquired (file handles, subprocesses, network connections,
temporary files, git worktrees) must be released via structured
cleanup (try/finally, context managers, defer) — not relied upon to
happen in the happy path.*

**Justification:** Resource leaks are cumulative. A leaked file
handle or orphaned subprocess may not cause immediate failure, but
under sustained operation (overnight campaigns, long-running
orchestrators) they accumulate until the system runs out of
resources. The fix is structural: acquire and release must be
syntactically paired.

**Evidence of implicit value:** Nous's `worktree.py` has
`remove_experiment_worktree` but it's called manually, not via
a context manager — and the comment "Safe to call even if already
removed" suggests cleanup has been missed in the past. sim2real
uses `try/finally` in orchestrator loops. BLIS is garbage-collected
(Go) but has explicit `Close()` calls on file handles.

**Gap risk:** Nous worktrees can be orphaned if the campaign crashes
between creation and cleanup. The `.nous-experiments/` directory
may accumulate dead worktrees over many failed runs.

---

#### IMP-IMPL-3: Errors carry context, not just messages

*Error values should carry enough context for the receiver to
understand what failed, where, and what input caused the failure
— without requiring access to the original call site. Stack traces
are for developers; structured context is for operators.*

**Justification:** An error message like "validation failed" forces
the operator to re-derive context (which file? which field? what
value?). Structured errors with context (file path, field name,
expected vs actual) enable automated error handling and human
diagnosis without log correlation. (Google SRE book, Chapter 17:
"Testing for Reliability")

**Evidence of implicit value:** Nous's validate.py returns structured
`{"status": "fail", "errors": [...]}` with per-field context. BLIS
SAFE-1 requires error paths to "panic with context." sim2real ERG-52
says "clear error messages over tracebacks." The value is present
but not universally applied.

**Gap risk:** Python exceptions in Nous often carry only a message
string. When these bubble up through retry loops, the original
context (which iteration, which arm, which file) may be lost.

---

#### IMP-IMPL-4: Concurrency safety is structural, not disciplinary

*When multiple processes or goroutines access shared state, safety
is enforced by the system's architecture (message passing, ownership
transfer, immutable data) — not by developer discipline ("remember
to hold the lock").*

**Justification:** Lock-based concurrency relies on every developer
at every call site remembering the protocol. A single violation
causes data races that manifest non-deterministically and are nearly
impossible to diagnose in production. Structural approaches (channel
ownership in Go, isolated worktrees in Nous, namespace isolation in
sim2real) make violations impossible by construction.

**Evidence of implicit value:** Nous isolates via separate
subprocesses (no shared memory). BLIS uses single-threaded
discrete-event simulation (no concurrency). sim2real uses
multi-process parallelism with namespace isolation. All three
*avoid* shared-memory concurrency rather than trying to do it
safely — which IS the structural approach.

**Gap risk:** As Nous moves toward parallel arm execution (#123)
and sim2real runs parallel orchestration, the temptation to add
shared state (progress counters, resource pools) will grow. The
principle of structural isolation should be stated before it's
violated.

---

### C. Language and Type Safety

#### IMP-LANG-1: Type annotations are contracts, not documentation

*In Python: all function signatures have complete type annotations.
In Go: interfaces are used at module boundaries. Types are the
machine-verifiable layer of the contract — not just hints for
IDEs.*

**Justification:** Type annotations enable static analysis (mypy,
pyright) to catch contract violations at development time rather
than runtime. They serve as executable documentation that can't
drift from the implementation. Studies (Ore & Birgisson, 2018)
show that type annotations in Python reduce type-related bugs by
~15%.

**Evidence of implicit value:** Nous uses Python type hints
throughout (e.g., `def dispatch(self, role: str, phase: str, *,
output_path: Path, iteration: int)`). BLIS uses Go's static type
system and interface types. sim2real likely uses type hints (Python
project). But none states this as an intent.

**Gap risk:** Without the explicit intent, type annotations may
be inconsistently applied (some functions typed, others not),
reducing the value of static analysis to whoever runs mypy.

---

#### IMP-LANG-2: Dependencies are pinned and bounded

*Production dependencies specify exact versions (pinned in lock
files). Development dependencies specify compatible ranges.
Dependencies are periodically reviewed for vulnerabilities and
updated deliberately — not auto-updated.*

**Justification:** Unpinned dependencies are the primary vector
for supply-chain attacks (e.g., event-stream, ua-parser-js,
colors.js). Even without malice, a minor version bump in a
transitive dependency can change behavior silently. Pinning
provides reproducibility; periodic review provides security.

**Evidence of implicit value:** Nous's pyproject.toml specifies
`>=` ranges (not pinned). No lock file is committed. BLIS uses
Go modules (which pin by default via go.sum). sim2real likely
has similar Python dependency management.

**Gap risk:** Nous's `pip install` with `>=` ranges means that
two installations a week apart may get different dependency
versions — potentially explaining non-reproducible behavior in
overnight campaigns.

---

### D. Documentation

#### IMP-DOC-1: Decisions are recorded with rationale

*When a non-obvious design decision is made (choosing A over B,
accepting a tradeoff, deferring a concern), the decision and its
rationale are recorded in a discoverable location. The record
answers "why is it this way?" for future readers.*

**Justification:** Code shows *what* was built. Commit messages
show *what changed*. But neither reliably captures *why this
approach and not that one*. Without decision records, future
developers re-derive the reasoning (wasting time), reverse the
decision (reintroducing solved problems), or are afraid to change
anything (calcification). ADRs (Architecture Decision Records,
Nygard 2011) are the standard practice.

**Evidence of implicit value:** BLIS has design docs and a four-tier
document hierarchy (PROC-5). Nous has docs/architecture.md explaining
design philosophy. sim2real has less visible decision documentation.
But none formally maintains ADRs.

**Gap risk:** Nous's architecture doc explains "why" for the current
design — but individual decisions within that design (why temp+rename
instead of write-ahead log? why two agents instead of one? why YAML
for bundles but JSON for findings?) are not recorded. When someone
asks "can we change this?", the answer requires archaeology.

---

#### IMP-DOC-2: Public interfaces have usage examples

*Every public function, CLI command, and configuration option has
at least one concrete usage example — not just a description of
what it does. Examples show the common case, not the edge case.*

**Justification:** Research on API usability (Myers & Stylos, 2016)
shows developers learn APIs primarily through examples, not
reference documentation. A function with a complete type signature
but no example is harder to use correctly than one with a single
concrete invocation showing typical inputs and outputs.

**Evidence of implicit value:** Nous's README has extensive CLI
examples. BLIS has `examples/` directory and workload presets.
sim2real has a guided workflow. But individual functions within
the codebases often lack examples (typical of internal code).

**Gap risk:** For projects intended to be used by others (Nous is
open source, BLIS is used by multiple teams), under-documented
internal APIs create friction for contributors.

---

### E. Process

#### IMP-PROC-1: Every bug fix has a regression test

*When a bug is fixed, a test is added that would have caught the
bug before the fix. The test fails on the pre-fix code and passes
on the post-fix code. This test is the permanent record that this
class of bug is guarded against.*

**Justification:** Without regression tests, bugs recur (sometimes
years later) when refactoring touches the same code path. The
cost of writing the test at fix time is minimal (you've already
diagnosed the bug and know the triggering condition). The cost of
NOT writing it is paid later in re-diagnosis time. (Google Testing
Blog: "The Beyoncé Rule" — if you liked it, you should have put
a test on it.)

**Evidence of implicit value:** Nous has test files for every
module. BLIS PROC-2 says "tests verify laws." sim2real PROC-46
requires test updates. But none explicitly states the regression
test requirement.

**Gap risk:** Bugs fixed by prompt changes in Nous (agent behavior
improvements) may not have corresponding tests because the
StubDispatcher doesn't exercise prompt quality. The /smoke-test
proposal (issue #97) addresses this gap.

---

#### IMP-PROC-2: Breaking changes are versioned and migrated

*When a change breaks existing contracts (schema changes, CLI flag
removals, API changes), the old behavior continues to work for a
transition period or a migration tool is provided. Breaking changes
are announced explicitly, not discovered by users.*

**Justification:** Hyrum's Law: "With a sufficient number of users,
every observable behavior of your system will be depended on by
somebody." Unannounced breaking changes erode trust and create
support burden. Semantic versioning (semver.org) codifies this:
breaking changes require a major version bump.

**Evidence of implicit value:** sim2real EXT-69 has manifest
versioning ("only v3 is accepted today"). Nous has schema versions
implied by JSON Schema `$schema` declarations. BLIS doesn't have
external consumers so this is less critical. But Nous's schemas
have evolved (e.g., adding `code_changes` to bundles) without
explicit versioning of the schema itself.

**Gap risk:** Nous campaigns from three months ago may have
`findings.json` files that don't conform to the current schema.
Without schema versioning, there's no way to know which schema
version a given artifact was written against.

---

#### IMP-PROC-3: Performance baselines exist and are monitored

*Key operations have established performance baselines (execution
time, memory usage, cost). Deviations beyond a threshold are
surfaced as alerts or CI failures — not discovered weeks later.*

**Justification:** Performance regressions are silent bugs. They
don't cause test failures, they don't produce errors — they just
make things slower or more expensive. Without baselines and
monitoring, performance degrades gradually until someone notices
"it used to be faster" but can't identify when or why.

**Evidence of implicit value:** Nous tracks LLM cost per campaign
(FEAT-15). BLIS OPT-1 and OPT-2 mention specific performance
targets (seconds not minutes; 60s test suite). sim2real OPT-59
minimizes GPU-hours. But none has automated regression detection
for these metrics.

**Gap risk:** Nous prompt bloat (identified in issue #77 — prompts
grew from 17K to 48K chars over iterations) was not caught until
a timeout. A baseline + alert would have surfaced this earlier.

---

#### IMP-PROC-4: Dependencies on external services are explicit and bounded

*Every external service the system depends on (LLM APIs, container
registries, Kubernetes clusters, git remotes) is explicitly
documented with: what happens if it's unavailable, how long the
system can operate without it, and what the fallback behavior is.*

**Justification:** External dependencies are the primary source of
production outages (Google SRE book, Chapter 22). Systems that
don't explicitly model their dependencies can't reason about
partial failures, can't test degraded modes, and can't set
expectations with operators about what "unavailable" looks like.

**Evidence of implicit value:** Nous's pre-flight check (FEAT-16)
validates the Claude CLI before starting. sim2real RES-35 handles
API failures. But the dependency inventory (what does the system
need, and what happens without each piece?) is not documented for
any of the three projects.

**Gap risk:** Nous depends on Claude CLI, OpenAI-compatible API
(for summaries), and git. If the OpenAI endpoint is down, summaries
are skipped (graceful). But if git is unavailable (worktree
creation fails), the campaign crashes — this asymmetry in failure
modes is not documented.

---

### F. Operational

#### IMP-OPS-1: Structured logging with consistent fields

*Log entries use structured format (JSON or key=value) with
consistent field names across the codebase. Common fields (timestamp,
component, severity, operation, duration) appear in every entry.
Free-text messages are supplementary, not the primary signal.*

**Justification:** Unstructured logs ("Processing file X...") cannot
be queried, aggregated, or alerted on. Structured logs enable
automated analysis: "show me all entries where component=executor
AND duration_ms > 30000" is trivial with structured data, impossible
with free text. Consistency of field names enables cross-component
correlation.

**Evidence of implicit value:** Nous logs to `llm_metrics.jsonl`
(structured) but uses Python's logging module with format strings
for operational logs (unstructured). sim2real OBS-37 mentions
"state-transition logging" but the format isn't specified. BLIS
outputs structured metrics (JSON) but operational logs are stderr
free text.

**Gap risk:** Debugging overnight Nous campaigns requires reading
free-text log output. A structured log with fields like
`{phase, iteration, event, duration_ms, error}` would make
`jq` queries trivial.

---

#### IMP-OPS-2: Graceful shutdown preserves in-flight work

*When the system receives a termination signal (SIGTERM, Ctrl-C,
timeout), it completes or checkpoints in-flight work before exiting.
Abrupt termination does not leave state that prevents restart.*

**Justification:** In containerized and cloud environments, processes
are routinely terminated (scale-down, preemption, deployment).
A system that can only be stopped between operations (not during
them) requires careful orchestration to avoid data loss. Graceful
shutdown is the difference between "kill and restart" being safe
(ops-friendly) versus dangerous (requires manual intervention).

**Evidence of implicit value:** Nous has checkpoint/resume (FEAT-5)
which handles the *consequence* of abrupt termination. sim2real
has state machine recovery (ARCH-18). But neither explicitly handles
SIGTERM — they assume crash recovery rather than graceful shutdown.
Nous's `gates.py` catches KeyboardInterrupt but `cli_dispatch.py`
may leave a `claude -p` subprocess orphaned on SIGTERM.

**Gap risk:** Overnight Nous campaigns killed by timeout may leave
orphaned processes or worktrees. The retry mechanism handles this
reactively (retries after restart) rather than proactively (clean
up before exit).

---

### Summary of Proposed Implicit Intents

| Category | ID | One-line statement |
|----------|----|--------------------|
| Security | IMP-SEC-1 | Least privilege for automated agents |
| Security | IMP-SEC-2 | No secrets in version-controlled artifacts |
| Security | IMP-SEC-3 | Untrusted input validated at system boundaries |
| Security | IMP-SEC-4 | Defense in depth for AI-generated actions |
| Implementation | IMP-IMPL-1 | All external calls have timeouts |
| Implementation | IMP-IMPL-2 | Resource cleanup is guaranteed, not best-effort |
| Implementation | IMP-IMPL-3 | Errors carry context, not just messages |
| Implementation | IMP-IMPL-4 | Concurrency safety is structural, not disciplinary |
| Language | IMP-LANG-1 | Type annotations are contracts, not documentation |
| Language | IMP-LANG-2 | Dependencies are pinned and bounded |
| Documentation | IMP-DOC-1 | Decisions are recorded with rationale |
| Documentation | IMP-DOC-2 | Public interfaces have usage examples |
| Process | IMP-PROC-1 | Every bug fix has a regression test |
| Process | IMP-PROC-2 | Breaking changes are versioned and migrated |
| Process | IMP-PROC-3 | Performance baselines exist and are monitored |
| Process | IMP-PROC-4 | Dependencies on external services are explicit and bounded |
| Operational | IMP-OPS-1 | Structured logging with consistent fields |
| Operational | IMP-OPS-2 | Graceful shutdown preserves in-flight work |

Of these 18, the ones with the highest gap risk (most likely to cause
real problems if left implicit) are:

1. **IMP-SEC-3** (untrusted input validation) — because all three
   projects consume LLM output or external data that is structurally
   validated but not semantically constrained.
2. **IMP-IMPL-2** (resource cleanup) — because Nous worktrees and
   sim2real namespace slots are known to leak under failure.
3. **IMP-PROC-2** (breaking changes) — because Nous schemas evolve
   and old campaign artifacts may become unreadable.
4. **IMP-OPS-2** (graceful shutdown) — because overnight unattended
   operation is an explicit goal of both Nous and sim2real.
