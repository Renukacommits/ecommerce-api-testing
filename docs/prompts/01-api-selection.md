# Prompt 01 — API Selection

**Tool:** Claude  
**Date:** April 2026  
**Stage:** Project Setup

---

## Context

Initial project plan specified FakeStoreAPI as the target API. On first exploration, the API returned an SSL certificate error in the browser.

## Prompt Used

> "I am building a portfolio API testing project using Postman. I was planning to use FakeStoreAPI but it's showing an SSL certificate error. I need a reliable public API that supports full CRUD operations across products, users, carts, and authentication. What would you recommend and why?"

## AI Output

Claude recommended DummyJSON as an alternative — reasons given:
- Stable and actively maintained
- Richer endpoint coverage (products, carts, users, auth, todos)
- Full CRUD support including PUT, PATCH, DELETE
- Returns realistic, structured JSON responses
- No authentication required for most endpoints (reduces friction for portfolio testing)

## Evaluation

**Accepted.** The recommendation was validated independently by exploring the DummyJSON documentation before committing. SSL reliability risk on FakeStoreAPI was a genuine concern for a portfolio project — an unstable API would undermine every test built on top of it.

## Outcome

Switched to DummyJSON. All subsequent test modules built on this API.
