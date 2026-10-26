---
id: skill.gate
parent: skill
owner: mohsenbostan
why: Code merges only when every approved Intent it could affect is verified by its pinned checks.
level: L2
status: draft
version: 2
---

# Merge gate

## Context
Implements the merge gate of spec A7. Runs in CI on every pull request,
against the PR's base branch. A code change does not say which intents it
touches, so the owner decided (2026-10-26) that the gate verifies every
approved intent on every change.

## Behaviour
```yaml
- id: all-green-passes
  given: lint has no errors, the lock is valid, every approved intent's checks pass, and the change does not mix intents or checks with other files
  when: the gate runs
  then: it exits 0 and reports every approved intent as pass
  examples:
    - { lint_errors: 0, lock_problems: 0, failing_checks: 0, mixed: false, expected_exit: 0 }
  verify: [example]
  severity: critical

- id: failing-check-blocks
  given: one check of one approved intent fails
  when: the gate runs
  then: it exits 1 and reports that intent as fail with the failing adapter
  examples:
    - { failing_checks: 1, expected_exit: 1, expected_failed_intents: 1 }
  verify: [example]
  severity: critical

- id: lint-or-lock-blocks
  given: lint reports an error, or the lock check reports a problem
  when: the gate runs
  then: it exits 1 and says which part failed
  examples:
    - { lint_errors: 1, expected_exit: 1 }
    - { lock_problems: 1, expected_exit: 1 }
  verify: [example]
  severity: critical

- id: no-mixed-changes
  given: files changed since the merge base include paths under intents or checks and also other paths
  when: the gate runs
  then: it exits 1 listing both groups, unless the mixed-change label is passed
  examples:
    - { changed: ["intents/a.intent.md", "src/app.py"], labels: [], expected_exit: 1 }
    - { changed: ["intents/a.intent.md", "src/app.py"], labels: [intents-mixed-change], expected_exit: 0 }
    - { changed: ["src/app.py"], labels: [], expected_exit: 0 }
    - { changed: ["intents/a.intent.md", "checks/a/r.example.test.py"], labels: [], expected_exit: 0 }
  verify: [example]
  severity: critical

- id: adapters-run-as-configured
  given: approved intents whose checks use adapters with a run command, with run_in_gate set to false, or with no run command
  when: the gate runs
  then: checks run with their adapter's command, run_in_gate false is reported as skipped, and a missing run command fails that intent
  examples:
    - { adapter: example, run: "a command that exits 0", expected: pass }
    - { adapter: slo, run_in_gate: false, expected: skipped }
    - { adapter: load, run: none, expected: fail }
  verify: [example]
  severity: major

- id: checks-every-approved-intent
  given: approved intents whose files and checks are untouched by the change
  when: the gate runs
  then: their checks run too, so every approved intent is verified on every change
  examples:
    - { approved_intents: 3, intents_changed: 0, code_changed: true, expected_intents_run: 3 }
  verify: [example]
  severity: critical

- id: unknown-base
  given: a base ref that does not exist
  when: the gate runs
  then: it exits 2 with a message on stderr
  verify: [example]
  severity: minor
```
