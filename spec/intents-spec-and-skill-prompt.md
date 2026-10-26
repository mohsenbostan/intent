# Intents — Specification v0.2 + Claude Code Skill Prompt

This file has two parts:

1. **Part A — The Intent Specification.** The format and rules.
2. **Part B — Claude Code prompt.** Paste it into Claude Code (or give it this whole file) to generate an `intents` Skill that implements Part A.

**Changes in v0.2:** new principle A2.7 *No open questions*. The `## Open questions` section is removed from A4.2, and I009 now rejects it in any status.

---

# Part A — Intent Specification v0.2

## A1. Purpose

An **Intent** is a human-owned, machine-verifiable statement of an outcome the software must deliver. Intents replace code as the artifact humans review and approve. Code is generated, disposable, and accepted only when every applicable Intent is verified.

> The code satisfies the Intent. Humans own the Intent. Machines verify it.

## A2. Core principles

1. **Outcome, not implementation.** An Intent describes observable behaviour at a boundary someone cares about (API, UI, event, data contract, budget, policy). Never function names, data structures, or libraries.
2. **Swap test.** If the entire implementation were replaced, the Intent must still be true and still matter. If not, it is too low-level.
3. **Separation of authority.** The agent that writes code never writes or edits the checks that verify it. Checks are generated from the Intent and the public interface only.
4. **Determinism after generation.** AI is non-deterministic once, when generating checks. Checks are then pinned, reviewed, versioned, and run deterministically.
5. **Every Intent has an owner and a why.** No orphan Intents.
6. **Humans read diffs, not the whole tree.** Scale comes from hierarchy and change budgets.
7. **No open questions.** Nothing undecided is ever written down: not in an Intent, a check, code, a comment, a document or a commit message. An agent that meets an undecided point stops and asks the user for an explicit answer, then records that answer as a rule.

## A3. Repository layout

```
intents/
  _index.yaml                 # optional: owners, defaults, adapter config
  checkout/
    _intent.md                # L0/L1 parent intent for the area
    payment/
      _intent.md
      authorize.intent.md     # leaf intents
      refund.intent.md
checks/                       # generated + pinned checks, mirrors intents/ tree
  checkout/payment/authorize/
    no-double-charge.property.test.ts
    ...
  checks.lock                 # hash of each intent rule -> hash of its check
```

- One Intent per file. Filename = last segment of `id`.
- Directory path must match the `id` (`checkout.payment.authorize` → `intents/checkout/payment/authorize.intent.md`).
- Parent Intents live in `_intent.md`.

## A4. File format

An Intent file is **Markdown with YAML front matter**. Machine-readable rules live in **fenced `yaml` blocks under known headings**. Any other prose is for humans and is ignored by tooling.

### A4.1 Front matter (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string, dot-separated | yes | Globally unique. Matches path. |
| `parent` | string | yes (except root) | `id` of parent Intent. |
| `owner` | string | yes | Team or person accountable. |
| `why` | string, one sentence | yes | The business reason. If you can't write it, the Intent shouldn't exist. |
| `level` | `L0`–`L3` | yes | See A5. |
| `status` | `draft` \| `approved` \| `deprecated` | yes | Only `approved` Intents gate merges. |
| `version` | integer | yes | Bumped on any rule change. |
| `reviewed_by` | string | required when `level` ≤ L2 and `status: approved` | Human approver. |
| `reviewed_at` | date | same as above | |
| `expires` | date | optional | Forces re-review; expired Intents fail lint. |
| `tags` | list | optional | e.g. `pci`, `gdpr`, `customer-facing`. |

### A4.2 Body sections

All sections optional except `Behaviour` (at least one of `Behaviour`, `Budgets`, `Policies` must exist).

| Heading | Contents |
|---|---|
| `## Context` | Free prose. Ignored by tooling. |
| `## Behaviour` | YAML list of behaviour rules (A4.3). |
| `## Budgets` | YAML map of non-functional limits (A4.4). |
| `## Policies` | YAML list of invariants that must always hold (A4.5). |
| `## Out of scope` | YAML list of strings or Intent ids. Stops agents wandering. |

