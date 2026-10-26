---
id: skill.lock
parent: skill
owner: mohsenbostan
why: A check only counts as verification if it is exactly the check a human reviewed, for exactly the rule they approved.
level: L2
status: draft
version: 2
---

# Pinned checks (lock)

## Context
Implements spec A2.4 (determinism after generation) and the pinning step of A7.
Check file naming and the lock format are in `interface/cli.md`. The owner
decided (2026-10-26) that every file under `checks/` is pinned, helpers included.

## Behaviour
```yaml
- id: pin-then-verify
  given: approved intents with a check for every non-manual adapter of every rule
  when: lock write runs and then lock check runs
  then: the lock file records a hash for every approved rule and every check file, and lock check exits 0
  examples:
    - { approved_rules: 3, checks_per_rule: 1, expected_check_exit: 0 }
  verify: [example]
  severity: critical

- id: rule-changed-after-pinning
  given: a pinned rule whose given, when, then, examples or verify list is edited afterwards
  when: lock check runs
  then: it exits 1 and names that rule as changed since its checks were pinned
  examples:
    - { edit: "then text of a pinned rule", expected_exit: 1 }
    - { edit: "Context prose only", expected_exit: 0 }
  verify: [example]
  severity: critical

- id: check-changed-after-pinning
  given: a pinned check file whose content is edited, deleted, or joined by a new unpinned check for the same rule
  when: lock check runs
  then: it exits 1 and names the rule and the file
  examples:
    - { change: "edit a pinned check", expected_exit: 1 }
    - { change: "delete a pinned check", expected_exit: 1 }
    - { change: "add an unpinned check", expected_exit: 1 }
  verify: [example]
  severity: critical

- id: missing-check
  given: an approved rule whose verify list names an adapter with no check file
  when: lock check runs
  then: it exits 1 naming the rule and adapter; the manual adapter never needs a check file
  examples:
    - { verify: [example, property], check_files: [example], expected_exit: 1 }
    - { verify: [example, manual], check_files: [example], expected_exit: 0 }
  verify: [example]
  severity: major

- id: orphans-and-stale-entries
  given: a check file for a rule that no longer exists, or a lock entry for a rule that is no longer approved
  when: lock check runs
  then: it exits 1 naming the rule
  verify: [example]
  severity: minor

- id: drafts-need-no-checks
  given: a draft or deprecated intent with no checks
  when: lock check runs
  then: that intent produces no problems
  verify: [example]
  severity: major

- id: helpers-are-pinned
  given: a helper file under the checks folder (a fixture, conftest or shared utility) that is edited, added or deleted after pinning
  when: lock check runs
  then: it exits 1 naming the file, so a helper can never silently change what a pinned check does
  examples:
    - { change: "edit a pinned conftest", expected_exit: 1 }
    - { change: "add a new fixture file", expected_exit: 1 }
    - { change: "add a file git ignores, such as a bytecode cache", expected_exit: 0 }
  verify: [example]
  severity: critical

- id: line-endings-do-not-matter
  given: a pinned check file whose line endings change between LF and CRLF with no other change
  when: lock check runs
  then: it exits 0
  examples:
    - { pinned_with: LF, checked_with: CRLF, expected_exit: 0 }
  verify: [example]
  severity: minor
```
