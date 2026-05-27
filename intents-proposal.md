# Intent Management System — Proposal

## The Idea

Teams build things — software, documents, systems, research. The things they build embody decisions: what matters, what's acceptable, what's out of bounds. Today, these decisions live in people's heads, in scattered documents, in code comments, in tribal knowledge. When a new person joins, they learn the hard way. When two teams collaborate, they discover misalignment after the work is done. When an AI agent is given a task, it has no way to know what constraints it's operating under beyond what's in the immediate prompt.

**Intents** make these decisions explicit and manageable. An intent is a statement about what should be true: "the system never loses data silently," "claims require evidence," "the common case requires minimal configuration." Intents exist at different levels — from organization-wide principles to individual working preferences — and they accumulate into a coherent body that guides all work within their scope.

An **Intent Management System (IMS)** is the infrastructure for working with intents at scale. It helps people state what they want, keeps the body of intents consistent, tracks whether intents are being satisfied, identifies when new work would violate existing commitments, and routes decisions to the right people. It works for any kind of output — code, research papers, proposals, deployments — because it manages the *goals*, not the artifacts.

### Why This Matters

Without explicit intent management:
- Teams re-derive the same principles independently, using different words, missing the shared commitment underneath
- Conflicts between goals surface late — during code review, in production incidents, or in customer complaints
- AI agents operate without guardrails beyond their immediate instructions, unable to respect broader commitments they were never told about
- "Why is it built this way?" has no discoverable answer — you have to find the right person and hope they remember

With it:
- A new team member can read the active intents for their scope and understand what matters before writing a line of code
- A conflict between "optimize for speed" and "validate all inputs" is detected when the intents are stated, not when the shortcut causes a bug
- An AI agent implementing a feature can check its plan against active intents and flag violations before producing artifacts
- A manager can ask "are we satisfying our reliability commitments?" and get a grounded answer, not a vague reassurance

### A Small Example

A team has three projects. Over time, each project independently arrives at the same principle: *the system should never silently discard data or swallow errors.* But they express it differently:

- Project A: "All failures logged, not swallowed. Nothing disappears silently."
- Project B: "Every request is accounted for at simulation end. No silent loss."
- Project C: "A failing subprocess must produce a visible error — never empty output interpreted as success."

These are the same intent, expressed three times in domain-specific language. Nobody realizes they share it because it's never been stated at the organizational level.

Now a fourth project starts. It has no reason to adopt this principle — nobody told them. They build a data pipeline that silently drops malformed records. It works fine until it doesn't, and the resulting incident takes a week to diagnose because no one can tell which records were lost.

With an IMS, the org-level intent "**No silent loss**: the system never discards information or proceeds past a failure without an explicit record" exists once, inherited by all projects. The fourth team sees it from day one. Their pipeline design includes an error log from the start — not because someone remembered to tell them, but because the intent was explicit and visible in their scope.

When they later propose an optimization ("drop records that fail validation three times to avoid infinite retry"), the IMS flags a tension with the no-silent-loss intent. The tension is real — but now it's surfaced *before* implementation, and resolved deliberately (e.g., "dropped records go to a dead-letter queue with full context") rather than discovered in an incident.

---

## Introduction (Technical)

An **intent** is a statement about what should be true — about a system, a process, a document, or any artifact that a team produces. Intents are not tasks (which describe work to do) or requirements (which describe acceptance criteria). They describe *desired properties* at varying levels of abstraction, from organizational values ("no silent loss") to specific mechanisms ("state.json written via temp+rename").

An **Intent Management System (IMS)** helps users state what they want, tracks progress toward satisfaction, identifies conflicts and ambiguities, and routes decisions to the right people. The IMS is artifact-agnostic — it manages the intent graph regardless of whether satisfaction produces code, a paper, a deployment, a policy document, or a presentation. The artifact-production system is downstream and pluggable.

The IMS serves multiple participants with different knowledge, authority, and concerns. It renders intents at the appropriate level of abstraction for each audience. It maintains consistency within and across scopes, detects when intents conflict, and ensures that conflicts are routed to someone with the authority and context to resolve them.

