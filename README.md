# Playwright Guardrails

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Version](https://img.shields.io/github/v/tag/harigovindarajan/playwright-guardrails?label=version)

A Claude Code plugin that keeps AI-driven test migrations to Playwright on-spec.
It bundles a canonical set of Playwright framework rules and an adversarial review
checklist, canonical shapes for the test contract and the locator map, and seven single-role
sub-agents (a planner, a contract drafter, a probe or a locator translator, a writer, a
reviewer, and a runner) that load those same sources, so planning, locator grounding, generation, review, and runtime
verification run against one authoritative source instead of improvising. A `/migrate` skill
drives them in order.

## Overview

Migrating tests with an AI agent tends to drift: the generator writes inconsistent
code and the reviewer has nothing authoritative to check against. Playwright
Guardrails fixes that with specialized agents that each do one job, take fixed inputs,
and share a single canonical rule set — so rule adherence is checkable rather than
implicit.

The agents are **agnostic to the source framework**. Selenium-to-Playwright is the
battle-tested path, but the same pipeline handles Cypress, WebDriver, or any other
origin: the probe grounds locators against the live DOM rather than translating old
selectors, so it never depends on what the source suite used.

The recommended starting point is a **test contract** — a plain-language description of
what each existing test proves: its intent, the journey and steps, the data it needs,
and the expected outcomes. You can migrate without one, but a contract makes the result
markedly more reliable, so the best first step is to draft a contract from the old test.
`playwright-contract-drafter` does exactly that, and every later agent takes the contract as
its shared input. Its shape is defined once, in the preloaded `contract` skill. See
[`examples/login-contract.md`](examples/login-contract.md) for a worked contract and
[`DESIGN.md`](DESIGN.md) for why it's built this way.

## Agents

### `playwright-migration-planner`

**Focus:** Inventories a source suite in any framework: every test file and its test
count, the shared dependencies (page objects, custom commands, fixtures, helpers) and who
uses them, and per-file risk flags. Returns a dependency-ordered migration sequence: shared
dependencies first, then simple tests, then complex ones. No time estimates.

**When to use:** Once, before migrating a suite.

**Invocation:** Dispatched with the source suite root(s), plus optional include/exclude
patterns and helper paths. Read-only. See the **Invocation Contract** in
[`agents/playwright-migration-planner.md`](agents/playwright-migration-planner.md).

### `playwright-contract-drafter`

**Focus:** Drafts one contract per source test, in the canonical shape. It reads the source
suite's helpers to see what a call like `cy.login()` actually does, copies source selectors
verbatim, flags fragile source mechanics as risk annotations, and turns anything it can't
determine into an explicit open question instead of guessing. It never touches the live app
and never writes Playwright code.

**When to use:** First stage for each test, whenever you don't already have a contract.

**Invocation:** Dispatched with source test path(s), an output directory, and optional
helper paths. See the **Invocation Contract** in
[`agents/playwright-contract-drafter.md`](agents/playwright-contract-drafter.md).

### `playwright-dom-probe`

**Focus:** Grounds a test contract's origin locators — whatever the source framework
used — against the live app.
It drives the contract's journey to reach each page state, snapshots it, and derives
each locator to the highest user-facing Playwright rung that matches exactly one
element — then writes a persisted `locator_map` file per test case.

**When to use:** After a contract exists, whenever its locators need to be resolved
against a running application.

**Invocation:** Dispatched with contract path(s), the live app base URL, a
`storageState` file for gated entry points, and an output directory (plus optional
reuse and block-list inputs). Its **Invocation Contract** section in
[`agents/playwright-dom-probe.md`](agents/playwright-dom-probe.md) is the authoritative
signature. Requires `playwright-cli` on `PATH` — it is the only agent that drives a
browser.

### `playwright-locator-translator`

**Focus:** The browser-free alternative to the probe. It converts a contract's origin
locators straight into the Playwright locators that select the same thing (a test-id
attribute to `getByTestId`, a link-text lookup to a link role), reading the target config's
`testIdAttribute`, and writes them into the same locator-map shape marked `translated`. It
never infers a role or name the source selector doesn't state.

**When to use:** Instead of the probe, when the source suite's locators are trusted and you
want every test migrated and running before spending time on live grounding. The writer
marks each translated locator `// carried: <source selector>` (Rule 28), and the test run is
what proves it.

**Invocation:** Dispatched with contract path(s), the framework root, and an output
directory, plus optional reuse inputs. See the **Invocation Contract** in
[`agents/playwright-locator-translator.md`](agents/playwright-locator-translator.md).

### `playwright-test-writer`

**Focus:** Generates the Playwright spec and supporting framework files from a contract
and the probe's `locator_map`, matching existing page-object, fixture, and config
conventions. It does not drive a browser, so its output is a pure function of its
inputs and reproducible.

**When to use:** After the probe has produced a locator map, and again on each feedback
round.

**Invocation:** Dispatched with contract path(s), locator-map path(s), and the target
framework root, plus the previous round's review and run reports on a feedback round. See
the **Invocation Contract** in
[`agents/playwright-test-writer.md`](agents/playwright-test-writer.md).

### `playwright-test-reviewer`

**Focus:** Reviews a completed spec against its contract and the canonical rules and
checklist, returning a structured PASS / FAIL verdict. It sees only the spec and
contract — never the locator map — so its verdict stays unbiased.

**When to use:** After the writer, before the spec is run or committed.

**Invocation:** Dispatched with spec path(s) and contract path(s) only — it loads the
rules itself. See the **Invocation Contract** in
[`agents/playwright-test-reviewer.md`](agents/playwright-test-reviewer.md).

### `playwright-test-runner`

**Focus:** Runs the spec with the project's existing Playwright install and a JSON reporter,
and reports each test as pass, fail, flaky, or skipped. Each failure gets a classification
(locator, timing, assertion, test data, auth/env, other), the evidence it rests on, and a
suggested route back to the probe, the writer, or a human. Run-only: it never edits files,
never installs anything, and never touches `playwright.config`.

**When to use:** After the reviewer passes a spec. It is the runtime check the reviewer
defers to.

**Invocation:** Dispatched with spec path(s), the framework root, and a run output
directory, plus optional config path, env file, `storageState`, locator map, and contract.
See the **Invocation Contract** in
[`agents/playwright-test-runner.md`](agents/playwright-test-runner.md).

## Skills

- **`migrate`** (user-invocable): the reference orchestrator. Run `/migrate` from your
  session and it drives the agents in order, gates on grounding, routes failures back, and
  stops for you when it should. See [`skills/migrate/SKILL.md`](skills/migrate/SKILL.md).
- **`rules`**, **`contract`**, and **`locator-map`** (preload-only): the canonical rules
  manifest, the contract shape, and the locator-map shape. Agents load them through their
  `skills:` frontmatter; you never invoke them directly.

## The pipeline

The seven agents are independent siblings: **none calls another**. An orchestrator (the
`/migrate` skill, your own session, or a migration pipeline) runs them in sequence:

```text
planner  source suite                    → plan: inventory, risks, order
drafter  source test + helpers           → contract file(s)
probe    contract + live app             → locator_map file(s)
  or translator (translate mode)
         contract + framework config     → locator_map file(s), unverified
writer   contract + locator_map          → spec + framework files
reviewer spec + contract                 → PASS / FAIL
runner   spec + framework                → pass / fail / flaky / skipped, routed

feedback loop, up to 3 rounds:
  writer → reviewer ── FAIL ──────────────────────────► next round
                    └─ PASS → runner ── all pass ─────► done
                                     ├─ route: writer ─► next round
                                     ├─ route: probe ──► re-probe, next round
                                     └─ route: human ──► stop
```

`/migrate` gates the probe's map on its grounding ratio (default 0.8) and stops for a human
after 3 feedback rounds. Both are configurable; [`DESIGN.md`](DESIGN.md) explains the
defaults.

Because the writer consumes a persisted map instead of probing, a stored map can be
replayed straight into the writer — skipping the probe — for deterministic,
browser-free generation.

## Installation

Add this repo as a marketplace, then install the plugin:

```text
/plugin marketplace add harigovindarajan/playwright-guardrails
/plugin install playwright-guardrails
```

For local development, load it directly without installing:

```bash
claude --plugin-dir .
```

The seven agents then appear in `/context` under Custom Agents. `/migrate` appears in the
slash-command menu. The other three skills (`rules`, `contract`, `locator-map`) are
preload-only, so none of them is invoked directly.

## Quick start

The shortest path is the orchestrator:

```text
/migrate cypress/e2e/login.cy.ts --framework-root=tests --base-url=https://staging.yourapp.example
```

It asks for anything it still needs (a `storageState`, an env file), then plans, drafts,
probes, writes, reviews, and runs, and reports what finished and what stopped.

To carry the source suite's locators over instead of probing the live app first, use
translate mode. No live probe runs; the test run is the gate, and every run keeps a trace:

```text
/migrate cypress/e2e/ --framework-root=tests --locators=translate
```

To run the stages yourself, dispatch the agents from your Claude Code session in sequence:

```text
Use playwright-contract-drafter on cypress/e2e/login.cy.ts with helpers in
cypress/support/, output to contracts/.
(Its manifest lists the files it wrote. Below, that is contracts/login-contract.md.)

Use playwright-dom-probe on contracts/login-contract.md against
https://staging.yourapp.example with auth/storageState.json, output to locator-maps/.

Use playwright-test-writer on contracts/login-contract.md with
locator-maps/login-locator-map.json, targeting the framework at tests/.

Use playwright-test-reviewer on tests/login.spec.ts against
contracts/login-contract.md.

Use playwright-test-runner on tests/login.spec.ts in the framework at tests/, run output
to runs/login/, with locator-maps/login-locator-map.json and contracts/login-contract.md.
```

For a whole suite, start with `Use playwright-migration-planner on cypress/` and migrate in
the order it returns.

Each agent's **Invocation Contract** section documents its full signature, including
the optional inputs.

## Requirements

- **Claude Code** with plugin support.
- **`playwright-cli`** on `PATH` — a runtime dependency of the probe for driving the
  app and grounding locators.
- **Playwright and its browsers installed in the target framework.** The runner runs specs
  with `npx --no-install playwright test` and never installs anything; if either is missing,
  it stops with a blocking failure. The planner, drafter, writer, and reviewer need no
  browser.
- A **`storageState`** file for gated entry points — the probe's only auth path; it
  never enters credentials into a form.

## Out of scope

The plugin migrates tests. Removing the old framework's dependencies, config files, CI jobs,
and the source tests themselves is left to you, once the migrated suite has proven itself.

## Learn more

- [`DESIGN.md`](DESIGN.md) — the architecture and the reasoning behind it.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — how to extend the canonical rules.
- [`CHANGELOG.md`](CHANGELOG.md) — release history.

## License

[MIT](LICENSE).
