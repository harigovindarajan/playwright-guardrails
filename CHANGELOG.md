# Changelog

All notable changes to the **playwright-guardrails** plugin are documented here.
This project follows [Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH`.

## [0.4.0] — 2026-07-25

Probe auth coverage, grounding-coverage reporting, and a shared locator-map skill.
Closes [#1](https://github.com/harigovindarajan/playwright-guardrails/issues/1),
[#2](https://github.com/harigovindarajan/playwright-guardrails/issues/2), and
[#5](https://github.com/harigovindarajan/playwright-guardrails/issues/5).

- **The probe now reaches gated states on a driven journey.** Previously
  `storageState` applied only at entry points a contract explicitly marked as
  authenticated, and the probe is (correctly) forbidden from typing credentials —
  so a contract describing a journey into a gated area authenticated at neither
  point and recorded every gated locator as an unverified fallback. The probe now
  drives the anonymous portion, then reopens with `state-load` and **replays** the
  journey under auth. It replays rather than navigating straight to the gated path
  because server-side journey state (a populated cart, wizard progress) does not
  survive the session swap.
- **Auth-context is assigned at preflight**, not at grounding time, so Tier 2 cache
  lookups — which run before the browser opens — key consistently with writebacks.
- **The persisted locator map now carries a required `grounding_summary`** with
  `grounded`, `grounded_this_run`, `contract_note`, and `unobserved_states`. The two
  grounded counts differ deliberately: Tier 1 and Tier 2 entries are `live-snapshot`
  without driving, so a fully cache-served map would otherwise report complete
  coverage.
- **Map `status` now distinguishes deliberate from blocked.** `completed` means every
  `contract-note` was a refusal (a destructive step, or a gated state with no
  `storageState`); `partial` means a state was blocked by a journey error, retry
  exhaustion, or an unreachable gated state.
- **Manifest surfaces partial maps** via a mandatory `state-unreachable` risk entry,
  since the orchestrator reads the manifest rather than each map. A pass-B run also
  records an `auth` risk entry noting the second session open.
- **PII hygiene extended to scripted-mode snapshot dumps**, which now contain
  authenticated account and payment DOM for the first time, and the scratch dir is
  removed on teardown.
- **The locator-map shape now lives once, in `skills/locator-map/`.** The map is the
  interface between the probe (which writes it) and the writer (which reads it), and
  its shape had been written down three times — the writer's copy documenting three
  provenance values where the probe emits four, `contract-note` having been missing
  since the initial public commit. Both agents now preload the skill through their
  `skills:` field, the same mechanism `skills/rules/` already uses, and neither
  restates the shape. Each gains a static failure gate for an absent shape plus a
  `missing-locator-map-shape` failure type. The reviewer deliberately does not
  preload it, which makes its blindness to the map a declared decision rather than
  an omission.

> **Upgrade note.** The `status` change is intentionally stricter: a caller gating on
> `status == "completed"` will now fail runs that previously passed while leaving
> states silently unobserved. That is the defect being corrected. Gate on a ratio over
> `grounding_summary` for a more precise check.

## [0.3.0] — 2026-07-18

Public-release preparation. No change to agent or skill runtime behavior — this
is repo hygiene plus documentation.

- **Removed the OpenCode port** (`.opencode/`) and the **eval harness** (`evals/`)
  from the shipped plugin; both were development-only and did not affect the
  installed agents or rules.
- **Added `LICENSE`** (MIT), matching the manifest's declared license.
- **Added `DESIGN.md`** — the probe/writer/reviewer architecture and its rationale —
  and **`CONTRIBUTING.md`** — how to extend the canonical rules.
- **Rewrote `README.md`** to the standard Claude Code plugin structure (per-agent
  Focus / When-to-use / Invocation, install steps, requirements), moving deep design
  rationale into `DESIGN.md` and dropping the 0.2.0 reuse-tier pipeline example and
  "Operating cost" sections.
- Version stays pre-1.0: the rule set is still evolving; `1.0.0` is reserved for
  after real external use.

## [0.2.0] — 2026-07-05

Consumer-facing behavior and configuration changes across the probe and reviewer
agents. All changes are backward-compatible: required invocation inputs are
unchanged, new inputs are optional, and output-schema changes are additive.

### Probe (`playwright-dom-probe`)

- **Scripted mode is now the default.** The probe writes the contract journey as a
  single `run-code` script, executes it in one call, and grounds locators offline
  from dumped snapshot files. It falls back to interactive driving only when the
  journey is not statically predictable (or a scripted run fails the one-retry).
  Existing callers invoke the probe the same way; the runtime default differs.
- **Turn-economy budget** — ≤4 CLI calls per page state, `generate-locator` as the
  first candidate source, one consolidated counting call; help discovery reduced to
  a single preflight probe.
- **Ad/analytics block-list** — optional `block-list` input with a built-in default,
  applied via `playwright-cli route` before the first `goto`.
- **Named-session isolation** — every `playwright-cli` command carries `-s=<run-id>`;
  concurrent probes are isolation-safe only with distinct session names.
- **Configurable output filename** (optional pattern, default preserved) and a new
  top-level `status: "completed" | "partial"` on the persisted locator map.
- **`tier_stats`** in the manifest, plus a mandatory reuse-miss risk entry when
  neither the page-objects nor the locator-cache directory is supplied.

### Reviewer (`playwright-test-reviewer`)

- **Reclassification discipline** — optional `prior report path` input enables a
  `findings_ledger`; reclassifying a finding to Blocking now requires citing changed
  artifact evidence (advisor opinion on unchanged facts is insufficient), with a
  raise-first-or-`escalation_deadline` rule.

### Agent configuration (Claude defs)

- **Per-role effort pinned in frontmatter** — probe `low`, writer `medium`,
  reviewer `high`.
- **Distinct agent colors** — probe `cyan`, writer `green`, reviewer `purple`.

### Tooling / docs

- Probe eval harness: static-app fixture, a pure snapshot-match + turn-budget scorer
  with unit tests, and a smoke-only live runner (`npm run eval:probe`).
- README gains a worked reuse-tier pipeline example and an "Operating cost" section.

## [0.1.0] — baseline

Implicit initial release: canonical Playwright rules + review checklist, the
`playwright-guardrails:rules` manifest skill, and the probe/writer/reviewer
sub-agents. This version predates the changelog and was never tagged; it is the
backward-compatibility baseline for `0.2.0`.
