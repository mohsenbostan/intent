---
id: skill.lint.governance
parent: skill.lint
owner: mohsenbostan
why: Intents stay few, readable and about outcomes as the repository grows.
level: L2
status: draft
version: 1
---

# Lint: governance (I011, I014, I015)

## Behaviour
```yaml
- id: implementation-leak-warning
  given: a behaviour rule whose given, when or then mentions a camel-case or Pascal-case identifier, a function call, a file path or a well-known library
  when: lint runs
  then: it reports an I011 warning quoting the suspicious text, without failing
  examples:
    - { then: "OrderService saves the row", expected: I011 }
    - { when: "authorizePayment() is called", expected: I011 }
    - { then: "the value is cached in redis", expected: I011 }
    - { then: "src/payments/charge.ts returns 200", expected: I011 }
    - { then: "exactly one charge exists at the provider", expected: none }
  verify: [example]
  severity: major

- id: change-budget
  given: a working tree compared with a base ref, and the change budget from the index file (default 10 added and 20 changed or removed rules)
  when: lint runs with that base
  then: exceeding either limit reports I014 unless the override label is passed
  examples:
    - { max_added_rules: 1, rules_added: 2, labels: [], expected: I014 }
    - { max_added_rules: 1, rules_added: 2, labels: [intents-change-budget-override], expected: none }
    - { max_added_rules: 1, rules_added: 1, labels: [], expected: none }
    - { base: none, rules_added: 50, expected: none }
  verify: [example]
  severity: major

- id: duplicate-warning
  given: two behaviour rules in different files whose given, when and then are nearly identical
  when: lint runs
  then: it reports an I015 warning naming the other rule, without failing
  examples:
    - { rule_a: "given an order / when paid / then a receipt is emailed", rule_b: "given an order / when paid / then a receipt is e-mailed", expected: I015 }
  verify: [example]
  severity: minor
```
