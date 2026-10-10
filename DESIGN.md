# Design

Why Playwright Guardrails is shaped the way it is. For what it does and how to
run it, see [`README.md`](README.md).

## The problem

Teams migrating an existing suite (Selenium, Cypress, WebDriver, …) to Playwright with
AI agents hit a consistent failure: even well-prompted agents drift from best
practices. The generator
produces inconsistent code, and a reviewer has nothing authoritative to check
against — so violations pass silently and the migration becomes a pile of
individually-reviewed one-offs instead of a consistent suite.

The fix is not a smarter prompt. It is **specialized single-role agents with fixed
inputs and one canonical rule set**, so rule adherence is checkable rather than
implicit and an agent can't prose its way around a violation.

## Canonical sources, loaded by preload

The framework rules and the review checklist live in exactly one place —
[`skills/rules/`](skills/rules/) — and are the single source the probe, writer, and
reviewer all consume. Nothing duplicates them. The checklist references rule numbers
rather than restating the rules. The planner, drafter, and runner don't preload them: none of
them writes or judges Playwright code.

Agents do **not** receive rule paths from the caller. Each lists the
`playwright-guardrails:rules` skill in its frontmatter `skills:` field. Preloading
that skill injects the manifest and the skill's base directory into the agent at
startup; the agent's first step is to read the two named files. If either is missing
or empty, the agent returns a blocking failure rather than generating or reviewing
against a partial rule set.

This keeps rules out of the orchestrator's context, removes the path-guessing that
once let reviews silently pass with no rules loaded, and avoids needing the `Skill`
tool — the agent loads rules with `Read` alone.

The same mechanism carries the **locator map's shape**. The map is the interface between the
probe (which writes it) and the writer (which reads it), so its definition lives in
[`skills/locator-map/`](skills/locator-map/) rather than inside either agent — both preload
it, and neither restates it. Keeping it in the producer's file was the obvious shortcut and
the wrong one: it would make the writer subordinate to a sibling it never calls, and the
reference would not be real, because an agent has no path to a sibling's definition — only to
the skills it preloads. Unlike the rules manifest, this skill carries its content inline, so
there is no file to read and no second load step. The runner preloads it too, read-only, to
route locator failures (see Runtime verification). The reviewer does **not** preload it, which
is what turns its blindness to the map into a declared decision rather than an omission.

The **contract's shape** is carried the same way, in
[`skills/contract/`](skills/contract/). The contract is the one input every agent shares, but
until 0.5.0 its shape existed only as an example, and each agent carried its own list of what
to look for in one. The probe described direct and driven steps, while the writer and the
reviewer each listed intent, steps, data, and outcomes, three copies that could drift the way
the locator map's copies did. The skill now defines the sections, the step markers, the
origin-locators table, the risk vocabulary, and open questions once. Every agent that reads a
contract preloads it, the reviewer included: the contract is the reviewer's own input, and only
the locator map is kept from it. The planner preloads it only for the risk vocabulary.

The contract skill is strict about what the drafter writes and lenient about what consumers
read. A hand-written contract with different headings still works, because consumers find
sections by meaning; only a contract with no findable intent, steps, or outcomes, or with an
unresolved `blocking` open question, is too ambiguous to use. That keeps existing contracts
working while giving drafted ones a single shape.

## Seven roles, kept as siblings

The pipeline is seven independent single-role sub-agents. **None of them calls another.** An
orchestrator runs them in sequence: your own session, a migration pipeline, or the bundled
[`migrate`](skills/migrate/SKILL.md) skill (see The reference orchestrator).

1. [`playwright-migration-planner`](agents/playwright-migration-planner.md): source suite →
   inventory, risk flags, and a dependency-ordered sequence.
2. [`playwright-contract-drafter`](agents/playwright-contract-drafter.md): source tests +
   helpers → one contract file per test case.
3. [`playwright-dom-probe`](agents/playwright-dom-probe.md): contract + live app → a
   persisted `locator_map` file per test case. In translate mode,
   [`playwright-locator-translator`](agents/playwright-locator-translator.md) takes this
   slot instead: contract + framework config → the same map shape, unverified.
4. [`playwright-test-writer`](agents/playwright-test-writer.md): contract + `locator_map` +
   framework → spec and supporting files.
