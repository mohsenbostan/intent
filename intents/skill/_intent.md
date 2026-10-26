---
id: skill
owner: mohsenbostan
why: Teams can accept AI-written code because humans own the outcomes and machines verify them.
level: L0
status: draft
version: 2
tags: [dogfood]
---

# The intents skill

## Context
The `intents` Claude Code skill implements the Intent Specification v0.2
(`spec/intents-spec-and-skill-prompt.md`, Part A). It has deterministic scripts
(lint, diff, lock, gate) and agent workflows described in SKILL.md.

This repository is specified with its own Intents. The public interface the
checks may rely on is `interface/cli.md`.

Policies and budgets here apply to every script below.

## Policies
```yaml
- id: stdlib-and-pyyaml-only
  rule: the scripts depend on nothing but the Python 3.10+ standard library and PyYAML
  verify: [static]

- id: exit-code-contract
  rule: every script exits 0 when nothing blocks, 1 when it finds violations, and 2 on usage or parse errors with a message on stderr
  verify: [contract]

- id: json-output
  rule: every script accepts --json and then prints exactly one JSON document on stdout, in the shape the CLI contract gives
  verify: [contract]

- id: never-edits-intents
  rule: no script changes files under intents/; the only file any script writes is the lock file, and only lock write writes it
  verify: [example]

- id: no-open-questions
  rule: nothing undecided is written down anywhere in the repository; no intent has an Open questions section and no file carries a to-do or to-be-decided marker
  verify: [static]
```

## Budgets
```yaml
latency_p99: 3s
verify: [load]
```
Budget is per script run on a repository with 500 intents and 2,000 rules.

## Out of scope
```yaml
- generating checks (an agent workflow, not a script)
- running production SLO monitors
- languages other than Python for the scripts
```
