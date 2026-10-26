---
id: skill.lint.approval
parent: skill.lint
owner: mohsenbostan
why: Nothing gates a merge unless a human has approved it and it is still current.
level: L2
status: draft
version: 2
---

# Lint: approval (I008, I009, I010)

## Behaviour
```yaml
- id: human-reviewer-required
  given: an approved intent at level L0, L1 or L2
  when: lint runs
  then: it reports I008 unless both reviewed_by and reviewed_at are set; approved L3 intents need neither
  examples:
    - { level: L2, status: approved, reviewed_by: none, expected: I008 }
    - { level: L2, status: draft, reviewed_by: none, expected: none }
    - { level: L3, status: approved, reviewed_by: none, expected: none }
  verify: [example]
  severity: critical

- id: no-open-questions
  given: an intent with an Open questions section, in any status and with any content
  when: lint runs
  then: it exits 1 and reports I009 telling the author to ask the user and record the answer as a rule
  examples:
    - { status: approved, open_questions: "Do refunds need approval?", expected: I009 }
    - { status: draft, open_questions: "Do refunds need approval?", expected: I009 }
    - { status: draft, open_questions: "(empty section)", expected: I009 }
    - { status: draft, open_questions: "(no such section)", expected: none }
  verify: [example]
  severity: critical

- id: expired-intents-fail
  given: an intent whose expires date is in the past
  when: lint runs
  then: it exits 1 and reports I010
  examples:
    - { expires: "yesterday", expected: I010 }
    - { expires: "tomorrow", expected: none }
  verify: [example]
  severity: major
```