There is no `## Open questions` section. Undecided points are resolved with the user before anything is written (A2.7).

### A4.3 Behaviour rule

```yaml
- id: no-double-charge          # unique within the file, kebab-case
  given: an order with a pending payment
  when: authorize is called twice with the same idempotency key
  then: exactly one charge exists at the provider
  examples:                     # optional, concrete human-chosen cases
    - { idempotency_key: "k1", calls: 2, expected_charges: 1 }
  verify: [property]            # one or more adapters (A6)
  severity: critical            # critical | major | minor
```

Rules:
- `given/when/then` are plain language, written against the public interface.
- `then` must be **observable** (response, state visible through an API, emitted event, UI state). Not "the cache is updated".
- `examples` are the strongest anchor against shared wrong assumptions. Required for `severity: critical`.

### A4.4 Budgets

```yaml
latency_p99: 800ms
latency_p50: 200ms
error_rate: "< 0.1%"
availability: "99.9% / 30d"
cost_per_call: "< $0.002"
throughput_min: 200rps
verify: [load, slo]
```

Units are mandatory. Budgets inherit from the parent; a child may only **tighten**, never loosen.

### A4.5 Policies

```yaml
- id: no-card-data-at-rest
  rule: no card PAN or CVV is logged or persisted
  verify: [policy, static]
- id: provider-access
  rule: only payments-service may call the payment provider
  verify: [network-policy]
```

Policies inherit downward and cannot be overridden by children.

## A5. Levels and hierarchy

| Level | Scope | Who writes | Who approves |
|---|---|---|---|
| L0 | Product / domain (e.g. `checkout`) | Human | Human (staff / product lead) |
| L1 | Capability (e.g. `checkout.payment`) | Human, AI-assisted | Human (owning team) |
| L2 | User-visible behaviour at a boundary | AI drafts, human edits | Human (owning team) |
| L3 | Edge cases / leaves derived from L2 | AI | AI, sampled human review |

- A child must not contradict its parent. Lint checks budget/policy inheritance; semantic contradictions are flagged by an agent review.
- L3 Intents must cite the L2 rule they refine (`refines: <file-id>#<rule-id>`).

## A6. Verification adapters

Each `verify` value maps to an adapter that compiles a rule into a deterministic check.

| Adapter | Use for | Example tooling |
|---|---|---|
| `example` | Concrete given/when/then cases | unit / integration test framework |
| `property` | Invariants over generated inputs | Hypothesis, fast-check, jqwik |
| `contract` | API/event shape and semantics | Pact, Schemathesis, OpenAPI |
| `e2e` | UI or multi-service flows | Playwright |
| `load` | Latency, throughput, error budgets | k6, Gatling |
| `slo` | Production budgets | Prometheus/Datadog SLO monitors |
| `policy` | Rules over config, infra, data | OPA / Conftest |
| `static` | Code-level forbidden patterns | Semgrep |
| `network-policy` | Who may talk to whom | K8s NetworkPolicy tests |
| `cost` | Spend per call / per day | billing export + threshold check |
| `manual` | Not yet automatable | Must have an owner + expiry; counts as debt |

Adapters are configured per repo in `intents/_index.yaml`. Unknown adapters fail lint.

## A7. Lifecycle

```
Requirement
   │
   ▼
1. Draft Intent         (human + AI)           status: draft
2. Lint                 (deterministic)
3. Review & approve     (human, reads the diff) status: approved
4. Generate checks      (independent agent: sees Intent + public interface, never the implementation)
5. Pin checks           (human reviews checks for L0–L2; writes checks.lock)
6. Prove checks bite    (targeted mutation: checks must fail on plausible bugs)
7. Implement            (coding agent: sees Intent, cannot edit checks/ or intents/)
8. Gate merge           (all checks for affected approved Intents pass)
9. Production feedback  (SLO breaches / incidents → new or amended Intent)
```

