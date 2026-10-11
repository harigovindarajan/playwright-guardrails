---
name: migrate
description: Drive a test migration to Playwright through the plugin's sibling agents in order (planner, drafter, translator, writer, reviewer, runner), carrying the source suite's locators over and grounding them from the traces of real runs instead of a live probe (or, only when asked, against the live app), routing review and runtime failures back to the right stage, and stopping for a human after a fixed number of rounds. Also upgrades a finished migration's carried locators from its saved traces (--upgrade). Run from the main session, which spawns each agent in turn; no agent calls another. Use when the user asks to migrate a suite, a test file, or a contract to Playwright end to end.
argument-hint: "<source suite root or test file> [--framework-root=tests] [--locators=trace|live] [--base-url=...] [--threshold=0.8] [--rounds=3] | --upgrade [--migration-dir=...]"
user-invocable: true
disable-model-invocation: true
---

# Migrate: the reference orchestrator

This skill is the **orchestrator** the agents' design assumes. Every agent in this plugin is a
single-role sibling, and none of them calls another. Sequencing belongs here: you, the main
session, run this procedure and spawn each agent in turn with the Agent tool, read the JSON it
returns, persist it, and decide the next step. You never do an agent's job yourself. If an
agent fails or returns something you can't use, you stop and report; you don't write the
contract, the locator map, or the spec by hand.

Each agent's **Invocation Contract** is its authoritative signature. This skill says which
inputs to pass at each stage and what to do with the result; it does not restate the agents'
output shapes.

## Settings

Collect these before starting. Ask the user for any required one that is missing; never
guess a URL, a path, or a credential.

