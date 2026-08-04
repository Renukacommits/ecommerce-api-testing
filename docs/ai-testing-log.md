# AI Testing Log

**Project 2 — E-commerce API Testing (DummyJSON)**

This log documents every instance where AI tools influenced a testing decision during this project — what was suggested, what was accepted, rejected, or modified, and why. It is evidence of AI-enabled QA practice, not AI-dependent execution.

**AI tools used:** Claude, ChatGPT, Postbot

**Strategy:** Design tests independently first. Then use AI to challenge the plan — evaluate gaps identified, accept what is valid, reject what is not, document both. AI assists efficiency; judgment remains with the QA engineer.

---

## April 2026 : Project Setup — API Selection

**Tool:** Claude

FakeStoreAPI was the initial choice based on the project plan. On first exploration, the API returned an SSL certificate error in the browser. Rather than forcing past the warning, flagged this as a reliability risk for a portfolio project — an unstable API would undermine every test built on top of it.

Raised with Claude. Switched to DummyJSON — better reliability, richer endpoints (products, carts, users, auth), and full CRUD support.

**Decision:** Accepted.

---

## 17-04-2026 : Products / Get All Products / Meta Dates Assertion

**Tool:** Postbot + Claude

Postbot generated an assertion that only validated the date portion of the `createdAt` and `updatedAt` timestamps. Two gaps identified:

1. Time format was not being validated at all — only the date part.
2. The date format check was partial — it would have accepted values like `9999-99-99T` as valid.

With help of Claude and Postbot, improved the assertion to validate the full timestamp using regex — including the time format alongside the date. The assertion now also checks whether the date is actually valid, not just whether it has the right number of digits.

**Decision:** Accepted the improved approach. Postbot's baseline was useful as a starting point but insufficient on its own.

---

## 18-04-2026 : Auth / Login / Valid Credentials

**Tool:** Postbot + Claude

The response had 9 fields to validate — checking both presence and non-null value for each. Initial approach was writing individual assertions per field, which was verbose and brittle.

With help of Postbot and Claude, refactored to store all required fields in an array and iterate using a `for` loop with a single inbuilt assertion. This is more maintainable — adding new fields only requires updating the array, not writing new assertions.

**Decision:** Accepted. Cleaner, more scalable approach.

---

## 18-04-2026 : Auth / Login / Valid Credentials / JSON Web Tokens

**Tool:** Claude (accepted) + Postman autocomplete suggestion (rejected)

Used Claude to understand JWT structure and generate the regex pattern for the assertion. Wrote the assertion independently after understanding it.

A suggestion was made to use the `jsonwebtoken` library. Rejected — this library is not available in the Postman sandbox and requires a secret key to verify, which is not accessible in this context.

**Decision:** Used the regex approach, written independently after understanding it from Claude.

---

## 18-04-2026 : Auth / Login / Valid Credentials / expiresInMins

**Tool:** None — self-identified limitation

`expiresInMins` is accepted by the API in the request body but not reflected in the response. Cannot directly assert token expiry duration without waiting for the token to expire, which is not practical in automated testing.

**Decision:** Test is limited to confirming the request succeeds and tokens are returned. Limitation documented honestly rather than writing a weak assertion to paper over it.

---

## 19-04-2026 : Wrong HTTP Method Scenarios

**Tool:** None — observed during exploration

DummyJSON returns an HTML error response for incorrect HTTP methods instead of a JSON error response. A well-designed API should return consistent JSON error responses regardless of error type.

**Decision:** Documented as a defect (later filed as DEF-009 once reconfirmed on a second module — see 06-07-2026 entry).

---

## 20-04-2026 : GET /products?limit=0 — Limit=0 Behaviour Discovery

**Tool:** None — self-identified during testing

Added a test case for `limit=0` expecting the API to return zero products. The API returned all 194 products. The response body showed `"limit": 194`, not `"limit": 0`. The API interpreted `limit=0` as "no limit — return everything."

**Key insight:** the `limit` field in the response body does not echo back the requested parameter — it reflects the number of items actually returned. `limit=0` in the request returns all 194 products, while `limit: 0` in the response body means 0 items were returned. These look identical but mean completely different things.

**Assertion decision:** did not assert `limit === 0` in the response. Asserted `limit === 194` to document actual behaviour.

**Bug flag:** `limit=0` as "no limit" is undocumented and non-standard. Flagged in the README defects section.

---

## 30-04-2026 : Sort Ascending Anomaly

**Tool:** ChatGPT

ChatGPT suggested using `.toLowerCase()` in the sort order assertion. On running the test, it failed. Investigation revealed the API uses case-sensitive sort. Removed `.toLowerCase()` — assertion now correctly validates actual API behaviour.

**Gap in AI suggestion:** ChatGPT assumed case-insensitive sort without verifying API behaviour first.

**Decision:** Rejected the ChatGPT suggestion. Corrected independently based on the actual API response.

---

## 02-05-2026 : URL Format Not Validated

**Tool:** None — self-identified gap

**Endpoint:** GET /products/categories

ChatGPT did not validate URL format in its suggestion — only checked that the value was a non-empty string. Added a regex test independently to verify URL structure.

**Decision:** Self-identified and corrected.

---

## 02-05-2026 : Weak Test Identified

**Tool:** None — self-identified

**Endpoint:** GET /products/category-list

Initial assertion used `include.members`, which would pass even with missing slugs. Corrected to `have.members` for exact-match validation.

**Decision:** Self-identified gap. Corrected independently.

---

## 03-05-2026 : Dynamic ID Extraction in CRUD Operations

**Tool:** Claude

**Endpoint:** /products/1

Instead of hardcoding `id: 1`, used Claude to identify `pm.request.url.path` and `parseInt()` to dynamically extract the product ID from the request URL at runtime. Makes the test reusable for any product ID.

**Decision:** Accepted and understood. Explained the logic independently before implementing.

---

## 07-05-2026 : Thumbnail URL Regex Failure

**Tool:** None — self-identified during testing

Thumbnail URL regex failed twice after initial build. First failure: an apostrophe in a product name. Second failure: an ampersand in a product name. Both fixed by inspecting actual console output rather than assuming. Final regex accounts for real-world special characters in DummyJSON URLs.

**Decision:** Self-identified and fixed through investigation.

---

## 07-05-2026 : Skip Beyond Total

**Tool:** Claude (gap missed)

**Endpoint:** GET /carts?skip=209

Claude focused on the `limit` parameter behaviour and did not identify the `skip > total` defect. When skip exceeds the total number of records, the API returns 200 with an empty array and `skip: 209` echoed back — an impossible position in the dataset.

**Decision:** Self-identified defect; Claude missed it entirely. Filed as DEF-013.

---

## 14-05-2026 : Update Cart / Folder + Request Scaffolding

**Tool:** Postbot

**Endpoint:** PUT/PATCH /carts/{id}

Update Cart required 29 requests across 5 subfolders. Rather than creating each request manually — a repetitive, low-value task — fed the agreed folder structure and request names to Postbot and asked it to scaffold the full structure. Postbot created 18 new requests (the remaining ones after 11 already existed), correctly named and placed in the right subfolders.

Two issues identified on review:

1. Postbot defaulted the HTTP method to GET on several requests — manually corrected to PUT or PATCH as appropriate.
2. Two pre-existing requests (cartId = 0, non-existing cartId = 9999) were misplaced inside `Invalid body/` instead of `Invalid cartId/` — moved manually.

**Decision:** Postbot used for scaffolding only — naming and placement. All request bodies and assertions written and validated independently. AI saved repetitive setup time; quality control remained with the QA engineer.

---

## 05-07-2026 : Filter Users / Invalid Key Behaviour

**Tool:** None — self-verified via live API call

**Endpoint:** GET /users/filter?key=invalidKey&value=Brown

**Expected:** 400 Bad Request (unknown filter key)
**Actual:** 200 OK, empty result set:

```json
{ "users": [], "total": 0, "skip": 0, "limit": 0 }
```

**Note:** `limit: 0` here means "0 items returned" — do not confuse with the separate `limit=0`-means-no-limit anomaly found on Products/Carts. Different meaning, same field name.

**Assertion impact:** do not assert 400. Assert `total === 0` and `users` is an empty array.

**Defect candidate:** DummyJSON doesn't validate `key` against known fields — permissive-input pattern consistent with DEF-001–003. (Written into the collection's assertions on 15-07-2026 — see Filter Users entry below.)

