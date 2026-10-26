---
id: skill.lint.structure
parent: skill.lint
owner: mohsenbostan
why: Tooling and reviewers can only trust Intents whose shape is well-formed.
level: L2
status: draft
version: 1
---

# Lint: structure (I001, I002, I004)

## Behaviour
```yaml
- id: front-matter-required
  given: an intent whose front matter is missing a required field or has an invalid value
  when: lint runs
  then: it exits 1 and reports I001 for that file, naming the field
  examples:
    - { problem: "no owner", expected: I001 }
    - { problem: "level: L5", expected: I001 }
    - { problem: "version: 0", expected: I001 }
    - { problem: "status: done", expected: I001 }
    - { problem: "non-root intent without parent", expected: I001 }
    - { problem: "no Behaviour, Budgets or Policies section", expected: I001 }
    - { problem: "front matter is not valid YAML", expected: I001 }
  verify: [example]
  severity: critical

- id: rule-fields-required
  given: a behaviour rule missing given, when, then or severity, or a policy missing its rule text
  when: lint runs
  then: it exits 1 and reports I001 naming the rule and the missing fields
  examples:
    - { rule: "behaviour without then", expected: I001 }
    - { rule: "severity: urgent", expected: I001 }
  verify: [example]
  severity: major

- id: id-matches-path
  given: an intent whose id does not match where its file lives
  when: lint runs
  then: it exits 1 and reports I002 with the id the path implies
  examples:
    - { path: "intents/checkout/payment/authorize.intent.md", id: checkout.authorize, expected: I002, expected_message_contains: checkout.payment.authorize }
    - { path: "intents/checkout/payment/_intent.md", id: checkout.payment, expected: none }
  verify: [example]
  severity: critical

- id: rule-ids-unique
  given: an intent with two rules sharing an id, or a rule id that is not kebab-case
  when: lint runs
  then: it exits 1 and reports I004 for that file
  examples:
    - { rule_ids: [declined-card, declined-card], expected: I004 }
    - { rule_ids: [DeclinedCard], expected: I004 }
  verify: [example]
  severity: major

- id: prose-is-ignored
  given: an intent whose Context and other free prose contain text that looks like rules or YAML
  when: lint runs
  then: only the front matter and the fenced yaml blocks under the known headings affect the result
  examples:
    - { context: "a yaml block with an invalid rule inside the Context section", expected_findings: 0 }
  verify: [example]
  severity: major
```
