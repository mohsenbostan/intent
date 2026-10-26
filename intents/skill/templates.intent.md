---
id: skill.templates
parent: skill
owner: mohsenbostan
why: Authors start from a shape that is already valid, so lint only finds real mistakes.
level: L2
status: draft
version: 1
---

# Templates

## Context
The skill ships templates for parent (L0/L1), leaf (L2) and L3 intents, plus an
example index file. Placeholders look like `<owner>`.

## Behaviour
```yaml
- id: filled-templates-lint-clean
  given: each intent template with every placeholder filled in and saved at a path matching its id, next to a filled parent
  when: lint runs
  then: it exits 0 with no errors
  examples:
    - { templates: [parent as L0, parent as L1, leaf, l3], expected_exit: 0 }
  verify: [example]
  severity: major

- id: index-template-loads
  given: the example index file copied into a repository's intents folder
  when: lint runs
  then: it loads without a usage error and its change budget is 10 added and 20 changed rules
  examples:
    - { file: "the example index file", expected_exit_not: 2 }
  verify: [example]
  severity: minor

- id: new-intents-start-as-draft
  given: any intent template
  when: it is used as written
  then: its status is draft and it carries no reviewer
  verify: [example]
  severity: major
```
