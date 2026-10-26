---
id: skill.lint.hierarchy.budget-units
parent: skill.lint.hierarchy
refines: skill.lint.hierarchy#budgets-only-tighten
owner: mohsenbostan
why: Budget comparisons stay correct when parent and child write the same limit in different units.
level: L3
status: draft
version: 1
---

# Budget units and direction

## Behaviour
```yaml
- id: time-units-normalised
  given: a parent latency budget and a child latency budget written in different time units
  when: lint runs
  then: both are compared as the same duration
  examples:
    - { parent: 800ms, child: 1s, expected: I007 }
    - { parent: 2s, child: 1500ms, expected: none }
  verify: [example]
  severity: major

- id: higher-is-tighter
  given: availability or throughput budgets on a parent and a child
  when: lint runs
  then: a lower child value is reported as I007 and a higher one passes
  examples:
    - { parent: "99.9% / 30d", child: "99.5% / 30d", expected: I007 }
    - { parent: "99.9% / 30d", child: "99.95% / 30d", expected: none }
  verify: [example]
  severity: major

- id: incomparable-units
  given: a parent and child budget with the same name but units or windows that cannot be compared
  when: lint runs
  then: it reports I007 saying the units differ
  examples:
    - { parent: 200rps, child: 200ms, expected: I007 }
    - { parent: "99.9% / 30d", child: "99.9% / 7d", expected: I007 }
  verify: [example]
  severity: minor

- id: comparators-and-currency
  given: budgets written with a leading comparator or a currency sign
  when: lint runs
  then: the comparator is ignored and the currency counts as the unit
  examples:
    - { parent: "< $0.002", child: "< $0.001", expected: none }
    - { parent: "< 0.1%", child: "< 0.5%", expected: I007 }
  verify: [example]
  severity: minor
```
