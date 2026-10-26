# Public interface: the `intents` skill CLI

This is the public interface of the `intents` skill. Agents that generate checks
(lifecycle step 4) may read **this file and `intents/` only**, never the script
source.

Everything here is observable from outside: arguments, files read and written,
exit codes, stdout/stderr, and JSON shapes.

## Location and runtime

- Scripts live at `.claude/skills/intents/scripts/{lint,diff,lock,gate}.py`.
- Run them as `python <script> [options]`. Python 3.10+. Dependencies: PyYAML only.
- Every script takes `--root DIR` (the repository root, default: current directory).
- Every script takes `--json`. With it, stdout is exactly one JSON document.
  Without it, output is human-readable text.

## Exit codes (all scripts)

| Code | Meaning |
|---|---|
| 0 | OK, nothing blocking |
| 1 | Violations found |
| 2 | Usage or parse error: bad arguments, no `intents/` directory, unknown git ref, malformed `intents/_index.yaml` or `checks/checks.lock`. A message goes to stderr. |

A malformed *intent file* is a lint violation (I001, exit 1), not a parse error.

## Files

### Intent files

Per spec A3/A4: `intents/**/<name>.intent.md` and `intents/**/_intent.md`.
Tooling reads the front matter plus the first fenced `yaml` block under each of
`## Behaviour`, `## Budgets`, `## Policies`, `## Out of scope`, and the text of
`## Open questions` (HTML comments ignored). Everything else is prose.

### `intents/_index.yaml` (optional)

```yaml
owners:                       # informational
  payments-team: "#payments"
change_budget:                # A9; defaults shown
  max_added_rules: 10
  max_changed_rules: 20
  override_label: intents-change-budget-override
mixed_change_label: intents-mixed-change
adapters:                     # the set of known adapters; a list of names is also accepted
  example:
    run: "python -m pytest -q {files}"   # {files} = space-separated check paths; appended if absent
  slo:
    run_in_gate: false        # checks are pinned but evaluated outside the merge gate
```

If the file is absent, or has no `adapters` key, the known adapters are the
eleven in spec A6, all without a `run` command.

### Rules and rule keys

A **rule** is each entry under `## Behaviour`, each entry under `## Policies`,
and the whole `## Budgets` block, which is one rule with the reserved id `budgets`.
A rule's key is `<intent-id>#<rule-id>`.

### Check files

`checks/<intent id with dots as slashes>/<rule-id>.<adapter>.<anything>`

Example: `checks/checkout/payment/authorize/no-double-charge.property.test.ts`
is a `property` check for `checkout.payment.authorize#no-double-charge`.

Files in `checks/` whose second dot-separated name segment is not a known
adapter are helpers. They are never treated as checks.

### `checks/checks.lock`

JSON, written only by `lock.py write`:

```json
{
  "version": 1,
  "rules": {
    "<intent-id>#<rule-id>": {
      "rule_hash": "<sha256 of the canonicalised rule>",
      "checks": { "<repo-relative check path>": "<sha256 of file, CRLF normalised to LF>" }
    }
  }
}
```

## lint.py

```
lint.py [--root DIR] [--base REF] [--label NAME]... [--json]
```

- Lints every intent file (codes I001–I015, see below).
- `--base REF` enables the change budget (I014): rules in the working tree are
  compared with rules at `REF`.
- `--label` (repeatable) names PR labels; the change-budget override label
  suppresses I014.
- Exit 1 if any finding has level `error`. Warnings alone exit 0.

JSON:

```json
{
  "findings": [
    { "code": "I006", "level": "error", "path": "intents/a/b.intent.md", "message": "..." }
  ],
  "errors": 1,
  "warnings": 0,
  "intents": 12
}
```

`path` is the repo-relative, `/`-separated intent file. Repo-wide findings (I014)
use the path `intents/`.

| Code | Level | Meaning |
|---|---|---|
| I001–I014 | as in spec A10 | I011 is a warning, the rest are errors |
| I015 | warning | Near-identical given/when/then in two files (spec A9 duplication) |

## lock.py

```
lock.py [check|write] [--root DIR] [--json]      # default: check
```

- `write` pins the current state of every rule of every `approved` intent and its
  check files into `checks/checks.lock`. Exit 0.
- `check` compares the working tree with the lock. Exit 1 on any problem.

`check` JSON:

```json
{ "ok": false, "problems": [ { "rule": "<intent-id>#<rule-id>", "message": "..." } ] }
```

## diff.py

```
diff.py [--base REF] [--head REF] [--full] [--root DIR] [--json]
```

- `--base` defaults to `origin/main`. `--head` defaults to the working tree,
  including uncommitted changes.
- Default output is Markdown suitable for a PR comment. L3 intents are summarised
  as counts unless `--full` is given.
- Exit 0 unless there is a usage error.

JSON:

```json
{
  "base": "origin/main",
  "head": "WORKTREE",
  "intents": [
    {
      "id": "checkout.payment.authorize",
      "level": "L2",
      "change": "added | removed | modified",
      "fields": { "status": ["draft", "approved"] },
      "rules": {
        "added":   [ { "id": "...", "...": "..." } ],
        "removed": [ { "id": "...", "...": "..." } ],
        "changed": { "<rule-id>": { "<field>": ["<old>", "<new>"] } }
      },
      "version_not_bumped": false
    }
  ]
}
```

`--json` always lists every intent, L3 included.

## gate.py

```
gate.py [--base REF] [--label NAME]... [--root DIR] [--json]   # --base default: origin/main
```

Runs the merge gate (spec A7) and exits 1 if any part fails:

1. `lint --base REF` has no errors.
2. `lock check` has no problems.
3. The files changed since the merge base of `REF` and `HEAD` (committed,
   uncommitted and untracked) do not include both `intents/` or `checks/` paths
   and other paths, unless `mixed_change_label` is passed with `--label`.
4. For every `approved` intent, the checks of each adapter run with that
   adapter's `run` command and exit 0. Adapters with `run_in_gate: false` are
   reported as skipped. A check whose adapter has no `run` command fails.

JSON:

```json
{
  "ok": false,
  "lint": { "errors": 0 },
  "lock": { "problems": [] },
  "mixed_change": { "blocked": false, "intent_paths": [], "other_paths": [] },
  "intents": [
    { "id": "...", "status": "pass | fail | skipped", "details": ["example: exit 1", "slo: skipped"] }
  ]
}
```
