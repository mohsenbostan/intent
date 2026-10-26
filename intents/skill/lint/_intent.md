---
id: skill.lint
parent: skill
owner: mohsenbostan
why: Malformed or unverifiable Intents are caught by a machine before a human spends time reviewing them.
level: L1
status: draft
version: 1
---

# Lint

## Context
Lint implements the deterministic rules of spec A10 (I001–I014) plus the A9
duplication warning (I015). Each leaf below owns a group of codes.

## Behaviour
```yaml
- id: clean-repo-passes
  given: a repository whose Intents break no lint rule, such as the complete example in spec A11 with its parents
  when: lint runs
  then: it exits 0 and reports zero errors and zero warnings
  examples:
    - { tree: "spec A11 example + its L0 and L1 parents", expected_exit: 0, expected_findings: 0 }
  verify: [example]
  severity: critical

- id: findings-are-located
  given: a repository with violations in several intent files
  when: lint runs with --json
  then: every finding carries its code, level, the repo-relative path of the offending file and a one-line message
  examples:
    - { broken_files: 2, expected_paths: "the 2 broken files, /-separated" }
  verify: [contract]
  severity: major

- id: readable-by-default
  given: a repository with violations
  when: lint runs without --json
  then: findings are printed as text grouped by file, followed by a total of errors and warnings
  verify: [example]
  severity: minor

- id: warnings-do-not-fail
  given: a repository whose only findings are warnings
  when: lint runs
  then: it exits 0 and still prints the warnings
  examples:
    - { findings: "one I011 warning", expected_exit: 0 }
  verify: [example]
  severity: major

- id: no-intents-directory
  given: a directory with no intents folder
  when: lint runs on it
  then: it exits 2 with a message on stderr
  examples:
    - { root: "empty directory", expected_exit: 2 }
  verify: [example]
  severity: minor
```

## Out of scope
```yaml
- semantic contradictions between parent and child (an agent review, spec A5)
```
