# Nous — Project Intents

A living catalog of what this system should do and be. Intents are
statements about desired properties, behaviors, and constraints.
They are not requirements in the traditional sense — they describe
*what we want to be true* about the system, at varying levels of
specificity and enforceability.

## Summary

This document contains **50 intents** organized into 8 categories:

| Category | Count | What it covers |
|----------|-------|----------------|
| **Invariants** | 10 | Hard constraints that must always hold — atomic writes, valid state transitions, schema validation, gate enforcement, worktree isolation. |
| **Features** | 18 | Capabilities the system provides — the experimentation loop, hypothesis bundles, code-change mode, checkpoint/resume, CLI, replay, inline dispatch. |
| **Optimizations** | 8 | Desired properties without hard SLAs — token discipline, parallel execution, knowledge compounding efficiency, minimal re-exploration. |
| **Safety** | 7 | Security and isolation — permission policies, worktree containment, credential hygiene, human-in-the-loop enforcement. |
| **Epistemology** | 10 | Scientific reasoning correctness — falsifiability, ground-truth independence, direction errors as highest-value signal, causal mechanisms, theory grounding. |
| **Process** | 7 | Development practices — PR workflow, schema-first design, smoke tests, failure logging. |
| **Architecture** | 8 | Structural properties — separation of orchestration and intelligence, pluggable dispatch, self-contained campaigns, reproducibility by default. |
| **Aspirational** | 12 | Desired future state from open issues — SDK port, parallel arms, critic phase, warm-start, MCP server, live progress TUI. |

The **Epistemology** section is distinctive to this project — few systems
have intents about *the quality of their own reasoning*. These derive from
real campaign failures (tautological experiments, unfalsifiable hypotheses)
and the protocol's emphasis on prediction error taxonomy.

---

## Taxonomy

Intents are grouped by *kind*:

| Kind | Meaning |
|------|---------|
| **Invariant** | Must hold in all circumstances; violations are bugs. |
| **Feature** | A capability or behavior the system provides. |
| **Optimization** | A desired property without a hard SLA — better is better. |
| **Safety** | Security, isolation, or harm-prevention constraints. |
| **Epistemology** | Correctness of reasoning and scientific validity. |
| **Process** | How the system is developed, tested, and operated. |
| **Architecture** | Structural properties of the codebase and deployment. |

Within each kind, intents are numbered for reference but not ranked.

---

## 1. Invariants

Things that must always hold. A violation means the system is broken.

### INV-1: Deterministic orchestration

The orchestrator (Python state machine) never calls an LLM. It owns
phase transitions, checkpointing, gate enforcement, and artifact
validation. AI agents are external processes invoked with structured
prompts and schema-governed outputs.

*Source: architecture.md, engine.py*

### INV-2: Atomic state writes

`state.json` is written via temp-file + fsync + rename. A crash at
any point during write leaves the previous valid state intact. The
in-memory state is only updated after the disk write succeeds.

*Source: engine.py, util.py*

### INV-3: Valid state transitions only

The engine enforces a fixed transition table. Any attempt to make an
invalid transition raises an error. There are no secret paths through
the state machine.

*Source: engine.py TRANSITIONS table*

### INV-4: Schema validation of all inter-component artifacts

Every artifact exchanged between orchestrator and agents is validated
against a JSON Schema (Draft 2020-12). The orchestrator trusts the
schema contract, not the content.

