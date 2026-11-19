# Playwright E2E + API Test Framework

[![Playwright Tests](https://github.com/<your-username>/playwright-e2e-framework/actions/workflows/playwright.yml/badge.svg)](https://github.com/<your-username>/playwright-e2e-framework/actions/workflows/playwright.yml)
![Playwright](https://img.shields.io/badge/Playwright-1.65-2EAD33?logo=playwright&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6?logo=typescript&logoColor=white)
[![Report](https://img.shields.io/badge/Test%20report-live-blue)](https://<your-username>.github.io/playwright-e2e-framework/)

A test automation framework built with **Playwright + TypeScript**. It covers UI, API and accessibility testing and runs on GitHub Actions with sharding.

| Layer             | Target app                                                    | What is tested                                                              |
| ----------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **UI**            | [SauceDemo](https://www.saucedemo.com)                        | Login, product sorting, cart, full checkout with price checks               |
| **API**           | [Restful-Booker](https://restful-booker.herokuapp.com/apidoc) | Auth, full CRUD, filtering, negative cases, schema checks                   |
| **Accessibility** | SauceDemo                                                     | WCAG 2.1 AA scan with axe-core, compared against a baseline of known issues |

**58 tests** run on Chromium, Firefox, WebKit, a mobile viewport (Pixel 7) and a browserless API project.

---

## Highlights

- **Page Object Model + custom fixtures.** Specs ask for `{ inventoryPage, checkoutPage }` and Playwright builds them, so there is no `new Page(page)` boilerplate in tests.
- **Log in once, reuse everywhere.** A `setup` project logs in through the UI and saves `storageState`. Every UI test then starts already signed in, which saves several seconds per test.
- **Typed API client with runtime contracts.** `BookingClient` wraps every endpoint, and **Zod** schemas check each response. If the API shape changes, the test fails with a clear message.
- **Validated config.** Environment variables are parsed with Zod at startup. A bad `BASE_URL` fails immediately with a readable error.
- **Fresh test data.** **Faker** factories create new names, dates and prices on every run, so tests don't collide on shared data.
- **Stable locators only.** The framework uses `getByTestId` (mapped to `data-test`), `getByRole` and filtered locators. It has no XPath and no `waitForTimeout`; ESLint blocks both.
- **Data-driven tests.** Negative login and checkout-validation cases come from one table each.
- **Tags.** `@smoke` marks the critical path. The mobile project runs only smoke tests.
- **Debugging built in.** Traces are recorded on first retry, screenshots on failure, and video is kept for failures.

## CI/CD pipeline

```
push / PR / nightly / manual
          │
   ┌──────▼──────┐
   │   quality   │  ESLint · tsc --noEmit · Prettier
   └──────┬──────┘
   ┌──────▼──────────────────────────────┐
   │ test  shard 1 │ 2 │ 3 │ 4  (parallel) │  each shard → blob report
   └──────┬──────────────────────────────┘
   ┌──────▼──────┐
   │   report    │  merge blobs → one HTML report + job summary table
   └──────┬──────┘
   ┌──────▼──────┐
   │ deploy (main)│  HTML report → GitHub Pages
   └─────────────┘
```

- **Sharding** across 4 runners shortens wall-clock time. `fail-fast: false` keeps one failing shard from hiding the results of the others.
- **Browser cache** is keyed by Playwright version, so later runs skip the browser download.
- **Concurrency control** cancels the older run when you push to the same branch again.
- **Nightly cron** runs the full regression. **workflow_dispatch** takes a `grep` input, so you can run only `@smoke` from the Actions tab.
- **Secrets**: passwords come from repository secrets when they are set, and fall back to the public demo credentials otherwise.

## Project structure

```
├── .github/workflows/playwright.yml   # CI pipeline
├── playwright.config.ts               # projects, reporters, retries
├── src/
│   ├── api/
│   │   ├── BookingClient.ts           # typed API client
│   │   └── schemas.ts                 # Zod response contracts
│   ├── data/
│   │   ├── factories.ts               # Faker test-data builders
│   │   └── users.ts                   # SauceDemo user personas
│   ├── fixtures/test.ts               # custom test with page-object fixtures
│   ├── pages/                         # Page Objects
│   │   ├── BasePage.ts
│   │   ├── LoginPage.ts
│   │   ├── InventoryPage.ts
│   │   ├── CartPage.ts
│   │   └── CheckoutPage.ts
│   └── utils/
│       ├── env.ts                     # validated environment config
│       └── paths.ts
└── tests/
    ├── setup/auth.setup.ts            # logs in once, saves session
    ├── ui/                            # login, inventory, checkout, a11y
    └── api/                           # booking CRUD + negative cases
```

## Getting started

**Requirements:** Node.js 20 or newer.

```bash
git clone https://github.com/<your-username>/playwright-e2e-framework.git
cd playwright-e2e-framework
npm ci
npx playwright install --with-deps
cp .env.example .env        # optional; defaults work out of the box
```

### Running tests

| Command                    | What it does                               |
| -------------------------- | ------------------------------------------ |
| `npm test`                 | Everything: all browsers + API             |
| `npm run test:smoke`       | Only `@smoke` tests                        |
| `npm run test:api`         | API tests only (no browser, fast)          |
| `npm run test:ui`          | UI tests on Chromium, Firefox and WebKit   |
| `npm run test:headed`      | Chromium with a visible browser            |
| `npm run test:debug`       | Step through with the Playwright Inspector |
| `npm run report`           | Open the last HTML report                  |
| `npx playwright test --ui` | Interactive UI mode, time-travel debugging |

### Code quality

```bash
npm run lint        # ESLint + eslint-plugin-playwright
npm run typecheck   # strict TypeScript
npm run format      # Prettier
```

## Design decisions

**Why fixtures over `beforeEach` setup?** Fixtures are lazy. A test that never uses `checkoutPage` never builds it. Fixtures also compose cleanly, and the test signature documents what each test depends on.

**Why validate API responses with Zod?** `expect(res.status()).toBe(200)` passes even when the body is wrong. Schema checks catch contract drift, such as a renamed field or a number sent as a string, which is the kind of bug that breaks frontends in production.

**Why an accessibility baseline?** The demo site has existing violations. Failing on all of them would make the test permanently red, and a red test gets ignored. The baseline fails only when a _new_ type of violation appears, which is how teams adopt a11y testing on legacy apps.

**Why `test.step`?** Long flows such as the CRUD lifecycle show up as named steps in the HTML report and in traces, so a failure points straight at "update (PUT)" instead of line 34.

## Adding a new test

1. Add locators and actions to a page object in `src/pages/` (or create one that extends `BasePage`).
2. Register it in `src/fixtures/test.ts`.
3. Import `test` and `expect` from `src/fixtures/test`, **not** from `@playwright/test`.
4. Tag critical-path tests with `@smoke`.

## License

MIT, © Maaz Ahmad
