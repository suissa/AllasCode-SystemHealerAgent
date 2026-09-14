# AllasCode SystemHealerAgent

The **SystemHealerAgent** is the AllasCode healing agent responsible for correcting **system-level configuration faults** without changing domain code, Action implementations, semantic contracts, tests, schemas, flows, or runtime binaries.

Its writable surface is intentionally minimal:

```text
WritableSet(SystemHealerAgent) = { configs/core.yml }
```

Everything else is read-only from the SystemHealerAgent point of view.

The SystemHealerAgent exists to solve failures whose causal mechanism is a wrong, insufficient, inconsistent, unsafe, or environment-incompatible value in the runtime configuration rather than a defect in an Action implementation.

The core principle is:

> **The SystemHealerAgent may change a system parameter only after a falsifiable causal thesis links observed runtime evidence to a specific configuration value and predicts how changing that value should alter the failed execution.**

It must never mutate configuration merely because a different value makes a visible test pass.

## Separation of authority

```text
CodeManager
  role: infer causal thesis
  write authority: none

CodeHealerAgent
  role: repair Action implementation
  write authority: target Action/implementation.zig only

SystemHealerAgent
  role: repair system configuration
  write authority: configs/core.yml only

Verifier / Runtime
  role: validate predictions, run integration/acceptance tests, promote or reject
  write authority: no healing mutation authority
```

The mutation domains are disjoint:

```text
WritableSet(CodeHealerAgent) ∩ WritableSet(SystemHealerAgent) = ∅
```

This preserves causal attribution: a successful healing experiment can be attributed either to a code intervention or to a configuration intervention, never to a simultaneous mixture of both.

## Scientific healing cycle

```text
Runtime Failure
  -> Events + Metrics + Traces + Logs + Error + Intent topology
  -> CodeManager causal thesis
  -> classify: code | system | unknown
  -> if system: bounded SystemHealingHypothesis
  -> SystemHealerAgent proposes configs/core.yml change
  -> Runtime applies candidate in isolated execution
  -> runtime checks
  -> hidden integration tests
  -> hidden acceptance/conformance tests
  -> observed effect compared with predicted effect
  -> supported | falsified | inconclusive
  -> promote | reject | quarantine
```

The hypothesis must exist before the mutation.

## Formalization and examples

See:

- [`FORMALIZATION.md`](FORMALIZATION.md) — normative model, causal inference contract, invariants, mutation rules and verification semantics.
- [`EXAMPLES.md`](EXAMPLES.md) — concrete system-fault examples showing how events, metrics, traces, logs and error messages are linked to a configuration cause and how the allowed configuration change is selected.

## Central rule

A SystemHealerAgent does not ask:

> "Which configuration value makes the failure disappear?"

It asks:

> "Which configuration value participates in the observed causal chain, what evidence supports that claim, what intervention follows from it, and what result would falsify the thesis?"