*Source: validate.py, schemas/*

### INV-5: Human gates cannot be bypassed

HUMAN_DESIGN_GATE and HUMAN_FINDINGS_GATE are hard stops. No code
path exists that skips them except the explicit `auto_approve` flag,
which itself requires `NOUS_ALLOW_AUTO_APPROVE=1` environment variable.

*Source: gates.py*

### INV-6: Iteration counter increments only on DONE → DESIGN

Loopbacks from HUMAN_DESIGN_GATE → DESIGN (rejection) do not
increment the counter. They are revisions within the same iteration.

*Source: engine.py:112-113*

### INV-7: Append-only ledger semantics

`ledger.json` rows are never modified or deleted once written.
Implementation reads, appends, and atomically rewrites. Idempotency
guard prevents duplicate rows for the same iteration.

*Source: ledger.py*

### INV-8: Principle merge is idempotent

Re-running principle merge for the same iteration produces a
duplicate (detectable by ID) rather than corruption. Principles are
upserted by ID.

*Source: iteration.py _merge_principles()*

### INV-9: Experiments run in isolated git worktrees

The executor never modifies the main working tree. Each experiment
gets a fresh worktree created from the target repo; the worktree is
cleaned up after execution completes.

*Source: worktree.py, iteration.py*

### INV-10: Artifacts survive worktree cleanup

All experiment output paths are absolute paths under `iter_dir/results/`.
Nothing important is written to the worktree's own file tree — only
to the campaign directory which persists.

*Source: execute_analyze.md prompt, validate.py output file check*

---

## 2. Features

Capabilities the system provides.

### FEAT-1: Hypothesis-driven experimentation loop

The core loop: form a falsifiable hypothesis, design a controlled
experiment, execute it, compare predictions to outcomes, extract
reusable principles. Both confirmed and refuted hypotheses produce
learning.

### FEAT-2: Hypothesis bundles with multiple arm types

Each experiment decomposes into arms: h-main, h-ablation,
h-super-additivity, h-control-negative, h-robustness. The bundle
structure forces the agent to think about *why* something works,
not just *that* it works.

### FEAT-3: Prediction error taxonomy

Wrong predictions are classified by type — direction, magnitude,
regime — determining what the system learns from the failure and how
principles are updated.

### FEAT-4: Compounding knowledge via principles

Principles extracted from iteration N constrain the design space of
iteration N+1. The principle store is a living knowledge base with
confidence levels, regime bounds, and supersession chains.

### FEAT-5: Checkpoint/resume

The campaign can be killed at any point and restarted from the last
committed state. Mid-flight campaigns resume at the correct iteration
and phase.

### FEAT-6: Two-role agent architecture

Planner (Opus) and Executor (Sonnet) are distinct roles with different
capabilities and prompts. The planner explores and designs; the executor
implements and measures.

### FEAT-7: Code-change experiments (evolve mode)

Bundle arms can include `code_changes` intents. The planner says
what and why; the executor implements as git patches in an isolated
worktree, enabling algorithmic experimentation beyond flag-tuning.

### FEAT-8: Handoff as living context document

`handoff.md` accumulates exploration context across iterations —
code maps, dead ends, warnings, validated commands. Each iteration's
designer curates it: keeps relevant entries, removes outdated ones,
adds new findings.

### FEAT-9: Gate summaries for human decision-making

Before each gate, an LLM-generated summary surfaces key numbers,
predictions, and results in plain language to help the human make
an informed approve/reject/abort decision.

### FEAT-10: Multi-iteration campaigns with continue gates

After each non-final iteration, a continue gate gives the human the
option to stop the campaign or advance. The campaign manages iteration
counting, ledger appends, and report generation.

### FEAT-11: Campaign report generation

At campaign end, an LLM generates `report.md` answering the research
question with specific evidence, principles, and limitations.

### FEAT-12: Unified CLI (`nous`)

A single entry point: `nous run`, `nous resume`, `nous status`,
`nous cost`, `nous report`, `nous replay`, `nous validate`.

### FEAT-13: Deterministic replay (`nous replay`)

Re-runs a prior iteration's `experiment_plan.yaml` commands
mechanically in a fresh worktree — no LLM needed. Verifies
reproducibility.

### FEAT-14: Inline dispatch mode

`--agent inline` emits prompts to stdout so Nous can run embedded
inside an existing agent framework. The calling agent writes artifacts
directly; no subprocess or API key needed.

### FEAT-15: LLM cost and usage tracking

Per-call metrics (tokens, cost, duration) logged to `llm_metrics.jsonl`.
`nous cost` displays aggregated summaries. Retry events logged to
`retry_log.jsonl`.

### FEAT-16: Pre-flight validation

At campaign start, validates that the CLI is installed and credentials
work via a quick test call. Environment problems are caught in seconds,
not hours into an overnight run.

### FEAT-17: Configurable model per phase

`defaults.yaml` specifies default models and max-turns per phase.
Campaign-level `models:` overrides defaults. CLI `--model` is the
fallback.

### FEAT-18: Validation CLI for agents

`nous validate design` and `nous validate execution` let agents
self-check their artifacts before claiming done. The agent retries
until validation passes.

---

## 3. Optimizations

Desired properties that improve the system but don't have hard
pass/fail criteria.

### OPT-1: Knowledge compounding efficiency

Each iteration should learn something the previous didn't know.
Marginal iterations (no new principles extracted) are a signal to
stop. The system should get smarter over time, not just longer.

### OPT-2: Token budget discipline

Prompts should not grow without bound. Handoff and principles
injection should be bounded or compacted. Static methodology prompts
should be cached when the SDK allows.

*Related: issue #122 (prompt caching), #131 (CLAUDE.md approach),
#77 (early stopping when question is answered)*

### OPT-3: Minimize wasted iterations

Fast-fail rules should cut wasted compute — if h-main is refuted,
remaining arms may be skipped. Campaigns should stop early when the
research question is answered.

*Related: issue #77, README fast-fail claims*

### OPT-4: Parallel execution when possible

Independent arms × seeds should be parallelizable. Currently
sequential.

*Related: issue #123 (SDK subagents), #82 (configurable parallelism),
#133 (harness-managed worktrees)*

### OPT-5: Resilient overnight operation

Long-running unattended campaigns should survive transient failures
(API timeouts, rate limits, network issues) via exponential backoff
and retry. Campaign-level resilience: failed iterations are recorded
and skipped, not fatal.

### OPT-6: Cross-campaign knowledge reuse

New campaigns against the same target should be able to warm-start
from prior campaigns' principles and handoff rather than re-discovering
everything.

*Related: issue #83 (warm-start)*

### OPT-7: Minimal re-exploration across iterations

The designer should build on previous handoffs, not re-explore the
same codebase from scratch. Dead ends from prior iterations must not
be repeated.

### OPT-8: Actionable reports over validation reports

When research discovers a better alternative to the investigated
approach, the report should clearly state "don't deploy X" rather
than burying it. Recommendation ≠ validation.

*Related: issue #101*

---

## 4. Safety

Security, isolation, and harm-prevention constraints.

### SAFE-1: Worktree isolation for experiments

The executor runs in a disposable git worktree. Experiments cannot
corrupt the main working tree. The worktree is force-removed after
execution.

### SAFE-2: No unscoped permissions

The current `--dangerously-skip-permissions` should be replaced with
fine-grained per-campaign permission policies that constrain the
executor to commands derivable from `experiment_plan.yaml`. Writes
outside the worktree should be denied by default.

*Related: issue #135 (permission policy), #128 (PreToolUse hook)*

### SAFE-3: Campaign directory stays inside target repo

When `repo_path` is set, campaign artifacts live at
`.nous/<run_id>/` inside the target repo. They don't leak into
arbitrary filesystem locations.

### SAFE-4: Auto-approve requires explicit opt-in

`auto_approve=True` requires `NOUS_ALLOW_AUTO_APPROVE=1` environment
variable. This prevents accidental bypass of human gates in production.

### SAFE-5: Human remains in the loop at decision points

The human sees both the design (what will be tested) and the findings
(what was learned) before the system advances. Domain expertise enters
the loop through gate decisions.

### SAFE-6: Executor constrained to its plan

The experiment plan (`experiment_plan.yaml`) should be the source
of truth for what the executor runs. Deviations from the plan should
be detectable and (in strict mode) blocked.

*Related: issue #128*

### SAFE-7: Sensitive credentials not embedded in artifacts

Campaign artifacts (YAML, JSON, prompts) should never contain API
keys, tokens, or other credentials. LLM authentication flows through
Claude CLI config, not through campaign files.

---

## 5. Epistemology

Correctness of reasoning. The system should do *good science*.

### EPIST-1: Hypotheses must be falsifiable

Every prediction must have a measurable success/failure threshold.
An experiment that cannot possibly fail is worthless.

*Related: issue #84, #87 (critic phase)*

### EPIST-2: Ground truth must be independent of the detector

The thing being tested and the thing used to judge it must be
different measurements. Tautological experiments (where the detector
and ground truth measure the same quantity) must be detected and
rejected.

*Related: issue #84, #85 (independence check in validator)*

### EPIST-3: Principles must distinguish empirical from analytical

A principle "proved with math" (algebraically guaranteed) is different
from one "discovered from data" (empirically observed). The system
should distinguish these so it doesn't mistake tautologies for
discoveries.

*Related: issue #86 (empirical_content flag)*

### EPIST-4: Direction errors are the most valuable

When a prediction is wrong in direction (not just magnitude or
regime), it reveals a fundamental flaw in the causal model. The
system should treat direction errors as the most serious and most
informative.

### EPIST-5: Principles are hard constraints on future designs

Active principles constrain subsequent iterations. The planner must
not design bundles that contradict active principles without explicit
justification.

### EPIST-6: Refuted hypotheses produce learning

The system must learn as much (or more) from refuted hypotheses as
from confirmed ones. A refutation is not a failure — it's a
correction to the model.

### EPIST-7: Predictions must be grounded in observed behavior

Base all experiment parameters on verified system behavior. If you
didn't probe it, don't assume it. No invented numeric thresholds
without evidence. Commands must be validated before inclusion in a
design.

### EPIST-8: Mechanisms must be causal, not correlational

Each arm requires a `mechanism` field — a causal explanation of
how/why the predicted effect occurs, grounded in the code. Correlation
without mechanism is not sufficient.

### EPIST-9: External theory references should anchor experiments

Campaigns investigating well-understood domains should declare the
external theory (queueing theory, information theory, etc.) that
provides independent ground truth.

*Related: issue #88 (theory_references)*

### EPIST-10: Campaign should stop when the question is answered

Continued iterations after the research question is answered waste
compute and risk overfitting the principle store. The planner should
be able to declare "research question answered" as a stop signal.

*Related: issue #77*

---

## 6. Process

How the system is developed, tested, and operated.

### PROC-1: Claude-based PR workflow

Contributors use the defined workflow: worktree → analyze → plan →
review → human gate → implement → review → PR. Plan files live in
`docs/plans/` (gitignored).

*Source: docs/contributing/workflow.md*

### PROC-2: Comprehensive test suite without LLM calls

The orchestrator is fully testable without LLMs via `StubDispatcher`.
Schema conformance, state transitions, ledger logic, validation —
all testable deterministically.

### PROC-3: Schema-first artifact design

New artifacts start as a schema. The schema is the contract between
components. Schema tests (`test_schemas.py`) verify that example
data conforms.

### PROC-4: Pre-merge confidence via smoke tests

Changes to prompts, validation logic, or orchestration should be
verifiable via a fast real campaign against a known target (BLIS)
before merging.

*Related: issue #97 (/smoke-test CI)*

### PROC-5: Atomic, focused commits

Each task in the implementation plan gets its own commit with a
descriptive message. No scope creep within a task.

### PROC-6: GitHub Actions for contributor invocation

`@claude` mentions in issues/PRs trigger Claude Code Action with
permission checks (admin/write/maintain only).

### PROC-7: All failures logged, not swallowed

Retry events go to `retry_log.jsonl`. Failed iterations go to
`ledger.json` with error details. Gate rejections go to
`human_feedback.json`. Nothing disappears silently.

---

## 7. Architecture

Structural properties of the codebase.

### ARCH-1: Separation of orchestration and intelligence

The orchestrator is a dumb pipe — it routes, validates, and
checkpoints. All reasoning lives in agent prompts and their
responses. This makes the system testable, auditable, and
model-swappable.

### ARCH-2: Pluggable dispatch backends

The system supports multiple dispatch backends (Stub, CLI, Inline)
behind a common interface (`LLMDispatcher`). New backends (SDK,
future agent frameworks) can be added without touching control flow.

*Related: issue #121 (Agent SDK port)*

### ARCH-3: Campaign artifacts are self-contained

A campaign directory (`.nous/<run_id>/`) contains everything needed
to understand what happened: config, state, ledger, principles,
per-iteration artifacts, metrics. No external dependencies for
auditability.

### ARCH-4: Prompt templates with placeholder rendering

Prompts are Markdown templates in `prompts/methodology/`. Context
is injected via `{{placeholder}}` rendering at dispatch time. Domain
knowledge comes from `campaign.yaml`, not hardcoded in templates.

### ARCH-5: Reproducibility by default

`experiment_plan.yaml` records exact commands. `nous replay` re-runs
them mechanically. Patches are saved as git diffs. Input files are
saved in `inputs/`. The full experiment is reconstructable from
artifacts alone.

### ARCH-6: The validator is an agent-accessible tool

Agents can run `nous validate` within their session to self-check.
The orchestrator's post-check is a safety net, not the primary
enforcement mechanism. This makes agents self-correcting.

### ARCH-7: Campaign directory inside target repo

When `repo_path` is set, `.nous/<run_id>/` lives inside the target
repo. This keeps campaigns co-located with the code they investigate,
enabling git-based versioning of research alongside code.

### ARCH-8: Living documents vs frozen snapshots

`handoff.md` and `principles.json` are living documents (updated
each iteration). Per-iteration snapshots (`handoff_snapshot.md`,
`principle_updates.json`) are frozen for audit. Both exist.

---

## 8. Aspirational (not yet implemented)

Intents that represent desired future state, not current reality.

### ASP-1: Per-campaign permission policies replace blanket skip

Replace `--dangerously-skip-permissions` with fine-grained settings.

*Issue #135*

### ASP-2: Plan enforcement via PreToolUse hooks

The experiment plan should be enforceable, not just descriptive.

*Issue #128*

### ASP-3: Critic phase catches unfalsifiable experiments

An adversarial review step before execution that asks "can this
experiment possibly fail?"

*Issue #87*

### ASP-4: SDK-native dispatch (no subprocess shelling)

Port from `subprocess(claude -p)` to the Claude Agent SDK for
proper lifecycle management, parallel subagents, and native caching.

*Issue #121*

### ASP-5: Parallel arm execution via subagents

One subagent per arm × seed, running concurrently in independent
worktrees.

*Issue #123, #133*

### ASP-6: Early stopping when research is saturated

Designer can declare "research question answered" instead of being
forced to produce another hypothesis bundle.

*Issue #77*

### ASP-7: Warm-start from prior campaigns

New campaigns inherit principles and handoff from completed prior
campaigns, acquiring only delta knowledge.

*Issue #83*

### ASP-8: MCP server exposing campaigns as resources

`nous-mcp` would let other agents query campaigns, principles, and
findings via the Model Context Protocol.

*Issue #126*

### ASP-9: Live progress and TUI for campaign monitoring

`--output-format stream-json` and `nous status --watch` for
real-time visibility into long-running campaigns.

*Issue #127*

### ASP-10: Methodology delivered via CLAUDE.md and auto-memory

Replace the 266-line prompt template with per-campaign CLAUDE.md
files and memory entries, reducing token bloat while preserving
methodology.

*Issue #131*

### ASP-11: Scheduled overnight campaigns via Routines

Integration with Claude Code Routines for fire-and-forget overnight
runs with async notification on completion.

*Issue #134*

### ASP-12: Campaign metadata and provenance at init time

Record target commit, Nous version, start time, and user-supplied
tags/goals in state.json for reproducibility and discoverability.

*Issue #115*

---

## Notes on this document

- Intents are not ordered by priority within a section.
- Some intents overlap (e.g., SAFE-1 and INV-9 both concern worktree isolation
  — the invariant states the hard constraint, the safety intent states the
  security motivation).
- Aspirational intents become features/invariants as they're implemented.
- This list should be revisited when: a new feature ships, a bug reveals a
  missing invariant, or a campaign failure exposes an epistemological gap.
