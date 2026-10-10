---
name: contract
description: Canonical definition of the test contract, the shared input to every migration agent. Defines the contract's sections, the step markers, the origin-locators table, the risk-annotation vocabulary, and open questions, plus how a consumer reads a contract that was written by hand. Preload into the drafter, planner, probe, writer, reviewer, and runner via their skills: field. Not for direct human use.
user-invocable: false
---

# Test Contract: Canonical Shape

This skill is the **single source** for the shape of a test contract. The drafter writes
contracts in this shape. The probe, writer, reviewer, and runner read them. The planner uses
only the risk vocabulary. None of them restates any of it: each preloads this skill, so the
definition is injected into the agent's context at startup and there is no second copy to
keep in sync.

A contract is a plain-language description of what one existing test proves. It is not a
translation of that test. It says what the test is for, how a user gets through it, what data
it needs, and what must be true at the end, so that every later stage works from intent rather
than from the source framework's mechanics.

This skill defines **what a contract is**. How to *draft* one from a source test belongs to the
drafter. How to *use* one (probing, generating, reviewing, running) belongs to each consumer.

## Sections

One Markdown file per test case. Sections appear in this order, under these headings.

### Header (required)

```markdown
# Contract: <short title of the behavior>

- Testcase: `<testcase-id>`
- Source: `<path/to/source/test>` (<source framework>, <test name or line>)
```

`Testcase` is the test-case identity. Consumers use it to name what they write (a locator map,
a spec). It is a lowercase slug, unique within a migration.

### Test intent (required)

`## Test intent`. One to three sentences: the behavior this test proves, in user or business
terms. Not the steps, not the selectors.

### Preconditions (required)

`## Preconditions`. A bullet list of what must be true before step 1: the session's auth state
(anonymous or authenticated, and as which kind of user) and anything that must already exist.

### Test steps (required)

`## Test steps`. A numbered list. Each step starts with its markers in bold, then says what the
user does and what they should see afterwards:

```markdown
1. **direct**: Navigate to `/`. Verify the home page is visible.
2. **driven**: Click "Sign in". Verify "Sign in to your account" is visible.
3. **driven, destructive**: Click "Delete account" and confirm. Verify the goodbye page.
```

Every step carries exactly one reach marker:

- `direct`: the step's page state is reached by loading a URL path on the app's base URL.
  The step names the path.
- `driven`: the step's page state is reached by performing the step's interaction (click,
  fill, submit, select) from the state the previous step left.

A step may also carry:

- `destructive`: performing the step changes or removes data irreversibly (delete, cancel,
  pay, send). Every page state after a destructive step depends on it.

Steps that log in are ordinary `driven` steps. The page states after a login step are gated:
reaching them needs an authenticated session.

### Test data setup / cleanup (required)

`## Test data setup / cleanup`. What data the test needs, where it comes from in the source
suite, and what is removed afterwards. Write "No cleanup" explicitly when nothing is.

Credential-typed values (passwords, tokens, API keys, personal data) are never written into a
contract. Name where the source keeps them and mark them credential-typed; the value itself
stays out.

### Origin locators (required)

`## Origin locators`. A table of every element the source test touches or asserts on:

```markdown
| Element | Source selector | Page state | User-facing description |
|---|---|---|---|
| Sign in button | `button[type='submit']` | sign-in | button "Sign in" |
```

- `Element`: a short name for the element, stable within the contract.
- `Source selector`: the source suite's selector, **verbatim**, exactly as the source test or
  its helper wrote it, including XPath, CSS, test ids, and framework-specific forms. It is a
  brittle starting point; the contract never upgrades it.
- `Page state`: the state the element lives in, named consistently with the steps (for
  example `home (anon)`, `sign-in`, `logged-in home`).
- `User-facing description`: what a user would call it, with its role and visible name where
  the source makes that knowable.

The table may be empty only when the source test touches no UI element.

### Expected outcomes (required)

`## Expected outcomes`. A bullet list of what must be true for the test to pass, taken from the
source test's assertions. Each outcome is something observable: visible text, a URL, a
persisted record, an API effect.

