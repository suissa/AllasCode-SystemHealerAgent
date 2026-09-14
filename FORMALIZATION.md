# Formalization of the AllasCode SystemHealerAgent

## 1. Purpose

The `SystemHealerAgent` is the self-healing component responsible for correcting failures whose causal mechanism lies in **runtime/system configuration**, not in domain semantics and not in Action implementation code.

The agent is therefore not a general-purpose administrator. It is a **causal configuration intervention agent** operating under a strict capability boundary.

Its only writable artifact is:

```text
configs/core.yml
```

All other artifacts are read-only.

Formally:

```text
WritableSet(SystemHealerAgent) = { configs/core.yml }
```

and:

```text
∀ p ∉ WritableSet(SystemHealerAgent): write(p) = DENIED
```

The runtime SHOULD enforce this at the OS/filesystem capability level rather than relying only on prompt instructions.

---

## 2. Architectural role

The SystemHealerAgent participates in a separated healing architecture:

```text
Observation
  -> CodeManager
  -> HealingHypothesis
      -> code    -> CodeHealerAgent
      -> system  -> SystemHealerAgent
      -> unknown -> diagnostics / no mutation
  -> Verifier
  -> Runtime promotion/rejection
```

Responsibilities are intentionally non-overlapping.

### CodeManager

- read-only;
- receives the complete failed Intent evidence;
- infers a falsifiable causal thesis;
- classifies the fault as `code`, `system`, or `unknown`;
- identifies the suspected configuration dimension when the fault is `system`;
- produces a bounded request to the SystemHealerAgent.

### SystemHealerAgent

- does not invent a new root cause after receiving the immutable thesis;
- checks whether the thesis is compatible with the configuration it can inspect;
- proposes the smallest admissible configuration intervention;
- changes only values in `configs/core.yml` that are declared healable;
- states the predicted effect before the candidate is executed.

### Verifier

- does not heal;
- compares declared predictions with observed effects;
- evaluates hidden integration and acceptance evidence;
- classifies the experiment as `supported`, `falsified`, or `inconclusive`.

### Runtime

- enforces capability boundaries;
- stages candidate configuration in isolation;
- executes validation;
- promotes, rejects, quarantines, or rolls back the candidate.

---

## 3. Fault taxonomy

The CodeManager MUST distinguish three fault classes.

### 3.1 Code fault

The causal mechanism lies inside an Action implementation.

Typical target:

```text
<Action>/implementation.zig
```

This must not be routed to SystemHealerAgent.

### 3.2 System fault

The Action behavior is semantically correct, but runtime behavior becomes incorrect because configuration values produce an invalid operating condition.

Typical classes include:

- retry timing;
- timeout windows;
- queue capacity;
- worker concurrency;
- rate-limit policy;
- circuit-breaker thresholds;
- backoff parameters;
- batching limits;
- cache TTL;
- resource budgets;
- observation cadence;
- snapshot/checkpoint cadence;
- scheduler limits;
- connection pool bounds;
- grace periods;
- lease durations;
- health-check thresholds.

These may be routed to SystemHealerAgent only if the relevant property is declared healable.

### 3.3 Unknown fault

The available evidence does not discriminate between code and system causes.

In that case:

```text
Mutation = forbidden
```

The next action is evidence acquisition, not healing.

---

## 4. Causal healing model

A SystemHealer candidate is not justified by correlation alone.

Let:

```text
O = observed runtime evidence
C = candidate configuration cause
M = causal mechanism
I = configuration intervention
P = predicted observable effect
```

A valid healing thesis has the structure:

```text
O supports C
C explains O through M
I modifies C
M predicts that I changes P
P is observable by independent verification
```

The SystemHealerAgent MAY mutate only if all of the following hold:

```text
SystemFault(O) = true
SpecificConfigCause(O, C) = plausible
Mechanism(C, O) = stated
Intervention(I, C) = bounded
Prediction(I, P) = predeclared
FalsificationCriterion = present
```

If any of these conditions is absent, the intervention must not be promoted as causal healing.

---

## 5. Evidence available to the thesis

The CodeManager may synthesize evidence from the complete failed Intent execution:

```text
E = {
  Intent goal,
  Intent invariants,
  Action descriptions,
  Action preconditions/postconditions,
  emitted events,
  metrics,
  traces/spans,
  logs,
  error message,
  current core configuration,
  previous configuration version,
  resource state,
  execution timing
}
```

