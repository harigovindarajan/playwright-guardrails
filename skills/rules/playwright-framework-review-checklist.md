# Playwright Framework Adversarial Review Checklist

## Purpose

Use this document to review a Playwright framework or PR against `playwright-framework-rules.md`.

The reviewer's job is not to restate the rules. The reviewer's job is to find brittleness, hidden intent, over-abstraction, and rule violations that would make the framework harder to maintain.

## Review Standard

Treat `playwright-framework-rules.md` as the canonical source of truth.

Prefer findings over praise. If there are no findings, say that explicitly and note any residual risk or untested area.

Each finding should include:

- Severity: Critical / Major / Minor
- File and line
- Rule violated
- Why it matters
- Minimal fix

## Critical Checks

- Are user-facing locators skipped in favor of test IDs, CSS, or XPath? See Rules 1 and 2.
- Does every fallback locator (`getByTestId()`, CSS/XPath, structural `.locator(...)` scoping, `.first()`, `.last()`, `.nth()`) have evidence that higher-priority user-facing locators failed under real Playwright locator resolution? See Rule 1.
- Is any fallback justified only by DOM duplication, snapshot inspection, or HTML structure review instead of Playwright locator behavior? Reject it. See Rule 1.
- Are business intent assertions hidden inside opaque POM journey methods? See Rule 10.
- Are page objects manually instantiated in tests instead of injected through fixtures? See Rule 13.
- Are hard waits, `networkidle`, force clicks, conditional UI branches, or lower-level waits used where a web-first assertion would express the expected state? See Rules 17, 18, and 19.
- Are tests or generated test data unsafe for parallel execution without explicitly declaring serial behavior or using collision-resistant data? See Rule 20.

## Locator Quality

- Are role, label, placeholder, text, or alt text locators used before test IDs? See Rule 1.
- Are test IDs used only when user-facing locators are unavailable? See Rule 2.
- Are CSS and XPath used only as a last resort? See Rule 1.
- If CSS is used only for scoping, is it reviewed as a structural fallback rather than harmless plumbing? See Rule 1.
- If `.first()`, `.last()`, or `.nth()` is used, is there evidence that semantic disambiguation with role/name, text, labels, `filter({ hasText })`, or `filter({ has })` was not possible? See Rules 1 and 24.
- Does the chosen locator prove user-facing intent rather than only DOM uniqueness? See Rule 1.
- Are lists narrowed with `filter()` instead of positional `nth()` where possible? See Rule 24.

## Test Readability

- Does each test read like a business journey rather than UI mechanics? See Rule 3.
- Do `test.step()` names describe business intent? See Rule 4.
- Are trivial one-line mechanical steps avoided? See Rule 5.
- Are action and immediate outcome kept together when they are the same journey moment? See Rule 6.

## Page Object Quality

- Do page objects own locators and low-level UI actions? See Rule 7.
- Do navigation and screen-changing methods leave the app ready for the next operation using the highest-priority stable destination signal available? See Rules 8 and 9.
- Does readiness use one stable, identity-tied web-first assertion rather than structural visibility or a lower-level wait when an assertion fits? See Rule 9.
- For a page whose submit/confirm/dialog behavior is bound by script on load, does readiness wait for the load lifecycle (handlers bound), not just element visibility? A visible element is not proof the page is interactive; interacting too early fires a native submit with no dialog and the post-action state never renders. See Rule 9 ("Visibility is not interactivity").
- Are stable user-visible elements used as readiness anchors when technical signals such as URL, title, or route are weak? See Rule 9.
- Are large pages decomposed into component objects where reuse or size justifies it? See Rule 23.

## Assertion Placement

- Are technical readiness assertions inside POM methods? See Rule 10.
- Are business outcome assertions visible in the test? See Rule 10.
- Are repeated business assertions wrapped only when genuinely reused? See Rule 11.
- Are one-off negative assertions kept explicit in the test? See Rule 11.

## Naming

- Are action methods named by user intent, not mechanics? See Rule 12.
- Are expectation methods named by asserted state, not vague verification verbs? See Rule 12.
- Are builders and data named for the business object they create? See Rule 12.

## Fixtures and Auth

- Are page objects injected through fixtures rather than manually created in specs? See Rule 13.
- Is fixture scope chosen deliberately, with test scope as the default? See Rule 14.
- Is login reused through `storageState` instead of repeated through the UI? See Rule 15.
- Is auth state isolated per role and per worker when tests mutate account state? See Rule 16.

## Reliability

- Are hard waits and `networkidle` absent? See Rule 17.
- Is conditional `if/else` logic kept out of test bodies? See Rule 18.
- Are Playwright auto-waiting and web-first assertions trusted instead of force actions or synchronous visibility reads? See Rule 19.
- Is each `locator.waitFor()` or other lower-level wait justified by a synchronization need that a retryable assertion would not express better? See Rules 17 and 19.
- Are ordered tests declared serial only when ordering is unavoidable? See Rule 20.

## Isolation and Data

- Is setup and teardown done through API where practical? See Rule 21.
- Is test data sourced from builders or data modules rather than inline literals? See Rule 22.
- Does API-created data use the fluent builder chain ending in `.build()`? See Rule 22.
- If runtime test data is generated, is the uniqueness strategy collision-resistant across workers instead of relying on `Date.now()` alone? See Rule 20.
- Are only third-party or unowned dependencies mocked? See Rule 27.

## Verify-Owned Risk

- Is any finding's only resolution "needs runtime confirmation" / "confirm at verify time" / "residual risk"? If so, it is **verify-owned** — record it as residual risk on a `PASS`, never as a blocking rejection. The `verify` stage runs the test and is the authoritative runtime check; blocking on what `verify` will decide is double-gating.
- A locator or behavior that can only be confirmed by running the test (for example, authenticated-only locators the probe grounded as `contract-note` because it could not reach the gated state) is verify-owned. Note it; do not block on it.
- This does not soften provable violations: a rule/checklist failure visible in the reviewed artifacts alone (unjustified fallback, uncovered contract step, an assertion that does not prove the outcome) stays blocking. See the reviewer's "Verify-owned findings are Non-blocking by construction" rule.

## Reviewer Output Template

```md
## Findings

1. Severity: Major
   File: `tests/checkout.spec.ts:42`
   Rule: Rule 10
   Issue: The main checkout confirmation assertion is hidden inside `checkoutPage.verifyCompleteCheckoutJourney()`.
   Why it matters: The test no longer shows the business outcome it proves.
   Minimal fix: Keep the checkout action in the POM, but assert the confirmation visibly in the spec.

## Residual Risk

- Note any area that was not reviewable from the provided diff.
```
