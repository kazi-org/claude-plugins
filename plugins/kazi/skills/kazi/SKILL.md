---
name: kazi
description: Drive kazi -- a reconciliation controller that converges a software goal to machine-checkable acceptance predicates -- as a tool from an orchestrating agent. A ROUTER over four verbs: plan (author the predicates), apply (converge the goal -- the reconcile loop), status (read convergence state), adopt (reverse-engineer a starter goal-file from a repo). Use when the user wants to author predicates for a goal, then have a cheap Claude model (in-family tiering, the default; local/BYOM via opencode is the secondary privacy option) grind until they objectively pass (and not declare victory early). Triggers include "have kazi drive this until done" (the canonical invocation phrase), "converge this goal with kazi", "drive kazi", "have kazi run the loop", "plan/apply with kazi", "author acceptance predicates and reconcile", or any task where you (a strong model) set the bar and a cheaper model should reach it under objective termination.
---

# Drive kazi from an orchestrating agent (router)

kazi is a reconciliation controller: you declare a goal as machine-checkable
acceptance predicates, and kazi drives a coding harness in a loop until those
predicates are objectively true, the loop is stuck, or it is over budget. kazi
is NOT a harness -- it drives one.

This skill ships as three files. Read the other two at their point of use:

- kazi/AUTHORING.md -- predicate authoring quality (the task brief,
  capability-vs-guard and the red-at-t0 rule, one requirement per predicate,
  provider inference). Read it BEFORE drafting any predicates.
- kazi/RECIPES.md -- operational recipes (the escalation ladder, streaming,
  parallel/standing, the check-only gate variant, the session bus).

## Site-specific routing: LOCAL.md

Operator wiring lives at a STABLE path OUTSIDE this skill directory:
`~/.claude/skills/kazi/LOCAL.md`. If that file exists, READ IT FIRST. It carries
the operator's own wiring -- e.g. which local orchestration skill owns
plan-driven work, house conventions, model policy overrides -- and it takes
precedence over the generic routing below. It lives outside the skill
directory ON PURPOSE (ADR-0077, the ADR-0074 amendment): a plugin update
replaces the skill directory wholesale, so customization kept at the stable
path survives plugin updates, re-installs, and any future relocation of the
skill directory. `kazi install-skill` never writes or overwrites it.

## How to drive kazi: MCP first, JSON-CLI fallback

PREFER the MCP server. If you speak MCP (Claude Code, Codex, Cursor), wire kazi
as an MCP server and drive its self-describing tools -- `kazi_plan`,
`kazi_approve`, `kazi_apply`, `kazi_status`, `kazi_list_proposed` -- whose
input/output schemas teach you the surface at ZERO prose cost. The installed
binary serves it over stdio via the `kazi mcp` verb (ADR-0044). The canonical
client config is the binary verb:

```json
{ "mcpServers": { "kazi": { "command": "kazi", "args": ["mcp"] } } }
```

`kazi init --with-mcp` writes exactly this `.mcp.json` into a repo for you, and
`mix kazi.mcp` is the development entry point that starts the SAME server.

FALLBACK -- the JSON-CLI shell-out. When MCP is unavailable, drive kazi by
shelling out and parsing its `--json` output (never its prose). The same
plan -> approve -> apply recipe applies; the rest of this skill teaches it as
CLI invocations, which map one-to-one onto the MCP tools above.

## The invocation phrase

When the user says "have kazi drive this until done" (the canonical
invocation phrase), that IS the request to drive kazi: author the acceptance
predicates for the task with the `plan` verb, then converge them with the
`apply` verb until they are objectively true. Treat it as "set the bar, then
reconcile to it".

This skill is a ROUTER. The user (or you) names a sub-skill verb; you route it
to the matching real `kazi` CLI command and drive it over `--json`:

