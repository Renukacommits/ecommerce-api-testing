# Prompt 06 — Users Module: Verify-Live-First Assertion Workflow

**Tool:** Claude  
**Date:** 15-07-2026  
**Stage:** Implementation — Search Users, Filter Users, Get User Carts (used repeatedly across all three folders)

---

## Context

Continuation of the Users module ("resume Users module assertions from Search Users folder"). Rather than a single one-off prompt, this describes a consistent working pattern applied across three folders in one session: verify actual API behaviour first, write assertions against what was observed (not assumed), and separate what could be checked independently from what needed a live Postman check.

## Prompt Pattern Used

> "Resume Users module assertions from Search Users folder — see memory entries for context." (repeated implicitly for Filter Users and Get User Carts as each folder was reached)

## AI Output (What Claude Did Each Time)

1. Fetched the folder's current scaffold from the Postman collection (via the Postman MCP connector) to confirm what existed and what didn't.
2. Verified live API behaviour directly where possible (public endpoints, no auth required) using a web-fetch tool — e.g. confirmed `Search Users`' `q` parameter matches firstName/lastName/username but not maidenName or company name, by testing `q=Smith` vs `q=Dooley` live rather than guessing at undocumented search scope.
3. Wrote assertions matching observed behaviour, using dynamic value extraction (e.g. resolving `key=hair.color` as a dot-path at runtime in Filter Users, instead of hardcoding `hair.color`).
4. For anything requiring an auth header — which the available fetch tool couldn't send — asked Renuka to run the specific request in Postman and report status/body, rather than assuming the result from a similar-looking endpoint.

## Issues Found on Review

- Initially wrote Search Users' "No Auth Token" test expecting 401 by pattern-matching against the already-secured Get All Users list endpoint. Self-flagged this as unverified before calling the folder complete, since DEF-005 had already proven auth enforcement is inconsistent across endpoints on this API — pattern-matching across routes isn't a safe substitute for checking. This turned out to matter: the pattern didn't hold, and Search Users had the same bug.
- Discovered mid-session that the fetch tool returns nothing for *any* non-200 response, not specifically for `/auth/*` paths as first assumed — corrected in the log rather than left as a standing (slightly wrong) explanation.

## Evaluation

**Accepted, with the self-correction folded back into the process.** The discipline of treating a confirmed pattern (DEF-005 on 2 endpoints, then 3) as a hypothesis to check rather than a fact to assume paid off directly — it caught DEF-005's spread to Filter Users and Get User Carts, and separately surfaced DEF-018 (a genuinely new IDOR finding, not just another DEF-005 instance) on the Get User Carts IDOR-specific test.

## Outcome

20 requests assertion-complete across the three folders (6 + 7 + 7). Two new/extended defect findings surfaced as a direct result of verifying rather than assuming: DEF-005 extended from 1 to 4 confirmed endpoints, and DEF-018 filed as a new finding. Full defect detail in `docs/ai-testing-log.md` and the README.
