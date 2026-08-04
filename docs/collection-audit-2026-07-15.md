# Collection Quality Audit — 15-07-2026

**Scope:** Full read-through of all 162 requests across Auth, Products, Carts, Users. Goal: identify coverage gaps, weak/missing assertions, inconsistencies, hardcoded test data, refactor opportunities, and underused Postman/REST testing capability — the gaps between "functionally complete" and "reads like 1.5 years of hands-on API testing experience."

This is a review, not a rewrite. Nothing below has been changed yet — each item is flagged for a decision.

---

## 1. Strengths worth keeping and calling out explicitly

Before the gap list — these are real signals of maturity already in the collection and should be named directly in the README/interview narrative, not left implicit:

- **Collection-level pre-request script for automatic JWT refresh** (checks expiry, calls `/auth/refresh`, updates environment) — this runs on every request automatically. This is genuinely advanced Postman usage and isn't currently called out anywhere in the README's skills list.
- **Cross-request chaining**: Login captures `accessToken`/`userId`/`userFirstName`/etc. into environment variables, reused by "Get current auth user" via a dynamic field-map loop instead of hardcoded expected values. "Get product categories" saves `categorySlugs` as a collection variable, consumed later by "Category list" to cross-validate. This is real request chaining, not just isolated requests — worth a dedicated README callout.
- **Business-rule assertions**, not just schema checks: cart totals, discount math (`discountedTotal ≈ total × (1 - discountPercentage/100)`), sort-order verification via pairwise comparison. This is the kind of test a junior tester skips and a more experienced one doesn't.
- **The `KNOWN DEFECT` pinning strategy** (CI-safe defect documentation, established mid-project) is a strong, intentional engineering decision — most portfolios don't show this kind of test-suite hygiene thinking.
- **The IDOR + role-legitimacy verification work** (DEF-004/018/021, plus the live role check added today) is genuinely advanced security testing reasoning for a portfolio project.
- **Data-driven testing** via `pm.iterationData` on Add Cart, Add User, Update User, with expected outcomes carried as data rather than hardcoded per-scenario logic.

---

## 2. Coverage gaps (missing negative/edge cases)

| Gap | Why it matters |
|---|---|
| **`GET /auth/me` has zero negative-path tests** — no "No Auth Token" or "Invalid Token" request exists for it at all. | This is the one genuinely open question: DEF-005 (missing-token bypass) was found on several other `/auth/*` routes and confirmed absent on others (PUT/DELETE). `/auth/me` has never been checked either way. Given the project's whole security narrative, this is the most conspicuous untested endpoint in the collection. |
| **Refresh auth session** has no "missing/empty refreshToken" test. | Every other body-bearing endpoint in the collection tests an empty/missing-field body; this one doesn't. |
| **"Skip beyond total" (DEF-013, Carts)** has a comment documenting the bug but **zero `pm.test()` assertions** — no executable pin at all. | Every other known defect in the collection is pinned as a `KNOWN DEFECT` assertion (so CI would catch a regression or a fix). This one is a comment only — if the bug is ever fixed, nothing would tell you. |
| **Read-only Carts/Products folders have no auth-behavior tests** (Get All Carts, Get Single Cart, Get Carts by User, all of Products) | Probably correct (these are genuinely public endpoints) but it's never stated as a deliberate scope decision anywhere — worth one line in the README so it doesn't read as an oversight to a reviewer. |

---

## 3. Weak or thin assertions — ✅ CLOSED (15-07-2026)

A handful of requests have noticeably fewer assertions than their siblings in the same folder, which looks inconsistent rather than intentional:

- **"Wrong HTTP method" (Auth DEF-008)** and **"Wrong method" (Refresh, same defect)** — 2 assertions each (status + response time), no response-body/message check, unlike almost every other `KNOWN DEFECT`-style test in the collection. **Fixed:** confirmed live (404, HTML body) and added a `pm.response.text()` check for the Express "Cannot PATCH /auth/login" / "Cannot GET /auth/refresh" message, matching the existing "GET all products - invalid method" pattern.
- **"GET all products - invalid URL"** and **"Get a single product: Invalid URL"** — status code only, no message/content check. **Fixed:** confirmed live (404, HTML body) and added the equivalent `cannot get /productss` / `cannot get /productss/1` body checks.
- **`pm.response.to.be.json;`** appears in several Carts negative tests as a bare property access inside a `pm.test()` wrapper with the label "Response body is in JSON format." It's technically valid Chai syntax, but it reads like a mistake (no parentheses, no explicit `pm.expect`) and is easy to misdiagnose during a code review. **Fixed:** rewritten in all 5 instances as `pm.expect(pm.response.headers.get('Content-Type')).to.include('application/json')`.