| sub-skill verb | routes to     | what it does                                            |
|----------------|---------------|---------------------------------------------------------|
| `plan`         | `kazi plan`   | author/refine the goal's acceptance predicates (authoring path). |
| `apply`        | `kazi apply`  | converge the goal -- the reconcile loop.                |
| `status`       | `kazi status` | read convergence/proposal state from the read-model (a pure read). |
| `adopt`        | `kazi init`   | reverse-engineer a starter goal-file from an existing repo.        |

The verb you TYPE, the skill, and the CLI read the same: `plan` and `apply`
are the CLI verbs (ADR-0032). `adopt` is the one human alias -- it routes to
`kazi init`. The legacy CLI verbs `run`/`propose` were REMOVED in v0.6.0
(T27.9): use `apply`/`plan`.

Confirm the live surface before you drive: `kazi help --json` emits the
command/flag table and `kazi schema [<command>]` emits the versioned result
schemas. They are generated from kazi's own command table, so they never drift.
Prefer them over this document when in doubt.

## Where kazi sits in your workflow

kazi is the EXECUTION layer for engineering goals: it makes "done" objective
and grinds until it holds. The INTENT layer -- deciding what to build,
strategy, work breakdown -- stays with you (or your own planning workflow).
When a work plan already carries machine-checkable acceptance criteria,
DERIVE the predicates from those criteria instead of inventing new ones
(caller-drafts, Step 1 below); when none exist, draft the predicates from the
goal directly.

For a CODE goal the on-ramp is exactly these four verbs. `kazi apply` IS the
reconcile loop, and "launch-ready" is the OBJECTIVE predicate vector
(including any live prod probe), not a heuristic to infer afterward
(ADR-0031) -- so you do not need a separate outer convergence loop or a
separate qualification pass around it.

### Not a kazi repo? Degrade cleanly

The router assumes `kazi` is on PATH. If it is not (a repo that has not
adopted kazi, or a non-engineering task), do NOT fabricate a `kazi`
invocation. Fall back to your own planning/execution workflow. Run `adopt`
(`kazi init <repo-dir>`) first only when the user wants to bring kazi into
the repo. kazi claims engineering/code goals; content, GTM, and ops work
stays outside it.

## The two-tier economics (why drive kazi at all)

kazi sits in the MIDDLE of a three-layer stack:

```
  you, the orchestrator   (FRONTIER model -- plan/design, AUTHOR the predicates)
        |  drive kazi as a tool  (this recipe)
        v
      kazi                (the controller -- objective predicates + convergence loop)
        |  drives the inner harness
        v
  cheap implementer       (a CHEAP Claude model -- the keystrokes)
```

Spend expensive reasoning ONCE on the part that needs judgment: what "done"
means -- the acceptance predicates. Spend cheap compute on the iterative grind
of editing until those predicates pass. kazi's objective termination makes the
split safe: the cheap implementer cannot declare victory on
plausible-but-wrong work, because truth lives in the controller (the predicate
vector), not in the model doing the keystrokes.

