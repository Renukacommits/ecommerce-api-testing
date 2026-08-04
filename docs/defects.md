# Defect Log — DummyJSON E-commerce API Testing

Full defect reports for the project. See the main [README](../README.md) for a summary table and project overview.

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

### Findings Under Review (not yet filed as defects)

- **Filter Users — invalid key silently accepted:** `GET /users/filter?key=invalidKey&value=Brown` returns 200 with an empty result set instead of 400. Same permissive-input pattern as DEF-001/002/003. Pending a decision on whether to file.
- **Search Users — empty query returns the full dataset:** `GET /users/search?q=` is not rejected and does not return zero results — it silently falls back to the full unfiltered user list (total 208), the same shape as `GET /users`. Widens the blast radius of DEF-006's plaintext-password exposure. Pending a decision on whether to file (would be DEF-022).
