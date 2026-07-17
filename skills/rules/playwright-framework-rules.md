# Playwright Automation Framework Rules

## Table of Contents

- [Purpose](#purpose)
- [Core Philosophy](#core-philosophy)
- [Recommended Folder Structure](#recommended-folder-structure)
- [Rule Format](#rule-format)
- **[Locator Strategy](#locator-strategy-rules)** — Rule 1: Prefer user-facing locators · Rule 2: Test IDs as the fallback
- **[Test Design](#test-design-rules)** — Rule 3: Tests read like business journeys
- **[test.step](#teststep-rules)** — Rule 4: Steps describe business intent · Rule 5: No valueless one-line steps · Rule 6: Action plus intent assertion
- **[Page Object Model](#page-object-model-rules)** — Rule 7: POM owns locators and UI actions · Rule 8: Navigation methods leave the app ready · Rule 9: Readiness uses one stable web-first assertion
- **[Assertion Placement](#assertion-placement-rules)** — Rule 10: Readiness in POM, business intent in tests · Rule 11: Wrap repeated assertions in POM methods
- **[Naming](#naming-rules)** — Rule 12: Names describe intent and state
- **[Fixtures & Dependency Injection](#fixtures-and-dependency-injection-rules)** — Rule 13: Inject page objects via fixtures · Rule 14: Choose fixture scope deliberately
- **[Authentication & State Reuse](#authentication-and-state-reuse-rules)** — Rule 15: Reuse login via storageState + setup project · Rule 16: Isolate auth state per role and worker
- **[Reliability & Determinism](#reliability-and-determinism-rules)** — Rule 17: Never hard-wait or networkidle · Rule 18: No conditional logic in test bodies · Rule 19: Trust auto-waiting, never force · Rule 20: Parallel-safe; declare serial explicitly
- **[Test Isolation & Data Setup](#test-isolation-and-data-setup-rules)** — Rule 21: Prefer API setup/teardown · Rule 22: Data from builders/modules, not inline literals
- **[Scaling & Robustness](#scaling-and-robustness-rules)** — Rule 23: Compose component objects · Rule 24: Narrow lists with filter() · Rule 25: Soft assertions and polling · Rule 26: Tag and gate declaratively · Rule 27: Mock only what you don't own
- [Out of Current Scope](#out-of-current-scope)
- [Example Implementation](#example-implementation)
- [Official Playwright References](#official-playwright-references)

## Purpose

This document defines the structure, coding rules, and review standards for a Playwright automation framework using TypeScript, Page Object Model, stable locator strategy, and business-readable test journeys.

The goal is to keep tests readable, reliable, and maintainable without hiding the real intent of the scenario.

## Core Philosophy

The framework should be thin and expressive.

Tests should read like business journeys.

Page objects hide low-level UI mechanics, not business intent.

Assertions are placed by ownership:

- POM methods should verify only technical readiness.
- Test specs should verify the actual business intent.
- Repeated business assertions can be wrapped in POM expectation methods.

Use `playwright-framework-review-checklist.md` for adversarial review against these rules.

## Recommended Folder Structure

```text
playwright-framework/
├── tests/
│   ├── smoke/
│   ├── regression/
│   └── api/
├── pages/
│   ├── base.page.ts
│   ├── home.page.ts
│   ├── login.page.ts
│   └── contact.page.ts
├── components/
│   ├── header.component.ts
│   ├── footer.component.ts
│   └── modal.component.ts
├── fixtures/
│   ├── test.fixture.ts
│   └── auth.fixture.ts
├── data/
│   ├── users.ts
│   └── enquiries.ts
├── utils/
│   ├── env.ts
│   ├── api-client.ts
│   └── test-helpers.ts
├── playwright.config.ts
└── README.md
```

## Rule Format

Every rule leads with a normative directive (`MUST` / `SHOULD` / `NEVER`) and a one-line `WHY`. The directive is the contract; the prose and Before/After examples only add detail the directive does not already carry. Infrastructure rules (fixtures and auth) state the rule rather than teach the implementation — the official Playwright docs own the how.

## Locator Strategy Rules

### Rule 1: Prefer User-Facing Locators

> MUST: Prefer user-facing locators (role > label > placeholder > text > alt text) over CSS and XPath.
> WHY: They mirror how users find elements and survive structural and style refactors. This is the foundation every other locator and POM rule builds on.

| Priority | Locator | Use for |
|---|---|---|
| 1 | `getByRole()` | Buttons, links, headings, checkboxes, menus |
| 2 | `getByLabel()` | Form fields with labels |
| 3 | `getByPlaceholder()` | Inputs where placeholder is stable |
| 4 | `getByText()` | Stable visible text |
| 5 | `getByAltText()` | Images with meaningful alt text |
| 6 | `getByTestId()` | Stable automation hooks (Rule 2) |
| Last | CSS or XPath | Only when no better locator exists |

Before using `getByTestId()`, CSS, XPath, structural `.locator(...)` scoping, or positional `.first()` / `.last()` / `.nth()` to disambiguate an element, validate that higher-priority user-facing locators do not work under real Playwright locator resolution. Raw DOM duplication, hidden template nodes, or snapshot inspection alone is not enough evidence to skip the locator ladder.

Structural scoping still counts as a fallback when it is used to make a locator unique; it couples the test to DOM shape even if the final target is user-facing. Positional disambiguation is allowed only when no meaningful semantic discriminator exists.

#### Good

```ts
this.submitButton = page.getByRole('button', { name: 'Submit' });
this.emailInput = page.getByLabel('Email');
this.contactHeading = page.getByRole('heading', { name: 'Contact Us' });
```

#### Avoid

```ts
this.submitButton = page.locator('#submit');
this.emailInput = page.locator('input:nth-child(2)');
this.contactHeading = page.locator('//h2[text()="Contact Us"]');
```

### Rule 2: Use test IDs as the Fallback When Rule 1 Cannot Be Followed

> SHOULD: When a user-facing locator is not available — text is volatile or duplicated, the element is role-less or icon-only, or a complex component needs an explicit automation contract — use `getByTestId()`.
> WHY: A stable test id is a deliberate contract for exactly the cases where Rule 1 cannot apply; it is the fallback, not a shortcut around accessible locators.

#### Example

```ts
this.cartCounter = page.getByTestId('cart-counter');
this.profileMenu = page.getByTestId('profile-menu');
```

## Test Design Rules

### Rule 3: Tests Must Read Like Business Journeys

> MUST: Write tests as business journeys — what the user does and what the test proves — not as a list of UI mechanics.
> WHY: Journey-shaped tests communicate intent, survive UI refactors, and are reviewable by people who did not author them.

#### Before

```ts
test('contact us form submission', async ({ page }) => {
  await page.goto('/');
  await expect(page.getByAltText('demo website for practice')).toBeVisible();

  await page.getByRole('link', { name: 'Contact us' }).click();
  await expect(page.getByRole('heading', { name: 'Contact Us' })).toBeVisible();

  await page.getByLabel('Name').fill('Hari');
  await page.getByLabel('Email').fill('hari@example.com');
  await page.getByLabel('Message').fill('I need more information');
  await page.getByRole('button', { name: 'Submit' }).click();

  await expect(page.getByText('Your enquiry has been submitted')).toBeVisible();
});
```

#### After

```ts
import { contactEnquiry } from '../../data/enquiries';

test('visitor can submit a contact enquiry', async ({ homePage, contactPage }) => {
  await test.step('visitor opens the contact form', async () => {
    await homePage.goto();
    await homePage.openContactUs();
  });

  await test.step('visitor submits the enquiry and sees the submitted confirmation', async () => {
    await contactPage.submitEnquiry(contactEnquiry);

    await expect(contactPage.successMessage).toHaveText('Your enquiry has been submitted');
  });
});
```

Test data (`contactEnquiry`) comes from the `data/` layer, not an inline literal — see Rule 22.

## test.step Rules

### Rule 4: test.step Must Describe Business Intent

> MUST: Name each `test.step()` for the business moment it represents, not the implementation detail.
> WHY: Step names become the readable spine of the report; mechanical names add noise without meaning.

#### Avoid

```ts
await test.step('visitor opens the storefront home page', async () => {
  await homePage.goto();
  await homePage.waitForLoaded();
  await expect(page.getByAltText('demo website for practice')).toBeVisible();
});

await test.step('visitor opens contact us', async () => {
  await homePage.openContactUs();
  await contactPage.waitForLoaded();
});
```

#### Preferred

```ts
await test.step('visitor opens the contact form', async () => {
  await homePage.goto();
  await homePage.openContactUs();
});
```

### Rule 5: Do Not Create One-Line Steps Without Business Value

> SHOULD: Combine an action and the check of that same action's result into one step instead of splitting trivial moments.
> WHY: Over-splitting fragments the journey and buries intent in ceremony.

#### Before

```ts
await test.step('visitor submits an enquiry', async () => {
  await contactPage.submitEnquiry(enquiry);
});

await test.step('visitor sees enquiry submitted confirmation', async () => {
  await expect(contactPage.successMessage).toHaveText('Your enquiry has been submitted');
});
```

#### After

```ts
await test.step('visitor submits the enquiry and sees the submitted confirmation', async () => {
  await contactPage.submitEnquiry(enquiry);

  await expect(contactPage.successMessage).toHaveText('Your enquiry has been submitted');
});
```

### Rule 6: A Step Can Contain Action Plus Intent Assertion

> SHOULD: Keep an action and its immediate outcome assertion together in one step when they are the same journey moment.
> WHY: The assertion reads as the point of the action, keeping cause and effect adjacent.

#### Good

```ts
await test.step('visitor signs in and lands on the dashboard', async () => {
  await loginPage.loginAs(validUser);

  await expect(dashboardPage.welcomeMessage).toContainText(validUser.firstName);
});
```

#### Also Good

```ts
await test.step('visitor submits invalid credentials and sees the validation message', async () => {
  await loginPage.login(invalidUser.email, invalidUser.password);

  await expect(loginPage.errorMessage).toHaveText('Invalid username or password');
});
```

## Page Object Model Rules

### Rule 7: POM Owns Locators and UI Actions

> MUST: Page objects own locators and low-level UI actions; tests should not locate elements unless the locator is the actual assertion target.
> WHY: Keeps test bodies about intent and confines selector churn to one place.

#### Before

```ts
await page.getByRole('link', { name: 'Contact us' }).click();
await page.getByLabel('Name').fill('Hari');
await page.getByLabel('Email').fill('hari@example.com');
```

#### After

```ts
await homePage.openContactUs();
await contactPage.submitEnquiry(enquiry);
```

### Rule 8: POM Navigation and Screen-Changing Methods Must Leave the App Ready

> MUST: Any POM method that navigates or changes screen state must leave the resulting page, form, modal, or component ready for the next operation.
> WHY: Tests should be able to chain POM methods without adding their own readiness plumbing.

Examples of the contract:

- `homePage.goto()` must ensure the home page is ready.
- `homePage.openContactUs()` must ensure the contact form is ready.
- `dashboardPage.openProfileMenu()` must ensure the profile menu is visible.
- `cartPage.removeItem(productName)` must ensure the item is removed or the cart state is stable.

#### Avoid

```ts
await homePage.goto();
await homePage.waitForLoaded();

await homePage.openContactUs();
await contactPage.waitForLoaded();
```

#### Preferred

```ts
await homePage.goto();
await homePage.openContactUs();
```

"Ready" means **interactive**, not merely rendered. If the destination binds its behavior on load — form `submit` handlers, `confirm`/`alert` dialogs, or script-initialized widgets — the navigation/readiness method must ensure those handlers are bound before it returns (see Rule 9, load lifecycle). A page that is visible but not yet wired up is not ready.

### Rule 9: POM Readiness Uses One Stable, Identity-Tied Web-First Assertion

> SHOULD: Express readiness with one stable, identity-tied web-first assertion, using the highest-priority stable signal available. Prefer user-facing signals; when URL, title, or route checks are weak or unreliable, use a stable user-visible element instead.
> WHY: One strong readiness signal is faster, less brittle, and more meaningful than many checks or structural fallback signals.

Readiness should identify the destination state, not just prove that something became visible. Choose from the locator ladder: role/label/placeholder/text/alt text before test id, and structural selectors only as a justified fallback.

Good readiness candidates: a stable heading, hero text, banner or landmark, page-specific form container, primary action, reliable URL pattern, empty-state message, or a unique count that is stable by design.

Use Playwright web-first assertions such as `await expect(locator).toBeVisible()` as the default readiness primitive. Use `locator.waitFor()` only when the intent is synchronization rather than asserting an expected UI state.

#### Before

```ts
async expectLoaded() {
  await expect(this.page).toHaveURL(/\/$/);
  await expect(this.heroBanner).toBeVisible();
  await expect(this.contactUsLink).toBeVisible();
  await expect(this.featuredProducts).toHaveCount(3);
}
```

#### After

```ts
async expectLoaded() {
  await expect(this.heroBanner).toBeVisible();
}
```

If one of the fields you would have over-asserted is the actual test intent, assert it in the test (Rule 10), not in readiness.

#### Visibility is not interactivity

A visible element does not prove the page's behavior is wired up. When a page binds interactive handlers on load — a form `submit` handler, a `confirm`/`alert` dialog, or a widget initialized by a script (e.g. `$("#form").on("submit", () => confirm(...))`) — a readiness assertion that only checks visibility can pass *before* those handlers exist. Interacting then does the wrong thing silently: clicking Submit fires a native form post with no dialog, and the expected post-action state (success banner, navigation) never appears. This is a frequent, hard-to-diagnose cause of "the locator never resolved" — the locator is fine; the action never really happened.

For a page whose interactivity is script-bound, gate readiness on the load lifecycle **as well as** the visible identity anchor:

```ts
async expectLoaded() {
  await this.page.waitForLoadState('load'); // handlers bound
  await expect(this.getInTouchHeading).toBeVisible(); // destination identity
}
```

`waitForLoadState('load')` is an event-based lifecycle signal, not a fixed timeout and not `networkidle` — it does **not** violate Rule 17. Use it only where script-bound interactivity demands it (forms with submit-time dialogs, script-initialized controls), not as a blanket pre-step on every page.

## Assertion Placement Rules

### Rule 10: Technical Readiness Belongs in POM; Business Intent Belongs in Tests

> MUST: Keep technical readiness assertions inside POM methods, and keep the assertion that proves the scenario's business purpose visible in the test.
> WHY: POMs should own interaction safety; tests should show what the scenario actually proves.

#### Example

```ts
async openContactUs() {
  await this.contactUsLink.click();
  await expect(this.page.getByRole('heading', { name: 'Contact Us' })).toBeVisible();
}
```

#### Good

```ts
await test.step('visitor submits the enquiry and sees the submitted confirmation', async () => {
  await contactPage.submitEnquiry(enquiry);

  await expect(contactPage.successMessage).toHaveText('Your enquiry has been submitted');
});
```

#### Avoid Hiding the Main Intent Too Early

```ts
await test.step('visitor submits the enquiry', async () => {
  await contactPage.submitEnquiryAndVerifySuccess(enquiry);
});
```

Only use that style when the assertion is genuinely reused and its wording is owned by the page object (Rule 11).

#### Avoid Collapsing the Whole Journey

```ts
await checkoutPage.verifyCompleteCheckoutJourney();
```

#### Prefer

```ts
await test.step('customer completes checkout and sees the order confirmation', async () => {
  await checkoutPage.placeOrder(order);

  await expect(orderConfirmationPage.orderNumber).toBeVisible();
});
```

### Rule 11: Repeated Business Assertions Can Be Wrapped in POM Expectation Methods

> SHOULD: Move an assertion reused across many tests into a clearly named POM expectation method — but only once it is genuinely repeated.
> WHY: Avoids copy-paste and centralizes wording changes; one-off assertions stay explicit in the test.

#### Explicit Assertion, Good for One-Off Negative Scenario

```ts
await loginPage.login(invalidUser.email, invalidUser.password);

await expect(loginPage.errorMessage).toHaveText('Invalid username or password');
```

#### POM Assertion, Good When Reused

```ts
await loginPage.login(invalidUser.email, invalidUser.password);

await loginPage.expectInvalidLoginMessage();
```

## Naming Rules

### Rule 12: Naming Conventions Describe Intent and State

> MUST: Name action methods for user intent (`openContactUs`, `loginAs`), expectation methods for the state they assert (`expectInvalidLoginMessage`), and builders/data for the business object they create (`createOrder`) — never mechanics (`clickContactLink`), vague verbs (`verifyError`), or generic data names.
> WHY: Intent, state, and data-purpose names keep tests readable and decouple them from implementation mechanics.

#### Good

```ts
await homePage.openContactUs();
await loginPage.loginAs(user);
await loginPage.expectInvalidLoginMessage();
await cartPage.expectProductInCart(productName);
const order = await orderManager.createOrder.withStatus('paid').withSku('TC-001').build();
```

#### Avoid

```ts
await homePage.clickContactLink();
await loginPage.enterUsernameAndPasswordAndClickButton();
await loginPage.verifyError();
await cartPage.checkCart();
const data = await manager.makeThing().build();
```

## Fixtures and Dependency Injection Rules

### Rule 13: Inject Page Objects Through Fixtures, Not Manual Instantiation

> MUST: Tests receive page objects from custom fixtures (`test.extend`); never call `new SomePage(page)` inside a test body.
> WHY: Centralizes wiring and readiness, keeps test bodies about intent, and lets fixtures handle setup and teardown even when a test throws.

#### Before

```ts
const homePage = new HomePage(page);      // manual wiring inside the test
const contactPage = new ContactPage(page);
```

#### After

```ts
test('visitor can submit a contact enquiry', async ({ homePage, contactPage }) => {
  // use injected page objects
});
```

### Rule 14: Choose Fixture Scope Deliberately

> SHOULD: Default to test-scoped fixtures; use worker scope only for an expensive resource shared safely across a worker's tests (auth state, DB connection, spawned server).
> WHY: Test scope preserves isolation; worker scope avoids paying a heavy setup cost on every test, but misusing it leaks state between tests.

## Authentication and State Reuse Rules

### Rule 15: Reuse Login via storageState and a Setup Project

> MUST: For apps with a logged-in state, authenticate once in a dedicated setup project, persist `storageState`, and have authenticated page objects assume the logged-in page. Do not log in through the UI in every test.
> WHY: UI login per test is slow and flaky; `storageState` keeps logged-in tests focused on the journey under test.

The login flow itself is exercised in one dedicated test, not re-run as setup everywhere else. Credentials come from environment variables; auth state files are git-ignored.

### Rule 16: Isolate Auth State Per Role and Per Worker When Tests Mutate It

> MUST: Use a separate `storageState` file per role; for tests that change server-side account state, give each parallel worker its own authenticated account.
> WHY: Sharing one account across parallel workers causes state collisions; per-role files keep admin and standard journeys cleanly separated.

## Reliability and Determinism Rules

### Rule 17: Never Use Hard Waits or networkidle

> NEVER: `page.waitForTimeout(...)` or `waitForLoadState('networkidle')` (including `waitUntil: 'networkidle'`).
> WHY: A fixed sleep is always too short or too long, and `networkidle` is officially discouraged as slow and unreliable. Assert the state you are actually waiting for.

#### Before

```ts
await page.click('text=Save');
await page.waitForTimeout(2000);
await page.goto('/list', { waitUntil: 'networkidle' });
```

#### After

```ts
await page.getByRole('button', { name: 'Save' }).click();
await expect(page.getByRole('status')).toHaveText('Saved');
await page.goto('/list'); // then assert the element you need
```

Acceptable waits when an assertion does not fit: `page.waitForURL(...)`, `locator.waitFor()`, `page.waitForResponse('**/api/**')`.

### Rule 18: No Conditional Logic in Test Bodies

> NEVER: branch test flow with `if/else` on live page state (e.g. "dismiss the dialog if it appears").
> WHY: Conditionals hide non-determinism and make coverage unpredictable. Make state deterministic in setup or a fixture, then assert unconditionally.

Ensure the precondition in setup/fixture so the test path is single and known, then act without a guard.

### Rule 19: Trust Auto-Waiting; Never Force Actions

> NEVER: pass `{ force: true }` to bypass actionability checks, and never wrap a synchronous `isVisible()` in `expect`.
> WHY: Playwright auto-waits for visible/stable/enabled/editable before acting. `force` papers over real bugs; a synchronous read does not retry and reintroduces flakiness.

When the intent is an expected UI state, prefer retryable assertion APIs such as `toBeVisible`, `toHaveText`, `toContainText`, and `toHaveURL` over lower-level waiting patterns. Lower-level waits are acceptable only when an assertion does not express the condition being synchronized.

#### Avoid

```ts
await page.getByRole('button', { name: 'Submit' }).click({ force: true });
expect(await page.getByRole('status').isVisible()).toBe(true);
```

#### Prefer

```ts
await page.getByRole('button', { name: 'Submit' }).click();
await expect(page.getByRole('status')).toBeVisible();
```

### Rule 20: Tests Must Be Parallel-Safe; Declare Serial Explicitly

> MUST: Write every test to run safely in parallel. If a file genuinely must run in order, declare `test.describe.configure({ mode: 'serial' })` so workers do not interleave it.
> WHY: `fullyParallel` spreads tests across workers; a file that assumes order without declaring serial mode fails intermittently when workers pick up its tests independently.

Serial mode is the documented exception, not the default — reach for it only when shared, ordered state is unavoidable, and prefer fixing the dependency first.

Runtime-generated test data must also be parallel-safe. Do not rely on `Date.now()` alone for data that must be globally unique; include worker/test identity or another collision-resistant component so separate workers cannot create the same account, order, email, or other unique value.

#### Example

```ts
test.describe('checkout wizard (ordered steps)', () => {
  test.describe.configure({ mode: 'serial' });

  test('step 1: add to cart', async ({ cartPage }) => { /* ... */ });
  test('step 2: pay', async ({ checkoutPage }) => { /* ... */ });
});
```

## Test Isolation and Data Setup Rules

### Rule 21: Prefer API Setup and Teardown Where Practical

> SHOULD: Create and destroy test data through the `request` fixture (API), not the UI, wherever practical.
> WHY: API setup is faster, more reliable, and keeps tests isolated; driving setup through the UI couples a test to screens it is not trying to verify.

"Where practical" is deliberate — when no API exists for a precondition, UI setup is acceptable as a fallback. The payload still goes through a builder, never an inline literal (Rule 22).

#### Example

```ts
test.beforeEach(async ({ productManager }) => {
  await productManager.createProduct.withSku('TC-001').build();
});

test.afterEach(async ({ productManager }) => {
  await productManager.deleteBySku('TC-001');
});
```

### Rule 22: Source Test Data from Builders and Data Modules, Not Inline Literals

> MUST: Source test data from shared data modules (`data/`) or fluent builders, never inline literals scattered across specs; data created through the API is constructed with a chained builder that ends in `.build()`.
> WHY: Centralizing data construction keeps test bodies readable, makes data reusable and overridable, and turns a schema or fixture change into one edit instead of a find-and-replace.

Static fixtures live as exports in `data/` (e.g. `contactEnquiry`); varying data uses a builder.

#### Avoid

```ts
await request.post('/api/orders', {
  data: { customerId: 'c-1', items: [{ sku: 'TC-001', qty: 1 }], status: 'paid' },
});
```

#### Prefer

```ts
const order = await orderManager.createOrder.withStatus('paid').withSku('TC-001').build();
```

The builder owns construction and the API round-trip; the test only chooses the meaningful differences.

## Scaling and Robustness Rules

### Rule 23: Compose Component Objects Instead of Monolithic Pages

> SHOULD: Model reusable UI regions (navigation bar, data table, modal) as component objects and compose them into page objects; avoid one giant class per page.
> WHY: A 200-method "god page object" is unmaintainable and breaks single responsibility. Components are reusable across pages and map to the existing `components/` folder.

#### Example

```ts
export class CheckoutPage {
  readonly cart: CartSummary;
  readonly address: AddressForm;

  constructor(private readonly page: Page) {
    this.cart = new CartSummary(page);
    this.address = new AddressForm(page);
  }
}
```

### Rule 24: Narrow Lists with filter(), Not nth()

> SHOULD: Pick an item from a list with `.filter({ hasText })` or `.filter({ has })`; use `.nth()` only when no semantic discriminator exists.
> WHY: `.nth(2)` silently targets a different element when the DOM reorders. Filtering by content survives reordering, and strict mode already protects you when a locator is ambiguous.

#### Before

```ts
await page.getByRole('row').nth(2).getByRole('button', { name: 'Edit' }).click();
```

#### After

```ts
await page.getByRole('row').filter({ hasText: 'Alice Smith' })
  .getByRole('button', { name: 'Edit' }).click();
```

Scope locators to a container to keep matches inside a component: `dialog.getByRole('button', { name: 'Cancel' })`.

### Rule 25: Use Soft Assertions and Polling Helpers Where They Fit

> SHOULD: Use `expect.soft` to collect several independent field checks in one go, and `expect.poll` / `expect(async () => {...}).toPass({ timeout })` for non-locator async conditions.
> WHY: Soft assertions report all field failures at once instead of stopping at the first; poll/toPass add retry-ability to things that are not element assertions. Always give `toPass` an explicit timeout — it does not inherit `expect.timeout`.

#### Example

```ts
await expect.soft(summary.price).toHaveText('$29.99');
await expect.soft(summary.sku).toHaveText('SKU-001');

await expect.poll(async () => (await api.jobStatus()).state, { timeout: 30_000 }).toBe('complete');
```

### Rule 26: Tag and Gate Tests Declaratively

> SHOULD: Tag suites with `@smoke` / `@regression` via the details object, and gate environment-specific tests with `test.skip(condition, reason)` / `test.fixme` — always with a reason.
> WHY: Tags let CI select suites (`--grep @smoke`); declarative skips document *why* a test does not run instead of silently commenting it out.

#### Example

```ts
test('user can checkout', { tag: ['@smoke', '@checkout'] }, async ({ checkoutPage }) => {
  test.skip(process.env.ENV === 'prod', 'Checkout uses a sandbox-only payment provider');
  // ...
});
```

### Rule 27: Mock Only What You Don't Own

> SHOULD: Intercept third-party or hard-to-trigger dependencies with `page.route(...)`; exercise your own backend for real.
> WHY: Mocking a service you control hides integration failures — the whole point of an end-to-end test. Mock payment, email, or maps providers; keep core journeys against the real backend.

#### Example

```ts
// stub an unreliable third-party endpoint
await page.route('**/payments.example.com/charge', route =>
  route.fulfill({ status: 200, json: { id: 'ch_test', status: 'succeeded' } }),
);
```

## Out of Current Scope

These are valuable but intentionally not yet codified as rules — add them when the framework needs them, following the official guidance:

- **Visual regression** — `toHaveScreenshot` with masking and thresholds. See [Visual comparisons](https://playwright.dev/docs/test-snapshots).
- **Accessibility testing** — `@axe-core/playwright` for WCAG rules and `toMatchAriaSnapshot()` for structural regression. See [Accessibility testing](https://playwright.dev/docs/accessibility-testing).
- **CI sharding** — `--shard` with the `blob` reporter and `merge-reports`. See [Sharding](https://playwright.dev/docs/test-sharding).
- **Debugging tooling** — trace viewer, UI mode, and `codegen` are run-time tools, not framework rules.

## Example Implementation

### Home Page

```ts
import { expect, type Locator, type Page } from '@playwright/test';

export class HomePage {
  readonly heroBanner: Locator;
  readonly contactUsLink: Locator;

  constructor(private readonly page: Page) {
    this.heroBanner = page.getByAltText('demo website for practice');
    this.contactUsLink = page.getByRole('link', { name: 'Contact us' });
  }

  async goto() {
    await this.page.goto('/');
    await this.expectLoaded();
  }

  async expectLoaded() {
    await expect(this.heroBanner).toBeVisible();
  }

  async openContactUs() {
    await this.contactUsLink.click();
    await expect(this.page.getByRole('heading', { name: 'Contact Us' })).toBeVisible();
  }
}
```

### Contact Page

```ts
import { type Locator, type Page } from '@playwright/test';

export type Enquiry = {
  name: string;
  email: string;
  message: string;
};

export class ContactPage {
  readonly nameInput: Locator;
  readonly emailInput: Locator;
  readonly messageInput: Locator;
  readonly submitButton: Locator;
  readonly successMessage: Locator;

  constructor(private readonly page: Page) {
    this.nameInput = page.getByLabel('Name');
    this.emailInput = page.getByLabel('Email');
    this.messageInput = page.getByLabel('Message');
    this.submitButton = page.getByRole('button', { name: 'Submit' });
    this.successMessage = page.getByText('Your enquiry has been submitted');
  }

  async submitEnquiry(enquiry: Enquiry) {
    await this.nameInput.fill(enquiry.name);
    await this.emailInput.fill(enquiry.email);
    await this.messageInput.fill(enquiry.message);
    await this.submitButton.click();
  }
}
```

### Test Data

```ts
// data/enquiries.ts
import { type Enquiry } from '../pages/contact.page';

export const contactEnquiry: Enquiry = {
  name: 'Hari',
  email: 'hari@example.com',
  message: 'I need more information',
};
```

For data that must vary per test, export a builder instead (Rule 22).

### Test Spec

```ts
import { expect } from '@playwright/test';
import { test } from '../../fixtures/test.fixture';
import { contactEnquiry } from '../../data/enquiries';

test('visitor can submit a contact enquiry', async ({ homePage, contactPage }) => {
  await test.step('visitor opens the contact form', async () => {
    await homePage.goto();
    await homePage.openContactUs();
  });

  await test.step('visitor submits the enquiry and sees the submitted confirmation', async () => {
    await contactPage.submitEnquiry(contactEnquiry);

    await expect(contactPage.successMessage).toHaveText('Your enquiry has been submitted');
  });
});
```

## Official Playwright References

- [Playwright Locators](https://playwright.dev/docs/locators)
- [Playwright Page Object Models](https://playwright.dev/docs/pom)
- [Playwright Best Practices](https://playwright.dev/docs/best-practices)
- [Playwright Auto-waiting](https://playwright.dev/docs/actionability)
- [Playwright Test Fixtures](https://playwright.dev/docs/test-fixtures)
- [Playwright Authentication](https://playwright.dev/docs/auth)
- [Playwright API Testing](https://playwright.dev/docs/api-testing)
- [Playwright Web-first Assertions](https://playwright.dev/docs/test-assertions)
- [Playwright Network and Mocking](https://playwright.dev/docs/network)
- [Playwright Test Annotations](https://playwright.dev/docs/test-annotations)
