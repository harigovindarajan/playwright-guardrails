---
name: locator-map
description: Canonical definition of the locator map — the artifact the probe (or, in translate mode, the translator) produces and the writer consumes. Defines the persisted file envelope, the per-entry shape, and every entry-level enum. Preload into the probe, the translator, the writer, and the runner via their skills: field; the reviewer must not preload it. Not for direct human use.
user-invocable: false
---

# Locator Map — Canonical Shape

This skill is the **single source** for the locator map's shape. The probe writes maps in
this shape, and so does the translator in translate mode; the writer reads them. Neither restates it — both preload this skill, so the
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
on coverage. `grounded + contract_note + translated` equals the number of entries; a map from
the probe always has `translated: 0`, and a map from the translator has `grounded: 0` and
`grounded_this_run: 0`. The writer consumes `entries`.

## Entry shape

One entry per source locator or contract page-state target:

```json
{
  "contract": "path/to/contract.md",
  "source": {
    "selector": "//original-or-css-selector",
    "source_type": "xpath | css | testid | contract-note | inferred",
    "contract_step": "Step 7: click Login"
  },
  "replacement": {
    "locator": "page.getByRole('button', { name: 'Login' })",
    "rung": "role | label | placeholder | text | alt | testid | structural | original",
    "resolves_to_one": true
  },
  "grounding": "live-snapshot | contract-note | translated",
  "tier_source": "page-object | cache | live | translation",
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

- `"live-snapshot"` — the locator matched exactly one element in the current snapshot of a
  reached page state. `resolves_to_one: true` and `state_reached: true` are mandatory. The
  writer may rely on it as snapshot-verified.
- `"contract-note"` — the locator was NOT snapshot-verified: the state was reachable only
  via a destructive step the probe does not perform, the journey to reach the state failed
  after one retry, a gated `storageState` was unavailable, or no candidate matched exactly
  one element in the snapshot. `resolves_to_one: false` and `state_reached: false` are
  mandatory, and `replacement.locator` is the contract's proposed locator verbatim,
  unverified. How the writer handles an unverified locator is the writer's rule, not this
  document's.
- `"translated"`: the locator was converted from the source suite's selector without any
  browser, in translate mode. It was never checked against a page: the source suite's history
  is the only evidence it works. `resolves_to_one: false` and `state_reached: false` are
  mandatory, `tier_source` is `"translation"`, `provenance` is `"translated"`, and
  `source.selector` holds the source selector verbatim. The generated code carries it as a
  carried-over locator (Rule 28 of the canonical rules), and the runtime run is what proves
  it.

`tier_source` records how *this run* resolved the locator: `page-object` (reused from
Tier 1), `cache` (reused from Tier 2), `live` (freshly driven this run), or `translation`
(converted from the source selector, with no page involved). It is auditing
metadata — it does not change how the writer treats the locator.

`page`, `state`, and `auth_context` are the Tier 2 cache key. They are required whenever the
cache is in use so an entry is addressable.

## What you MUST do

- **Treat this shape as authoritative.** Do not restate these field names or enum values in
  an agent definition, and do not accept a locally remembered variant over what is written
  here.
- **Never fall back to a remembered shape.** If what you recall disagrees with this document,
  this document wins. The case where this content is absent entirely is caught by the static
  failure gates in the probe, the translator, the writer, and the runner, not here — a guard cannot live inside the content
  whose absence it guards.
