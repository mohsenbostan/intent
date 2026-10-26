---
id: skill.workflows
parent: skill
owner: mohsenbostan
why: The agent following the skill keeps the separation of authority that makes the checks trustworthy.
level: L2
status: draft
version: 1
expires: 2027-04-26
---

# Agent workflows

## Context
These rules describe what the agent does when it follows SKILL.md (new, lint,
review, checks, mutate, implement, gate, prune). Agent behaviour is not yet
automatically verifiable, so every rule here is `manual` and counts as debt
(spec A6). The gate's mixed-change rule backs up `implement-never-edits-spec`
mechanically.

## Behaviour
```yaml
- id: never-self-approves
  given: an intent ready for review
  when: the agent runs the review workflow
  then: it shows the rule-level diff and the review checklist and leaves status, reviewed_by and reviewed_at for a human to set
  verify: [manual]
  severity: critical
  examples:
    - { agent_sets_reviewed_by: false }

- id: check-writer-is-isolated
  given: approved intents that need checks
  when: the agent runs the checks workflow
  then: checks are written by a separate subagent given only the intents and the public interface, which never reads implementation source
  verify: [manual]
  severity: critical
  examples:
    - { subagent_inputs: "intents + public interface", implementation_read: false }

- id: implement-never-edits-spec
  given: a coding task under the implement workflow
  when: the agent finds an intent that looks wrong
  then: it stops and proposes an intent change instead of editing intents or checks
  verify: [manual]
  severity: critical
  examples:
    - { intent_looks_wrong: true, expected: "proposal, no edit under intents/ or checks/" }

- id: mutations-reverted
  given: targeted mutation testing of changed rules
  when: the agent runs the mutate workflow
  then: it reports surviving mutants per rule and leaves the working tree as it found it
  verify: [manual]
  severity: major

- id: prune-asks-first
  given: orphaned, expired, unverified or duplicate intents
  when: the agent runs the prune workflow
  then: it proposes deletions and deletes nothing without the user's confirmation
  verify: [manual]
  severity: major
```