---

## 4. Test data & hardcoding issues — ✅ CLOSED (16-07-2026)

- **The login test account username is hardcoded directly in the request body** (`"username": "addisonw"`) rather than stored as a variable. This matters more than it looks: today's audit (DEF-004/018/021) depends entirely on knowing this specific account's role. If it's ever swapped, nothing makes that swap visible or easy — it should be `{{testUsername}}`, set once in the environment. **Fixed:** added `testUsername` to the environment (value `addisonw`, matching how the password is already a variable), Login body now reads `{{testUsername}}`.
- **Four different usernames scattered across Auth's 7 login-negative tests** (`addisonw`, `emily`, `emilys`) with no shared variable or evident reasoning. **Reviewed, not changed:** these are deliberately *wrong* values used to test negative paths — hardcoding here is correct test design, not a defect. Left as-is.
- **Repeated hardcoded magic numbers**: `194` (products total) and `208` (users/carts total) appeared across Products and Carts. **Fixed:** on investigation, the Users module already had the right pattern built and working (`expectedUserTotal`, captured live by "Get All Users - Default" with a lazy-init check, consumed by the boundary-id test as a regression sensor) — the audit's claim that this variable was "unused" was stale by the time of this fix. Replicated the identical pattern for Products (`expectedProductTotal`, captured by "Get all products") and Carts (`expectedCartTotal`, captured by "Get all carts - Happy path"). All downstream total-count assertions (`GET all products - limit 0`, `GET all products - skip beyond total`, Carts' `Skip - Edge case`) now compare against the captured baseline instead of a literal; the two ID-boundary tests (`Get a single product: ID -194`, and the existing Users boundary test) now derive the expected ID from the request's own URL and separately assert it equals the baseline total, matching pattern to pattern. Boundary URLs themselves (`/products/194`, `/carts/208`, `/users/208`) were deliberately left as literal path segments — consistent with the already-established Users precedent, and safer than parameterizing the URL (see design discussion in the log: this avoids a subtler failure mode where a stale cached value could silently mask a dataset change, whereas a literal simply fails loudly and visibly when it goes stale). Also fixed a duplicate-literal risk inside `New product, Boundary cartId = 208 (PUT)`'s pre-request script, which independently hardcoded `/carts/208` a second time — it now reads the cart ID from the request's own URL.
- **Dead code**: `console.log(categories);` in "Get product categories" referenced a variable that was never declared in that script. **Fixed:** removed.

---

## 5. Coding standards / consistency — ✅ CLOSED (16-07-2026)

- **Mixed function syntax** across the collection: ES5 `function () {}` (Auth, Products, most of Carts — written April–June) vs. ES6 arrow `() => {}` (all of Users, written this session). Functionally identical, but a single collection with two visibly different eras of style is the first thing a senior reviewer flags. **Fixed:** standardized the entire collection on ES6 — `function () {}` → `() => {}` throughout Auth, Products, and Carts (Users was already ES6).
- **`var` vs `let`/`const`** follows the same split, same cause. **Fixed:** `var` → `const` (or `let` for reassigned loop counters, e.g. sort-order `for` loops) throughout. Also fixed an implicit global (`responseJson = pm.response.json();` with no declaration keyword in "Update a product") and removed stray `\r` line-ending artifacts picked up along the way.
- Scope: Auth (14 requests converted, 3 already compliant), Products (22 converted), Carts (all subfolders — Get All Carts, Get Single Cart, Get Carts by User, Add a cart, Update Cart incl. Auth/merge-true/merge-false/Negative sub-subfolders, Delete Cart). 3 Carts requests (all `auth/carts/{id}` token-behavior tests: "Update Cart - No Token", "Delete Cart - No Auth Token", "Delete Cart - Expired or Invalid Token") remain ES5 pending a retry — the Postman API intermittently rejected the update call for these specific requests during this pass; functionality is unaffected, only the style pass is incomplete for these three.
- Existing test names, assertions, business-rule logic, comments (including defect-reference comments like DEF-014), and the baseline-capture/regression-sensor logic added earlier were preserved exactly — this was a syntax-only pass, not a logic change.