| Setting | Required | Default | Used by |
|---|---|---|---|
| Source: a suite root, test file(s), or existing contract(s) | yes | | planner, drafter |
| Source helper paths (page objects, custom commands, fixtures) | no | none | drafter, planner |
| Playwright framework root | yes | | writer, runner |
| Page-objects dir inside the framework root | no | `<framework-root>/pages` if it exists | probe (Tier 1) |
| Locator source: `trace` (carry the source suite's locators over, then ground them from the traces of real runs; see step 3) or `live` (ground every locator against the live app, only when the user asks for it) | no | `trace` | steps 3 and 4 |
| App base URL | in `live` mode | | probe (live) |
| `storageState` path, and the env var name the framework reads it from | when any state is gated | | probe, runner |
| Env file and required variable names | no | none | runner |
| Migration dir (contracts, maps, cache, reports) | no | `.playwright-migration/` | all |
| Grounding threshold | no | `0.8` | live probe gate (not used in `trace` mode) |
| Round cap | no | `3` | feedback loop |

`translate` and `probe`, the 0.6.0 names for the locator source, still work for this release
as aliases of `trace` and `live`. Trace mode needs Playwright 1.63 or later in the framework
root; the probe checks, and fails statically with `missing-trace-cli` on an older install. Tell
the user the version it needs.

From the migration dir, use these sub-paths **for every round of every test case**, so that
reports, maps, and the cache line up across rounds:

- `contracts/`: drafter output, filename pattern `{testcase}-contract.md`
- `locator-maps/`: probe output, one file per probe run, filename pattern
  `{testcase}-locator-map.r<n>.json`, where `<n>` is the round the probe ran for (`r1` for the
  first probe). You fill in `<n>` before passing the pattern, so the probe sees, for example,
  `{testcase}-locator-map.r2.json`. A re-probe never replaces an earlier map; the highest
  version is the **current map**, and every earlier one stays as evidence of what a past round
  ran with.
- `locator-cache/`: the probe's Tier 2 cache, shared by every test case in this migration
- `reports/<testcase>/`: every agent's JSON for that test case, written by you as
  `<stage>-round-<n>.json`. An agent you run on several test cases at once (the planner, or a
  batched drafter, translator, probe, writer, or reviewer) writes one report for the batch:
  save it once, at `reports/batch-<b>/<stage>-round-<n>.json`, and save the runner's reports
  per test case as above. `<b>` counts the `/migrate` invocations in this migration dir: one
  more than the highest existing `reports/batch-<b>/`, so a later batch never overwrites an
  earlier batch's reports.
- `runs/<testcase>/round-<n>/`: the runner's run output directory for that round. The runner
  creates one `attempt-<k>/` folder inside it per Playwright invocation, so every run's report,
  log, and traces are kept side by side and nothing is overwritten.
- `runs/index.jsonl`: your append-only index of every run attempt, one JSON line each (see below)
- `scratch/`: the probe's scratch directory in trace mode. It extracts each trace here and
  removes it with `trace close`; its snippet files stay, as the record of each trace grounding.

Create these directories (and each round's `runs/` directory) before the agent that needs them
runs. The drafter, the probe, and the runner require their output directories to exist and
fail statically when they don't; creating them is your job, not theirs.

You persist every agent's JSON response to `reports/` before acting on it, **verbatim**:
the exact JSON object the agent returned, never a summary, an excerpt, or a reformatted
version. Later stages take those files as inputs (the writer reads the review and run reports
as feedback, and the reviewer reads its own prior report for its findings ledger), so a
trimmed report silently breaks the next round. The files are also the audit trail of the
migration.

## Procedure

### 1. Plan (once per migration)

Skip when the source is a list of contracts.

Spawn `playwright-migration-planner` with the source suite root(s) and any helper paths
outside them. Persist the plan. Show the user its summary, unresolved dependencies, and
sequence, then migrate test files in the plan's `sequence` order: each `shared` entry's seed
test file first, then `simple`, then `complex`. When the source is a single test file, skip
the planner and treat that file as the whole sequence.

### 2. Draft

For each test file in order, spawn `playwright-contract-drafter` with the file, the helper
paths, and `contracts/` as the output directory. Persist the manifest.

**Gate: open questions.** For each contract in the manifest:

- **Helper preflight.** When every `blocking` question on a contract comes from an unresolved
  helper (a call into shared code the drafter was not given; the manifest lists them in
  `unresolved_helpers`), search the source root for each helper's definition. If you find all
  of them, tell the user which files define them and offer to re-run the drafter with those
  files added as helper paths. Re-draft only after they agree, and save its manifest as the
  next draft round (`draft-round-2.json`). Never add files without asking.
- Any `blocking` open question: **stop this test case** and put the questions to the user.
  Continue only after the contract has been edited to answer them. Other test cases may go
  ahead.
- `non-blocking` questions: show them to the user and continue. The writer carries them as
  risks.

### 3. Ground the first map

**Default (`trace`): translate.** Spawn `playwright-locator-translator` with the contract(s),
the framework root, `locator-maps/` as the output directory, the `r1` filename pattern,
`locator-cache/` and the page-objects dir when they exist (both read-only for it), and the
`storageState` path when there is one (only its name is used). Persist the manifest.

**Gate: none on grounding.** A translator map is unverified by design, so the grounding ratio
does not apply; the runner is the gate. Stop only when the manifest's `status` is `FAILED`. The
writer marks every translated locator under Rule 28, and the reviewer treats a marked locator as
residual risk the run settles. Every run keeps a trace (see Run below), so the probe can ground
from what the runs observed instead of driving the app: a failing locator in the feedback loop
(step 4), and the carried locators of passing tests in an upgrade (see Upgrade carried
locators).

**`live`, only when the user asks for it.** Live probing drives the app for every page state,
which costs far more time and tokens than translating and grounding from traces. Never choose it
yourself: use it only when the user passed `--locators=live` or asked for it in words. Spawn
`playwright-dom-probe` with `mode: live`, the contract(s), the base URL, the `storageState`,
`locator-maps/` as the output directory, the `r1` filename pattern above, `locator-cache/`, and
the page-objects dir when it exists. Persist the manifest.

**Gate: grounding (live only).** Stop the test case for a human when any of these is true:

- The manifest's `status` is `FAILED`.
- The test case's map has `status: "partial"`: a state was blocked, not deliberately skipped,
  whatever the ratio says.
- `grounding_summary.grounded / (grounded + contract_note)` is below the threshold.

Gate on `grounded`, not `grounded_this_run`. Cache and page-object reuse are the point of the
shared cache; a map served from it is not less grounded.

**Why 0.8, and why `partial` stops regardless.** A threshold of `1.0` would stop every
contract with a legitimate deliberate fallback, such as a step behind a destructive action the
probe refuses to take, or a confirmation page after it. Those fallbacks are the probe working
as designed, and in a typical contract they are a small share of the locators. At `0.8`, a map
must have grounded at least four of every five locators against a live snapshot, so the
writer works mostly from verified locators and the unverified rest are visible to the
reviewer and the runner. The `partial` rule covers what a ratio cannot: a single blocked state
can hold few locators and still mean the journey is broken.

### 4. Write, review, run: the feedback loop

Run rounds `n = 1, 2, ...` up to the round cap.

1. **Write.** Spawn `playwright-test-writer` with the contract, the current map, the framework root,
   and `locator-cache/`. From round 2 on, also pass the previous round's review report and
   run report as its feedback reports. Persist the manifest. A `FAILED` manifest stops the
   test case.
2. **Review.** Spawn `playwright-test-reviewer` with the spec(s) from the writer's manifest
   and the contract. **Never pass it the locator map.** From round 2 on, pass the previous
   round's review report as its prior report path, so its `findings_ledger` and
   reclassification discipline apply. Persist the report.
   - `FAIL`: if any blocking finding is about the contract itself (ambiguous, missing, an
     unresolved open question), stop for a human. Otherwise go to the next round; the
     writer gets this report as feedback. Do not run the runner on a spec that failed review.
   - `PASS`: continue to the runner.
3. **Run.** Spawn `playwright-test-runner` with the spec(s), the framework root,
   `runs/<testcase>/round-<n>/` as the run output directory, the current map and contract, the
   `storageState` and its env var name, and the env file with its variable names. In `trace`
   mode also pass trace `on`, so every attempt keeps a trace of each test. Persist the
   report, then append one entry per attempt in the report's `run.attempts` (a `FAILED`
   report can carry attempts too) to
   `runs/index.jsonl`: the test case, the round, the attempt number and folder, the attempt's
   result, the per-test outcomes (`null` for an `infra-failure` attempt), the locator-map
   version the run used, the trace files the report lists for that attempt (`[]` when
   tracing was off), and the path of the runner report you saved.
   Never rewrite or remove earlier entries; the index is how a later step finds a test case's
   runs without walking folders. One JSON object per line, for example:

   ```json
   {"testcase": "adds-a-todo-and-completes-it", "round": 2, "attempt": 1, "dir": "runs/adds-a-todo-and-completes-it/round-2/attempt-1", "result": "tests-ran", "outcomes": {"pass": 1, "fail": 0, "flaky": 0, "skipped": 0}, "map": "locator-maps/adds-a-todo-and-completes-it-locator-map.r2.json", "traces": [], "report": "reports/adds-a-todo-and-completes-it/run-round-2.json"}
   ```

   A `FAILED` report (missing Playwright, browsers, env, or
   inputs) stops everything: no later round can fix it.
4. **Route** on the runner's tests:
   - Every test `pass` (or `skipped` by design): **done** for this test case.
   - Any failure routed `human`: stop for a human with the evidence. For a failure classified
     `obstructed` (something covered the page, such as an ad interstitial or a consent banner),
     say what covered it and propose the environment fix, for example blocking the same ad
     hosts the probe blocks by default in the user's Playwright config. Re-run only after the
     user has made the change; never route an obstructed page to the probe or the writer.
   - Any failure routed `probe`: ground the failing locator again before the next round,
     writing the **next** round's map (after round 2, `{testcase}-locator-map.r3.json`), so the
     earlier map is kept and the new one becomes the current map. Pass **neither** the
     page-objects dir nor the locator cache, so the probe grounds afresh instead of replaying
     the locator that just failed.
     - `trace` mode: spawn the probe with `mode: trace` and `scope: failing`, the contract, its
       current map, this round's runner report, the framework root, and `scratch/`. It grounds
       only the implicated element(s), from the snapshot the run's trace recorded at the
       failure, with no URL and no live app. If any implicated entry comes back a
       `contract-note`, stop for a human: the trace could not resolve it. Say so, and offer
       `--locators=live` as the way to ground it against the app; never switch to live
       yourself. Otherwise go to the next round. There is no ratio gate here: the map still
       holds translated entries that the runs prove.
     - `live` mode: spawn the probe with `mode: live`, the base URL, and the `storageState`. If
       the new map's entry for the failing element is still a `contract-note`, stop for a
       human: the probe cannot reach it either. Then apply the grounding gate to the new map
       and go to the next round.
   - Any failure routed `writer`, or any `flaky` test: go to the next round; the writer gets
     this run report as feedback. A flaky test is not done.

   When failures in one run carry different routes, `human` wins over `probe`, and `probe`
   wins over `writer`. There is no point fixing the spec against a map that is about to change.

**Round cap: 3.** Stop for a human when round 3 ends without a clean run. Three rounds give
the writer two chances to act on feedback. The reviewer's raise-first rule means a blocking
finding on unchanged code had to be raised in the round that first saw it, so new blocking
findings after round 2 should come only from new code. A failure that survives three rounds is
usually a contract that doesn't match the app, an environment problem, or a real app defect,
and none of those gets better with a fourth pass at the spec.

### 5. Report

**Re-run done tests whose files changed.** A test case is proven only against the files it ran
with. If a later writer manifest modified a file a done test case's spec uses (a shared page
object, fixture, data module, or the config), that test case is no longer proven. Before the
summary, spawn the runner once over all such specs together, with `runs/batch-<b>-final/` as
the run output directory, and save its report as `reports/batch-<b>/run-final.json`. Index its
attempts like any other run, with `"testcase": "batch-<b>-final"`. A failure here goes in the summary with its route; it
does not start another round.

