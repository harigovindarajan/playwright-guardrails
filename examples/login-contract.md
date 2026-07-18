# Contract: TC2 — Login with correct email and password

Source: `selenium-source-reference/tests/LoginTest.java` (Feature "User", CRITICAL).

## Test intent

A registered user can sign in with correct credentials and is recognised as
logged in under their account name.

## Preconditions

- Anonymous session.
- A pre-existing registered account (the "existing user").

## Test steps

1. **direct** — Navigate to `/`. Verify the home page is visible.
2. **driven** — Click "Sign in". Verify "Sign in to your account" is visible.
3. **driven** — Enter the existing account's email and password, click "Sign in".
   Verify the header shows "Logged in as &lt;name&gt;" matching the account name.

## Test data setup / cleanup

- **Existing user** from `resources/testData/ExistingUser.json` (name/email/
  password). In Playwright these are credential-typed and MUST come from
  `process.env.*` (Rule 15) — see `EXISTING_USER_*`. No cleanup (read-only login;
  the account is not modified).

## Origin locators

| Element | Selenium selector | Page state | User-facing description |
|---|---|---|---|
| home hero image | `div[class='hero-banner'] img` | home (anon) | image, alt "seasonal promotion" |
| Sign in nav link | `a[href='/signin']` | home (anon) | link "Sign in" |
| "Sign in to your account" heading | `div[class='signin-form'] h2` | sign-in (anon) | heading "Sign in to your account" |
| login email input | `input[name='email']` | sign-in | textbox "Email Address" |
| login password input | `input[name='password']` | sign-in | textbox "Password" |
| Sign in button | `button[type='submit']` | sign-in | button "Sign in" |
| logged-in username | `//*[@id='header']//li[10]/a/b` | logged-in home | bold username inside "Logged in as …" |

## Expected outcomes

- Home page hero is visible.
- "Sign in to your account" is visible.
- The header username equals the existing account's name.