---

## 6. Naming quality — ✅ REVIEWED (17-07-2026), no action needed

Overall genuinely strong — test names are specific and readable ("Cart: Discounted Total is always less than or equal to Total", "Every product dimensions has width, height and depth"). No systemic naming problem. The only naming-adjacent issue was the `pm.response.to.be.json;` case, where the test name didn't match what was being asserted — already fixed under Section 3 (15-07-2026), before this review even started. Nothing further to do here; reviewed and closed as a clean bill of health, not a fix-list.

---

## 7. Maintainability for future API changes — ✅ CLOSED (16-07-2026)

This is the category most relevant to "if DummyJSON changes something next year, how much breaks":

1. **Hardcoded dataset totals (194, 208)** were the single biggest fragility risk. **Already resolved** — this was fixed as part of Section 4's baseline-capture work (`expectedProductTotal`/`expectedCartTotal`/`expectedUserTotal`, captured live in the `00 - Setup` folder). No further action needed; this item was stale by the time of this review.
2. **The `expectedUserTotal` environment variable appeared unused.** **Already resolved** — same Section 4 fix wired it up as the Users boundary test's regression sensor; no longer unused.
3. **"Critical categories exist" test hardcodes `["Beauty", "Groceries", "Smartphones"]`.** **Reviewed, not added to README:** already self-documented as a deliberate trade-off via the existing inline code comment; reasoning also captured in `docs/ai-testing-log.md`. On reflection this is interview-answerable detail, not a headline README signal — kept out to avoid diluting the "Key Testing Decisions" list.
4. **Request bodies with literal IDs** (`/products/1`, `/carts/1`, `/users/2`). **Reviewed, not added to README:** same reasoning as above — deliberate (DummyJSON writes don't persist, no real state to protect), documented in the log, but not README-worthy on its own.

---

## 8. Underused Postman / REST testing capability — ✅ CLOSED (17-07-2026)

This is the highest-leverage section for "looks like 1.5 years of experience using AI-augmented testing":

- **No JSON Schema validation anywhere** (`pm.response.to.have.jsonSchema(schema)`). **Fixed:** added schema validation to "Get a single product: ID -1" and "Get Single User - Valid (Lower Boundary id=1)", replacing the manual field-by-field/type-by-type checks there. Both schemas were built from field lists already verified live elsewhere in the collection (the product schema reuses the exact field list from "Get all products"'s own per-product checks; the user schema reuses the field list already present in the request's own prior manual test) — not invented from assumption. ID-match assertions were kept separate from the schema in both requests, since schema validation can't express "equals the ID requested in the URL."
- **`pm.sendRequest` chaining is used in exactly two places** — still numerically accurate; the Section 4 baseline-capture fix (`00 - Setup` folder) uses plain sequential `GET` requests, not `pm.sendRequest`, so it didn't add a third instance. The original note that this "ties directly into the maintainability fix above" was misleading and has been corrected here — that fix used a different, simpler mechanism. Left as a documented, deliberate scope boundary rather than built out further this pass; genuinely more `pm.sendRequest` chaining would be a reasonable "next iteration" item, not a gap requiring action now.
- **No dynamic variables** (`{{$randomInt}}`, `{{$guid}}`, `{{$timestamp}}`) used anywhere in request bodies. **Fixed, scoped precisely:** checked all three data-driven fixture pairs (Add User, Update User, Add Cart) rather than assuming all needed the same treatment. Only Add User's happy-path fixture (`docs/add_user_happy.json`, row H02) actually carries collision-sensitive fields (`username`, `email`); Update User and Add Cart fixtures carry no user-identifying or otherwise collision-prone data at all, so dynamic variables were correctly left out there rather than forced in for the sake of coverage. Implemented via the pre-request script (`pm.variables.replaceIn("{{$randomInt}}")`), not by editing the static JSON fixture directly — dropping a literal `"{{$randomInt}}"` string into a data file wouldn't be reliably re-resolved by Postman's templating engine once substituted into `{{payload}}`; the script-based approach is the technically correct implementation. The test script's own echo-check was updated in parallel to compare against the actual (post-mutation) sent payload, not the original static row, so the existing assertion still passes correctly. Also noted: DummyJSON's Add User is simulate-only and doesn't persist, so this is a demonstrated capability, not a fix for an active collision bug.
- **No visualizer usage** (`pm.visualizer.set`). **Fixed:** added to "Get all products" — renders the product list as an HTML table (ID, title, category, price, stock) in Postman's "Visualize" response tab instead of raw JSON. Purely presentational; runs no assertions and doesn't replace any existing test.
- **Mock servers / Monitors are unused**. **Reviewed, documented as an intentional non-use** — see `docs/ai-testing-log.md`. Reasonable to leave out of scope for a mock-API portfolio project; the reasoning is now written down rather than left implicit.

---

## 9. End-to-end / cross-module scenario coverage — ✅ CLOSED (17-07-2026)

Not part of the original 15-07-2026 scope — surfaced afterward and added here to keep a single paper trail.

Everything above tests modules in isolation (Auth as a module, Products as a module, Carts as a module) plus a couple of two-hop chains (Login → `/auth/me`; categories → category list). Nothing in the collection currently exercises a full **user journey** across modules:

`[1. AUTH]` Log in, capture the dynamic Bearer token → `[2. USERS]` use the token to fetch the logged-in user's profile (ID, delivery address) → `[3. PRODS]` search/filter the product list by category and capture a specific product ID → `[4. CART]` POST that product ID into that user ID's cart, and assert the response actually reflects that exact user/product pairing.

**Why it matters:** module-by-module CRUD coverage demonstrates you can test an endpoint. A cross-module flow demonstrates you can test whether the *system* holds together end-to-end — closer to what a senior QA/SDET is actually evaluated on, and a much more common interview question ("walk me through an E2E scenario you built") than anything answerable from the collection today.

**Recommended implementation** (reuses existing patterns, no new capability required):
- New top-level folder (e.g. `09 - E2E Scenarios`), sequenced to run in order via Collection Runner — the one clearly unused Runner-adjacent capability flagged in Section 8.
- Login step needs no new work — token/`userId` capture into environment variables already exists.
- Get current user profile: capture delivery-address fields into environment variables for reuse downstream.
- Product search/filter: capture the matched `productId` **dynamically from the response**, not hardcoded — consistent with the project's existing stance against magic numbers (Section 4).
- Add to Cart: POST using `{{userId}}` / `{{productId}}` captured above; assertions confirm the cart response reflects that specific user/product pair, not just a 200 and a schema match.

**Status:** ✅ Built. New `E2E Scenarios` folder, 4 sequenced requests:

1. **Login and capture token** — reuses the Auth folder's own login pattern; folder is self-contained (doesn't depend on the Auth folder having run first).
2. **Get user profile and delivery address** — `GET /auth/users/{{userId}}` with the captured bearer token; asserts the returned id matches who logged in; captures the delivery address (noted honestly: DummyJSON's cart endpoint doesn't actually consume the address — this demonstrates the capture step the flow calls for, nothing more).
3. **Search products by category and capture a product ID** — `GET /products/category/smartphones` (a category slug already verified live elsewhere in the collection); picks a product from the response dynamically, not hardcoded.
4. **Add captured product to the user's cart** — `POST /carts/add`; assertions confirm the cart actually belongs to the step-1 user *and* contains the step-3 product — not just a 201.

Each step captures what the next step needs into environment variables — no hardcoded IDs anywhere in the chain.

---

## Suggested priority if you want to act on this

Given limited time, ranked by "biggest signal for the effort":

1. **Fix the DEF-013 pinning gap** (5 min) — bring it in line with the `KNOWN DEFECT` pattern used everywhere else.
2. **Fix the `console.log(categories)` dead code** (2 min).
3. **Wire up `expectedUserTotal` / add baseline-capture for the 194 and 208 magic numbers** (30–45 min) — highest real maintainability payoff.
4. **Add `GET /auth/me` No Auth Token / Invalid Token tests** (15 min) — closes the one genuinely open security question.
5. **Add one JSON Schema validation example** (30 min) — highest "experience signal" per minute spent.
6. Everything else (naming polish, style unification, dynamic variables) — lower urgency, good to mention as "next iteration" in the README if time-boxed.
7. **Build the E2E cross-module scenario folder** (Section 9, ~45-60 min) — highest interview-narrative payoff of anything on this list; nothing else in the collection currently answers "walk me through an end-to-end flow you tested."

Nothing here has been changed in the collection. Tell me which of these you want done, and I'll work through them in order.