### Risk annotations (optional)

`## Risk annotations`. Places where the source test's intent depends on its framework's
behavior in a way that may not survive the move. These are annotations, never translations:
they say where to look, not what to write.

```markdown
| Risk | Where | Note |
|---|---|---|
| `network-alias` | step 3; `login.cy.ts:22` | Waits on `@loginRequest` before asserting the header. |
```

`Risk` takes one value from the vocabulary below. `Where` names the step and the source
location. `Note` says what the source does, in one line.

### Open questions (optional)

`## Open questions`. Anything the contract's author could not determine from the source.
Each question carries a severity in bold:

```markdown
- **blocking**: `cy.loginAs('admin')` is not in the supplied helpers; the user role and how
  it is set up are unknown. (step 2; `admin.cy.ts:8`)
- **non-blocking**: The source asserts `.toast` exists but not its text; the outcome may need
  a specific message.
```

- `blocking`: the test's intent, a step, the data it needs, or an expected outcome cannot be
  determined without an answer.
- `non-blocking`: the contract is usable as written, but an answer would make it more precise.

A question is resolved by editing the contract to answer it and deleting the question.

## Risk vocabulary

This is the only list of risk values. Contracts use it in `Risk annotations`; the planner uses
it for per-file risk flags.

| Value | What it flags | Typical source signals |
|---|---|---|
| `retry-ability` | The source relied on its framework retrying a query or assertion until it passed, so the waiting is implicit in the source. | Cypress chained `.should(...)`, `.contains(...)` doing double duty as a wait. |
| `session-cache` | Login or setup is cached and restored across tests rather than performed per test. | `cy.session`, a shared logged-in driver, a `beforeAll` login. |
| `cross-origin` | The journey leaves the app's origin (identity provider, payment page). | `cy.origin`, a redirect to another domain. |
| `network-alias` | A step waits on, stubs, or asserts a specific network request. | `cy.intercept` with `cy.wait('@alias')`, request interception in Selenium proxies. |
| `explicit-wait` | The source waits for a specific condition by hand. | `WebDriverWait`, `ExpectedConditions`, polling helpers. |
| `hard-wait` | The source sleeps for a fixed time. | `Thread.sleep`, `cy.wait(<ms>)`, `time.sleep`, `browser.pause`. |
| `iframe` | An element lives inside a frame. | `switchTo().frame`, `cy.iframe`, frame helpers. |
| `multi-tab` | The journey opens or switches to another tab or window. | `getWindowHandles`, `switchTo().window`, `target="_blank"` handling, stubbed `window.open`. |
| `test-order` | The test only passes if another test ran first. | Reliance on a previous `it`'s side effects, ordered suites, numbered test names. |
| `shared-state` | The test reads or mutates state other tests also use. | A shared account, a fixed record id, global variables between tests. |
| `other` | A fragility outside the values above. The note must say what it is. | |

## Reading a contract

Contracts written by the drafter follow the sections above exactly. Contracts written by hand
may not, and consumers accept them on these terms:

- **Find sections by meaning, not by heading text.** A section titled "Goal" is the test
  intent; a "Locators" table with a selector column is the origin-locators table.
- **A missing required element makes the contract ambiguous.** If a consumer cannot find the
  test intent, the steps, or the expected outcomes at all, the contract is too ambiguous to
  use. Missing preconditions, data, or locators are treated as "none stated".
- **Missing step markers are inferred, not invented.** A step that names a URL to load is
  `direct`; any other step that changes the page state is `driven`. A step that deletes,
  cancels, pays, or sends is treated as `destructive` when in doubt.
- **An unresolved `blocking` open question makes the contract ambiguous** for every consumer
  that depends on the part of the contract it is about.

## What you MUST do

- **Treat this shape as authoritative.** Do not restate these sections, markers, columns, or
  risk values in an agent definition, and do not accept a locally remembered variant over what
  is written here.
- **Never fall back to a remembered shape.** If what you recall disagrees with this document,
  this document wins. The case where this content is absent entirely is caught by the static
  failure gate in each agent that preloads it, not here: a guard cannot live inside the
  content whose absence it guards.
