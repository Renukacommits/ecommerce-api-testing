# Prompt 04 — Postbot: Update Cart Request Completion (URLs, Bodies, Methods)

**Tool:** Postbot (Postman)  
**Date:** 14-05-2026  
**Stage:** Implementation — Update Cart module

---

## Context

After scaffolding the 29 Update Cart requests (see Prompt 03), Postbot was used again to complete each request — setting the correct URL, HTTP method, and request body — using the DummyJSON API documentation as input. This avoided repetitive manual data entry across 29 requests.

## Prompt Used (fed to Postbot)

> "I have an Update Cart folder in my Postman collection with requests already named and organised into subfolders. I need you to complete each request by adding the correct URL, HTTP method, and request body. I will provide you the API documentation. Ask me for any details you need before proceeding."

DummyJSON API documentation for Update Cart was then provided to Postbot on request.

## AI Output

- Set `{{baseURL}}/carts/{id}` as the URL for all requests, with the correct cart ID per scenario
- Set HTTP method to PUT or PATCH as appropriate per scenario
- Added request bodies with correct `merge` value and `products` array per scenario

## Issues Found on Review

Not tracked at the per-request level separately from the scaffolding pass (see Prompt 03) — the URL/method/body corrections that came out of this step were folded into the same review as the naming/placement corrections, rather than logged as a distinct set of issues. This is a documentation gap, not a claim that the Postbot output needed zero corrections — flagged here rather than backfilled with invented specifics.

## Evaluation

**Accepted, gap acknowledged.** The Update Cart module is complete and all 29 requests have independently written and validated assertions (confirmed in `docs/ai-testing-log.md`), so the work this prompt describes was done and reviewed — the review notes for this specific step just weren't preserved in enough detail to reconstruct honestly after the fact.

## Outcome

URLs, methods, and bodies completed by Postbot for all 29 Update Cart requests, reviewed alongside the scaffolding corrections in Prompt 03. All request bodies and assertions were written and validated independently afterward — see the Update Cart entries in `docs/ai-testing-log.md`.