### Merge gate (must all pass)
- Lint clean.
- Every rule of every `approved` Intent touched by the change has a pinned check whose hash matches `checks.lock`.
- All those checks pass.
- No changes to `checks/` or `intents/` in the same PR as implementation changes, unless explicitly split and approved.

## A8. Trusting the checks

- **Targeted mutation testing:** for each rule, an LLM proposes 3–5 realistic bugs ("charge twice on retry", "swallow decline"). Apply to changed code only. Each mutant must be killed by at least one check of that rule. Survivors = rule not actually verified.
- Full mutation runs nightly, not per PR.
- **Seeded-bug audits:** periodically inject known bugs; measure catch rate.

## A9. Governance and scale

- **Change budget:** a single PR may add at most N rules (default 10) or change at most M (default 20). Exceeding requires an explicit override label.
- **Pruning:** an Intent with no owner, an expired `expires`, or zero verified rules for 30 days is flagged for deletion.
- **Duplication:** lint flags rules with near-identical `given/when/then` across files.
- **Human load:** reviewers see a rendered diff of rules, not raw files. L3 changes are summarised, not listed.

## A10. Lint rules (deterministic)

| Code | Rule |
|---|---|
| I001 | Front matter valid; required fields present. |
| I002 | `id` matches path. |
| I003 | `parent` exists. No cycles. |
| I004 | Rule ids unique within file. |
| I005 | Every rule has ≥1 known adapter in `verify`. |
| I006 | `severity: critical` rules have `examples`. |
| I007 | Child budgets only tighten; child policies don't override parent. |
| I008 | `approved` + level ≤ L2 has `reviewed_by` and `reviewed_at`. |
| I009 | Any `## Open questions` section → error, in every status (A2.7). |
| I010 | `expires` in the past → error. |
| I011 | Implementation-leak heuristics in `given/when/then` (class/function names, file paths, library names) → warning. |
| I012 | `manual` adapter without owner + expiry → error. |
| I013 | L3 rule has valid `refines`. |
| I014 | Change budget exceeded → error unless overridden. |

## A11. Complete example

````markdown
---
id: checkout.payment.authorize
parent: checkout.payment
owner: payments-team
why: Customers must be charged exactly once per order.
level: L2
status: approved
version: 3
reviewed_by: mohsen
reviewed_at: 2026-10-26
tags: [pci, customer-facing]
---

# Authorize payment

## Context
Called by checkout after the user confirms the order. Provider retries are common.

## Behaviour
```yaml
- id: no-double-charge
  given: an order with a pending payment
  when: authorize is called twice with the same idempotency key
  then: exactly one charge exists at the provider and both calls return the same result
  examples:
    - { idempotency_key: "k1", calls: 2, expected_charges: 1 }
  verify: [property, example]
  severity: critical

- id: declined-card
  given: a card the provider declines
  when: authorize is called
  then: order status is "payment_failed" and the response offers a retry
  examples:
    - { card: "4000000000000002", expected_status: "payment_failed" }
  verify: [contract, e2e]
  severity: major
```

## Budgets
```yaml
latency_p99: 800ms
error_rate: "< 0.1%"
cost_per_call: "< $0.002"
verify: [load, slo]
```

## Policies
```yaml
- id: no-card-data-at-rest
  rule: no card PAN or CVV is logged or persisted
  verify: [policy, static]
```

## Out of scope
```yaml
- checkout.payment.refund
- currency conversion
```
````

---

# Part B — Claude Code Prompt (generates the `intents` Skill)

Copy everything between the lines below into Claude Code, run from the repo root. Keep this file in the repo so Claude Code can read Part A.

---

````text
You are building a Claude Code Skill named `intents` that implements the
Intent Specification in Part A of `intents-spec-and-skill-prompt.md`
(this file). Read Part A fully before writing anything. Part A is the
source of truth; if anything below conflicts with it, Part A wins.

## Goal
A Skill that lets engineers and agents author, lint, review, verify and
maintain Intents, while enforcing separation between the agent that writes
code and the agent that writes checks.

