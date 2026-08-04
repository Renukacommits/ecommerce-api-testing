# DummyJSON E-commerce API Testing Project

![Postman](https://img.shields.io/badge/Postman-FF6C37?style=for-the-badge&logo=postman&logoColor=white)
![Newman](https://img.shields.io/badge/Newman-66B135?style=for-the-badge&logo=postman&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)
![API Testing](https://img.shields.io/badge/API%20Testing-Automated-blue?style=for-the-badge)

## Project Overview

This project tests the API of a simulated e-commerce platform, covering the full user journey — authentication, products, carts, and users. It exercises every endpoint in each module to confirm expected behaviour, and documents every place the API deviates from that expectation.

## What This Project Demonstrates

Full CRUD coverage across Auth, Products, Carts and Users; data-driven testing using JSON datasets; security testing that surfaced two real authorization vulnerabilities (IDOR and Broken Object Level Authorization); and an AI-assisted testing workflow using Postbot and Claude — with critical evaluation of AI output documented throughout, not accepted at face value.

---

## Test Coverage Summary

### API Endpoints Tested

- **Base URL:** `https://dummyjson.com/`
- **Modules:** `/auth`, `/products`, `/carts`, `/users`

Counts below are pulled directly from the live Postman collection, not estimated.

| Module       | Requests | Assertion Status                                                              |
| ------------ | -------- | ------------------------------------------------------------------------------ |
| **Auth**     | 17       | ✅ Complete                                                                     |
| **Products** | 23       | ✅ Complete                                                                     |
| **Carts**    | 63       | ✅ Complete                                                                     |
| **Users**    | 62       | ✅ Complete                                                                     |
| **Total**    | **165**  | **165/165 requests have completed assertions — collection is 100% assertion-complete** |

**Also in the collection (not counted above — infrastructure and integration, not per-module CRUD):**

- **`00 - Setup`** (3 requests) — captures dataset totals (`expectedProductTotal`, `expectedCartTotal`, `expectedUserTotal`) into environment variables before any module runs, so boundary/total-count assertions read a live-captured value instead of a hardcoded magic number.
- **`E2E Scenarios`** (4 requests) — a single chained business flow: log in and capture the token → fetch the logged-in user's profile → search products by category and capture a product ID → add that exact product to that exact user's cart, with assertions confirming the final cart genuinely belongs to that user and contains that product. Every step captures what the next one needs dynamically — no hardcoded IDs anywhere in the chain. Demonstrates the system holds together across modules, not just at each endpoint tested in isolation.

### Defects Found

Format: Endpoint / Expected / Actual / Severity. IDs are stable — several are referenced by name inside the Postman test scripts themselves (e.g. `KNOWN DEFECT DEF-005`), so numbering is never reused or reshuffled once assigned.

#### DEF-001 — Update Cart: quantity = 0 accepted silently

**Endpoint:** PUT /carts/{id}
**Expected:** 400 with validation error — quantity 0 is not a valid cart quantity
**Actual:** 200 returned, product added to cart with quantity 0, counted in totalProducts
**Severity:** Medium

#### DEF-002 — Update Cart: invalid input accepted silently

**Endpoint:** PUT /carts/{id}
**Expected:** 400 with validation error
**Actual:** 200 returned for both negative quantity (-1) and a non-existing productId — invalid input silently accepted or dropped with no error
**Severity:** High

#### DEF-003 — Update Cart: integer value accepted for merge field

**Endpoint:** PUT /carts/{id} [merge = 1 (integer)]
**Expected:** 400 with validation error — merge must be boolean
**Actual:** 200 returned, integer 1 treated as truthy, merge behaviour applied
**Severity:** Medium

#### DEF-004 — Delete Cart: IDOR — regular user can delete another user's cart

**Endpoint:** DELETE /auth/carts/{id}
**Expected:** 403 — a user with role USER should only be able to delete their own cart
**Actual:** 200 — a valid USER token can delete any cart regardless of ownership
**Severity:** Critical
**Note:** Tested with a non-admin user token to rule out admin privileges. Vulnerability confirmed.

#### DEF-005 — Broken Object Level Authorization (BOLA) on GET /auth/me, /auth/users/{id}, /search, /filter, /{id}/carts, and POST /auth/users/add, /auth/carts/add

**Endpoint:** GET /auth/me, GET /auth/users/{id}, GET /auth/users/search, GET /auth/users/filter, GET /auth/users/{id}/carts, POST /auth/users/add, POST /auth/carts/add — 7/7 tested `/auth/*` routes matching this signature
**Expected:** 401 Unauthorized when no bearer token is supplied
**Actual:** 200/201 OK with the full user object/list/cart contents (GET routes) or a real created record (POST routes) — with no token at all
**Severity:** Critical
**OWASP Classification:** API1:2023 — Broken Object Level Authorization, #1 on the OWASP API Security Top 10.
**Root cause:** the auth middleware validates a token only when one is present. There is no else-branch to reject a request that omits the Authorization header entirely — an *invalid* token is correctly rejected with 401 on every route tested, but a *missing* token falls straight through to the handler every time.
**Inconsistency:** GET /auth/users (list, no id) correctly returns 401 with no token, and both `PUT /auth/users/{id}` and `DELETE /auth/users/{id}` correctly return 401 with no token — the bug is specific to `/auth/me`, item-level GET, search, filter, nested-resource GET, and the two add routes; it does not affect the collection-level list or the update/delete verbs.
**Write-route note:** POST /auth/users/add and POST /auth/carts/add are the two non-GET routes confirmed vulnerable. Add User is simulate-only (no real record persists), but **Add Cart genuinely creates a real cart with no authentication at all** — a real record, not a simulated one, making this the highest-impact write-route instance found so far. Found 18-07-2026 after the test's original assertion (401) turned out to have been written from an assumption never verified live — surfaced by a script crash (`jsonData.message` was undefined against the actual 201 response) rather than caught proactively.
**Confirmed at 7/7:** reproduced independently on seven separate routes (five reads, two writes) with the identical signature. At this sample size, treated as a property of the auth middleware itself for GET/POST verbs specifically — PUT and DELETE on the same resources are confirmed unaffected, so the assumption is scoped to matching verbs, not the whole `/auth/*` surface.
**Test strategy:** assertions pin the current (insecure) behaviour under `KNOWN DEFECT DEF-005` test names rather than staying permanently red, so the suite stays CI-safe once gated into GitHub Actions. If this is ever patched, the pinned tests fail — that failure is the trigger to flip them back to the secure expectation and close the defect. Full reasoning in `docs/ai-testing-log.md`.
**Related:** compounds the plaintext-password anti-pattern (DEF-006) — the same PII is now reachable with zero authentication on four separate GET routes.

#### DEF-006 — Auth: plaintext password returned in API responses

**Endpoint:** GET /auth/me (also present on every /users and /auth/users response)
**Expected:** password field absent or masked in any API response
**Actual:** password returned in plaintext
**Severity:** High
**Note:** this is the underlying data-exposure issue that DEF-005 turns into a critical vulnerability — without auth, plaintext passwords are reachable with no token at all.

#### DEF-007 — Auth: invalid expiresInMins returns 500 instead of 400

**Endpoint:** POST /auth/login, POST /auth/refresh
**Expected:** 400 Bad Request for an invalid expiresInMins value (e.g. -1)
**Actual:** 500 Internal Server Error
**Severity:** Medium

#### DEF-008 — Auth: wrong HTTP method returns 404 instead of 405

**Endpoint:** /auth/login, /auth/refresh (wrong method sent)
**Expected:** 405 Method Not Allowed (unconfirmed against a formal spec — DummyJSON has no public API contract to verify against)
**Actual:** 404 Not Found
**Severity:** Low
**Note:** a status-code choice issue, distinct from DEF-009's response-format issue below.

#### DEF-009 — Wrong HTTP method / invalid URL returns HTML instead of JSON

**Endpoint:** confirmed on /products (wrong method, invalid path) and independently reconfirmed on /users (`POST /users`, invalid path) — same defect pattern recurring across modules
**Expected:** a consistent JSON error response regardless of error type
**Actual:** a fully-branded DummyJSON HTML error page
**Severity:** Low — doesn't break functionality, but breaks error-contract consistency for any client parsing error responses as JSON

#### DEF-010 — Products: incorrect sort order for titles with a common prefix

**Endpoint:** GET /products?sortBy=title&order=asc
**Expected:** "Apple Airpods" sorts before "Apple AirPods Max Silver" (shorter prefix first)
**Actual:** "Apple AirPods Max Silver" appears before "Apple Airpods"
**Severity:** Low
**Likely cause:** sort/collation logic rather than true alphabetical prefix ordering.

#### DEF-011 — Products: invalid sortBy value silently ignored

**Endpoint:** GET /products?sortBy=invalid
**Expected:** 400 Bad Request with a descriptive error
**Actual:** 200 OK, silently falls back to default ordering with no warning
**Severity:** Low

#### DEF-012 — Products: invalid category returns 200 instead of 404

**Endpoint:** GET /products/category/invalid
**Expected:** 404 Not Found
**Actual:** 200 OK, `{"products": [], "total": 0, "skip": 0, "limit": 0}` — indistinguishable from a valid-but-empty category
**Severity:** Medium

#### DEF-013 — Carts: skip beyond total not rejected

**Endpoint:** GET /carts?skip=209 (total = 208)
**Expected:** 400 Bad Request, or skip clamped to the true total
**Actual:** 200 OK, empty array, skip echoed back as 209 — an impossible dataset position
**Severity:** Low
**Note:** same permissive-input pattern later reconfirmed on the Users module (see `docs/ai-testing-log.md`).

#### DEF-014 — Inconsistent error codes for invalid IDs across routes

**Endpoint:** GET /carts/{invalid} vs GET /carts/user/{invalid}
**Expected:** the same status code for structurally invalid input across equivalent routes
**Actual:** invalid cart ID → 404; invalid user ID → 400
**Severity:** Low

#### DEF-015 — Inconsistent field naming for the same value across endpoints

**Endpoint:** GET /carts/{id} vs POST /carts/add
**Expected:** the same field name for the same concept across endpoints
**Actual:** GET /carts returns `discountedTotal` per product; POST /carts/add returns `discountedPrice` per product for the equivalent value
**Severity:** Low
**Impact:** breaks client-side consistency for any consumer of both endpoints.

#### DEF-016 — Carts: invalid productId silently dropped on add

**Endpoint:** POST /carts/add
**Expected:** 400 Bad Request naming the invalid productId
**Actual:** 201 Created — invalid productId silently dropped, only valid products included
**Severity:** Medium

#### DEF-017 — Carts: duplicate product entries not consolidated (data quality)

**Endpoint:** GET /carts/7
**Expected:** duplicate productId entries consolidated into one line with combined quantity
**Actual:** product id 56 appears twice, each with quantity: 1, instead of once with quantity: 2
**Severity:** Low
**Classification:** static dataset data-quality defect, not an API logic defect — flagged separately since it affects any test asserting unique product IDs per cart.

#### DEF-018 — IDOR: GET /auth/users/{id}/carts (sibling of DEF-004)

**Endpoint:** GET /auth/users/{id}/carts
**Expected:** 401/403 when a valid token requests a different user's carts than the one the token belongs to
**Actual:** 200 OK — a valid token for one user can read a completely different user's cart contents (products, prices, totals) by simply changing the `{id}` in the URL
**Severity:** Critical
**OWASP Classification:** API1:2023 — Broken Object Level Authorization / IDOR.
**Note:** same vulnerability class as DEF-004 (Delete Cart IDOR), now confirmed on a GET route as well as a DELETE route. The endpoint validates *who is asking* (a real token) but never checks whether that person is allowed to see *this particular* user's data — a missing ownership check, not a missing auth check (contrast with DEF-005, which is the absence of auth entirely).
**Role check:** confirmed the token owner (`addisonw`, id 30) holds a plain `"user"` role and the target account (id 6, `oliviaw`) holds `"moderator"` — ruling out "the token owner is an admin with legitimate cross-account access" as an innocent explanation. Verified live via a runtime lookup in the test script, not assumed.
**Test strategy:** pinned under `KNOWN DEFECT DEF-018`, same CI-hygiene approach as DEF-005.

#### DEF-019 — Add User: invalid/incomplete input accepted silently

**Endpoint:** POST /users/add
**Expected:** 400 Bad Request for an empty body, missing required fields (firstName/lastName), or wrong data type on a field (age as a string)
**Actual:** 201 Created for all three cases — the endpoint is simulate-only with no server-side request validation, echoes the (invalid) input back with a fake new `id`
**Severity:** Medium
**Note:** same permissive-write pattern already confirmed on Update Cart (DEF-001/002/003) and Add Cart (DEF-016), now confirmed on a Users-module write endpoint. Data-driven — all three scenarios and their confirmed statuses live in `docs/add_user_negative.json`, pinned under `KNOWN DEFECT DEF-019` in "Add User - Negative Scenarios (Data-Driven)".

#### DEF-020 — Update User: invalid/incomplete input accepted silently

**Endpoint:** PUT /users/{id}
**Expected:** 400 Bad Request for an empty body or wrong data type on a field (age as a string)
**Actual:** 200 OK for both cases — the endpoint accepts the (invalid) body and echoes it back with no server-side validation
**Severity:** Medium
**Note:** same permissive-write pattern as DEF-001/002/003, DEF-016, and DEF-019. Contrast: this endpoint *does* correctly validate that the target user ID exists — `PUT /users/9999` returns 404, unlike Add User which has no existence concept at all. The gap here is specifically body-content validation, not ID validation. Data-driven — all scenarios (including the 404 case, which is correct behaviour, not a defect) live together in `docs/update_user_negative.json`, pinned under `KNOWN DEFECT DEF-020` only for the two permissive rows in "Update User - Negative Scenarios (Data-Driven)".
**Also confirmed on this endpoint (not a defect):** `PUT /auth/users/{id}` with no token correctly returns 401 — DEF-005's missing-token bug does **not** extend here, unlike POST /auth/users/add. Verified live rather than assumed; the auth middleware bug is route-specific, not universal across `/auth/*`.

#### DEF-021 — IDOR: DELETE /auth/users/{id} (sibling of DEF-004, DEF-018)

**Endpoint:** DELETE /auth/users/{id}
**Expected:** 401/403 when a valid token attempts to delete a different user's account than the one the token belongs to
**Actual:** 200 OK — a valid token for one user can delete a completely different user's account by changing `{id}` in the URL
**Severity:** Critical
**OWASP Classification:** API1:2023 — Broken Object Level Authorization / IDOR.
**Note:** third confirmed instance of the same missing-ownership-check vulnerability, after DEF-004 (Delete Cart) and DEF-018 (Get User Carts) — now on Users, on DELETE. `DELETE /auth/users/{id}` with no token correctly returns 401 (DEF-005 does not apply here), and an invalid token correctly returns 401 — the endpoint validates *who* is asking, just never *what* they're allowed to touch.
**Role check:** same as DEF-018 — token owner (`addisonw`, id 30) is a plain `"user"`, target account (id 6, `oliviaw`) is `"moderator"`. A lower-privileged account deleted a higher-privileged one with no role check anywhere, ruling out legitimate admin access as an explanation. Verified live, not assumed.
**Test strategy:** pinned under `KNOWN DEFECT DEF-021`, same CI-hygiene approach as DEF-005/DEF-018.

---

### Findings Under Review (not yet filed as defects)

- **Filter Users — invalid key silently accepted:** `GET /users/filter?key=invalidKey&value=Brown` returns 200 with an empty result set instead of 400. Same permissive-input pattern as DEF-001/002/003. Pending a decision on whether to file.
- **Search Users — empty query returns the full dataset:** `GET /users/search?q=` is not rejected and does not return zero results — it silently falls back to the full unfiltered user list (total 208), the same shape as `GET /users`. Widens the blast radius of DEF-006's plaintext-password exposure. Pending a decision on whether to file (would be DEF-022).

### Features Validated

- ✅ Authentication & JWT lifecycle — login, token refresh, expiry handling
- ✅ Full CRUD coverage across Products, Carts, and Users
- ✅ Pagination & boundary behaviour — limit, skip, and the undocumented `limit=0` "no-limit" anomaly
- ✅ Negative and boundary input validation across every module
- ✅ Security testing — IDOR (DEF-004, DEF-018) and Broken Object Level Authorization (DEF-005, confirmed across both read and write routes) discovery, OWASP API Top 10 awareness
- ✅ Error-contract consistency checks — status codes, JSON vs. HTML error responses
- ✅ Data-driven testing using JSON fixtures (Add Cart, Add User, Update User scenarios)
- ✅ Cross-module E2E business flow — Auth → Users → Products → Cart, chained dynamically with no hardcoded IDs

---

## Tech Stack

- **API Testing:** Postman v11+
- **Test Automation:** Newman CLI
- **Scripting Language:** JavaScript
- **Version Control:** Git/GitHub
- **Reporting:** Newman HTML Extra

---

## How to Run the Tests

### Prerequisites

- **Node.js & npm** installed
- **Postman** installed
- No API key required — DummyJSON is a public mock API

### Installation

1. **Clone the repository:**

   ```bash
   git clone https://github.com/YOUR_USERNAME/ecommerce-api-testing.git
   cd ecommerce-api-testing
   ```

2. **Install Newman:**

   ```bash
   npm install -g newman newman-reporter-htmlextra
   ```

3. **Set up the environment:**
   - Open `postman/environments/DummyJSON.postman_environment.json`
   - The environment exports with an empty `baseURL` value — set it to `https://dummyjson.com` before running

### Run Tests in Postman

1. **Import the collection:**
   - Open Postman
   - Import `postman/collections/Project 2 - E-commerce API- DummyJSON.postman_collection.json`

2. **Import the environment:**
   - Import `postman/environments/DummyJSON.postman_environment.json`
   - Confirm `baseURL` is set

3. **Run Auth → Login first** — it populates `accessToken` and `refreshToken`, which every authenticated request in the collection depends on. A collection-level pre-request script auto-refreshes the token when it expires.

4. **Run the collection:**
   - Click "Run" to execute everything, or run folder-by-folder

### Run Tests via Newman CLI

```bash
newman run "postman/collections/Project 2 - E-commerce API- DummyJSON.postman_collection.json" \
  -e "postman/environments/DummyJSON.postman_environment.json" \
  -r htmlextra,cli \
  --reporter-htmlextra-export reports/newman-report.html
```

## Project Structure

```
ecommerce-api-testing/
├── postman/
│   ├── collections/        # Postman collection — all requests + assertions
│   └── environments/       # DummyJSON environment (baseURL, accessToken, refreshToken)
├── docs/
│   ├── ai-testing-log.md   # Every AI-assisted decision, logged live as it happens
│   ├── prompts/            # Saved prompts from the AI-assisted workflow
│   └── *.json              # Data-driven test fixtures (Add Cart, Add User, Update User — happy/negative datasets)
├── newman/                 # Newman CLI run artifacts
├── reports/                # Newman HTML Extra reports
└── README.md
```

---

## Key Testing Decisions & Learnings

- Verify against live API behaviour before writing an assertion — never assume documented or "expected" behaviour. This is how the `limit=0`, `skip > total`, and empty-query anomalies were caught.
- Extract IDs and expected values dynamically from the request/response — no hardcoded fixture values.
- Known defects are pinned as clearly-labeled `KNOWN DEFECT` assertions documenting actual behaviour, rather than left permanently failing. Keeps the suite CI-safe once gated into GitHub Actions, while the finding stays fully documented in the test itself and in the log.
- Reduced duplication using array-driven and folder-level assertions instead of one assertion per field.
- Every AI-assisted decision — Postbot scaffolding, Claude-assisted regex/logic, ChatGPT suggestions accepted or rejected — is logged in `docs/ai-testing-log.md` with the reasoning, not just the outcome.
- Focused on risk-based coverage (auth, security, boundary values, permissive-input patterns) over exhaustive enumeration.
- Read-only GET endpoints on Products and Carts (Get All Products, Get Single Product, Get All Carts, Get Single Cart, Get Carts by User) have no auth-behavior tests, deliberately — these are genuinely public endpoints with no `/auth/` variant. Auth coverage is applied everywhere an authenticated variant of a route actually exists.
- Dataset totals (products, carts, users) are captured live into environment variables (`expectedProductTotal`, `expectedCartTotal`, `expectedUserTotal`) rather than hardcoded — a dedicated **`00 - Setup`** folder captures each one before any module runs, so downstream boundary and total-count assertions never depend on a repeated magic number. Design rationale in `docs/ai-testing-log.md`.
- Standardized on modern ES6 JavaScript (arrow functions, `const`/`let`) across the entire collection — earlier folders (Auth, Products, most of Carts) were originally written in ES5 style before a deliberate collection-wide rewrite; the reasoning is documented in `docs/ai-testing-log.md`.
- Schema-driven contract testing (`pm.response.to.have.jsonSchema()`) is used on Get Single Product and Get Single User as a deliberate demonstration, replacing manual field-by-field/type checks there. The rest of the collection still uses manual assertions on purpose — schema validates contract *shape*, but can't express business-rule math (discount totals, sort order) or request-specific checks (a returned ID matching the one requested), so both approaches coexist rather than one replacing the other everywhere.

---

## Technical Skills Demonstrated

- ✅ Risk-based API test design
- ✅ JavaScript test scripting in Postman
- ✅ Security testing — IDOR/BOLA discovery, OWASP API Top 10
- ✅ Newman CLI automation
- ✅ HTML report generation
- ✅ Git/GitHub version control
- ✅ AI-assisted testing workflow (Postbot, Claude) with critical evaluation of AI output

## Limitations of Using a Mock API

DummyJSON does not validate stock levels or enforce business rules around inventory. Scenarios testing stock-based constraints (out of stock, exceeding stock) cannot be meaningfully tested against this API.

---

## Contact

**Renuka**

- LinkedIn: www.linkedin.com/in/renukaporje
- Email: renuka.suresh224@gmail.com

---

## Acknowledgments

- **DummyJSON** for providing the free API
- **Postman** for the testing platform
- **Newman** for CLI automation capabilities

---

**⭐ If you find this project helpful, please consider giving it a star!**