5. [`playwright-test-reviewer`](agents/playwright-test-reviewer.md): spec + contract →
   PASS / FAIL.
6. [`playwright-test-runner`](agents/playwright-test-runner.md): spec + framework → a
   per-test runtime outcome, with each failure classified and routed.

Two design choices make this work:

- **The probe is the only agent that explores the live app.** It drives the contract
  journey to reach each page state, snapshots it, and grounds each locator to the
  highest user-facing rung matching exactly one element, then stops. It does not
  re-drive to verify individual locators after the snapshot; that post-snapshot
  verification loop is the failure mode this design avoids. Isolating the one
  live-DOM step keeps the rest deterministic and makes locator quality its own
  measurable signal instead of hiding inside the writer's output. The runner also
  launches a browser, but only by running the generated spec. It never explores the app
  or grounds a locator.

- **The writer is a pure function of its inputs.** Because it consumes a persisted map
  instead of probing, its output is reproducible, and a stored map can be replayed straight
  into the writer, skipping the probe, for browser-free generation. A sub-agent driving
  another sub-agent would re-introduce the live-DOM dependency the split exists to remove,
  so the writer never invokes the probe. In a feedback loop it also takes the previous
  round's review and run reports as inputs; it stays a pure function of what it is given.

The reviewer sees **only** the spec and contract — never the locator map — so its
verdict stays unbiased by how the probe resolved a locator.

