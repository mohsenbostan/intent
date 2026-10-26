# Intents

> The code satisfies the Intent. Humans own the Intent. Machines verify it.

**Intents** are human-owned, machine-verifiable statements of the outcomes software
must deliver. They replace code as the thing humans review and approve. This repo
holds the specification and a Claude Code skill that implements it.

This repository is built **with Intents, from the beginning**. The skill's own
requirements are written as Intents and approved before any of its code exists.

**No open questions, ever (spec A2.7).** Nothing undecided is written down: not in
an Intent, a check, code, a comment, a document or a commit message. An agent that
meets an undecided point asks the user and records the answer as a rule. Lint
rejects any `## Open questions` section (I009).

## Where this is in the lifecycle (spec A7)

| Step | | Status |
|---|---|---|
| 1. Draft Intents | `intents/` | ✅ 13 intents, 62 rules |
| 2. Lint | | ✅ clean (bootstrap linter) |
| 3. Review & approve | human | ⏳ **awaiting review** |
| 4. Generate checks | isolated agent, sees `intents/` + `interface/` only | — |
| 5. Pin checks | `checks/checks.lock` | — |
| 6. Prove checks bite | targeted mutation | — |
| 7. Implement | `.claude/skills/intents/` | — |
| 8. Gate merge | `gate.py` | — |

## Layout

```
spec/intents-spec-and-skill-prompt.md   the Intent Specification v0.2 (Part A) + skill prompt (Part B)
interface/cli.md                        public interface of the skill's scripts (what checks may rely on)
intents/                                this repo's own Intents
  _index.yaml                           adapters and change budget
  skill/_intent.md                      L0: the intents skill
  skill/lint/…                          L1 lint + L2 leaves + one L3
  skill/{lock,gate,diff,templates,workflows}.intent.md
```

## Bootstrap notes

- The first commit added 58 rules, over the default change budget of 10. It is the
  bootstrap commit, so the change budget does not apply; later PRs use the
  `intents-change-budget-override` label when they must exceed it.
- The linter used for step 2 is a draft kept outside the repo. The real one is
  written in step 7 and has to pass the pinned checks like everything else.
