---
name: locator-map
description: Canonical definition of the locator map — the artifact the probe (grounding from a saved trace or, on request, the live app) and the translator produce and the writer consumes — and of the Tier 2 locator cache. Defines the persisted file envelope, the per-entry shape, every entry-level enum, the Tier 1 rule, and the cache file. Preload into the probe, the translator, the writer, and the runner via their skills: field; the reviewer must not preload it. Not for direct human use.
user-invocable: false
---

# Locator Map — Canonical Shape

This skill is the **single source** for the locator map's shape and the Tier 2 cache's. The
probe writes maps in this shape, whether it grounds from a saved trace or from the live app, and
so does the translator; the writer reads them. None of them restates it — all preload this skill, so the
definition is injected into each agent's context at startup and there is no second copy to
keep in sync. The runner also preloads it, read-only, to tell a probe fault from a writer
fault when a locator fails at runtime.

The **reviewer does not preload this skill, and must not.** It sees only the spec and the
contract — never the locator map — so its verdict stays unbiased by how the probe resolved
a locator. See [`DESIGN.md`](../../DESIGN.md).

This skill defines **what the artifact is**. How to *produce* it (driving, grounding,
persistence, filename patterns, coverage accounting) belongs to the probe. How to *consume*
it (resolution order, fallback handling, reporting) belongs to the writer. How to *diagnose* a runtime failure against it belongs to
the runner.

## File envelope

One JSON file per contract:

```json
{
  "contract": "path/to/contract.md",
  "testcase": "tc-002-login-user",
  "status": "completed | partial",
  "grounding_summary": {
    "grounded": 26,
    "grounded_this_run": 18,
    "contract_note": 2,
    "translated": 0,
    "unobserved_states": []
  },
  "entries": [ /* one entry per source locator or contract page-state target */ ]
}
```

`status` and `grounding_summary` are written by the map's producer and read by callers gating
on coverage. `grounded + contract_note + translated` equals the number of entries.
`grounded_this_run` counts the entries this run grounded itself (`tier_source` `live` or
`trace`), as opposed to reusing them. A map from the translator has `grounded_this_run: 0`. A
live probe map has `translated: 0`. A trace probe map carries forward every entry of the map it
started from, so it can still hold `translated` entries it could not ground. The writer
consumes `entries`.

## Entry shape

One entry per source locator or contract page-state target:

```json
{
  "contract": "path/to/contract.md",
  "source": {
    "selector": "//original-or-css-selector",
    "source_type": "xpath | css | testid | strategy | contract-note | inferred",
    "contract_step": "Step 7: click Login"
  },
  "replacement": {
    "locator": "page.getByRole('button', { name: 'Login' })",
    "rung": "role | label | placeholder | text | alt | testid | structural | original",
    "resolves_to_one": true
  },
  "grounding": "live-snapshot | contract-note | translated",
  "tier_source": "page-object | cache | live | trace | translation",
  "page": "HomePage",
  "state": "default",
  "auth_context": "anonymous",
  "provenance": "contract-hint-confirmed | derived-fresh | fell-back-with-reason | contract-note | translated",
  "fallback_reason": null,
  "state_reached": true,
  "notes": []
}
```

## Field semantics

`grounding` tells the writer how the locator was established, and pins the accompanying
field values:

- `"live-snapshot"` — the locator matched exactly one element in a snapshot of a reached page
  state of the real app: one the probe took live, or one recorded in the saved trace of a real
  test run. `resolves_to_one: true` and `state_reached: true` are mandatory. The writer may rely
  on it as snapshot-verified; `tier_source` says which kind of snapshot it was.
- `"contract-note"` — the locator was NOT snapshot-verified: the state was reachable only
  via a destructive step the probe does not perform, the journey to reach the state failed
  after one retry, a gated `storageState` was unavailable, no candidate matched exactly one
  element in the snapshot, or, grounding from a trace, the run never recorded the element's
  state. `resolves_to_one: false` and `state_reached: false` are
  mandatory, and `replacement.locator` is the contract's proposed locator verbatim,
  unverified. How the writer handles an unverified locator is the writer's rule, not this
  document's.