That isolation has a price, and it is worth naming. Because the reviewer never sees the
map, it cannot tell you that a spec asserts against a page state nothing ever observed.
The probe reports coverage instead: every persisted map carries a `grounding_summary`
distinguishing locators grounded from a live snapshot, locators grounded fresh *this
run*, and locators carried over as unverified contract fallbacks — plus the page states
where nothing was observed at all. A caller can gate on that ratio. Whether the reviewer
should receive a counts-only summary — enough to flag an unobserved page, not enough to
ratify the writer's locator choices — is an open question tracked in
[#3](https://github.com/harigovindarajan/playwright-guardrails/issues/3).

Since 0.5.0 the runner narrows that gap from the other side. A spec that asserts against a
page state nothing observed will usually fail when it runs, and the runner reports it with
evidence, without the reviewer ever seeing the map.

## Drafting contracts is its own role

The README has always said the best first step is to have an LLM draft a contract from the old
test. [`playwright-contract-drafter`](agents/playwright-contract-drafter.md) makes that step a
sibling with fixed inputs: source tests in any framework, plus the source suite's shared
helpers so it can see what a call like `cy.login()` actually does. It writes contracts and
returns a manifest of what it wrote and what it could not determine.

It is deliberately not a translator. Mechanical selector-for-selector porting carries the
source's brittleness straight into the new suite, which is the problem the probe exists to
solve. So the drafter copies source selectors verbatim into the origin-locators table and
leaves upgrading them to the probe, and it marks where the source leaned on its framework
(Cypress retry-ability, `cy.session`, aliased network waits, Selenium sleeps, frames, tabs,
order dependence) as **risk annotations** rather than rewriting them. What it cannot determine
from the source becomes an explicit **open question**, never an invented step: a guessed step
in a complete-looking contract would be reproduced faithfully by every stage after it.

The drafter never touches the live app and never writes Playwright code, so it needs neither
a browser nor the rules.

## Translate mode: carry locators over first, ground them later

The probe grounds every locator against the live app before the writer generates anything.
That is the right default, but it is not always the right first step. An established suite's
selectors already work against the real app; they only fail Playwright's locator strategy.
Probing every one of them up front costs time, and it needs a live environment with test data
and auth before a single test has been migrated.

Translate mode is the alternative first pass. [`playwright-locator-translator`](agents/playwright-locator-translator.md)
converts each origin locator into the Playwright locator that selects the same thing, with no
browser, and writes it into the same locator-map shape with `grounding: "translated"`. The
writer is unchanged except that it marks every translated locator `// carried: <source
selector>`, and Rule 28 makes a marked locator residual risk the test run settles, not a
Rule 1 violation. The pipeline runs through to execution, and a test that runs green is
migrated.

The translator is a separate agent rather than a probe flag because it is a different role
with different tools: the probe grounds against observed DOM and needs a browser, the
translator converts text and needs neither a shell nor the app. It translates only what a
source selector already states. Turning a CSS selector into a role locator because the
contract's description names a role would be grounding without evidence, so the translator
leaves that to a later grounding step and records the description for it.

The runner keeps a trace of every run in translate mode. A later release grounds carried
locators from those traces, offline, and writes an upgraded map version; see
[#8](https://github.com/harigovindarajan/playwright-guardrails/issues/8).

## Planning a suite is a separate role from drafting a test

A contract describes one test. A migration is a suite, and the order matters: a suite whose
every test logs in through one custom command should migrate a test that exercises that
command first, so its page objects and fixtures land once and every later probe can reuse them.
[`playwright-migration-planner`](agents/playwright-migration-planner.md) inventories the source
suite (every test file, its test count, the shared dependencies and who uses them) and returns
a dependency-ordered sequence: shared dependencies first, each with the simplest test that
seeds it, then simple test files, then complex ones.

Its per-file risk flags come from the contract skill's risk vocabulary, which it preloads for
that alone. One vocabulary means a flag the planner raises for a file reappears as an
annotation in the contracts the drafter writes for it, rather than as a near-synonym from a
second list.

The plan carries **no time estimates**. A per-test minute figure cannot be traced to anything
in the source, and a plan that invents numbers trains people to trust the parts that are also
invented. Complexity is a sequencing input, not a cost.

Planning stays separate from drafting so that each agent keeps one role: the planner reads
the whole suite shallowly, the drafter reads one test deeply.

## Runtime verification, and why the runner may read the map

Nothing upstream executes a spec. The writer has no shell and the reviewer reads files, so a
spec can earn a PASS and still fail the first time it runs. The reviewer already defers
anything it can only confirm by running the test to a `verify` stage; that stage is
[`playwright-test-runner`](agents/playwright-test-runner.md). It runs the specs with the
project's own Playwright install, reads the JSON report, and returns a per-test outcome. For
each failure it gives a classification, the evidence behind it, and a suggested route: back
to the probe, back to the writer, or to a human. It is run-only, the way the reviewer is
review-only. It never edits a file, never installs anything, and never touches
`playwright.config`; every option goes on the command line.

**Two retry layers, kept apart.** Playwright retries a failing test once by default, passed
as `--retries` so the result does not depend on the config. That one retry is what makes
Playwright report `flaky`, and the runner reports a flaky test as flaky, never as a pass. The
runner itself reruns the whole invocation at most once, and only when infrastructure failed
before any test body ran. Letting the agent rerun failing tests would stack a second retry
layer on the first and hide exactly the flakes the first layer exists to surface.

**The runner preloads the locator map; the reviewer still doesn't.** The reviewer is blind
to the map so that its judgment of the writer's locator choices is not anchored on the
probe's. The runner makes no such judgment. Its evidence is what the runtime did, and the map
cannot change that. What the map does change is the route. A locator that fails at runtime
is a probe fault when the map grounded it from a snapshot that no longer matches, or when the
map only carried it as an unverified `contract-note`. It is a writer fault when the spec did
not use the locator the map grounded. Without the map those cases are indistinguishable, and
every locator failure would have to go to a human, which defeats the point of routing. So the
map path is an optional runner input, the runner preloads the shape like the probe and the
writer do, and without a map it routes locator failures to a human rather than guessing.

The cost is that the runner's routing now trusts the map's record of what was grounded. If a
map is wrong about its own provenance, the runner will route to the wrong stage. That is a
bounded cost: a misrouted failure comes back on the next round, and the orchestrator's round
cap stops the loop.

**The runner does not preload the rules.** It reports outcomes, not rule adherence, and
loading the full rule set would invite it to issue findings that belong to the reviewer.

## The reference orchestrator

Siblings need something to sequence them, and until 0.5.0 that was left entirely to the
caller. [`skills/migrate/`](skills/migrate/SKILL.md) is a user-invocable skill that documents
and drives the whole sequence. The main session runs it and spawns each agent in turn, so it
is a session following a procedure, not an agent calling agents. The procedure is plan, draft,
probe, then a write → review → run loop, with three gates:

- **Open questions.** A contract with a `blocking` open question stops before the probe. The
  drafter was told never to guess; the orchestrator must not guess in its place.
- **Grounding.** The probe's map must have `grounded / (grounded + contract_note)` of at least
  `0.8` by default, and a `partial` map stops whatever its ratio. A threshold of `1.0` would
  stop every contract with a legitimate deliberate fallback, such as a step past a
  destructive action the probe refuses to take. `0.8` means the writer works mostly from
  verified locators. The `partial` rule catches what a ratio cannot: one blocked state can
  hold few locators and still mean the journey is broken.
- **Rounds.** The feedback loop stops for a human after `3` rounds. That gives the writer two
  chances to act on feedback. The reviewer's raise-first rule means new blocking findings
  after round 2 should only come from new code, so a failure that survives three rounds is
  usually a contract that doesn't match the app, an environment problem, or an app defect,
  none of which another pass at the spec can fix.

Routing follows the runner's suggested route, with `human` over `probe` over `writer` when
failures in one run disagree. Two agent changes made routing possible. The writer gained an
optional feedback-reports input, because a pure function given the same inputs twice
produces the same spec twice. And on a `probe` route, the orchestrator re-runs the probe
without its reuse tiers, because the page objects and cache would otherwise hand back the
very locator that just failed.

**Nothing a round produces is overwritten.** Each runner invocation keeps its report, log,
and traces in its own `runs/<testcase>/round-<n>/attempt-<k>/` folder, and the locator map is
versioned by round (`{testcase}-locator-map.r<n>.json`), so a re-probe adds a version instead
of replacing one. An append-only `runs/index.jsonl` records, for every attempt, the test case,
the round, the outcome, and the map version it ran with. A failure in round 3 can then be
compared with what round 1 saw, and later tooling can work from the full history of every
test case instead of only its last state.

From round 2, the reviewer receives its own prior report, so the `findings_ledger` and its
reclassification discipline apply across rounds. It never receives the map or a run report.

**Out of scope.** The orchestrator migrates tests. Removing the old framework's
dependencies, config files, CI jobs, and the source tests themselves is left to the user.

## Mechanical vs. reasoning roles

The probe, translator, and runner are mostly **mechanical**: the probe extracts (drive →
snapshot → ground), the translator converts a selector to its equivalent, and the runner
runs a command and reads its report. Running them at high reasoning
effort buys little and dominates wall-clock. The other four carry the migration's quality and
run at `high`. The drafter reads what each existing test intends, and every later stage
inherits that reading, so a shallow contract costs more than the extra reasoning does. The
writer turns contract and map into code that has to fit the framework's conventions and pass
review. The reviewer weighs findings and issues a verdict, and the planner orders a whole
suite.

Per-role model and effort are pinned in each agent's frontmatter so they are enforced by the
plugin rather than left to the session default:

| Agent | `model` | `effort` | Why |
| --- | --- | --- | --- |
| `playwright-migration-planner` | `opus` | `high` | Suite-wide dependency resolution and ordering is reasoning, and a wrong order costs every later stage. It runs once per migration, so the cost is paid once. |
| `playwright-contract-drafter` | `sonnet` | `high` | Synthesizing what an existing test intends, through its helpers, is the input every later stage builds on. |
| `playwright-locator-translator` | `sonnet` | `medium` | Converting a selector to its equivalent is mostly mechanical, but it reads the target config and must refuse every conversion the source does not support. |
| `playwright-dom-probe` | `sonnet` | `low` | Mechanical drive → snapshot → ground; scripted and deterministic. |
| `playwright-test-writer` | `sonnet` | `high` | Generation that must match framework conventions, respect the rules, and act on review and run feedback. |
| `playwright-test-reviewer` | `sonnet` | `high` | Weighs tradeoffs: classification and the verdict. |
| `playwright-test-runner` | `sonnet` | `medium` | Running is scripted, but classifying a failure and choosing its route weighs evidence within a fixed set of classes. |

The session advisor is a separate, harness-level lever the plugin cannot pin. For
mechanical probe/writer loops, downgrade or disable it via a project-level
`.claude/settings.json` override so an extraction run doesn't silently inherit a
heavyweight advisor.

## Principle

The generated framework should stay thin, expressive, and readable as business
journeys. The rules document is canonical; everything else references it.
