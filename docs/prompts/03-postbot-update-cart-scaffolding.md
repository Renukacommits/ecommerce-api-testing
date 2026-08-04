# Prompt 03 — Postbot: Update Cart Request Scaffolding

**Tool:** Postbot (Postman)  
**Date:** 14-05-2026  
**Stage:** Implementation — Update Cart module

---

## Context

Update Cart required 29 requests across 5 subfolders. Creating each request manually — naming it, placing it in the correct folder, setting the URL — was repetitive and low-value work. Postbot was used to handle the scaffolding.

## Prompt Used (fed to Postbot)

> "Create the following folder structure and requests in my Postman collection under the Update Cart folder. Do not add test scripts — only create the requests with the correct names, HTTP methods, and folder placement.
>
> merge - true/
> - PUT - Add multiple new products
> - PUT - All products already exist in cart
> - PUT - Empty products array
> - PUT - Boundary cartId = 1
> - PUT - Boundary cartId = 20
>
> merge - false/
> - PUT - merge field absent
> - PUT - Empty products array
>
> Negative/ → Invalid cartId/
> - PUT - cartId = -1
> - PUT - cartId = 21
> - PUT - cartId = abc
> - PUT - cartId = 1.5
>
> Negative/ → Invalid merge value/
> - PUT - merge = 'true' (string)
> - PUT - merge = 1 (integer)
> - PUT - merge = null
>
> Negative/ → Invalid body/
> - PUT - No body
> - PUT - Empty body
> - PUT - products field missing
> - PUT - Product missing id
> - PUT - Product missing quantity
> - PUT - Product quantity = 0
> - PUT - Product quantity = -1
> - PUT - Non-existing productId"

## AI Output

Postbot created 18 requests across the 5 subfolders as specified.

## Issues Found on Review

1. **Wrong HTTP methods:** Several requests in merge-true and merge-false were created with GET instead of PUT. Corrected manually.
2. **Misplaced requests:** Two pre-existing requests (cartId = 0, Non-existing cartId = 9999) appeared inside Invalid body/ instead of Invalid cartId/. Moved manually.

## Evaluation

**Accepted with corrections.** Postbot handled the repetitive scaffolding work correctly for the majority of requests. The method and placement errors were caught on visual review before any work was done on bodies or assertions. 

**Decision:** Postbot is effective for mechanical setup (naming, placement, volume creation) but requires manual review of every request before proceeding. It is not trusted to set HTTP methods correctly without verification.

## Outcome

18 requests scaffolded by Postbot. 2 method corrections and 2 placement corrections made manually. All request bodies and assertions written independently.
