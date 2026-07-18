# Changelog

All notable changes to the **playwright-guardrails** plugin are documented here.
This project follows [Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH`.

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
