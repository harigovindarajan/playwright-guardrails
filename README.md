# Playwright Guardrails

A Claude Code plugin that keeps AI-driven test migrations to Playwright on-spec.
It bundles a canonical set of Playwright framework rules and an adversarial review
checklist, and three single-role sub-agents — a probe, a writer, and a reviewer — that
all load those same rules, so locator grounding, generation, and review run against one
authoritative source instead of improvising.

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
markedly more reliable, so the best first step is to have an LLM draft a contract from
the old test. The three agents then take that contract as their shared input. See
[`DESIGN.md`](DESIGN.md) for why it's built this way.

## Agents

### `playwright-dom-probe`

**Focus:** Grounds a test contract's origin locators — whatever the source framework
used — against the live app.
It drives the contract's journey to reach each page state, snapshots it, and derives
each locator to the highest user-facing Playwright rung that matches exactly one
element — then writes a persisted `locator_map` file per test case.

**When to use:** First stage of a migration, whenever a contract's locators need to be
resolved against a running application.

**Invocation:** Dispatched with contract path(s), the live app base URL, a
`storageState` file for gated entry points, and an output directory (plus optional
reuse and block-list inputs). Its **Invocation Contract** section in
[`agents/playwright-dom-probe.md`](agents/playwright-dom-probe.md) is the authoritative
signature. Requires `playwright-cli` on `PATH` — it is the only agent that drives a
browser.

### `playwright-test-writer`

**Focus:** Generates the Playwright spec and supporting framework files from a contract
and the probe's `locator_map`, matching existing page-object, fixture, and config
conventions. It does not drive a browser, so its output is a pure function of its
inputs and reproducible.

**When to use:** Second stage, after the probe has produced a locator map.

**Invocation:** Dispatched with contract path(s), locator-map path(s), and the target
framework root. See the **Invocation Contract** in
[`agents/playwright-test-writer.md`](agents/playwright-test-writer.md).

### `playwright-test-reviewer`

**Focus:** Reviews a completed spec against its contract and the canonical rules and
checklist, returning a structured PASS / FAIL verdict. It sees only the spec and
contract — never the locator map — so its verdict stays unbiased.

**When to use:** Third stage, before committing a migrated test.

**Invocation:** Dispatched with spec path(s) and contract path(s) only — it loads the
rules itself. See the **Invocation Contract** in
[`agents/playwright-test-reviewer.md`](agents/playwright-test-reviewer.md).

## The pipeline

The three agents are independent siblings — **none calls another**. An external
orchestrator (your calling session or a migration pipeline) runs them in sequence:

```text
probe    contract + live app      → locator_map file(s)
writer   contract + locator_map   → spec + framework files
reviewer spec + contract          → PASS / FAIL
```

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

The three agents then appear in `/context` under Custom Agents. The rules skill
is preload-only — the agents load it themselves, so it isn't invoked directly.

## Requirements

- **Claude Code** with plugin support.
- **`playwright-cli`** on `PATH` — a runtime dependency of the probe for driving the
  app and grounding locators. The writer and reviewer need no browser.
- A **`storageState`** file for gated entry points — the probe's only auth path; it
  never enters credentials into a form.

## Learn more

- [`DESIGN.md`](DESIGN.md) — the architecture and the reasoning behind it.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — how to extend the canonical rules.
- [`CHANGELOG.md`](CHANGELOG.md) — release history.

## License

[MIT](LICENSE).