---

## 06-07-2026 : Users / Invalid Method & Invalid URL — Confirmed Defect

**Tool:** None — self-verified via live API call

Both return HTML, not JSON — the same pattern already documented on Products (19-04-2026 entry above). Logging this as a repeat of that finding, now confirmed on Users too.

**Endpoint:** POST /users (wrong method); any nonexistent `/users/*` path

**Actual:**
- `POST /users` → HTML page: "Cannot POST /users"
- Nonexistent path → HTML 404 page (fully branded DummyJSON error page)

**Expected:** a consistent JSON error response regardless of error type
**Actual:** HTML in both cases

**Assertion impact:** don't assert a JSON schema on these two — assert `Content-Type` is NOT `application/json`, or assert on the HTML string itself if pinning the defect.

**Decision:** Confirmed as a repeat of the Products finding. Filed together as DEF-009.

---

## 14-07-2026 : DEF-005 — Broken Object Level Authorization on GET /auth/users/{id}

**Tool:** None — self-identified during Get Single User assertion testing

**Endpoint:** GET /auth/users/{id}

**Discovery:** "Get Single User - No Auth Token" was written expecting 401 (mirroring the correctly-secured `GET /auth/users` list endpoint). On execution, the test failed — the API returned 200 OK and the complete user object: full name, email, plaintext password, phone, SSN, credit card, IP, address, macAddress — with no bearer token supplied.

**Severity:** Critical

**OWASP Classification:** API1:2023 — Broken Object Level Authorization (BOLA), #1 on the OWASP API Security Top 10.

**Inconsistency:** same DummyJSON server, same `/auth/` prefix:
- `GET /auth/users` (list) → returns 401 as expected
- `GET /auth/users/1` (single) → returns 200 with full PII

The `/auth/` prefix implies authorization is required. The list endpoint honours that. The single-resource endpoint does not — a classic BOLA pattern where authentication is enforced at the collection level but skipped at the item level.

**Decision (revised 15-07-2026):** originally kept assertions expecting 401 so the test would fail as documentation of the vulnerability. Revised after weighing CI/GitHub Actions implications with Claude — a test that is *expected* to stay red permanently breaks the green = deployable signal once the suite is wired into a CI gate, and makes it impossible to distinguish this known issue from a genuine new regression at a glance. Assertions now pin the current (insecure) 200-with-PII behaviour, with test names explicitly labeled `KNOWN DEFECT DEF-005` and a comment block explaining the vulnerability, root cause, and OWASP classification. If DummyJSON ever patches this, the pinned test will start failing — that failure is the trigger to flip the assertions back to the secure expectation and close the defect. Known defects belong in the defect log, not as permanently-failing CI assertions.

**Related findings:** compounds the earlier "password returned in plaintext" observation logged for Get All Users (DEF-006) — the same PII is now accessible without any authentication at all.

**Extension confirmed 15-07-2026:** same root-cause bug (auth middleware validates a token only when one is present, with no else-branch to reject a request that sends no Authorization header at all) reproduced independently on a second endpoint — `GET /auth/users/search` with no token also returns 200 with the full matching user list, plaintext passwords included, instead of 401. Verified live by Renuka in Postman (Claude's fetch tool could not check independently — see 15-07-2026 entries below). Not filed as a separate defect — widens DEF-005 from a single route to a middleware-wide finding. Note: the "Invalid Token" scenario on `/auth/users/{id}` correctly returns 401, confirming the validator itself works when a token is present — the gap is specifically the missing-token branch.

---

## 15-07-2026 : Search Users Assertions + DEF-005 Pinning Strategy

**Tool:** Claude

Wrote pm.test assertions for all 6 requests in the Search Users folder (Valid Query, No Match, Empty Query - Edge, No Auth Token, Invalid Token, Valid Token). Verified live API behaviour first rather than assuming: confirmed `q` matches against firstName/lastName/username but NOT maidenName or company name (tested `q=Smith` vs `q=Dooley` live), so the "results are relevant to the query" assertion checks name/username/email fields instead of guessing at DummyJSON's undocumented search scope.

Found that `q=` (empty string) is not rejected and does not return zero results — it silently falls back to the full unfiltered user list (same shape as GET /users: total 208, limit 30). Logged as a defect candidate, pending Renuka's decision on whether to file it (would be DEF-019).

Separately, revisited DEF-005: Claude initially wrote the Search Users "No Auth Token" test expecting 401 by pattern-matching against the correctly-secured Get All Users list endpoint. Flagged this as unverified before claiming the folder complete — DEF-005 already proved auth enforcement is inconsistent across endpoints on this API, so pattern-matching across routes isn't a safe substitute for checking. Asked Renuka to verify live in Postman. Confirmed: the same BOLA bug reproduces on `/auth/users/search` — 200 OK, full user list leaked, 3 assertions failed as predicted.

**Decision:** Rewrote both the original `/auth/users/{id}` DEF-005 test and the new `/auth/users/search` No Auth Token test to pin the current (insecure) behaviour under clearly labeled `KNOWN DEFECT DEF-005` test names instead of leaving them permanently red. Known defects now live in the defect log with full detail; the pinned tests will flip red again automatically if the bug is ever fixed.

**AI limitation acknowledged:** Claude could not independently verify the `/auth/users/search` no-token behaviour — its fetch tool returned nothing for `/auth/*` paths at the time, and it has no ability to execute authenticated Postman requests (Postman MCP is management-API only). Renuka ran the verification manually. Logged as an explicit gap rather than an assumed result.

---

## 15-07-2026 (cont'd) : Filter Users Assertions

**Tool:** Claude

Wrote assertions for all 7 requests in Filter Users. Verified live first: `GET /users/filter?key=hair.color&value=Brown` does exact-match filtering (every one of the 23 returned users genuinely has `hair.color === "Brown"`), and `invalidKey`, missing `value`, and missing `key` all return 200 with an empty result rather than erroring — same permissive-input family as DEF-001/002/003.

The "match" assertion resolves the `key` query param dynamically (`key.split('.').reduce(...)`) instead of hardcoding `hair.color` — works for any dot-path key the request uses, not just this one.

Noted a genuine contrast worth keeping in mind: Filter Users returns an *empty* result when a required param is missing, while Search Users' empty `q=` falls back to the *full* dataset (see entry above). Two similarly-shaped endpoints, two different permissive-input behaviours — not assumed to be consistent just because they're both under `/users`.

**Auth trio:** No Auth Token / Invalid Token / Valid Token written against the secure expectation (401 / 401 / 200), matching house pattern. Given DEF-005 had already reproduced on 2/2 tested `/auth/users/*` routes with the same root cause, No Auth Token was flagged as a strong candidate for the same bug and sent to Renuka for live confirmation rather than assumed.

**Confirmed 15-07-2026:** Renuka verified — same behaviour. `GET /auth/users/filter` with no token returns 200 with the full filtered user list leaked. This is now **3/3 tested `/auth/users/*` routes** exhibiting the identical root cause (`{id}`, `search`, `filter`). At this sample size it's reasonable to treat it as a property of the auth middleware itself rather than a per-route defect — every `/auth/users/*` route should be assumed vulnerable until proven otherwise, not verified one at a time as new folders are built. Rewrote "Filter Users - No Auth Token" to pin the current behaviour under `KNOWN DEFECT DEF-005`, same pattern as the other two.

---

## 15-07-2026 (cont'd) : Get User Carts Assertions — DEF-005 (4th confirmation) + New DEF-018

**Tool:** Claude

Wrote assertions for all 7 requests. One live-verifiable directly (Valid User With Carts — confirmed the strict 1:1 userId:cartId mapping holds on this sub-resource too), but three needed Renuka to check in Postman: Claude's fetch tool returns nothing for *any* non-200 response, not specifically `/auth/*` paths as previously logged — this general limitation was only discovered this turn, by cross-checking against the already-passing `/users/9999` test (also blank via fetch, despite having nothing to do with auth). The 15-07-2026 Search Users entry above should be read with that correction in mind.

Renuka reported:

- **Boundary (209):** 404, `{"message": "User with id '209' not found"}` — same message format as the existing Get Single User 404 test, written to match.
- **Invalid User ID (abc):** 400, `{"message": "Invalid user id 'abc'"}` — different status *and* message shape from the 404 case. Worth noting as *good* design, in contrast to DEF-014: this endpoint correctly distinguishes malformed input (400) from valid-but-missing input (404).
- **No Auth Token:** 200, full cart data leaked. **DEF-005, 4th confirmed endpoint** (`{id}`, `search`, `filter`, now `{id}/carts`). At 4/4 this is unambiguously a middleware property, not a per-route coincidence.
- **IDOR test:** 200, and the leaked cart genuinely belongs to userId=6 — not the token owner. **New defect, DEF-018** — same vulnerability class as DEF-004 (Delete Cart IDOR), now confirmed on a GET route too. No ownership check anywhere on this endpoint, only a validity check on the token itself.

**Design note flagged, not fixed unilaterally:** "Get User Carts - Valid Token" and "Get User Carts - Another User's Carts - Security (IDOR)" both hit the exact same hardcoded URL (`/auth/users/6/carts`). Since userId 6 isn't the logged-in test account, these two tests currently exercise the *same* scenario — "Valid Token" isn't actually testing "a user can see their own data," it's silently re-confirming DEF-018. Flagged to Renuka; would need "Valid Token" repointed to `{{userId}}` to be a genuine, distinct positive-path test.

**Decision:** DEF-005 and DEF-018 both pinned under `KNOWN DEFECT` test names per the established CI-hygiene strategy — pin current behaviour, let a future fix flip the test red as the signal to close the defect.

---

## 15-07-2026 (cont'd) : Documentation Cleanup

**Tool:** Claude

Renuka requested an ongoing standard: documentation (this log, the README, and `docs/prompts/`) gets updated in the same turn as the Postman work, not as a separate ask each time — and defect numbering must always be re-derived from actual doc/collection content, never assumed from memory.

Acted on both immediately: removed a duplicate/stale draft log file (`docs/ai-testing-log-1.md` — superseded by this file), rewrote the README's defects section into consistent `DEF-XXX` / Endpoint / Expected / Actual / Severity format, and replaced leftover boilerplate from an earlier Weather-API project template (setup instructions, project structure, "Features Validated" list) with content matching the actual DummyJSON project. Coverage table numbers (182 total requests, per-module breakdown) computed directly from the live Postman collection rather than estimated.

**Self-caught inconsistency:** DEF-018 had been inserted into the README immediately after DEF-005 (chronological insertion point) instead of after DEF-017 (numeric order) — exactly the kind of drift that happens when relying on "next number I remember" instead of re-scanning actual content. Corrected during this cleanup pass. Also found and fixed a genuine date error while auditing (not just formatting): the DEF-005 discovery entry above was dated "4-07-2026" — a dropped digit; every other reference to the same discovery (including the comment inside the live Postman test script) says 14-07-2026.

---

## 15-07-2026 (cont'd) : Scope Correction — Get User Posts/Todos Removed

**Tool:** Claude

Renuka challenged the inclusion of the Get User Posts and Get User Todos folders — questioned their relevance to an e-commerce API testing project, since DummyJSON's Posts and Todos resources are generic blog/task-management filler data with no connection to products, carts, or orders. Confirmed via DummyJSON's own documentation: Posts is described as "sample blog post data... for content management and social media features," Todos as "sample to-do items... for task management and productivity applications." Neither has any e-commerce relevance.

Both folders removed from the Postman collection. The Postman MCP connector has no delete capability (confirmed via tool search — create/read/update/duplicate/spec-generation only), so the deletion itself was done manually by Renuka in the Postman UI; Claude updated the README's request counts afterward (182 → 168 total, Users 82 → 68).

**Decision:** Accepted. Get User Posts had 6/7 requests with assertions already written at the time of the scope challenge — the correction came after partial work, not before. Worth naming directly: the scope question should have been raised before writing those assertions, not after.

---

## 15-07-2026 (cont'd) : Add User — Data-Driven Refactor + DEF-005 Extended to a Write Route + New DEF-019

**Tool:** Claude

Add User had 8 requests already scaffolded (URL/method/body, no assertions) with hardcoded payloads baked individually into each request — flagged as a plan deviation, since the project was meant to use data-driven testing (JSON fixture + `pm.iterationData`) the same way Add Cart already does, not one-off hardcoded requests per scenario.

Renuka asked a sharp question before agreeing to consolidate: how do you combine happy, negative, and edge cases into one data-driven request when expected outcomes differ per row? Answer, demonstrated from the existing Add Cart implementation: the expected outcome (`expectedStatus`) is itself a field in the data row, read at runtime via `pm.iterationData.get("expectedStatus")` — the script never hardcodes a status. Response-*shape* assertions (which legitimately differ between success and error responses) are the one place a data-driven script branches, and that branch condition is still derived from the row's own expected value, not from which scenario happens to be running.

Before writing any `expectedStatus` values, asked Renuka to run all 8 existing requests live rather than assume behaviour from DummyJSON's docs (which hint at permissiveness but aren't authoritative). Results: 7 of 8 returned 201, one — "Add User - No Auth Token" — returned 201 where 401 was expected.

