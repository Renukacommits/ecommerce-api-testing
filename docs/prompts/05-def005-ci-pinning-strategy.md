# Prompt 05 — DEF-005: Pin Known-Bad Behaviour or Leave the Test Red?

**Tool:** Claude  
**Date:** 15-07-2026  
**Stage:** Users module — Get Single User / Search Users / Filter Users / Get User Carts

---

## Context

DEF-005 (Broken Object Level Authorization on `GET /auth/users/{id}`, discovered 14-07-2026) had its "No Auth Token" test written to assert the *secure* expectation (401, no leaked data) — meaning the test was expected to fail forever as a way of documenting the vulnerability. This is a defensible testing philosophy (treat the spec as source of truth), but it collides with an upcoming milestone: wiring the collection into GitHub Actions.

## Question Posed

> "Before that, what about the Get Single User - No Auth Token request — it fails 3 assertions due to the recent security issues that you can see in memory, should we keep the failed assertions as is - or what is the ideal way?"

## AI Output

Claude recommended against leaving the assertions permanently red, reasoning:

- A CI gate only works if red reliably means "something broke." A test that is *expected* to stay red forever destroys that signal — a reviewer (or a future regression) can no longer tell a known issue from a new one at a glance.
- Known defects belong in a defect tracker / log, not as permanently-failing test assertions — that's the whole reason bug trackers and test suites are kept separate.
- The professional pattern: **pin the current (insecure) behaviour** under test names that scream the defect ID (`KNOWN DEFECT DEF-005`), with a full comment block explaining the vulnerability, root cause, and OWASP classification. If the bug is ever fixed, the pinned assertion starts failing — and *that* failure is the correct trigger to flip it back to the secure expectation and close the defect.
- Also diagnosed the root cause more precisely from the existing data: since "Invalid Token" on the same endpoint correctly returns 401, the auth middleware's validator works fine — the gap is specifically a missing `else` branch for requests that send no Authorization header at all.

## Evaluation

**Accepted.** This became the standing strategy applied consistently for the rest of the session — DEF-005 (4 endpoints) and the new DEF-018 (Get User Carts IDOR) were both pinned this way, not left red.

## Outcome

- Rewrote "Get Single User - No Auth Token" to pin the 200-with-PII behaviour under `KNOWN DEFECT DEF-005`.
- Same pattern applied to three more confirmed instances of the same bug as they were found (Search Users, Filter Users, Get User Carts), plus the new DEF-018 (Get User Carts IDOR).
- Every pinned test carries an in-line comment explaining the defect, so the finding stays fully visible without needing the suite to be red to prove it exists.
