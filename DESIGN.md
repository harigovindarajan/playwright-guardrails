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

## One canonical rule set, loaded by preload

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

The session advisor is a separate, harness-level lever the plugin cannot pin. For
mechanical probe/writer loops, downgrade or disable it via a project-level
`.claude/settings.json` override so an extraction run doesn't silently inherit a
heavyweight advisor.

## Principle

The generated framework should stay thin, expressive, and readable as business
journeys. The rules document is canonical; everything else references it.
