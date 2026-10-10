---
name: migrate
description: Drive a test migration to Playwright through the plugin's sibling agents in order (planner, drafter, probe, writer, reviewer, runner), gating on locator grounding, routing review and runtime failures back to the right stage, and stopping for a human after a fixed number of rounds. Run from the main session, which spawns each agent in turn; no agent calls another. Use when the user asks to migrate a suite, a test file, or a contract to Playwright end to end.
argument-hint: "<source suite root or test file> [--framework-root=tests] [--base-url=...] [--threshold=0.8] [--rounds=3]"
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
| App base URL | yes | | probe |
| `storageState` path, and the env var name the framework reads it from | when any state is gated | | probe, runner |
| Env file and required variable names | no | none | runner |
| Migration dir (contracts, maps, cache, reports) | no | `.playwright-migration/` | all |
| Grounding threshold | no | `0.8` | probe gate |
| Round cap | no | `3` | feedback loop |

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
- `reports/<testcase>/`: every agent's JSON, written by you as `<stage>-round-<n>.json`
- `runs/<testcase>/round-<n>/`: the runner's run output directory for that round. The runner
  creates one `attempt-<k>/` folder inside it per Playwright invocation, so every run's report,
  log, and traces are kept side by side and nothing is overwritten.
- `runs/index.jsonl`: your append-only index of every run attempt, one JSON line each (see below)

Create these directories (and each round's `runs/` directory) before the agent that needs them
runs. The drafter, the probe, and the runner require their output directories to exist and
fail statically when they don't; creating them is your job, not theirs.

You persist every agent's JSON response to `reports/` before acting on it. Later stages take
those files as inputs, and the files are the audit trail of the migration.

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

- Any `blocking` open question: **stop this test case** and put the questions to the user.
  Continue only after the contract has been edited to answer them. Other test cases may go
  ahead.
- `non-blocking` questions: show them to the user and continue. The writer carries them as
  risks.

### 3. Probe

Spawn `playwright-dom-probe` with the contract(s), the base URL, the `storageState`,
`locator-maps/` as the output directory, the `r1` filename pattern above, `locator-cache/`,
and the page-objects dir when it exists. Persist the manifest.

**Gate: grounding.** Stop the test case for a human when any of these is true:

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
   `storageState` and its env var name, and the env file with its variable names. Persist the
   report, then append one entry per attempt in the report's `run.attempts` (a `FAILED`
   report can carry attempts too) to
   `runs/index.jsonl`: the test case, the round, the attempt number and folder, the attempt's
   result, the per-test outcomes (`null` for an `infra-failure` attempt), the locator-map
   version the run used, and the path of the runner report you saved.
   Never rewrite or remove earlier entries; the index is how a later step finds a test case's
   runs without walking folders. One JSON object per line, for example:

   ```json
   {"testcase": "adds-a-todo-and-completes-it", "round": 2, "attempt": 1, "dir": "runs/adds-a-todo-and-completes-it/round-2/attempt-1", "result": "tests-ran", "outcomes": {"pass": 1, "fail": 0, "flaky": 0, "skipped": 0}, "map": "locator-maps/adds-a-todo-and-completes-it-locator-map.r2.json", "report": "reports/adds-a-todo-and-completes-it/run-round-2.json"}
   ```

   A `FAILED` report (missing Playwright, browsers, env, or
   inputs) stops everything: no later round can fix it.
4. **Route** on the runner's tests:
   - Every test `pass` (or `skipped` by design): **done** for this test case.
   - Any failure routed `human`: stop for a human with the evidence.
   - Any failure routed `probe`: re-run the probe for this contract before the next round,
     with the same output directory and the **next** round's filename pattern (after round 2,
     `{testcase}-locator-map.r3.json`), so the earlier map is kept and the new one becomes the
     current map. Run it **without** the page-objects dir and the locator cache, so it
     re-grounds live instead of replaying the locator that just failed. If the new map's
     entry for the failing element is still a `contract-note`, stop for a human: the probe
     cannot reach it either. Then apply the grounding gate to the new map and go to the next
     round.
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

When every test case is done or stopped, give the user one summary:

- per test case: done, or stopped with the stage, the reason, and the report that shows it;
- the specs and files the writer created or modified;
- every `flaky` test and every non-blocking open question still standing;
- the paths of `reports/`, `runs/`, and `runs/index.jsonl`.

## Rules for the orchestrator

- **Spawn, don't do.** Every contract, map, spec, review, and run comes from its agent.
- **Don't widen inputs.** Pass each agent exactly what its Invocation Contract lists. In
  particular, the reviewer never receives a locator map or a runner report.
- **Never handle secret values.** Pass the env file and `storageState` as paths. Don't read
  them, and don't paste their contents into an agent prompt.
- **Keep the migration dir out of version control.** The runner redacts what it reports, but
  Playwright writes its raw report, log, traces, and screenshots under `runs/` unredacted.
  Add the migration dir to `.gitignore` before the first run.
- **Stop means stop.** When a gate stops a test case, tell the user why and what would let it
  continue. Don't relax a threshold or skip a stage to get to green.

## Out of scope

This skill migrates tests. Removing the old framework's dependencies, config files, CI jobs,
and the source tests themselves is left to you, once the migrated suite has proven itself.