**Two findings from the live results:**

1. **DEF-005 extended to a 5th route, the first write route.** POST `/auth/users/add` with no token returns 201 instead of 401; the sibling "Invalid Token" request correctly returns 401 — identical root-cause signature to the 4 already-confirmed GET routes (validator only runs when a token is present, no else-branch for a missing header). Impact is narrower here since Add User is simulate-only (no real record created or exposed), so this is a broken-authentication-*enforcement* finding rather than a data-exposure one, and the README's DEF-005 entry was updated to say so explicitly rather than implying identical impact across all 5 routes.
2. **New defect, DEF-019.** The 3 "negative" scaffolded scenarios (empty body, missing required fields, wrong data type for age) all returned 201 instead of 400 — same permissive-write pattern as DEF-001/002/003 (Update Cart) and DEF-016 (Add Cart), now confirmed on a Users-module endpoint. Filed as its own defect (not folded into DEF-001/002/003) since it's a different resource, consistent with how DEF-018 was kept separate from DEF-004 despite sharing a root cause.

**Refactor:** collapsed the 8 hardcoded requests down to 5 working ones — 3 auth-specific (No Auth Token, Invalid Token, Valid Token, kept as individually named requests, mirroring how Add Cart's "Auth" subfolder is *not* data-driven) plus 2 new parameterized requests ("Add User - Happy Path (Data-Driven)" and "Add User - Negative Scenarios (Data-Driven)") driven by new fixtures `docs/add_user_happy.json` (6 rows) and `docs/add_user_negative.json` (3 rows, all confirmed live before being written — no invented status codes). The happy-path script asserts every submitted field is echoed back generically via `Object.keys(payload).forEach(...)`, so it works unchanged regardless of which fields a given row sends — no per-scenario hardcoding.

The 3 remaining hardcoded requests that are now redundant (Valid Payload - Full Fields, Negative - Missing Required Fields, Negative - Wrong Data Type for Age) can't be deleted via the Postman MCP connector — flagged to Renuka for manual deletion in the UI, same limitation as the Posts/Todos removal above.

**Decision:** Accepted. DEF-019 pinned under `KNOWN DEFECT DEF-019` per the established CI-hygiene strategy (see Prompt 05). DEF-005's README entry rewritten to 5/5 with the write-route impact caveat rather than silently generalizing the GET-route framing to the new route.

---

## 15-07-2026 (cont'd) : Update User — Data-Driven Refactor + New DEF-020, DEF-005 Does Not Extend Here

**Tool:** Claude

Same 8-hardcoded-requests-no-assertions starting state as Add User. Before writing anything, needed live results for all 8 — Renuka flagged that manually pasting 8 statuses one at a time was tedious. Checked whether Claude could send the PUT/POST requests directly instead: the web-fetch tool is GET-only (no method parameter in its schema), and routing around that via the sandbox's shell (`curl`) was blocked at the network layer (`403 blocked-by-allowlist` from the sandbox's outbound proxy) — confirmed by testing, not assumed. Postman remains the only tool with a live, unrestricted connection to the API. Landed on a lower-friction alternative: run the whole folder once via Postman's Collection Runner and paste the summary table, instead of opening and reading each request individually.

Results: Single Field 200, Multiple Fields 200, No Auth Token 401, Invalid Token 401, Valid Token 200, Negative - Invalid User ID (`/users/9999`) 404, Negative - Empty Body 200, Negative - Invalid Data Type 200.