The DEFAULT recipe is in-family Claude tiering (ADR-0033/0035, amended
2026-07-08 on fleet data): you author the predicates in this very session;
run the grind on the DEFAULT grind tier via
`kazi apply --harness claude --model claude-sonnet-5 --json [--stream]`.
The default tier is `claude-sonnet-5` (step up to `claude-opus-4-8` for
harder slices). Haiku is an explicit OPT-DOWN for a slice you already KNOW
is trivial (a one-line fix, a lint pass), never the default -- cheap-tier
grinding produced the vacuous-convergence failure mode (#924). Local/BYOM is
the SECONDARY privacy add-on: `kazi apply --harness opencode --model
<local-model>` keeps the grind on your hardware. When in doubt about a
tiering call, consult `kazi economy --json` -- measured cost/outcome
percentiles per model and goal shape from YOUR OWN fleet's run registry --
rather than trusting any frozen figure (the dated finding that flipped this
default lives in ADR-0035's dated amendment).

## The loop the verbs sit inside: plan -> approve -> apply

### Step 1 -- author the goal-set (`plan` verb -> `kazi plan --json`)

`plan` AUTHORS or REFINES a goal-set -- the acceptance `predicates`, plus the
optional `[[groups]]` that partition a larger goal and the `needs` edges that
order the groups into dependency waves -- and persists it as a reviewable
PROPOSAL that HOLDS for approval: `plan` runs NOTHING and dispatches NO
harness. A deterministic clarify FLOOR flags a missing live-verification
target and an unscoped goal, so an under-specified goal is surfaced, never
silently accepted.

As the orchestrator use CALLER-DRAFTS mode (ADR-0023): you already reasoned
about the goal, so YOU supply the candidate predicates and kazi spawns NO
second model -- it only validates, applies the floor, and persists:

```sh
kazi plan --json --predicates '{
  "name": "ship a /healthz endpoint",
  "predicates": [
    {"id": "cap-healthz-route", "provider": "custom_script", "description": "..."},
    {"id": "cap-healthz-live",  "provider": "http_probe",  "description": "GET /healthz returns 200 in prod"}
  ],
  "rationale": "a health probe for the deploy target"
}'
```

**Before drafting, read kazi/AUTHORING.md and follow it** -- predicate
quality is the single biggest determinant of convergence honesty and cost.
It covers the task brief the grind model needs, one requirement per
predicate, capability-vs-guard classification and the red-at-t0 rule,
negative-space companions, and provider inference (never the deprecated
`test_runner`).

For a human or a thin script that has only a prose idea, kazi-drafts mode
spawns a harness to draft the predicates instead:
`kazi plan "a /healthz endpoint that returns 200" --json --yes`. Under
`--json` kazi is NON-INTERACTIVE: an underspecified idea is a JSON error and
a non-zero exit, never a hang (`--strict` refuses instead of best-effort;
`--adr` also writes an ADR-lite doc; `--workspace <path>` scopes the repo).

### Step 2 -- review and approve (`kazi approve --json`)

Read the proposed `predicates` and the `clarify` gaps
(`{id, prompt, recommended}` entries). If a gap matters (e.g. no
live-verification predicate), re-run `kazi plan` with it closed. Run
`kazi lint <goal-file>` -- read-only, advisory, exit 0 even with warnings --
then approve: `kazi approve <proposal-ref> --json`. (`kazi reject
<proposal-ref> --json` declines, kept for audit.) Browse the queue with
`kazi list-proposed --json` (optionally `--status proposed|approved|rejected`).

### Step 3 -- observe t0, then converge (`apply` verb)

First observe the t0 predicate vector (the check-only gate variant in
kazi/RECIPES.md, or `kazi apply --explain` for the schedule alone). A
CAPABILITY predicate that already passes at t0 is SUSPECT -- fix it before
burning a converge (kazi/AUTHORING.md, red-at-t0). Then:

```sh
kazi apply <proposal-ref> --workspace <path> --harness claude --model claude-sonnet-5 --json
```

`apply` takes the APPROVED `prop-...` ref from Step 2 (loaded straight from
the read-model, ADR-0049) or a goal-file path, and drives the goal to a
TERMINAL VERDICT. The exit code mirrors convergence: 0 only on `converged`.

Respect kazi's three safety refusals -- do NOT reflexively override them. It
refuses (a) a `--workspace` that is a git repo's PRIMARY worktree root (the
dispatched agent's shell can reset/clean the whole checkout) -- run against a
dedicated task worktree (`git worktree add <path> <branch>`), and pass
`--allow-primary-workspace` only for a throwaway checkout you accept losing;
and (b) a goal the run registry already shows LIVE (fresh heartbeat) --
`--allow-duplicate-run` only for a deliberate re-run; and (c) a `--workspace`
a DIFFERENT live goal already holds (two goals on one directory bleed commits
into each other) -- `--allow-workspace-collision` only for co-tenancy you know
is safe. When a refusal surfaces, fix the CONDITION, not the flag.

For a LONG convergence add `--stream` (JSONL progress); for a partitioned
goal-set add `--parallel`; for a continuous hold-true reconciler add
`--standing`; `--explain` (alias `--dry-run`) prints the wave schedule and
exits without dispatching. For a supervised checkpoint between needs-DAG
waves (#936, T50.3) add `--pause-between-waves`: stops starting new groups
once the current frontier settles, persists a resume checkpoint, and exits
0 with a `paused` collective carrying a `resume_token` -- continue later
with `--resume <token>`. Details for all five flags: kazi/RECIPES.md.

### Step 4 -- parse the result and branch on `next_action`

`apply --json` emits ONE terminal result object (pin `schema_version`,
currently 2 -- see kazi/RECIPES.md):

| `status`      | `next_action`  | exit | What you do |
|---------------|----------------|------|-------------|
| `converged`   | `done`         | 0    | Finished. Ship / report. |
| `stuck`       | `investigate`  | != 0 | Same slice failed N times -> escalate the ladder. |
| `over_budget` | `raise_budget` | != 0 | Raise the budget and re-run, or step up. |
| `error`       | `investigate`  | != 0 | Pre-loop misconfig (vacuous goal, unknown harness): read `error`, FIX THE GOAL -- never escalate the model. |

`next_action` is an orchestration HINT -- you own the policy.

**Escalation ladder (bounded).** On `stuck`/`over_budget` for the SAME
slice/goal_id, re-dispatch one rung up. Two rungs, default tier first:

```
claude-sonnet-5  ->  claude-opus-4-8   (STOP; do not escalate past Opus)
```

`claude-haiku-4-5` is NOT a rung on this ladder -- it is an explicit
OPT-DOWN you pin yourself for a known-trivial slice. The rung counter is
YOUR state, keyed by `goal_id`; a fresh slice resets to the default rung.
Full trigger mapping and a copy-paste POSIX recipe: kazi/RECIPES.md. When a
goal keeps landing on stuck, `kazi economy --rediscovery <goal>` names
which predicate is burning the repeat attempts.

Alternatively, let kazi own the ladder: declare an `[escalation]` block in
the goal-file (ADR-0056) and the loop re-dispatches the SAME goal at the
next model in the declared list on `stuck`/`over_budget`, all inside one
`kazi apply` -- no rung counter of your own. Drive the ladder by hand
(above) when you want to inspect between rungs; use the block when you do
not. Details + the TOML shape: kazi/RECIPES.md.

## Roadmap scope: a project is a goal DAG (ADR-0056)

`plan`/`apply` author and converge ONE goal; a project is an ordered SET of
goals with dependencies. The SAME verbs lift one level, so you drive the whole
engineering surface -- roadmap planning, discovery, the plan document -- from
the binary alone, with NO external plan/apply layer assumed.

- **Author a roadmap** -- `kazi plan --project '<goals-json>'` carries a
  multi-goal payload (a JSON object with a `"goals"` array; each goal a
  per-goal predicate payload plus optional `needs` edges). kazi persists N
  linked proposals under ONE roadmap ref and emits the roadmap ref + per-goal
  proposal refs. Caller-drafts, exactly like `--predicates` one level down.
- **Discovery on-ramp** -- `kazi plan --discover` (opt-in) attaches best-effort
  discovery evidence (stack detection, `.feature` use-cases, a public-surface
  codebase scan) to a kazi-drafts proposal, visible via
  `kazi status <proposal-ref> --json`. Caller-drafts (`--predicates`/`--project`)
  bypass it; any step failing degrades to a plain draft with a warning.
- **Converge a roadmap** -- `kazi apply <roadmap-file>` runs the whole goals in
  topological `needs` frontiers (the same engine `--fleet` uses), each goal in
  its own task worktree with its own `[integration]` landing, to a
  roadmap-level collective verdict. `--explain` prints the schedule and exits.
- **Render the plan** -- `kazi plan render <roadmap-file> [--out <path>]` emits
  the human-readable plan (WBS, waves, progress) as GENERATED markdown from the
  read-model. It is OUTPUT, never input -- regenerate, never hand-edit, so the
  document cannot drift. Details: kazi/RECIPES.md.

## Landing: `[integration]`, `[conventions]`, and the process contract

Convergence is not the end: a goal whose code predicates pass but whose fix is
still uncommitted is not done. kazi treats landing as part of the objective bar
(ADR-0055) and owns the universal working rules so goal-files stay declarative.
Full how-to: docs/landing.md.

- **`[integration]` -- how work LANDS.** A block declaring `mode` (default
  `none`; one of `none | commit | branch | pr | merge`). When `mode != none`,
  kazi SYNTHESIZES a `landed` predicate evaluated against the LIVE working tree
  (clean tree plus the mode-appropriate committed / pushed / PR-open /
  rebase-merged state), so "code-green but uncommitted" stays UNSATISFIED. The
  `:integrate` action then verifies-then-ships (the inner agent owns its
  commits; a dirty tree is a distinct error, never a silent bulk commit). Under
  `--parallel`, each group lands on its own branch; `mode = "merge"` over a
  `needs`-DAG merges in topological order with `git cherry` silent-revert
  verification.
- **`[conventions]` -- the process contract.** kazi appends a small, versioned
  block of UNIVERSAL working rules to every dispatch prompt (small conventional
  commits scoped to one directory; commit as you go; no stubs; grep
  docs/lore.md before debugging; migration-number safety under parallelism;
  network-retry; prefer graph tools). It is byte-identical across a goal's
  iterations (a cacheable head) and harness-agnostic. `process_contract = false`
  disables it; `extra_rules = [...]` appends repo-specific lines verbatim.
- **Tier-0 pattern (older binaries).** If your goal-file targets a kazi binary
  that PREDATES the `[integration]` block, hand-write the equivalent `landed`
  predicate as a `custom_script` -- "clean tree AND HEAD ahead of origin/main",
  the manual equivalent of `mode = "commit"`. Keep the commits small and scoped
  to one directory (matching the process contract). Copy-pasteable:

  ```toml
  [[predicate]]
  id = "landed"
  provider = "custom_script"
  description = "clean tree AND HEAD ahead of origin/main -- manual equivalent of [integration] mode = commit"
  cmd = "sh"
  args = ["-c", "git status -s | grep -q . && exit 1; git diff origin/main HEAD | grep -q . || exit 1; exit 0"]
  verdict = "exit_zero"
  ```

- **The routing decision (ADR-0055).** Do NOT paste prose discipline blocks into
  a goal-file -- each concern has one home: objectively-checkable rules become
  PREDICATES (the `landed` predicate, the validation ladder, zero-stub);
  universal how-to-work guidance is carried by the PROCESS CONTRACT (never
  restated per goal); mechanics (worktree isolation, branch creation, merge
  ordering, PR opening) are CONTROLLER behavior. A goal-file stays a short
  declarative statement of done.

## status / adopt / waiting

- `kazi status <ref> --json` reads convergence or proposal state (a pure
  read; run goal-id first, else proposal-ref). `kazi dashboard` renders the
  same read-model for a human watching a fleet.
- `adopt` = `kazi init <repo-dir>`: bootstrap a starter goal-set once per
  repo by deterministic stack detection, then refine via `plan`
  (kazi/RECIPES.md). For notes outside kazi's own surfaces,
  `kazi export <goal-file> --obsidian <dir>` snapshots a goal's group tree
  + verdicts to an Obsidian vault.
- Waiting on another session with a `kazi daemon` up: `kazi bus` -- peek
  checks without consuming, read ACKS what it pulls (landmine), watch is
  the no-poll blocking wait. Taxonomy: kazi/RECIPES.md.

## Feedback

Friction while driving kazi -- a bug, a gap, a workaround you needed -- is
signal: file it at https://github.com/kazi-org/kazi/issues (search for an
existing issue first).