When every test case is done or stopped, give the user one summary:

- per test case: done, or stopped with the stage, the reason, and the report that shows it;
- the specs and files the writer created or modified;
- every `flaky` test and every non-blocking open question still standing;
- the paths of `reports/`, `runs/`, and `runs/index.jsonl`.

## Upgrade carried locators: `/migrate --upgrade`

A test that passes on carried locators is migrated: the run proved them. Upgrading them to
grounded, user-facing locators is optional polish, run as its own invocation over an existing
migration dir, never inside a migration. It drives nothing: the probe grounds from the traces
the passing runs already saved.

1. **Find what to upgrade.** For each test case, take its current map (the highest `r<n>` in
   `locator-maps/`), and skip the test case unless the map has `translated` entries. From
   `runs/index.jsonl`, take the test case's latest entry with at least one pass, no `fail`, no
   `flaky`, and a non-empty `traces` list, and the runner report it points to. A test case with
   no such run has no passing trace: report it as not upgradable and skip it.
2. **Ground.** Spawn one `playwright-dom-probe` for all of them with `mode: trace` and
   `scope: carried`: each contract with its current map and that passing run report, the
   framework root, `scratch/`, `locator-cache/`, and the page-objects dir when it exists. Each
   test case's map is written as `r<n'>`, where `n'` is one more than the highest round in its
   index entries. Save the manifest as `reports/batch-<b>/upgrade-round-1.json`.