The important rule is that system diagnosis must use the **whole execution**, not only the terminal error.

For example, a timeout is not enough to infer that `timeout_ms` is too small. The traces must show that the underlying operation is still making valid progress and that the timeout expires before the operation's normal completion window.

---

## 6. SystemHealingHypothesis

Before any mutation, the following artifact must exist and be immutable for that experiment:

```text
SystemHealingHypothesis
- hypothesis_id
- intent_id
- execution_id
- fault_class = system
- suspected_config_path
- thesis
- mechanism
- evidence_for[]
- evidence_against_or_missing[]
- alternative_causes[]
- causal_chain[]
- falsification_criteria[]
- predicted_intervention_effect
- safety_bounds
- requested_mutation_scope = configs/core.yml
```

The SystemHealerAgent must receive this artifact before it receives write capability.

---

## 7. Configuration mutability model

`configs/core.yml` is the only writable file, but not every property in it is automatically healable.

The configuration model should distinguish:

```text
immutable
operator_mutable
healable
```

Only `healable` properties may be changed autonomously.

Conceptually:

```text
MutableBy(SystemHealerAgent, key)
  iff
key ∈ configs/core.yml
and key.healable = true
and proposed_value ∈ key.safety_bounds
```

Therefore:

```text
WritableFile != UnlimitedSemanticAuthority
```

The runtime must validate both the path and the property-level authorization.

---

## 8. Minimal intervention principle

The SystemHealerAgent should prefer the smallest change capable of testing the thesis.

If a retry failure is attributed to `retry.initial_backoff_ms`, changing retries, timeout, concurrency, queue size, and circuit-breaker thresholds at the same time would destroy causal attribution.

Therefore:

```text
PreferredIntervention = argmin(change_surface)
subject to thesis_testable = true
```

A healing experiment should change one causal dimension at a time whenever operationally safe.

---

## 9. Prediction-before-mutation rule

Before writing the candidate configuration, the SystemHealerAgent must declare:

```text
predicted_effects[]
preserved_invariants[]
expected_metric_changes[]
expected_event_changes[]
risks[]
refutation_conditions[]
```

This prevents post-hoc rationalization.

The Verifier then compares:

```text
PredictedEffect
vs
ObservedEffect
```

Disposition:

```text
supported
falsified
inconclusive
```

---

## 10. Required SystemHealer response contract

Before mutation, the agent should produce:

```text
accepted_thesis:
  <restatement of the immutable system thesis>

configuration_cause:
  <exact property or bounded property set>

causal_explanation:
  <why current values produce the observed behavior>

proposed_intervention:
  <old value -> candidate value or policy transformation>

mutation_scope:
  configs/core.yml only

safety_bounds:
  <permitted min/max or policy constraints>

predicted_effects:
  - <observable consequence>

preserved_invariants:
  - <system/domain property that must remain true>

risks:
  - <possible regression>

verification_predictions:
  - <what hidden integration/acceptance tests should observe>

refutation_condition:
  - <result that would falsify the configuration thesis>
```

Only after this response may the runtime grant the candidate write capability.

---

## 11. Configuration-causality rules

A configuration value may be considered causal only when evidence supports a path from that value to the failure.

### Timing relation

Example structure:

```text
configured timeout = 500 ms
trace shows valid operation duration = 780-840 ms
no downstream semantic error observed
failure occurs exactly at timeout boundary
```

Causal thesis:

```text
Timeout policy terminates a valid operation before its normal completion window.
```

### Capacity relation

Example structure:

```text
queue_capacity = 100
queue_depth reaches 100
rejections begin only at depth 100
consumers remain healthy
```

Causal thesis:

```text
Configured queue capacity is the binding constraint causing admission failure.
```

### Concurrency relation

Example structure:

```text
workers = 64
DB pool = 16
traces show many workers blocked waiting for DB connections
latency rises as active workers exceed pool capacity
```

Causal thesis:

```text
Worker concurrency exceeds the bounded downstream connection resource, producing queueing and timeout amplification.
```

### Retry relation

Example structure:

```text
provider retry-after = 2000 ms
configured retry schedule = 50, 100, 200 ms
all retries receive same 429
Action reports transient rate limit correctly
```

Causal thesis:

```text
Retry policy violates the provider recovery window and deterministically re-enters the same throttled state.
```

### Threshold relation

Example structure:

```text
circuit opens at 3 failures
traffic has expected transient bursts of 3-4 failures
successful requests resume immediately after
```