---

## How Intents Evolve

### Three Stores

The IMS maintains three distinct bodies of information:

| Store | Contains | Character |
|-------|----------|-----------|
| **What we want** | Intents — desired properties, constraints, goals | Prescriptive |
| **What we're learning** | Discoveries, failed approaches, constraints found, refined understanding | Descriptive |
| **What we're producing** | Artifacts — code, documents, deployments, whatever satisfies intents | Concrete |

These are genuinely different. An intent can outlive many artifacts. An artifact may satisfy multiple intents. Learning feeds back into both — causing intents to sharpen and informing how artifacts are produced.

### Intent Lifecycle

An intent has two independent dimensions of state: the stability of its *definition* and the status of its *satisfaction*. These are orthogonal — an intent can be well-defined but unsatisfied, or satisfied but evolving (because we're raising the bar).

**Definition state** — is the intent itself stable?

| State | Meaning | Transitions |
|-------|---------|-------------|
| **Draft** | Stated but not yet reviewed for consistency | → Accepted (after review) |
| **Accepted** | Consistent with the intent set, stable enough to work toward | → Evolving (when refinement needed) |
| **Evolving** | Being refined — clarified, decomposed, or sharpened | → Accepted (when stable again) |
| **Retired** | No longer active — superseded, withdrawn, or permanently achieved | terminal |

An intent can move between Accepted and Evolving repeatedly. Implementation often reveals that an intent needs refinement; refinement doesn't mean work stops, but the IMS should flag that downstream artifacts may need revisiting when the definition re-stabilizes.

**Satisfaction state** — is the intent being met?

| State | Meaning | Transitions |
|-------|---------|-------------|
| **Unsatisfied** | No artifacts yet fulfill this intent | → In Progress (work begins) |
| **In Progress** | Work is underway toward satisfaction | → Satisfied (artifacts produced) |
| **Satisfied** | Artifacts exist that fulfill the intent | → Verified (independently confirmed) |
| **Verified** | Satisfaction independently demonstrated | → Unsatisfied (regression) |

Satisfaction can regress at any point: Verified → Unsatisfied, Satisfied → Unsatisfied. This happens when a new or modified intent enters the set and invalidates the existing satisfaction — by raising the bar, introducing a conflict, or changing the context.

**Blocked** is a condition that can apply in either dimension, not a state in its own right:
- *Definition blocked*: the intent can't stabilize because it conflicts with another intent and the conflict is unresolved
- *Satisfaction blocked*: the intent is well-defined but work can't proceed because satisfying it would violate another intent

In both cases, the system cannot proceed without a decision. Any conflict is technically resolvable — at some cost (duplicate the system, weaken one intent, narrow the scope, reframe the problem). The question is never "can we?" but "at what price and who decides?"

**What causes satisfaction to regress:**

- *Direct conflict* — a new intent contradicts something the satisfied intent depends on
- *Raised bar* — a new intent strengthens criteria such that existing satisfaction no longer qualifies
- *Dependency invalidation* — a sub-intent becomes unsatisfied, propagating upward to the parent

**Severity of regression** depends on category:
- An Invariant or Safety intent regressing is a stop-the-line event — the system may currently be in a state it should not be in
- An Optimization or Ergonomics intent regressing may be acceptable temporarily — it represents degradation, not danger
- The introducing intent may itself need to be blocked until the conflict it created is resolved

The IMS must prevent new intents from silently degrading the system. At minimum, any satisfaction regression caused by a new introduction must be surfaced to affected participants before work proceeds.

### Intent Relationships and Transformations

Intents don't exist in isolation. They relate to each other through several distinct kinds of transformation, each producing a different structural relationship:

**Decomposition** — breaking an intent into parts

The parent intent breaks into sub-intents that collectively satisfy it. The parent remains as the "why"; sub-intents represent the parts or steps. The parent is satisfied when its sub-intents are.

> "The system is resilient under failure" decomposes into: "transient failures are retried with backoff," "permanent failures are surfaced immediately," and "no operation blocks indefinitely."

Relationship: parent ← parts. Parts are *different concerns* that together constitute the whole.

**Refinement** — making an intent concrete for a specific context

A general intent is expressed in more specific, actionable terms for a particular scope, language, or domain. The refined version is *the same concern* made testable. The general and refined versions are linked — satisfying the refinement contributes to satisfying the general form.

> "No silent loss" (org) refines into "every error path returns an error or increments a counter" (Go repo) and "all exceptions logged to structured output; bare `except: pass` is forbidden" (Python repo).

Relationship: general ← specialization. The IMS should flag general intents that have no refinements in active repos — is that a gap or an intentional exception?

**Derivation** — an intent that exists because of another intent

One intent implies or necessitates another. The derived intent wouldn't exist independently — it serves the parent. But unlike decomposition, it may be a *consequence* rather than a *part*.

> "Experiments must be reproducible" derives "all random seeds are recorded" and "dependency versions are pinned." These aren't parts of reproducibility — they're things that must be true *for* reproducibility to hold.

Relationship: intent ← consequence. Derived intents are necessary conditions, not components.

**Tension** — two intents that trade off against each other

Both intents are valid and desired, but satisfying one fully makes the other harder or more expensive. Unlike a conflict (which blocks), a tension requires explicit prioritization or a creative resolution that partially satisfies both.

> "Minimize wasted compute" is in tension with "validate all inputs exhaustively." More validation costs more compute; less validation risks wasted work downstream.

Relationship: intent ↔ intent. The IMS should track known tensions and the resolution strategy chosen.

**Supersession** — a new intent replaces an old one

The old intent is retired; the new one takes its place. The link preserves history — why was the old form abandoned? What changed?

> "Experiments run in isolated Docker containers" superseded by "experiments run in isolated git worktrees" — same goal (isolation), different mechanism, for reasons worth recording.

Relationship: old ← new. The old intent moves to Retired; the new one carries forward.

**Speculative introduction** — a candidate intent evaluated without commitment

A hypothetical intent is introduced into the graph to assess impact: what would it conflict with? What currently-satisfied intents would regress? What refinements would be needed? The candidate is not committed unless the user accepts the assessed cost.

> "What if we add: all API calls must complete within 5 seconds?" — the IMS reports which existing intents this would conflict with and what work it would imply, without changing anything.

Relationship: none (yet). The candidate is evaluated against the graph but doesn't participate until committed.

---

These relationships form a graph. An intent may have decomposition children, refinement children at lower scopes, derived intents it implies, tensions with siblings, and a supersession history. The IMS navigates this graph to answer questions like "why does this intent exist?" (follow derivation/decomposition upward) and "what would break if we changed this?" (follow refinement/derivation downward).

### The Feedback Loops

```
User states intents
    → IMS checks consistency (conflicts, ambiguity, gaps)
    → User resolves (clarify, qualify, prioritize, withdraw)
    → Work proceeds toward satisfaction (producing artifacts, generating learning)
    → User observes progress (high-level or drill-down)
    → Discoveries cause intents to evolve
    → User questions, redirects, adds constraints
    → Cycle continues
```

The user's primary mechanism for controlling what happens is **modifying intents**, not modifying artifacts directly. Redirecting implementation means adding or changing an intent — and the system figures out what that implies for artifacts.

---

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────┐
│                    PARTICIPANTS                       │
│  (humans and AI agents with different roles/scopes)  │
└──────────────────────┬──────────────────────────────┘
                       │ state intents, observe, resolve conflicts
                       ▼
┌─────────────────────────────────────────────────────┐
│               INTENT MANAGEMENT                      │
│                                                      │
│  • Intent store (what we want)                       │
│  • Consistency engine (conflict detection)           │
│  • Attention router (who needs to know what)         │
│  • Progress tracker (satisfaction state per intent)  │
└──────────────┬───────────────────┬──────────────────┘
               │                   │
               ▼                   ▼
┌──────────────────────┐  ┌───────────────────────────┐
│   KNOWLEDGE BASE     │  │    EXECUTION ENGINE        │
│                      │  │                            │
│  • Discoveries       │  │  • Artifact production     │
│  • Failed approaches │  │  • Hypothesis/experiment   │
│  • Constraints found │  │  • Verification            │
│  • Refined models    │  │  • (pluggable per domain)  │
└──────────┬───────────┘  └─────────────┬─────────────┘
           │                            │
           │    ┌───────────────────┐   │
           └───►│    ARTIFACTS      │◄──┘
                │                   │
                │  Code, documents, │
                │  deployments,     │
                │  reports, etc.    │
                └───────────────────┘
```

### Core IMS Functions

**1. Consistency maintenance.** Given intents within a scope, detect:
- *Conflicts* — two intents that cannot both be satisfied simultaneously
- *Tensions* — two intents that trade off against each other (both satisfiable but at increased cost)
- *Gaps* — implied intents that aren't stated (discovered when sub-intents leave the parent partially covered)

**2. Progress tracking.** For each intent, report its lifecycle state. Aggregate up: a parent intent's progress is derived from its sub-intents. Present at whatever granularity the participant asks for.

**3. Attention routing.** Not everything needs human action. The IMS triages and escalates:
- Conflicts → route to someone with authority to resolve
- Ambiguities → route to the intent author for clarification
- Stalled progress → route to whoever is responsible for the blocked work
- Discoveries that affect intents → route to the intent owner

**4. Conflict resolution.** When a conflict is detected:
1. **Assess** — what are the resolution options and their costs?
2. **Route** — who can make this decision? (determined by scope of the conflicting intents and the nature of the conflict)
3. **Resolve** — decision is made, intents are modified, work unblocks

Resolution authority follows the scope hierarchy. If both intents are in the same scope, the scope owner resolves. If they're in different scopes, it escalates to the lowest common ancestor. Some conflicts are technical (an engineer decomposes differently). Some are value-based (a manager or architect decides). Some are economic (a cost function or budget owner decides).

### Relationship to Categorization

The ten intent categories (Invariant, Architecture, Capability, Process, Safety, Optimization, Extensibility, Ergonomics, Epistemics, Aspiration) inform how the IMS operates:

- **Conflict detection** is category-aware. An Invariant conflicting with an Optimization is almost always resolved in favor of the Invariant. Two Optimizations in tension require explicit prioritization. A Safety intent conflicting with an Ergonomics intent needs a risk assessment.

- **Satisfaction criteria** differ by category. Invariants are binary (holds or doesn't). Optimizations are directional (better or worse). Aspirations may be satisfied by being converted into active intents of another category.

- **Attention routing** uses category to determine urgency. A blocked Safety intent is more urgent than a blocked Optimization. A conflict involving an Invariant is higher priority than one involving an Aspiration.

- **Decomposition patterns** are category-typical. Capabilities decompose into sub-capabilities and enabling Architecture. Process intents decompose into gates and checkpoints. Invariants rarely decompose — they either hold or they don't.

---

## Participants

Participants are humans or AI systems that interact with the IMS. They differ in:
- **Scope of authority** — what scopes they can create/modify intents in
- **Depth of concern** — whether they operate at high-level goals or concrete mechanisms
- **Nature of contribution** — whether they state intents, implement toward them, verify satisfaction, or resolve conflicts

### Roles

| Role | Scope of authority | Primary actions | Cares about |
|------|-------------------|-----------------|-------------|
| **Strategist** | Org / Project | State high-level intents, resolve cross-project conflicts, set priorities | Direction, consistency across projects, cost |
| **Architect** | Project / Repo | Decompose intents, detect structural conflicts, propose resolution options | Feasibility, coherence, tradeoffs |
| **Implementer** | Repo | Produce artifacts toward satisfaction, discover constraints, propose sub-intents | Clarity of intent, freedom to choose mechanism |
| **Verifier** | Repo / Project | Confirm satisfaction, challenge claims, define "done" concretely | Criteria, evidence, edge cases |
| **AI Agent** | Delegated | Execute toward assigned intents, respect inherited constraints, report discoveries | Active intents that constrain its work, what it must not violate |

These are roles, not people — one person may hold multiple roles. An AI agent may act as implementer or verifier depending on delegation.

### What Participants Need from the IMS

| Need | How the IMS serves it |
|------|----------------------|
| Understand current state | Progress dashboard at appropriate granularity |
| Focus on what matters | Attention routing — surface conflicts, blocks, decisions needed |
| Drill down when curious | Navigate intent → sub-intents → artifacts → learning |
| Redirect implementation | Modify or add intents; the system propagates implications |
| Know what's blocked and why | Conflict graph with resolution options and costs |
| Trust the system is consistent | Continuous consistency checking; violations surfaced immediately |
| Communicate across roles | Intents rendered at appropriate abstraction for each audience |

---

## Intent Categories

### 1. Invariant

Hard guarantees that must hold unconditionally. A violation means the system is broken — not degraded, not suboptimal, but incorrect. Invariants are binary: they hold or they don't.

Invariants are testable as assertions. They constrain all implementations and cannot be traded away for performance, convenience, or schedule.

### 2. Architecture

Structural properties of how the system is organized. Architecture intents describe the shape of the system — what depends on what, where boundaries exist, how information flows. They are about *form*, not behavior.

Architecture intents are stable over time. Changing them is a deliberate, high-cost decision that cascades through the system.

### 3. Capability

What the system does — the features and behaviors it provides to its users. Capabilities describe observable functionality from outside the system boundary.

Capabilities are the primary axis along which systems grow. They accumulate over time and represent the delivered value of the project.

### 4. Process

How work is performed — by humans developing the system, by operators running it, or by the system itself when it orchestrates multi-step workflows. Process intents describe sequences, gates, reviews, and workflows.

Process intents are the most social of the categories. They encode team agreements about how to collaborate and maintain quality.

### 5. Safety

Constraints that prevent harm — to data, to users, to other systems, or to the correctness of results. Safety intents are defensive: they describe what the system must *not* do, or what it must guarantee even under adversarial or unexpected conditions.

Safety intents overlap with Invariants but are distinguished by *motivation*: invariants preserve internal consistency; safety intents prevent external harm.

### 6. Optimization

Desired properties without hard pass/fail criteria. Better is better. Optimization intents describe directions rather than thresholds — reduce cost, improve speed, minimize waste — without creating a binary success/failure test.

Optimization intents are the most likely to conflict with each other and require explicit prioritization.

### 7. Extensibility

How the system accommodates growth and change. Extensibility intents describe where new capability can be added without modifying existing code, how interfaces are designed for pluggability, and where seams exist for future variation.

Extensibility intents are architectural in character but forward-looking: they describe capacity for change rather than current structure.

### 8. Ergonomics

Properties that serve the human operator or consumer. Ergonomics intents describe the experience of using the system — clarity of output, sensibility of defaults, forgiveness of errors, discoverability of features.

Ergonomics intents are defined by the human's perspective, not the system's. They may conflict with implementation simplicity.

### 9. Epistemics

How the system reasons about the validity of its own claims. Epistemics intents describe standards of evidence, falsifiability requirements, independence of measurement, and the distinction between correlation and causation.

Epistemics intents are most relevant to systems that make empirical claims — about performance, about correctness, about the effect of changes. They apply wherever the system says "X is true" and someone might ask "how do you know?"

### 10. Aspiration

Desired future states that are not yet achieved. Aspirations describe where the project is heading — capabilities planned but not built, properties desired but not yet enforced, directions that guide prioritization without constraining current behavior.

Aspirations are the most volatile category. They are refined, achieved, or abandoned over time.

---

## Examples

Each example synthesizes a single statement from how the intent manifests across projects — elevated to the org or project level rather than quoted per-repo.

### Invariant

**No silent loss.** The system never discards information, swallows errors, or proceeds past a failure without an explicit record. Every input is accounted for; every failure is surfaced. Whether the unit of accounting is a request, an iteration, or a subprocess invocation, nothing disappears without a trace.

**Atomic state transitions.** Persistent state moves from one valid state to another with no observable intermediate. A crash at any point during a write leaves the previous valid state intact. The mechanism may vary (temp-file + rename, single API call, transactional write) but the guarantee is the same: no partial state is ever visible.

### Architecture

**Separation of orchestration and intelligence.** Control flow is separated from domain logic. The orchestration layer is deterministic, auditable, and testable in isolation. Domain intelligence — whether AI reasoning, simulation physics, or algorithm translation — is invoked through well-defined dispatch points, never entangled with the control machinery. The orchestrator's job is to make domain work reliable and resumable, not to perform it.

**Self-contained units of work.** The unit of work carries everything needed to understand, reproduce, or audit it. No external context is required to make sense of a completed run. Configuration, inputs, outputs, and metadata live together.

### Capability

**Hypothesis-driven experimentation.** The system supports forming falsifiable claims, designing controlled experiments to test them, executing under isolation, comparing predictions to outcomes, and extracting reusable learning. Both confirmed and refuted results produce value.

**Automated translation between representations.** The system bridges representations — from simulation to production, from algorithm to deployment artifact — with AI assistance, human review gates, and structured intermediate formats.

### Process

**Plan before implement.** The plan is a separate artifact from the implementation. Work begins with understanding and design; code follows only after the approach is reviewed. The specific ceremony varies (design docs, issues, plan files) but the commitment is the same: think before doing.

**Automated quality gates block merge.** Lint, tests, and validation run on every change. Failures prevent integration. The gate is mechanical and unskippable — not advisory.

### Safety

**Human judgment preserved at decision points.** The system automates mechanical work but requires explicit human approval where judgment, domain expertise, or risk assessment is needed. Automation never silently makes decisions that belong to humans. The enforcement mechanism (hard gate, review protocol, approval workflow) varies, but bypass is never the default.

**Secrets never appear in persistent artifacts.** Credentials, API keys, and tokens flow through environment variables or dedicated secret stores — never through version-controlled files, logs, or error messages.

### Optimization

**Minimize wasted compute.** The system should detect early when work will fail or is unnecessary, and avoid spending resources on it. Fast-fail rules, capacity-aware scheduling, staleness detection, and early stopping all serve this goal. The specific resource (GPU-hours, LLM tokens, CI minutes) varies by project.

**Fast feedback loops.** The test suite and validation pipeline should run in seconds to low minutes, not hours. Speed of feedback directly affects development velocity. If the system is CPU-only or locally testable, there is no excuse for a slow loop.

### Extensibility

**Pluggable implementations behind stable interfaces.** Policy is separated from mechanism. New implementations (algorithms, dispatch backends, translation strategies) can be added without modifying orchestration or control flow. The coupling mechanism (Go interfaces, Python class inheritance, file-based boundaries) is chosen per project, but the property is the same: swap without surgery.

**Versioned contracts for forward compatibility.** External-facing schemas and manifests carry version identifiers. New versions can introduce new semantics without breaking tooling that understands older versions. Evolution is deliberate, not accidental.

### Ergonomics

**Sensible defaults minimize required configuration.** The system works out of the box with minimal input. Complexity is opt-in. The common case requires only the essential inputs; everything else has a reasonable default. A new user should be productive before understanding every option.

**Errors are sentences, not stack traces.** User-facing errors communicate what went wrong in plain language. Implementation details (tracebacks, internal state dumps) indicate a bug in error handling, not a useful diagnostic. Operators should be able to act on errors without reading source code.

### Epistemics

**Claims must be falsifiable.** Every assertion the system makes about behavior — predictions, performance comparisons, parity validations — must have a measurable success/failure threshold defined before the test runs. An experiment that cannot possibly fail produces no information.

**Measurement must be independent of the thing measured.** The detector and the ground truth must be different instruments. When the system validates its own output, the validation mechanism must not be derived from the generation mechanism. Tautological validation is worse than no validation — it creates false confidence.

### Aspiration

**Parallel execution of independent work.** Work units that don't depend on each other (experiment arms, namespace slots, test cases) should execute concurrently. Sequential execution of independent work wastes wall-clock time proportional to the parallelism available.

**Declarative over imperative orchestration.** The system should converge toward describing *desired state* rather than prescribing *steps to get there*. Declarative orchestration simplifies recovery (just reconcile) and makes the system self-healing rather than requiring manual intervention after failures.

---

## Scope Hierarchy

Intents bind at different levels of organizational structure. The hierarchy is:

```
Org  ⊃  Project  ⊃  Repo  ⊃  Individual
```

Each level inherits intents from its parent. A repo inherits all project intents, which in turn inherits all org intents. Exceptions must be explicit and justified.

### Org

Organization-wide statements of intent that apply to all projects under the umbrella unless an explicit exception is granted. Org intents express shared engineering culture — values so fundamental that violating them in any project would be considered a defect in judgment, not a local tradeoff.

**Characteristics:**
- Stable over years, not quarters
- Changed through deliberate cultural decisions, not project pressure
- Express *why* and *what*, never *how* (implementation is always local)
- Exceptions require explicit justification visible to the org

**Examples:** No silent loss. Secrets never in version control. Human judgment preserved at decision points.

### Project

Intents that apply to all work within a project — across its repos, contributors, and operational environments — but not necessarily to other projects in the org. Project intents encode the specific engineering discipline that this project has adopted based on its domain, risk profile, and history.

**Characteristics:**
- Shaped by domain constraints and learned failures
- Apply to all repos within the project equally
- May be stricter than org intents (never weaker)
- Reflect the project's specific tradeoff decisions

**Examples:** Byte-identical determinism (BLIS). Hypothesis falsifiability (Nous). Per-pair cost derivation (sim2real).

### Repo

Intents that apply to a specific repository — its code, its artifacts, its CI pipeline. Repo intents are the most concrete: they often specify mechanisms, name specific files, and reference implementation details. A project with multiple repos may have different repo-level intents for each (e.g., stricter performance intents for the hot path, relaxed ergonomics intents for internal tooling).

**Characteristics:**
- Reference specific files, interfaces, and mechanisms
- Directly testable and enforceable in CI
- May specialize project intents to the local context
- Can only strengthen inherited intents, not weaken them

**Examples:** `state.json` written via temp+rename (Nous). Simulation clock never decreases (BLIS). ConfigMap is authoritative over local files (sim2real).

### Individual

Personal intents that govern how a single contributor interacts with the system. Individual intents describe preferences, working style, and personal quality standards that augment (but cannot weaken) the inherited intents from above.

**Characteristics:**
- Defined by the individual, not imposed by the org
- Cannot override org/project/repo intents
- May add stricter personal standards
- Govern interaction style, tooling preferences, review thoroughness

**Examples:** "A question does not mean fix it — investigate but don't change." Preferred review depth. Personal testing standards beyond CI requirements.

---

## Category × Scope Affinity

Each category has natural affinities with certain scopes — levels where it most commonly and naturally appears. This doesn't mean a category *cannot* appear at other levels, but that it requires more justification or takes a different character when it does.

| Category | Org | Project | Repo | Individual |
|----------|-----|---------|------|------------|
| **Invariant** | Rare | Natural | Most natural | No |
| **Architecture** | Unusual | Natural | Most natural | No |
| **Capability** | No | Natural | Natural | No |
| **Process** | Very natural | Natural | Less natural | Natural |
| **Safety** | Very natural | Natural | Natural | Weak |
| **Optimization** | Weak | Natural | Natural | Weak |
| **Extensibility** | No | Natural | Most natural | No |
| **Ergonomics** | Possible | Natural | Natural | Very natural |
| **Epistemics** | Natural | Most natural | Weak | Possible |
| **Aspiration** | Natural | Natural | Weak | Possible |

### Commentary

**Org-natural categories: Process, Safety, Epistemics**

These are culture-level concerns. An org's process intents say "we do code review" without specifying the tool. An org's safety intents say "no secrets in version control" without specifying the scanning mechanism. An org's epistemics intents say "claims require evidence" without specifying what counts as evidence in each domain.

The common thread: org-level intents describe *how we behave* — they are about people and judgment, not about systems and mechanisms.

**Repo-natural categories: Invariant, Architecture, Extensibility**

These are system-level concerns. Invariants describe properties of running code. Architecture describes the structure of a specific codebase. Extensibility describes where a specific system can grow.

The common thread: repo-level intents describe *what the artifact is* — they are about the system's properties, testable against the code itself.

**Individual-natural categories: Process, Ergonomics**

These are personal concerns. An individual's process intents describe how they prefer to work ("investigate before changing"). Their ergonomics intents describe how they prefer to be communicated with ("terse output, no trailing summaries").

The common thread: individual-level intents describe *how I interact* — they govern the interface between the person and the system/team.

**Project as the versatile middle**

Project scope accommodates nearly every category naturally. This is because a project is the level where domain meets implementation — abstract enough to express principles, concrete enough to enforce them. When in doubt about where an intent belongs, project is often the right default.

### How character changes across scopes

The same category takes on different character at different scopes:

| Category | At Org scope | At Repo scope |
|----------|-------------|---------------|
| Safety | Policy ("no secrets in VCS") | Implementation constraint ("validate all LLM output before exec") |
| Process | Cultural norm ("plan before implement") | CI enforcement ("lint must pass before merge") |
| Invariant | Rare — org "invariants" are really policies | Concrete — names files, specifies mechanisms |
| Epistemics | Cultural value ("claims require evidence") | Methodology ("hypotheses must be falsifiable") |

The org version is governance. The repo version is engineering. The project version bridges the two.

---

## Inheritance and Exceptions

The scope hierarchy follows strict inheritance with explicit exception:

1. **Inheritance is automatic.** A repo inherits all project intents. A project inherits all org intents. No action required.

2. **Strengthening is free.** Any level can add stricter intents than its parent. BLIS requiring byte-identical determinism is stricter than an org intent of "reproducibility" — no exception needed.

3. **Weakening requires explicit exception.** If a repo needs to violate a project intent, or a project needs to violate an org intent, the exception must be documented with:
   - What intent is being weakened
   - Why the exception is justified
   - What compensating controls exist
   - When the exception should be revisited

4. **Exceptions are visible upward.** An exception at repo level should be visible at project level. An exception at project level should be visible at org level. Hidden exceptions defeat the purpose of the hierarchy.

---

## Open Questions

### Taxonomy
- Should the category list be closed (these 10, no more) or open (new categories can be proposed)?
- Is "Consistency" (from sim2real) a separate category or a sub-concern of Invariant?
- Is "Resilience" (from sim2real) a separate category or a sub-concern of Safety?

### Lifecycle
- Is the Draft → Active → Satisfied → Verified progression the right set of states, or is something missing/redundant?
- How long can an intent remain in "Evolving" before it needs to be split or withdrawn?
- Should Aspirations have an explicit expiry or review date?
- When an intent is satisfied, what triggers re-verification? (code changes, dependency changes, time?)

### Conflict Resolution
- What is the priority ordering across categories? (Invariant > Safety > ... > Aspiration? Or is it context-dependent?)
- When two intents in the same scope conflict, what's the escalation path if the scope owner can't resolve?
- How are cost assessments for resolution options represented? (TBD)
- Can an AI agent propose conflict resolutions, or only surface them?

### Knowledge Base
- What is the boundary between "knowledge base" and "intent store"? Is a discovered constraint an intent or a fact?
- How does learning from one project propagate to others? (Through org-level intent evolution? Through shared knowledge base?)
- What's the retention policy for failed approaches? (Forever? Until superseded?)

### Multi-User
- How are concurrent edits to the same intent handled? (Locking? Merge? Last-write-wins?)
- Can participants see each other's draft intents before they become active?
- How does the IMS render the same intent differently for different roles without losing fidelity?

### Execution
- How tightly coupled is the execution engine to the IMS? (Fully integrated? Loosely coupled via APIs? Completely separate with human bridge?)
- Can the execution engine autonomously create sub-intents, or only propose them?
- What's the feedback mechanism from execution failure back to intent evolution?