3. **Show the result.** Per test case: each carried locator upgraded (old → new, or confirmed as
   it stands) and each left carried, with its reason. If nothing was upgraded, stop here.
4. **Ask before applying.** Applying rewrites page objects and re-runs specs. Suggest that the
   user commit first, so they can revert the upgrade with git (you never touch git), and wait
   for their go-ahead.
5. **Apply.** For each test case with upgraded entries, run the feedback loop (step 4) from round
   `n'` with the new map as the current map, the round cap counted from `n'`. The writer swaps
   each grounded locator and removes its marker (Rule 28); the reviewer and the runner check the
   result, and failures route as in step 4. Then finish with step 5, which re-runs every done
   test whose shared files the upgrade changed.

## Rules for the orchestrator

- **Spawn, don't do.** Every contract, map, spec, review, and run comes from its agent. That
  includes the framework: never edit its config, specs, page objects, fixtures, or data
  yourself, even to fix an environment problem a run exposed. Propose the change to the user
  and continue only after they make it or tell you to.
- **Follow the runner's route.** If you think a route is wrong, tell the user why, with the
  evidence; don't act on your own reading instead.
- **Don't widen inputs.** Pass each agent exactly what its Invocation Contract lists. In
  particular, the reviewer never receives a locator map or a runner report.
- **Never handle secret values.** Pass the env file and `storageState` as paths. Don't read
  them, and don't paste their contents into an agent prompt.
- **Keep the migration dir out of version control.** The runner redacts what it reports, but
  Playwright writes its raw report, log, traces, and screenshots under `runs/` unredacted.
  Add the migration dir to `.gitignore` before the first run.
- **Ask before a destructive run.** The runner runs against the real app. Before the first run,
  in each `/migrate` invocation, of a spec whose contract has a `destructive` step (it creates, posts, pays for, or deletes
  something), tell the user what it will change and wait for their go-ahead. Read-only specs
  run without asking.
- **Live only on request.** Never pass `mode: live` to the probe unless the user asked for live
  probing. When trace grounding stops a test case, offer live; don't take it.
- **Stop means stop.** When a gate stops a test case, tell the user why and what would let it
  continue. Don't relax a threshold or skip a stage to get to green.

## Out of scope

This skill migrates tests. Removing the old framework's dependencies, config files, CI jobs,
and the source tests themselves is left to you, once the migrated suite has proven itself.