- `"translated"`: the locator was converted from the source suite's selector without any
  browser, by the translator. It was never checked against a page: the source suite's history
  is the only evidence it works. `resolves_to_one: false` and `state_reached: false` are
  mandatory, `tier_source` is `"translation"`, `provenance` is `"translated"`, and
  `source.selector` holds the source selector verbatim. The generated code carries it as a
  carried-over locator (Rule 28 of the canonical rules), and the runtime run is what proves
  it.

`source.selector` holds the source selector exactly as the contract records it. `source_type`
says what kind it is: `strategy` for a framework locator call such as `By.id("name")`,
`By.linkText("Sign in")`, or `cy.contains('Save')`, kept whole so the strategy is not lost.

`tier_source` records how *this run* resolved the locator: `page-object` (reused from
Tier 1), `cache` (reused from Tier 2), `live` (grounded this run in a snapshot the probe took by
driving the app), `trace` (grounded this run in a DOM snapshot from a saved Playwright trace of a
test run), or `translation` (converted from the source selector, with no page involved). It is
auditing metadata — it does not change how the writer treats the locator. A `trace` entry's
`notes` name the trace, the action, and the snapshot phase it was grounded from.

`page`, `state`, and `auth_context` are the Tier 2 cache key. They are required whenever the
cache is in use so an entry is addressable.

## Tier 1: page objects

A locator the framework's page objects already define for an element is a Tier 1 source: a
producer may reuse it instead of grounding or translating the element again. **A locator marked
`// carried:` (Rule 28 of the canonical rules) is never a Tier 1 source.** It was never checked
against a page, so reusing it as grounded would relabel an unverified locator as verified, and
hide it from every later step that grounds carried locators. Treat a marked locator as a Tier 1
miss.

## Tier 2: the locator cache

The locator cache is a directory shared by every test case in a migration. Producers read it
before grounding; only the probe writes it, after grounding a locator itself. One file per page,
named `<page>-locator-cache.json` (a slug of the page identity), accumulates that page's
grounded locators across states and auth-contexts:

```json
{
  "page": "HomePage",
  "entries": [
    {
      "state": "default",
      "auth_context": "anonymous",
      "element": "subscriptionEmailInput",
      "replacement": {
        "locator": "page.getByRole('textbox', { name: 'Your email address' })",
        "rung": "role",
        "resolves_to_one": true
      },
      "provenance": "derived-fresh",
      "tier_source": "live | trace",
      "grounded_at": "2026-06-30T00:00:00Z"
    }
  ]
}
```

- **Key.** An entry is identified by `state` + `auth_context` + `element` within its page file.
  A fresh grounding for an existing key overwrites that entry. An `anonymous` entry never
  answers an authenticated lookup, or the reverse: the rendered DOM differs by auth.
- **Only grounded locators are cached.** Write an entry only for a `live-snapshot` result
  grounded this run (`tier_source` `live` or `trace`). Never cache a `contract-note` or a
  `translated` entry: neither was checked against a page.
- **No secrets or PII.** The locator map's hygiene applies: never write credential values,
  tokens, cookies, or DOM-captured PII into a cache file. Describe PII-bearing elements
  generically.
- **Read-tolerant.** A missing or unparseable cache file is an empty cache, not a failure. The
  cache is an optimization, never a correctness dependency.

## What you MUST do

- **Treat this shape as authoritative.** Do not restate these field names or enum values in
  an agent definition, and do not accept a locally remembered variant over what is written
  here.
- **Never fall back to a remembered shape.** If what you recall disagrees with this document,
  this document wins. The case where this content is absent entirely is caught by the static
  failure gates in the probe, the translator, the writer, and the runner, not here — a guard cannot live inside the content
  whose absence it guards.