**Two things worth noting, one a non-finding:**

1. **DEF-005 does NOT extend to `PUT /auth/users/{id}`.** No Auth Token correctly returned 401, not 200/201 like the 5 previously-confirmed routes. This is exactly why the project's "verify live before assuming, even after a pattern holds N times" discipline exists — after DEF-005 held on 5/5 tested routes including the first write route (Add User), it would have been easy to assume it holds everywhere under `/auth/`. It doesn't. The auth middleware bug is route-specific, not a blanket `/auth/*` failure — a more precise and more credible claim for the README than "everything under /auth/ is broken" would have been.
2. **New defect, DEF-020.** Empty body and wrong-type `age` both returned 200 instead of 400 — same permissive-write family as DEF-001/002/003/016/019. But the non-existent-user-ID case (`/users/9999`) correctly 404'd — unlike Add User, this endpoint does validate that the target resource exists. The defect is scoped precisely to body-content validation, not ID validation.

**Refactor:** same shape as Add User — collapsed 8 requests to 5 (3 auth requests kept individually, 2 new data-driven requests). The negative fixture (`docs/update_user_negative.json`) deliberately mixes a correct-behaviour row (404, `isKnownDefect: false`) with two defective-behaviour rows (200, `isKnownDefect: true`) in the same data file — this is the direct answer to the question Renuka raised earlier about combining scenarios with genuinely different expected outcomes in one data-driven request: the expected status *and* whether a row represents a known defect both travel as data, and the test script branches on the row's own `isKnownDefect` flag rather than hardcoding which scenario is "the special one." 3 now-redundant hardcoded requests (Multiple Fields, Negative - Invalid User ID, Negative - Invalid Data Type) flagged for manual deletion in Postman — same MCP delete limitation as before.

**Decision:** Accepted. DEF-020 pinned under `KNOWN DEFECT DEF-020`, applied only to the two permissive rows — the 404 row gets a plain (non-defect) assertion instead.

---

## 15-07-2026 (cont'd) : Delete User — New DEF-021 (Third IDOR Confirmed)

**Tool:** Claude

8 requests, same shape as Delete Cart (individually named, not data-driven — no varying body on a DELETE call). Renuka ran the folder once via Collection Runner and pasted all 8 statuses.

Results: Valid ID 200, No Auth Token 401, Invalid Token 401, Valid Token 200, IDOR test 200, Negative - Invalid ID 400, Delete Twice 200, GET after Delete 200.

**Key finding: DEF-021, a third confirmed IDOR.** The "Another User's Account" test — a valid token deleting a different user's account (id=6) — returned 200 instead of being rejected. Same missing-ownership-check vulnerability as DEF-004 (Delete Cart) and DEF-018 (Get User Carts), now on a third resource/verb combination. `No Auth Token` correctly returned 401 (DEF-005 doesn't apply here either, consistent with Update User's PUT route), so this endpoint's problem is entirely ownership, not authentication — validates *who's* asking, never *what* they're allowed to touch.

Everything else confirmed expected/documented behaviour, not new findings: invalid ID format correctly 400s; repeat delete and GET-after-delete both succeed because DummyJSON's DELETE is simulate-only (same documented limitation as Delete Cart).

**Decision:** Accepted. DEF-021 pinned under `KNOWN DEFECT DEF-021`, sanity-checked against the token owner's own userId first (same pattern as DEF-018's IDOR test), so the test provably exercises a genuine cross-user case.

**Users module now functionally complete** — all 7 folders have assertions. 3 leftover hardcoded requests from the Update User refactor remain pending manual deletion in Postman.

---

## 15-07-2026 (cont'd) : Role-Legitimacy Gap Caught on All Three IDOR Findings

