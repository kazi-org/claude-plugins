# kazi/AUTHORING.md -- predicate authoring quality

Reference for anything that drafts kazi predicates: this skill's `plan`
verb, or an orchestrating workflow deriving predicates from a work plan's
acceptance criteria. Predicate quality determines convergence honesty and
cost.

## Author for the grind tier

The predicate `description` fields are effectively the ONLY brief the grind
model receives: kazi's dispatch prompt is the goal name + failing predicates
+ evidence. The grind model never sees your session or your strategy doc. A
thin description makes a mid-tier model flail or converge vacuously; a dense
one converges in a dispatch or two. Every payload carries:

- **A TASK BRIEF in the first predicate's description**: one sentence of
  WHY, the exact files/modules to touch (read the repo first if you do not
  know them), the pieces known to be missing, what NOT to change, and
  issue/ADR numbers with "read these first". Write it like a ticket a new
  hire could execute without asking anything.
- **A PROCESS contract in the same brief** (branch `task/<goal-id>`, small
  conventional commits, push with `-u`) plus a `landed` predicate (clean
  tree AND `HEAD == @{u}`), so convergence means PUSHED work.
- **ONE requirement per predicate** (or per named assertion). Never fold N
  requirements into a single "the new test file passes" check: the grind
  model authors that test, and a self-authored test can satisfy one of N
  requirements and still go green. Enumerate every requirement.
- **Negative-space companions** for any text-presence check: a bare
  `grep -q` is satisfiable by string-stuffing or an unrelated pre-existing
  match (the clarify floor warns on this; #924 is the failure mode).
- **Guard predicates** for what must not break (full suite, formatter),
  hermetically wrapped when the checker boots the app
  (`sh -c 'env -i HOME="$HOME" PATH="$PATH" LANG="$LANG" ...'`).

If you catch yourself writing a one-line description, expand it: tokens
spent on the brief are repaid by the dispatches the grind tier does not
burn.

## Capability vs guard, and the red-at-t0 rule

Classify every predicate when you author it:

- **capability** -- proves NEW behavior this goal is supposed to create (the
  endpoint exists, the test passes, the flag works). A capability predicate
  MUST be observed RED at t0 (before any grind dispatch). If it already
  passes, one of these is true: the work is already done (drop the goal),
  the predicate is vacuous (a grep matching pre-existing text, an
  always-true script), or it tests the wrong thing. Fix or reclassify --
  never converge on it and call that progress.
- **guard** -- protects EXISTING behavior (full suite green, formatter
  clean, no regression). Guards are expected green at t0; that is their job.

Tag the classification in the predicate id (e.g. `"id": "cap-healthz"` /
`"id": "guard-suite"`) until kazi grows a first-class `kind` field (#1128).
Consequence for check-only gates: a `vacuous_goal` verdict (all predicates
pass at t0) is a GREEN result ONLY when every predicate is guard-shaped; if
any capability predicate is in the set, vacuous means the goal never
measured the new work -- treat it as a verification FAILURE, not a pass.

## Provider inference (drafting from acceptance-criterion lines)

When a work plan carries machine-checkable acceptance-criterion lines, emit
one predicate per line --
`{"id": "<task-id>", "provider": "<provider>", "description": "<criterion + brief>"}`:

- `http_probe` when the text names an HTTP verb + path, or mentions "prod",
  "deployed", "live", or a URL.
- `swift_test` when the text names an Xcode/XCTest/`.xcresult` acceptance
  criterion (a Swift test suite passing, an `.xcresult` bundle showing zero
  failures). Requires `xcresult_path`; see `kazi schema swift_test`.
- `custom_script` for everything else (test assertions, build/lint
  conditions, CLI behavior). Do NOT emit `test_runner`: deprecated
  (ADR-0040, removal in v2.0.0); every load prints a deprecation warning.

A task without a machine-checkable criterion contributes no predicate; its
free-text acceptance criteria are simply not kazi-covered.

## Runtime introspection

Confirm the payload shape against the live CLI before drafting:
`kazi help --json`, `kazi schema plan`. If this document and the schema
disagree, the schema wins.
