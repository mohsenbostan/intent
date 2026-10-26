---
id: skill.diff
parent: skill
owner: mohsenbostan
why: Reviewers approve Intents by reading what changed in the rules, not by rereading whole files.
level: L2
status: draft
version: 1
---

# Rule-level diff

## Context
Implements spec A2.6 and A9 (human load). The Markdown output is meant to be
pasted into a pull request comment.

## Behaviour
```yaml
- id: rule-level-changes
  given: a base ref and a head where intents were added, removed and modified
  when: diff runs
  then: it lists each changed intent once, with its changed front-matter fields as old and new values and its rules as added, removed or changed with old and new values per field
  examples:
    - { change: "then text of rule declined-card edited", expected: "declined-card changed, then: old -> new" }
    - { change: "new intent file", expected: "intent added with all its rules" }
    - { change: "status draft -> approved", expected: "status: draft -> approved" }
  verify: [example, contract]
  severity: critical

- id: unchanged-is-silent
  given: an intent whose only change is free prose
  when: diff runs
  then: that intent is not listed
  verify: [example]
  severity: major

- id: version-bump-flagged
  given: an intent whose rules changed but whose version did not
  when: diff runs
  then: that intent is flagged as needing a version bump
  examples:
    - { rules_changed: 1, version: "3 -> 3", expected: flagged }
    - { rules_changed: 1, version: "3 -> 4", expected: not flagged }
  verify: [example]
  severity: major

- id: l3-summarised
  given: changes to L3 intents
  when: diff runs without --full
  then: L3 changes appear only as counts of intents and rules; with --full or --json they are listed like any other level
  verify: [example]
  severity: minor

- id: working-tree-by-default
  given: uncommitted edits to an intent
  when: diff runs without --head
  then: the uncommitted edits are included
  verify: [example]
  severity: major
```