Causal thesis:

```text
Circuit-breaker threshold is below normal transient burst behavior, converting recoverable noise into system-wide unavailability.
```

---

## 12. Healing invariants

### INV-SYSHEAL-001 — Single writable artifact

```text
WritableSet(SystemHealerAgent) = { configs/core.yml }
```

### INV-SYSHEAL-002 — Property-level authorization

Only properties declared `healable` may be modified.

### INV-SYSHEAL-003 — No Action mutation

The SystemHealerAgent must never modify `implementation.zig`.

### INV-SYSHEAL-004 — No semantic-contract mutation

The SystemHealerAgent must not modify Intents, Actions, schemas, invariants, tests, policies, or behavioral contracts to make the error disappear.

### INV-SYSHEAL-005 — Thesis before mutation

Every autonomous configuration mutation must reference a pre-existing immutable `SystemHealingHypothesis`.

### INV-SYSHEAL-006 — Falsifiability

Every thesis must include at least one observable result that would refute it.

### INV-SYSHEAL-007 — Bounded intervention

Every proposed value must remain inside declared safety bounds.

### INV-SYSHEAL-008 — Independent verification

The entity proposing the configuration mutation cannot promote its own candidate.

### INV-SYSHEAL-009 — No speculative mutation under uncertainty

If evidence cannot distinguish code from system cause, no mutation is permitted.

### INV-SYSHEAL-010 — Causal provenance

Every promoted configuration change must retain:

```text
hypothesis_id
AgentID
ExecutionID
previous config hash
candidate config hash
evidence hash
predictions
observed effects
verifier disposition
```

### INV-SYSHEAL-011 — Disjoint healer authority

```text
WritableSet(CodeHealerAgent) ∩ WritableSet(SystemHealerAgent) = ∅
```

### INV-SYSHEAL-012 — Same Intent

Healing may change system conditions, but must not silently weaken or replace the original Intent.

---

## 13. Runtime enforcement

The Runtime should enforce the boundary through defense in depth:

```text
1. Dedicated OS security principal/UID for the SystemHealerAgent
2. Read-only workspace by default
3. configs/core.yml mounted writable
4. property-level config validator
5. no privilege escalation
6. no write access to Git metadata
7. candidate staged in isolated execution
8. hidden integration and acceptance validation
9. Runtime-controlled commit/promotion
10. rollback to last accepted configuration on rejection
```

Git is provenance and recovery infrastructure, not the primary security boundary.

---

## 14. Verification semantics

Let:

```text
H = SystemHealingHypothesis
C0 = current configuration
C1 = candidate configuration
P(H) = predicted observations
O(C1) = observed candidate-run evidence
```

The verifier computes conceptually:

```text
Support(H, C1) iff
  mutation_scope_valid(C0, C1)
  ∧ safety_bounds_hold(C1)
  ∧ required_invariants_hold(C1)
  ∧ predictions_match(P(H), O(C1))
  ∧ original_failure_class_absent(O(C1))
```

Passing a visible test is not sufficient.

Promotion requires full-flow evidence.

---

## 15. Distinguishing healing from tuning

System healing is not unrestricted auto-tuning.

### Healing

Triggered by a specific failed execution and tied to a causal thesis.

```text
failure -> thesis -> bounded intervention -> verification
```

### Tuning

Optimizes healthy behavior according to objectives such as cost, latency, throughput, or resource utilization.

```text
healthy system -> optimization objective -> search -> fitness evaluation
```

The two mechanisms may share infrastructure, but their evidence and promotion rules must remain distinct.

---

## 16. Knowledge accumulation

Each completed healing experiment should become reusable causal knowledge.

A retained record may encode:

```text
failure signature
context
configuration state
causal thesis
intervention
predicted effect
observed effect
support/falsification
regressions
final disposition
```

Future CodeManager runs can use this as prior evidence, but previous success must not be treated as an unconditional rule. Context remains part of the causal claim.

Therefore:

```text
Past healing result = evidence
not = universal command
```

---

## 17. Summary

The SystemHealerAgent is intentionally powerful only inside a very small mutation surface.

Its design principle is:

> **Maximize causal reasoning while minimizing mutation authority.**

The agent may inspect broad system evidence, but it may write only to `configs/core.yml`, only to authorized healable properties, and only after a predeclared falsifiable thesis connects the observed failure to the proposed configuration intervention.
