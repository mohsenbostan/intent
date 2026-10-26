---
id: skill.lint.verification
parent: skill.lint
owner: mohsenbostan
why: Every rule names how it will be verified, so nothing approved is silently unverifiable.
level: L2
status: draft
version: 1
---

# Lint: verification (I005, I006, I012)

## Behaviour
```yaml
- id: known-adapters
  given: a rule whose verify list is empty, missing, or names an adapter the index file does not configure
  when: lint runs
  then: it exits 1 and reports I005 naming the rule
  examples:
    - { verify: [], expected: I005 }
    - { verify: [fuzz], configured: "spec A6 defaults", expected: I005 }
    - { verify: [property], configured: "adapters: [example]", expected: I005 }
    - { verify: [property], configured: "no index file", expected: none }
  verify: [example]
  severity: critical

- id: critical-needs-examples
  given: a behaviour rule with severity critical and no examples
  when: lint runs
  then: it exits 1 and reports I006 naming the rule
  examples:
    - { severity: critical, examples: none, expected: I006 }
    - { severity: major, examples: none, expected: none }
  verify: [example]
  severity: critical

- id: manual-is-debt
  given: a rule verified by the manual adapter
  when: lint runs
  then: it reports I012 unless the rule or its intent has an owner and an expires date that has not passed
  examples:
    - { verify: [manual], rule_expires: none, intent_expires: none, expected: I012 }
    - { verify: [manual], rule_owner: alice, rule_expires: "today + 30 days", expected: none }
    - { verify: [manual], rule_expires: "yesterday", expected: I012 }
  verify: [example]
  severity: major
```
