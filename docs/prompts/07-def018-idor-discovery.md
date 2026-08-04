# Prompt 07 — DEF-018: IDOR Discovery on GET /auth/users/{id}/carts

**Tool:** Claude  
**Date:** 15-07-2026  
**Stage:** Implementation — Get User Carts

---

## Context

The Get User Carts folder scaffold (built in an earlier session) already included a dedicated request — "Get User Carts - Another User's Carts - Security (IDOR)" — with a description flagging exactly what to check: whether a valid token for one user could read a *different* user's cart contents. This wasn't an AI-generated test idea; it was pre-planned as a deliberate security check, mirroring the same question already asked (and answered "yes, vulnerable") on Delete Cart as DEF-004.

## Question Posed

Claude could not execute the request itself (Postman MCP is management-API only, no execution capability), so the actual check required Renuka:

> "Get User Carts - Another User's Carts (IDOR) — this is the one I'd flag as most interesting: does a valid token for one user let them read a different user's cart contents? If yes, that's DEF-004's sibling on a GET route instead of DELETE."

Renuka's report: `200`.

## AI Output

Given the confirmed 200 with cart data returned, Claude:

- Wrote a "sanity check" assertion first — confirming programmatically that the requested `userId` in the URL does *not* match the logged-in token's own `userId` (via `pm.environment.get('userId')`), so the test provably exercises a genuine cross-user scenario rather than accidentally testing legitimate self-access.
- Wrote the leak assertion to confirm the returned cart data genuinely belongs to the *other* user (`cart.userId` matches the URL's `{id}`, not the token owner) — proving actual data exposure, not just a non-error status code.
- Classified it as a new defect (DEF-018) rather than folding it into DEF-004, since it's a different endpoint and a different HTTP verb, even though the underlying flaw (missing ownership check on an authenticated request) is the same class of bug.
- Pinned the assertions under `KNOWN DEFECT DEF-018`, using the same CI-hygiene reasoning already established for DEF-005 (see Prompt 05).

## Evaluation

**Accepted.** Also flagged a related design issue during this same review, unprompted: the "Valid Token" request in the same folder hits the *identical* URL as the IDOR test (`/auth/users/6/carts`), which means it isn't actually testing "a user can see their own data" — it's silently re-confirming DEF-018. Flagged to Renuka rather than changed unilaterally, since repointing the URL to `{{userId}}` is a test-design decision, not a bug fix.

## Outcome

DEF-018 filed — a second confirmed IDOR vulnerability (Broken Object Level Authorization, OWASP API1:2023), distinct from but structurally identical to DEF-004. Strengthens the project's security-testing narrative: the same missing-ownership-check pattern found on two different verbs (DELETE and GET) is a stronger portfolio signal than either finding alone.
