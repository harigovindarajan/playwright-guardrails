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
rather than restating the rules.

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
there is no file to read and no second load step. The reviewer does **not** preload it, which
is what turns its blindness to the map into a declared decision rather than an omission.

The **contract's shape** is carried the same way, in
[`skills/contract/`](skills/contract/). The contract is the one input every agent shares, but
until 0.5.0 its shape existed only as an example, and each agent carried its own list of what
to look for in one. The probe described direct and driven steps, while the writer and the
reviewer each listed intent, steps, data, and outcomes, three copies that could drift the way
the locator map's copies did. The skill now defines the sections, the step markers, the
origin-locators table, the risk vocabulary, and open questions once. Every agent that reads a
contract preloads it, the reviewer included: the contract is the reviewer's own input, and only
the locator map is kept from it.

The contract skill is strict about what the drafter writes and lenient about what consumers
read. A hand-written contract with different headings still works, because consumers find
sections by meaning; only a contract with no findable intent, steps, or outcomes, or with an
unresolved `blocking` open question, is too ambiguous to use. That keeps existing contracts
working while giving drafted ones a single shape.

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

## Three roles, kept as siblings

The pipeline is three independent single-role sub-agents. **None of them calls
another** — an external orchestrator runs them in sequence:

1. [`playwright-dom-probe`](agents/playwright-dom-probe.md) — contract + live app →
   a persisted `locator_map` file per test case.
2. [`playwright-test-writer`](agents/playwright-test-writer.md) — contract +
   `locator_map` + framework → spec and supporting files.
3. [`playwright-test-reviewer`](agents/playwright-test-reviewer.md) — spec + contract
   → PASS / FAIL.

Two design choices make this work:

- **The probe is the only agent that touches a browser.** It drives the contract
  journey to reach each page state, snapshots it, and grounds each locator to the
  highest user-facing rung matching exactly one element — then stops. It does not
  re-drive to verify individual locators after the snapshot; that post-snapshot
  verification loop is the failure mode this design avoids. Isolating the one
  live-DOM step keeps the rest deterministic and makes locator quality its own
  measurable signal instead of hiding inside the writer's output.

- **The writer is a pure function of (contract + locator map + framework).** Because
  it consumes a persisted map instead of probing, its output is reproducible, and a
  stored map can be replayed straight into the writer — skipping the probe — for
  browser-free generation. A sub-agent driving another sub-agent would re-introduce
  the live-DOM dependency the split exists to remove, so the writer never invokes the
  probe.

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

## Mechanical vs. reasoning roles

The probe and writer are **mechanical**: the probe extracts (drive → snapshot →
ground), the writer generates from fixed inputs. Neither weighs tradeoffs, so running
them at high reasoning effort buys nothing and dominates wall-clock. The reviewer is
the one role that classifies and weighs evidence.

Per-role effort is pinned in each agent's frontmatter so it is enforced by the plugin
rather than left to the session default:

| Agent                      | `effort` | Why                                                          |
| -------------------------- | -------- | ------------------------------------------------------------ |
| `playwright-dom-probe`     | `low`    | Mechanical drive → snapshot → ground; scripted and deterministic. |
| `playwright-test-writer`   | `medium` | Mechanical generation with some convention-matching.         |
| `playwright-test-reviewer` | `high`   | The only role that weighs tradeoffs (classification, verdict). |
| `playwright-contract-drafter` | `medium` | Reading intent out of source tests and their helpers is interpretive, but bounded by the rule to ask rather than guess. |
| `playwright-test-runner`   | `medium` | Running is scripted, but classifying a failure and choosing its route weighs evidence within a fixed set of classes. |

The session advisor is a separate, harness-level lever the plugin cannot pin. For
mechanical probe/writer loops, downgrade or disable it via a project-level
`.claude/settings.json` override so an extraction run doesn't silently inherit a
heavyweight advisor.

## Principle

The generated framework should stay thin, expressive, and readable as business
journeys. The rules document is canonical; everything else references it.