**Tool:** Claude (caught by Renuka's question, not self-identified)

Renuka asked directly: for DEF-021 (and by extension DEF-004, DEF-018), did the IDOR check confirm the token owner wasn't an admin with legitimate cross-account access? It hadn't — every IDOR test to date only checked that the requested ID differed from the token owner's ID, never whether the token owner held an elevated role that would make the access legitimate rather than a bug.

Investigated rather than assumed: pulled the collection's login request body to find the fixed test account (`addisonw`), looked up that account via a live `GET /users/search?q=addisonw` (id 30), and separately looked up the target account used in both DEF-018 and DEF-021 (id 6, `oliviaw`) via `GET /users/6`. Result: `addisonw` holds a plain `"user"` role; `oliviaw` holds `"moderator"`. A lower-privileged account deleting/reading a higher-privileged one rules out "the token owner is an admin" as an explanation — the findings hold.

Also checked DEF-004 (Delete Cart IDOR): Carts have no role/ownership concept beyond `userId`, so a role check doesn't apply there — narrowed the fix to the two User-resource findings only, rather than over-applying it.

**Fix, not just a note:** added a runtime role check to both DEF-018 and DEF-021's test scripts — a live `pm.sendRequest` to `GET /users/{tokenOwnerId}` asserting `role === 'user'`, rather than hardcoding the now-known role as a static value. This was a deliberate call: the role is verified today, but a live check means if the test account is ever swapped for one with a different role, the assertion catches it instead of silently going stale.

**Decision:** Accepted. This was a real gap in the original test design across all three IDOR findings, not just DEF-021 — logged as-is rather than reframed as something already covered.

---

## 15-07-2026 (cont'd) : Collection Audit + Coverage Gap Closure (Section 2)

**Tool:** Claude

Full read-through of all 162 requests across every module to check test coverage, assertion quality, naming, hardcoded test data, coding-standard consistency, maintainability, and Postman-feature usage. Findings written up in `docs/collection-audit-2026-07-15.md`. Renuka chose to work through it section by section, starting with coverage gaps.

**Fixed in this pass:**

1. **DEF-013 (Carts: skip beyond total) had zero executable assertions** — only a code comment documented the bug. Pinned properly under `KNOWN DEFECT DEF-013`, matching the standard used everywhere else in the collection.
2. **`GET /auth/me` had no negative-path tests at all** — the one endpoint the audit flagged as genuinely untested either way, given DEF-005's exact bug signature. Checked live (Renuka via Collection Runner): No Auth Token → 200 (bug confirmed), Invalid Token → 401 (correctly rejected). **DEF-005 extended to a 6th confirmed route.** Also confirmed PUT/DELETE on the same user resource are NOT affected — the assumption is now scoped to matching-verb routes, not the whole `/auth/*` surface, which is a more precise and defensible claim than before.
3. **Refresh auth session had no missing-refreshToken test** — added. Returns 401 (not 400), which is a defensible REST convention (401 = no credentials at all, vs. 403 for the existing invalid-refreshToken case) rather than a new defect — logged as a design observation, not filed.
4. **Read-only GET endpoints on Products/Carts have no auth tests** — confirmed this is intentional (they're genuinely public, no `/auth/` variant exists) and added one line to the README stating so explicitly, so it doesn't read as an oversight.

**Decision:** Accepted. Coverage-gaps section of the audit closed. Remaining sections (weak assertions, hardcoded test data, coding standards, maintainability, underused Postman features) queued for subsequent passes.

---

## 15-07-2026 (cont'd) : Collection Audit — Section 3, Weak/Thin Assertions Closed

**Tool:** Claude

Second audit section: requests with noticeably fewer assertions than their folder siblings, flagged as inconsistent rather than intentional.

**Fixed in this pass (9 requests total):**

1. **`pm.response.to.be.json;` bare-property pattern (5 Carts negative-path requests)** — technically valid Chai syntax but reads like a typo (no parentheses, no explicit `pm.expect`), and the test name didn't match what was actually being checked. Rewritten in all 5 to an explicit `pm.expect(pm.response.headers.get('Content-Type')).to.include('application/json')`.
2. **"Wrong HTTP method" (Auth, DEF-008) and "Wrong method" (Refresh, same defect)** — previously asserted only status code + response time. Before adding a body check, needed to know whether the response was JSON or HTML — DummyJSON runs on Express, and DEF-009 already established elsewhere in the collection that unmatched-route/method requests return an HTML error page, not JSON. Renuka confirmed live: both return 404 with an HTML body. Added `pm.response.text()` + a lowercase `.include("cannot patch /auth/login")` / `.include("cannot get /auth/refresh")` check, matching the existing pattern already used in "GET all products - invalid method."
3. **"GET all products - invalid URL" and "Get a single product: Invalid URL"** — same gap, same fix. Renuka confirmed both are 404/HTML live. Added the equivalent `cannot get /productss` / `cannot get /productss/1` body checks.

**Decision:** Accepted, no defects filed — this section was about assertion strength, not new bugs. All 4 body-check additions were written only after live confirmation of response shape (HTML vs JSON), not assumed from the DEF-009 precedent alone. Section 3 of the audit closed. Remaining sections (hardcoded test data, coding standards, maintainability, underused Postman features) queued next.

---

## 16-07-2026 : Collection Audit — Section 4, Test Data & Hardcoding Closed

**Tool:** Claude

Third audit section. Renuka flagged from memory that she thought a username had already been variablized in some of the newer folders and asked for it to be checked before assuming the audit's finding still held.

**Checked, not assumed:** pulled the live collection and searched every occurrence of the literal `addisonw` and the `username` field. Confirmed both things were true at once: the Login request itself still hardcoded the username inline in its body (password was already a variable, username wasn't); but everything *downstream* of Login was already fully dynamic — the test script captures `{{userName}}` from the response, and "Get current auth user" compares against `{{userName}}` via a field-map loop rather than re-hardcoding it. So the gap was real but narrower than the original audit line implied: one literal at the very top of the chain, not a pattern repeated throughout.

**Fixed:** added `testUsername` to the environment (mirrors how the password is already a variable, `auth_password_0pef`) and pointed the Login body at `{{testUsername}}`.

**Magic numbers (194/208) — design discussion before implementation:** Renuka asked directly whether making boundary tests depend on an earlier request's run order was a good idea or a risk. Walked through the trade-off explicitly rather than just picking an approach: full-collection runs preserve order (no issue); a single isolated request run cold fails loudly (safe, just confusing); the one real risk is a stale cached value silently surviving after the underlying dataset changes — but that's no worse than the *existing* hardcoded-literal risk, just relocated. Proposed reusing the collection's own established precedent (Login → downstream chaining) rather than inventing heavier machinery. Renuka pushed back on the offered "production-grade self-healing" (pre-request auto-refresh, mirroring the JWT-refresh script) as overkill for this project's scope, but wanted the simpler version to still read as deliberate and well-considered rather than half-done.

**Investigated before building anything — found the target pattern already existed.** Expected to build a baseline-capture mechanism from scratch, but a live check of the Users module showed `expectedUserTotal` was already wired up correctly: "Get All Users - Default" captures it with a lazy-init guard (`if (!baselineTotal && ...)`, documented in-script with an explicit "how to reset the baseline" comment), and "Get Single User - Valid (Upper Boundary id=208)" already derives its expected ID from the URL dynamically and separately checks that ID against the captured baseline as a "dataset continuity" regression sensor. The audit's claim that `expectedUserTotal` was "unused" was accurate when written but had already been overtaken by newer work — worth noting as a reminder that audit findings can go stale mid-project and should be re-verified against the live collection before acting on them, not just trusted from the document.

**Fix:** replicated the exact same pattern for the other two resources rather than designing something new:
- Products: `expectedProductTotal` captured by "Get all products" (first request in the folder); consumed by "GET all products - limit 0" (total-count assertion, plus made the array-length check self-referential against the response's own `total` field so it needs no baseline at all), "GET all products - skip beyond total" (total-count assertion, and the skip-echo assertion now derives the expected skip from the request's own URL instead of a second hardcoded number), and "Get a single product: ID -194" (expected ID derived from the URL, plus a new continuity check against the baseline).
- Carts: `expectedCartTotal` captured by "Get all carts - Happy path"; consumed by "Skip - Edge case"'s total-count assertion.
- Fixed a related, smaller issue found along the way: "New product, Boundary cartId = 208 (PUT)" had the cart ID hardcoded twice — once in the request URL, once independently in its pre-request script's `pm.sendRequest` call. The two could silently drift apart if one was ever updated without the other. Pre-request script now reads the cart ID from the request's own URL instead.
- Boundary URLs themselves (`/products/194`, `/carts/208`, `/users/208`) were deliberately left as literal path segments, not converted to `{{variable}}` — matching the already-established Users precedent, and intentionally avoiding the ordering fragility Renuka asked about for the one place where it would have mattered most (the request configuration itself, not just its assertions).
- Also fixed the unrelated dead code flagged in the same audit section: `console.log(categories)` in "Get product categories" referenced an undeclared variable — removed.

**Decision:** Accepted. Section 4 of the audit closed. Remaining sections (coding standards, maintainability, underused Postman features) queued next.

---

## 16-07-2026 (cont'd) : Dedicated Setup Folder — Closing the Run-Order Gap Properly

**Tool:** Claude

Continued the design discussion on the baseline-capture approach before finalizing it.

Laid out the real spectrum: (1) what was already built — capture on each module's own first request, works for a full run, fragile for an isolated single-folder run; (2) a dedicated Setup folder at the top of the collection that owns the capture explicitly, a well-known pattern in professional Postman/Newman suites; (3) full self-healing via the existing collection-level pre-request script (the one already doing JWT refresh), extended to also check-and-populate the totals on any request, regardless of order. Recommended (2): (3) is genuinely production-grade infrastructure that doesn't earn its complexity on a mock API portfolio project, and the actual skill worth demonstrating is making and explaining a deliberate trade-off, not building resilience machinery nothing here needs. Renuka agreed on (2).

**Constraint hit immediately:** the Postman MCP connector can create requests but has no folder-creation or item-reordering capability — same category of limitation as the earlier Posts/Todos folder deletion. Asked Renuka to create an empty `00 - Setup` folder and drag it above Auth in the Postman UI herself; she confirmed once done.

**Built:** 3 new lightweight requests inside `00 - Setup` — "Setup: Capture Product Total Baseline" (`GET /products?limit=1`), "Setup: Capture Cart Total Baseline" (`GET /carts?limit=1`), "Setup: Capture User Total Baseline" (`GET /users?limit=1`). Each does the minimal work needed: assert 200, capture `total` into the matching environment variable, assert the capture succeeded. `limit=1` keeps each request's payload minimal since the `total` field is present regardless of page size — a small but deliberate performance-conscious choice worth naming as such.

The existing lazy-init captures inside "Get all products," "Get all carts - Happy path," and "Get All Users - Default" were left in place rather than removed — with Setup now the guaranteed primary source, those become a harmless redundant fallback (defense in depth), not dead logic.

**Decision:** Accepted. This is the version of the fix that reads as deliberately engineered rather than "hardcoding relocated to a variable" — the folder makes the dependency a visible, first-class part of the collection's structure instead of an implicit side effect of an unrelated functional test happening to run first.

---

## 16-07-2026 (cont'd) : Collection Audit — Section 5, Coding Standards Closed (ES5 → ES6)

**Tool:** Claude

Fifth audit section: the collection had two visibly different eras of JavaScript style — ES5 `function () {}` / `var` across Auth, Products, and most of Carts (written April–June), versus ES6 arrow functions and `const`/`let` across all of Users (written more recently). Renuka tested the standard directly by asking why `const`/`let` had been the advice given for new work when some of that same session's own new scripts (the `00 - Setup` folder requests) had been written in `var`. Caught and owned specifically, not generically: those Setup requests were brand new with zero legacy-style justification, which is exactly why they should have been `const`/`let` from the first draft. That inconsistency became the deciding factor for choosing a full rewrite over documenting the split as an acceptable trade-off.

**Options weighed:** (A) document the split as a deliberate choice, change nothing; (B) rewrite the entire collection to ES6; (C) rewrite one representative module and note the rest as a scoped follow-up. Renuka chose B given the scale (~150+ scripts) was worth doing properly rather than partially.

**Reconciliation given on the underlying rule, since it looked contradictory at first:** `const`/`let` over `var`, and arrow functions over `function () {}`, are always the right default for new code or any code already being touched for another reason. Retrofitting a large volume of already-working, untouched legacy code purely for cosmetic consistency is a separate call — the standard "boy scout rule" distinction (enforce on new/touched code; don't force unrelated big-bang rewrites of stable code). Choosing B here means the whole collection is being touched deliberately, so the distinction collapses back into "just use the standard everywhere."

**Executed folder by folder** (Postman's API only supports full script replacement, not line-level patches, so every request needed its complete `test`/`prerequest` script read and rewritten in full): Auth (14 requests converted, 3 already ES6-compliant), Products (22 converted, including fixing one implicit global — `responseJson = pm.response.json();` with no declaration keyword, in "Update a product"), Carts (every subfolder — Get All Carts, Get Single Cart, Get Carts by User, Add a cart, Update Cart with its Auth/merge-true/merge-false/Negative sub-subfolders, Delete Cart). Stray `\r` line-ending artifacts from the original scripts were dropped along the way as a side effect of the rewrite, not a separate pass.

**Scope discipline:** this was strictly a syntax pass — `function` → arrow, `var` → `const`/`let`, nothing else. Existing test names, assertions, business-rule math, defect-reference comments (e.g. the DEF-014 explanatory comment in the Carts-by-user-id negative test), and the baseline-capture/regression-sensor logic added in the Section 4 work were all preserved exactly as-is. A pre-existing weak assertion pattern (`pm.response.to.be.json;`, a bare property access) was spotted again in a handful of Delete Cart requests during this pass — left untouched, since Section 3 (weak/thin assertions) is already closed and re-opening it wasn't part of this section's scope; worth a follow-up note rather than scope creep here.

**Known gap:** 3 Carts requests — "Update Cart - No Token," "Delete Cart - No Auth Token," "Delete Cart - Expired or Invalid Token" (all exercising the same `auth/carts/{id}` no-token/invalid-token pattern) — hit repeated 403s from the Postman API itself on the update call during this pass, even after spacing out retries. Left as ES5 pending a retry; this is a tooling/API reliability issue, not a decision or a skipped step.

**Decision:** Accepted. Section 5 of the audit closed (with the 3-request carve-out noted above). Remaining sections (maintainability for future API changes, underused Postman/REST testing capability) queued next.

---

## 16-07-2026 (cont'd) : Collection Audit — Section 7, Maintainability Closed

**Tool:** Claude

Seventh audit section (skipped 6, "Naming quality," for now — that one's a clean bill of health with nothing to fix, not an action list). Four items flagged for future-change resilience:

**Items 1 & 2 (hardcoded 194/208 totals; `expectedUserTotal` looking unused) were already stale by the time of this review** — both were fully resolved earlier as part of the Section 4 baseline-capture fix (`expectedProductTotal`/`expectedCartTotal`/`expectedUserTotal`, captured live via the `00 - Setup` folder, `expectedUserTotal` wired into the Users boundary test as its regression sensor). No new work needed; closing them here formally rather than leaving them dangling as open items in a section that reads "unclosed."

**Items 3 & 4 — hardcoded category list, literal IDs in request bodies:** initially added a README bullet for each (matching the existing pattern from the Section 4 baseline-capture bullet). Renuka pushed back on this directly — asked why audit closures were being routed into README by default rather than by judgment. Correct challenge: README is the recruiter-skim layer and should hold genuine differentiators (dynamic baseline capture, IDOR/BOLA discovery, AI-workflow critique); these two items are minor, interview-answerable trade-offs already covered by inline code comments and this log, and adding them diluted the signal-to-noise of the "Key Testing Decisions" list rather than strengthening it. **Reverted** — both bullets removed from README, reasoning kept here and in the audit doc only. Also reassessed the Section 5 ES6-standardization README bullet under the same lens and judged it a defensible keep (it forecloses a real negative signal — visibly inconsistent code style — rather than adding a marginal one), so it was left in place.

**Decision:** Accepted, with the README self-correction above. Section 7 closed. Section 6 (naming) intentionally left open as a "reviewed, nothing to do" note rather than force-closed. Section 9 (new E2E cross-module scenario finding, added same day) still queued, pending go-ahead.

---

## 17-07-2026 (cont'd) : Collection Audit — Section 8, Underused Postman Capability Closed

**Tool:** Claude

Eighth audit section, four items, each handled on its own merits rather than as a uniform batch.

**JSON Schema validation:** added to "Get a single product: ID -1" and "Get Single User - Valid (Lower Boundary id=1)", replacing their manual field-by-field/type-by-type checks. Rather than inventing schemas from assumption, both were built strictly from field lists already verified live elsewhere in the collection — the product schema reuses the exact required-field list from "Get all products"'s own per-product assertions; the user schema reuses the field list already present in the request's own prior manual test (itself built from a live-verified response). Both requests keep a separate, non-schema assertion for "response id equals the id requested in the URL" — schema validation checks shape and type, not cross-field business logic, so the two approaches sit side by side rather than one fully replacing the other.

**`pm.sendRequest` chaining — verified before writing anything, not assumed stale.** Checked whether the Section 4 baseline-capture fix (the `00 - Setup` folder) had incidentally added a third usage, since its own recommendation text implied that fix was linked to this one. It hadn't — Setup uses plain sequential `GET` requests, not `pm.sendRequest`. Corrected the audit doc's own "ties directly into the maintainability fix above" framing, which had gone stale/inaccurate once Section 4 landed via a different mechanism. Left as a named, deliberate scope boundary rather than expanded — more chaining is a reasonable next-iteration item, not something this pass needed to force in.

**Dynamic variables — scoped by actually checking the fixtures, not applied uniformly.** Reviewed all three data-driven fixture pairs (Add User, Update User, Add Cart). Only Add User's happy-path fixture (row H02) has collision-sensitive fields at all (`username`, `email`); Update User and Add Cart carry no user-identifying data, so nothing was forced in there for the sake of appearing thorough. Implemented via the pre-request script using `pm.variables.replaceIn("{{$randomInt}}")`, not by writing a literal `"{{$randomInt}}"` string into the static JSON fixture — that wouldn't have worked reliably, since Postman's template engine resolves `{{payload}}` once against the fixture's already-stringified value and doesn't recursively re-parse a nested `{{...}}` token inside it. The test script's own echo-check (`Every submitted field is echoed back correctly`) was updated in the same pass to compare against the actual post-mutation payload (stored separately as `sentPayload`) rather than the untouched fixture row, so the existing assertion keeps working correctly instead of silently failing against a value that was never sent. Noted honestly: DummyJSON's Add User is simulate-only, so this demonstrates the capability rather than fixing a real collision risk today.

**Visualizer — reconsidered after initial "skip" call.** Originally documented as a deliberate non-use. Renuka asked directly whether it would actually fit this project rather than accepting the blanket "optional, skip it" framing — fair challenge, since list-returning endpoints (products, carts, users) are exactly the classic Visualizer use case. Added to "Get all products": a Handlebars template renders the product list as an HTML table (ID, title, category, price, stock) in Postman's "Visualize" tab. One implementation detail worth remembering: the template is itself written as a JS template literal (backticks), so a literal `${{this.price}}` would be misread by JS as its own interpolation syntax before Handlebars ever sees it — escaped as `\${{this.price}}` to keep it literal text for Handlebars to process. Purely presentational; adds no assertions and doesn't touch any existing test.

**Mock Servers/Monitors — reviewed, confirmed as deliberate non-use, not built.** Out of scope for a mock-API portfolio project by design — DummyJSON already serves as the mock layer, and monitoring/scheduled runs aren't a claim this project needs to make. Documented as an explicit "not used, here's why" statement rather than a silent gap.

**Decision:** Accepted. Section 8 closed. Only Section 9 (E2E cross-module scenario) remains open, still pending go-ahead.

---

## 17-07-2026 (cont'd) : Collection Audit — Section 9, End-to-End Scenario Built

**Tool:** Claude

Ninth item (a new finding, not part of the original 15-07-2026 audit — surfaced when Renuka asked whether a real business flow, Auth → Users → Products → Cart, should be reflected in the collection at all, since everything built so far tests modules in isolation).

**Built:** new `E2E Scenarios` folder, 4 sequenced requests, each capturing what the next one needs:

1. Login — captures `accessToken`/`userId`. Deliberately duplicated (not referenced) from the Auth folder's own login, so this folder can be run standalone via Collection Runner without depending on Auth having run first in the same pass.
2. Get user profile via `GET /auth/users/{{userId}}` with the bearer token — asserts the returned id matches the logged-in user, captures the delivery address. Noted honestly: DummyJSON's cart endpoint doesn't use the address at all, so this step demonstrates the capture the business flow calls for, not a real downstream dependency.
3. Search products by category (`smartphones` — reused a slug already verified live elsewhere in the collection rather than guessing a new one) — picks a product id from the response dynamically.
4. Add that product to that user's cart — assertions check the cart response actually belongs to the exact user from step 1 and contains the exact product from step 3, not just a 201.

**Decision:** Accepted. Section 9 closed — this closes out the full audit (all 9 sections now resolved: 1 was informational, 2/3/4/5/7/8/9 fixed, 6 reviewed with no action needed).

---

## 17-07-2026 (cont'd) : Full-Collection Run Investigation — DEF-005 Tests Flip to 401 Under Volume

**Tool:** Claude + Renuka (live verification at each step)

Running the full collection (162+ requests) surfaced 23 failing tests. First one investigated: the four `KNOWN DEFECT DEF-005` pinned tests (Get Single User, Search Users, Filter Users, Get User Carts — all "No Auth Token") failed with 401 instead of their pinned 200-with-leaked-PII expectation.

**Hypotheses tested in order, each ruled out with a live check rather than assumed:**
1. *DummyJSON patched the vulnerability* — ruled out: running each of the 4 individually still reproduces 200 with leaked data. Bug is still real.
2. *Config/scripting bug on these 4 requests* — ruled out: pulled the live request definitions for all four; each has `"auth": {"type": "noauth"}` explicitly set with empty headers. Identical and correct in every run.
3. *Cookie leakage from the Login step* — ruled out: ran Login + these 4 requests as an isolated 5-request sequence; still 200 (bug reproduces). A prior Login in the same run doesn't cause the flip.
4. *Request speed / rate-limiting by pace* — ruled out: added a 300ms and then 1000ms delay between requests on a full run; 401 persisted at both, and the delay broke unrelated response-time assertions (collateral damage, not a fix).
5. *Token-refresh script spamming `/auth/refresh`* — ruled out: checked the Postman Console during a full run; "Token refreshed successfully" logged exactly once, working as designed.

**What the evidence does show:** the flip is tied to total request *volume* within a run, not speed, not what ran immediately before, and not a collection bug. 5 requests → real bug (200). ~70 requests (Users folder alone) → 401. 160+ (full collection) → 401. This points to a network/infrastructure-layer protection (Cloudflare or DummyJSON's own hosting) that appears to throttle unauthenticated hits to these specific PII-exposing routes once a request-count threshold is crossed in a session — a mechanism outside this project's visibility or control, and not fixable from the test side.

**Decision:** Rather than keep chasing DummyJSON/Cloudflare's internal defensive logic, the practical fix is execution strategy, not test logic: run the DEF-005 security-pin checks as a separate, smaller, dedicated pass, distinct from the full regression run. Both stay meaningful; both stay green. This is a legitimate test-architecture decision (splitting execution profiles), not a workaround — and it directly informs the GitHub Actions setup planned next (separate CI jobs per execution profile).

---

## 18-07-2026 : Full-Collection Run Investigation, Continued — Stale 404 Copy + New DEF-005 Write-Route Instance

**Tool:** Claude + Renuka (live verification at each step)

Continuing through the 23 full-collection-run failures, in order.

**Point 3 — stale exact-text assertions on DummyJSON's 404 page.** Six `pm.test()` assertions across five requests (Auth > Login "Wrong HTTP method", Auth > Refresh "Wrong method", Products > "GET all products - invalid URL", "GET all products - invalid method", "Get a single product: Invalid URL", and Users > "Get All Users - Invalid Method") checked for an exact substring like `"cannot get /productss"` or `` `Cannot ${method}` ``. Renuka pasted the live response bodies for two of them — DummyJSON has fully redesigned its 404 page: a branded page ("This page floated away," "Error 404") that doesn't echo the requested path or method anywhere, replacing what was presumably a plain Express default message. This is a genuine third-party site change, not a test-writing mistake. **Fixed:** rewrote all six to check structurally instead of on exact copy — status code, `Content-Type: text/html` (the actual defect this test family documents — HTML instead of a consistent JSON error contract), and a generic `<!doctype html>` marker. This matches the pattern "Get All Users - Invalid URL" already used correctly, so the fix brings the rest of the collection in line with an existing good example rather than inventing a new approach. One process note: an early edit attempt on "GET all products - invalid method" mistakenly labeled the new script as a `prerequest` event instead of `test` — caught and corrected in the same turn before moving on.

**Point 4 — "Add a cart - No Token" TypeError, which turned into a new security finding.** The test crashed with `Cannot read properties of undefined (reading 'toLowerCase')`. Root cause: the test asserted 401 based on a comment ("`/auth/carts/add` enforces authentication — no token returns 401") that reads as an assumption, never actually verified live. Renuka ran it and got 201 with a fully real cart created (id 209, real product, real total) — meaning the endpoint doesn't enforce auth at all, the same DEF-005 root cause (auth middleware has no else-branch for a missing token) confirmed on a **new route**: `POST /auth/carts/add`. The crash was just a symptom — `jsonData.message` doesn't exist on a successful cart-creation response.

**Significance:** this is a higher-impact instance than the earlier write-route finding (Add User). Add User is simulate-only, so nothing real gets created. Add Cart is not — this genuinely creates a persisted-looking cart record with zero authentication. **Fixed:** rewrote the test to pin the current (insecure) 201-with-real-cart-data behavior under `KNOWN DEFECT DEF-005`, same CI-hygiene pattern as every other confirmed route. README's DEF-005 entry updated from 6/6 to 7/7 confirmed routes, with this write-route's higher impact called out explicitly rather than folded in as identical to Add User's.

**Decision:** Both fixed and verified against live behavior, not assumed. Two more points remain from the original 23-failure list to work through.

---

## 19-07-2026 : Full-Collection Run Investigation, Continued — Remaining 15 Failures Triaged, E2E State-Artifact Confirmed

**Tool:** Claude + Renuka (live verification at each step)

A second full-collection run still showed 15 failing tests after the previous session's fixes. Triaged into 4 buckets before touching anything, since not all 15 shared the same cause.

**Bucket 1 — DEF-005 pin recurrence (10 fails, 5 requests).** Get Single User, Search Users, Filter Users, Get User Carts, Add User (all "No Auth Token") flipped to 401 against their pinned 200-with-leaked-PII expectation. Identical signature to the 17-07-2026 investigation above — same conclusion applies, no new work needed.

**Bucket 2 — Delete a product, response time (1 fail).** 3037ms against a 2000ms threshold, full-run only. Same execution-load category as Bucket 1.

**Bucket 3 — E2E Scenarios, steps 2 & 4 (2 fails).** `pm.expect(responseJson.id).to.equal(Number(pm.environment.get("userId")))` failed with "expected 1 to equal 30." Initial hypothesis — hardcoded user id in the request — was checked against the live request definitions and was wrong: both step 2's URL (`{{baseURL}}/auth/users/{{userId}}`) and step 4's body (`"userId": {{userId}}`) correctly use the dynamic value captured in step 1. Traced the full variable chain instead of guessing further: `testUsername` is a fixed, never-reassigned value referenced in only 2 places in the whole collection; `userId` is only ever written by login test scripts; nothing executes between step 1's test script and step 2's request except the collection-level auto-refresh pre-request script, which never touches `userId`. No conflict found on our side.

Verified the API endpoint itself next, since the sandbox has no network access to dummyjson.com (`curl` blocked — `403 blocked-by-allowlist`, confirmed by testing, not assumed) and this MCP's Postman tools can't send live requests either. Renuka ran two manual calls with the same token: `GET /auth/users/1` → returned id 1 (Emily); `GET /auth/users/30` (or `{{userId}}`) → returned id 30 (Addison). Endpoint correctly honors the URL id in both directions — not broken.

Final confirming check: ran the **E2E Scenarios** folder alone (isolated Collection Runner run, nothing else). Passed clean. This rules out both a broken test and a broken endpoint — the failure only exists as an artifact of full-collection execution, same category as Bucket 1/2.

**Decision:** Buckets 1, 2, and 3 all resolved to the same root cause — full-collection execution load, not a defect in the API or a mistake in test design. No code changes made to the E2E folder; the earlier hardcoded-id hypothesis was explicitly wrong and corrected rather than quietly dropped. This reinforces the existing decision (17-07-2026 entry) to split CI execution profiles rather than force a "fix" onto tests that are already correct. Bucket 4 (Refresh auth session — Missing refreshToken, 2 fails, still unverified as real-defect-vs-assertion-error) remains open for the next pass.

---

## 19-07-2026 (cont'd) : Refresh Auth Session "Missing refreshToken" — Cookie Jar Test-Isolation Gap, Not a Defect

**Tool:** Claude + Renuka (live verification at each step)

Last of the 15 full-collection failures. "Missing refreshToken" (empty-body `/auth/refresh`) was pinned to expect 401 back on 15-07-2026, verified live at the time. Now returning 200 with a full valid token pair for the correct test user (addisonw, id 30) — reproducible even when run alone, ruling out the full-run-execution-artifact explanation used for the other 14.

**Investigated rather than assumed a genuine API regression.** Checked the request's Auth tab first: "Inherit auth from parent" resolves to "No Auth" — confirmed no Authorization header is being sent, so the request genuinely has zero header-based credentials. That narrowed it to one remaining channel: cookies. Postman's cookie manager showed 2 stored cookies for `dummyjson.com` — `accessToken` and `refreshToken` — left over from an earlier login in the same Postman session. DummyJSON's `/auth/login` apparently sets these as cookies in addition to returning them in the response body, and Postman was silently reattaching them on every later request to the domain, including this one.

**Confirmed:** deleted both cookies and resent the identical request — `401`, `{"message": "Refresh token required"}`, exactly the original expectation. The test, the assertion, and the API were all correct the whole time; the request was never actually credential-less, it was carrying credentials through a channel outside the request definition itself.

**Fix:** rather than rely on manually clearing cookies before every run (fragile, not automation-safe, easy to forget), added `pm.cookies.jar().clear(pm.environment.get('baseURL'), ...)` to the top of the collection-level pre-request script — the same script that already handles JWT auto-refresh — so every request in the collection is guaranteed cookie-free and self-contained regardless of what ran before it, matching the same "don't depend on run order" principle already used elsewhere (`00 - Setup` folder, E2E's duplicated login). Could not apply this via the Postman API: `putCollection` requires resubmitting the entire 160+ request collection with every ID preserved to change anything at the collection root, which was judged too risky to do blind through this tool for a one-script edit — Renuka added it manually in the Postman UI instead.

**Decision (revised same day — see follow-up below):** originally closed as a test-isolation fix via a collection-level guard. That fix did not hold up under further testing and was reverted; see the follow-up entry immediately below for the corrected conclusion.

---

## 19-07-2026 (cont'd) : Cookie-Clear Fix Reverted — Postman Script Permission Wall, Not a Code Bug

**Tool:** Claude + Renuka (live verification at each step)

The `pm.cookies.jar().clear()` collection-level fix (previous entry) did not work in practice, across several attempted corrections:

1. First full-collection re-run after adding it: "Missing refreshToken" still returned 200, unchanged.
2. Checked whether the script had actually saved correctly — it had, but appended after the JWT-refresh block instead of before. Moved it to the top and added a success-path `console.log` for visibility, since the original version only logged the error case and gave no proof either way of whether it was running.
3. Re-ran with the Postman Console open. Found the real cause: every request logged `"Cookie clear failed:" {type: "Error", name: "Error", message: "CookieStore: programmatic access to \"dummyjson.com\" is denied"}` — Postman blocks script-based cookie access to a domain by default, a documented security restriction, independent of the collection/request logic entirely.
4. Renuka whitelisted the domain via Manage Cookies > "Add domain" (`dummyjson.com`), per the error message's own instruction. Re-ran — still failing, same result.

**Decision:** Stopped chasing Postman's script-level cookie permission model rather than continuing to iterate blindly against an increasingly narrow, tool-specific wall — six rounds in without a working fix is a signal to reassess the approach, not push harder on the same one. Reverted the `pm.cookies.jar().clear()` addition entirely (Renuka removed it manually from the collection-level pre-request script) since a non-functional script was actively adding console noise (errors logged on every single request) without providing any benefit.

**Key realization that made reverting the right call, not a retreat:** Newman — the CLI tool this collection is about to run through in GitHub Actions — starts every invocation with a completely fresh, non-persistent cookie jar. The entire failure mode (a login's cookie surviving into a later, supposedly-credential-less request) is specific to the interactive Postman desktop app carrying session state across manual runs in the same window. It structurally cannot happen in CI. Chasing a Postman-desktop-only permission quirk with a Postman-desktop-only scripting workaround wasn't solving the problem for where this collection actually needs to run correctly.

**Final documented state:**
- **Root cause (unchanged from the original diagnosis):** Postman's cookie jar silently carries `accessToken`/`refreshToken` cookies set by DummyJSON's `/auth/login` across later requests in the same interactive session.
- **Interactive/manual mitigation:** clear cookies for `dummyjson.com` via Manage Cookies before re-running "Missing refreshToken" by hand, same as the original manual verification earlier in this log.
- **CI mitigation:** none needed — Newman's per-run cookie jar isn't persistent, so this cannot reproduce there.
- Not filed as a defect (API behavior was never at fault); not carrying an active script-level workaround in the collection (proved unreliable and unnecessary given the CI target).

This closes out all 15 failures from the second full-collection run: 13 were full-run execution artifacts (already explained, no code changes needed), and this one is a documented, understood test-isolation limitation of the interactive tool, correctly scoped to not matter for CI.

---

---

## 23-07-2026 : Documentation Sync Audit — README Updated, No Collection Drift

**Tool:** Claude

Full audit of the README and this log against the live Postman collection, ahead of starting GitHub Actions/CI work.

**Verified:** module request counts (Auth 17, Products 23, Carts 63, Users 62) match the live collection exactly. Previously-flagged manual cleanups (redundant Add User/Update User requests, Get User Posts/Todos removal) confirmed complete. `00 - Setup` and `E2E Scenarios` present with counts matching the 16-07 and 17-07 entries. No undocumented collection changes since the last log entry.

**Gap found and fixed (README only — the collection itself was correct):** `E2E Scenarios`, closed out in the 17-07-2026 entry, was missing from "What This Project Demonstrates," "Features Validated," and the project structure notes. Added. Also corrected the data-driven testing bullet (missing Update User) and the fixture-file description in the project structure section.

**Decision:** Accepted. Docs match the live collection.

---

## 23-07-2026 (cont'd) : Committed Collection and Environment Exports Rebuilt From Live Postman

**Tool:** Claude

Extended the audit to every file in the repo, not just docs against the API — a stronger check, since the previous pass only verified prose, not the actual files a cloned repo would contain.

**Found:** the committed collection export (`postman/collections/Project 2 - E-commerce API- DummyJSON.postman_collection.json`) was dated 14-05-2026 — 52 requests across 4 folders. The live collection has 172 requests across 6 folders. Everything from Update Cart onward existed only in Postman and in this log, never in the file itself.

**Fixed:** rebuilt the committed collection file from the live Postman collection. Verified after rebuild: 6 top-level folders in the correct order (`00 - Setup`, `Auth`, `Products`, `Carts`, `Users`, `E2E Scenarios`), 172 requests, counts matching folder-by-folder (3/17/23/63/62/4). Given the file's size (~13,500 lines), used a script-based extraction rather than manual editing to remove transcription risk.

**Also found and fixed:** the committed environment export (dated 13-05-2026) was missing 8 variables added since, including `testUsername` — its absence would silently break Login for anyone importing the file fresh, since the login request reads `{{testUsername}}` directly. Rebuilt from the live environment.

**Also found and fixed:** "Get User Carts - Valid Token," flagged as a design gap in Prompt 07 (15-07-2026) and never corrected, was still pointed at the same hardcoded URL as the IDOR test — silently re-confirming DEF-018 instead of testing genuine self-access. Repointed to `{{userId}}` and re-synced the committed file. No script change needed; the existing assertion already derived its expected value from the URL.

**Not touched:** fixture JSON files (validated, no drift), `docs/prompts/*` (historical snapshots, correctly left as-is), defect list (re-verified, no drift).

**Also found and resolved:** a stray 0-byte `newman` file and a `postman - Shortcut.lnk` file at the repo root — neither part of the project, both deleted manually (file permissions blocked automated removal).

**Decision:** Accepted. Committed artifacts now match live Postman exactly. Repo root is clean.

---

_Log maintained throughout project. Last updated: 23-07-2026._