## Output layout (create exactly this)
.claude/skills/intents/
  SKILL.md
  reference/
    spec.md                  # Part A copied verbatim
    adapters.md              # how each verify adapter compiles to a check, per language
    review-checklist.md      # what a human checks when approving an Intent diff
  templates/
    leaf.intent.md           # L2 template
    parent.intent.md         # L0/L1 template (_intent.md)
    l3.intent.md             # L3 template with `refines`
    _index.yaml              # adapter config + defaults (change budget N=10, M=20)
  scripts/
    lint.py                  # implements lint rules I001–I014
    diff.py                  # renders a rule-level diff between two git refs
    lock.py                  # computes/validates checks/checks.lock
    gate.py                  # merge gate from A7; exit non-zero on failure
    tests/                   # pytest tests for every script

## SKILL.md requirements
- YAML front matter with `name: intents` and a `description` that triggers
  when the user talks about intents, specs, acceptance criteria, outcome
  specs, verifying AI-generated code, or runs /intents.
- Keep SKILL.md under ~250 lines; push detail into reference/.
- Define these workflows as clear numbered procedures:
  1. `new`      – draft an Intent from a requirement. Ask for owner and `why`
                  if missing. Ask the user about every undecided point and
                  write the answer as a rule; never record open questions
                  (A2.7). Apply the swap test (A2.2). Start as draft.
  2. `lint`     – run scripts/lint.py; explain each failure in one line.
  3. `review`   – run scripts/diff.py; present the rule-level diff and
                  reference/review-checklist.md. Never mark approved yourself;
                  only a human sets reviewed_by.
  4. `checks`   – generate checks for approved Intents. MUST be run as a
                  separate subagent that is given ONLY the Intent files and the
                  public interface (OpenAPI, type signatures, UI routes). It
                  must not read implementation source. Write to checks/, then
                  run scripts/lock.py.
  5. `mutate`   – targeted mutation (A8): propose 3–5 realistic bugs per
                  changed rule, apply to changed code only, run checks, report
                  survivors. Revert all mutations afterwards.
  6. `implement`– coding workflow: read relevant Intents, write code, run gate.
                  MUST NOT modify intents/ or checks/. If an Intent looks wrong,
                  stop and propose an Intent change instead.
  7. `gate`     – run scripts/gate.py; summarise pass/fail per Intent.
  8. `prune`    – list orphaned, expired, unverified or duplicate Intents
                  (A9). Propose deletions; never delete without confirmation.
- Include a short "Hard rules" section: separation of authority, no
  implementation details in Intents, humans approve L0–L2, checks are pinned,
  no open questions anywhere (ask the user and record the answer, A2.7).

## Script requirements
- Python 3.10+, stdlib + PyYAML only.
- Parse front matter and the fenced ```yaml blocks under the known headings.
  Ignore all other prose.
- Machine-readable output via `--json`; human-readable by default.
- Exit codes: 0 ok, 1 violations, 2 usage/parse error.
- lint.py: implement I001–I014. I011 is a heuristic warning (CamelCase
  identifiers, `foo()` calls, file paths, known library names).
- lock.py: hash each rule (canonicalised YAML) and each check file; detect
  rules without checks and checks whose rule changed since pinning.
- gate.py: enforce every bullet of the merge gate in A7, including
  "no intents/ or checks/ changes in the same diff as implementation changes"
  (compare against a base ref, default `origin/main`).
- Write pytest tests covering at least one pass and one fail per lint rule,
  plus lock and gate behaviour. All tests must pass.

## Templates
- Must pass lint.py once placeholders are filled.
- Leaf template mirrors the example in A11.

## Verification before you finish
1. Create a throwaway fixture tree under scripts/tests/fixtures with the A11
   example plus deliberately broken Intents; confirm lint catches each.
2. Run the full pytest suite; fix until green.
3. Print the final file tree and a 5-line summary of how to use the Skill.

Do not invent features outside Part A. Keep everything concise.
````
