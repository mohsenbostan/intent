---
id: skill.lint.hierarchy
parent: skill.lint
owner: mohsenbostan
why: A child Intent can never quietly weaken what its parent promised.
level: L2
status: draft
version: 1
---

# Lint: hierarchy (I003, I007, I013)

## Behaviour
```yaml
- id: parent-exists
  given: an intent whose parent does not exist, or whose parent chain loops back to itself
  when: lint runs
  then: it exits 1 and reports I003 for that file
  examples:
    - { parent: checkout.missing, expected: I003 }
  verify: [example]
  severity: critical

- id: budgets-only-tighten
  given: a child intent with a budget that an ancestor also sets
  when: lint runs
  then: a looser or unitless child budget is reported as I007 and an equal or tighter one passes
  examples:
    - { parent: { latency_p99: 800ms }, child: { latency_p99: 900ms }, expected: I007 }
    - { parent: { latency_p99: 800ms }, child: { latency_p99: 700ms }, expected: none }
    - { parent: { throughput_min: 200rps }, child: { throughput_min: 100rps }, expected: I007 }
    - { parent: {}, child: { latency_p99: "800" }, expected: I007 }
  verify: [example]
  severity: critical

- id: policies-not-overridden
  given: a child intent that declares a policy with the same id as an inherited policy but different content
  when: lint runs
  then: it exits 1 and reports I007 naming the ancestor
  examples:
    - { parent_policy: "no-card-data-at-rest = no PAN logged", child_policy: "no-card-data-at-rest = PAN may be logged masked", expected: I007 }
  verify: [example]
  severity: critical

- id: l3-refines
  given: an L3 intent that cites no L2 rule, or cites one that does not exist
  when: lint runs
  then: it exits 1 and reports I013 for that file
  examples:
    - { refines: none, expected: I013 }
    - { refines: "checkout.payment.authorize#no-such-rule", expected: I013 }
    - { refines: "checkout.payment.authorize#no-double-charge", expected: none }
  verify: [example]
  severity: major
```
