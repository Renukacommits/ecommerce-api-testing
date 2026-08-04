# DummyJSON E-commerce API Testing Project

![Postman](https://img.shields.io/badge/Postman-FF6C37?style=for-the-badge&logo=postman&logoColor=white)
![Newman](https://img.shields.io/badge/Newman-66B135?style=for-the-badge&logo=postman&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)
![API Testing](https://img.shields.io/badge/API%20Testing-Automated-blue?style=for-the-badge)

## Project Overview

This project tests the API of a simulated e-commerce platform, covering the full user journey — authentication, products, carts, and users. It exercises every endpoint in each module to confirm expected behaviour, and documents every place the API deviates from that expectation.

## What This Project Demonstrates

Full CRUD coverage across Auth, Products, Carts and Users; data-driven testing using JSON datasets; security testing that surfaced real authorization vulnerabilities across four confirmed defects (IDOR and Broken Object Level Authorization, OWASP API1:2023); and an AI-assisted testing workflow using Postbot and Claude — with critical evaluation of AI output documented throughout, not accepted at face value.

---

## Test Coverage Summary

### API Endpoints Tested

- **Base URL:** `https://dummyjson.com/`
- **Modules:** `/auth`, `/products`, `/carts`, `/users`

Counts below are pulled directly from the live Postman collection, not estimated.

| Folder            | Requests | Purpose                                                                        |
| ----------------- | -------- | ------------------------------------------------------------------------------- |
| **Auth**          | 17       | ✅ CRUD + assertions complete                                                    |
| **Products**      | 23       | ✅ CRUD + assertions complete                                                    |
| **Carts**         | 63       | ✅ CRUD + assertions complete                                                    |
| **Users**         | 62       | ✅ CRUD + assertions complete                                                    |
| **00 - Setup**    | 3        | Captures dataset totals into environment variables before any module runs      |
| **E2E Scenarios** | 4        | Chained cross-module business flow (login → browse → cart, dynamic end-to-end) |
| **Total**         | **172**  | —                                                                              |

The `00 - Setup` folder captures `expectedProductTotal`, `expectedCartTotal`, and `expectedUserTotal` into environment variables so downstream boundary/total-count assertions read a live-captured value instead of a hardcoded magic number. `E2E Scenarios` is a single chained flow — log in and capture the token → fetch the logged-in user's profile → search products by category and capture a product ID → add that exact product to that exact user's cart, with assertions confirming the final cart genuinely belongs to that user and contains that product. Every step captures what the next one needs dynamically; no hardcoded IDs anywhere in the chain. Demonstrates the system holds together across modules, not just at each endpoint tested in isolation.

### Defects Found

21 confirmed defects, including 4 authorization vulnerabilities (IDOR / Broken Object Level Authorization, OWASP API1:2023). IDs are stable — several are referenced by name inside the Postman test scripts themselves (e.g. `KNOWN DEFECT DEF-005`), so numbering is never reused or reshuffled once assigned.

Full write-ups (root cause, verification steps, OWASP classification) are in **[`docs/defects.md`](docs/defects.md)**. Summary:

| ID | Summary | Severity |
| --- | --- | --- |
| DEF-001 | Update Cart: quantity = 0 accepted silently | Medium |
| DEF-002 | Update Cart: invalid input accepted silently | High |
| DEF-003 | Update Cart: integer value accepted for merge field | Medium |
| DEF-004 | Delete Cart: IDOR — regular user can delete another user's cart | Critical |
| DEF-005 | BOLA — 7 `/auth/*` routes return data with no token at all | Critical |
| DEF-006 | Auth: plaintext password returned in API responses | High |
| DEF-007 | Auth: invalid expiresInMins returns 500 instead of 400 | Medium |
| DEF-008 | Auth: wrong HTTP method returns 404 instead of 405 | Low |
| DEF-009 | Wrong HTTP method / invalid URL returns HTML instead of JSON | Low |
| DEF-010 | Products: incorrect sort order for titles with a common prefix | Low |
| DEF-011 | Products: invalid sortBy value silently ignored | Low |
| DEF-012 | Products: invalid category returns 200 instead of 404 | Medium |
| DEF-013 | Carts: skip beyond total not rejected | Low |
| DEF-014 | Inconsistent error codes for invalid IDs across routes | Low |
| DEF-015 | Inconsistent field naming for the same value across endpoints | Low |
| DEF-016 | Carts: invalid productId silently dropped on add | Medium |
| DEF-017 | Carts: duplicate product entries not consolidated (data quality) | Low |
| DEF-018 | IDOR: GET /auth/users/{id}/carts (sibling of DEF-004) | Critical |
| DEF-019 | Add User: invalid/incomplete input accepted silently | Medium |
| DEF-020 | Update User: invalid/incomplete input accepted silently | Medium |
| DEF-021 | IDOR: DELETE /auth/users/{id} (sibling of DEF-004, DEF-018) | Critical |

Two additional findings are under review, not yet filed as defects — see `docs/defects.md`.

### Features Validated

- ✅ Authentication & JWT lifecycle — login, token refresh, expiry handling
- ✅ Full CRUD coverage across Products, Carts, and Users
- ✅ Pagination & boundary behaviour — limit, skip, and the undocumented `limit=0` "no-limit" anomaly
- ✅ Negative and boundary input validation across every module
- ✅ Security testing — IDOR (DEF-004, DEF-018, DEF-021) and Broken Object Level Authorization (DEF-005, confirmed across both read and write routes) discovery, OWASP API Top 10 awareness
- ✅ Error-contract consistency checks — status codes, JSON vs. HTML error responses
- ✅ Data-driven testing using JSON fixtures (Add Cart, Add User, Update User scenarios)
- ✅ Cross-module E2E business flow — Auth → Users → Products → Cart, chained dynamically with no hardcoded IDs

---

## Tech Stack

- **API Testing:** Postman v12+
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
   git clone https://github.com/Renukacommits/ecommerce-api-testing.git
   cd ecommerce-api-testing
   ```

2. **Install Newman:**

   ```bash
   npm install -g newman newman-reporter-htmlextra
   ```

3. **Set up the environment:**
   - Open `postman/environments/DummyJSON.postman_environment.json`
   - `baseURL` is pre-filled to `https://dummyjson.com` — verify it after importing, no changes needed

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
│   ├── defects.md          # Full defect write-ups (root cause, OWASP classification, verification)
│   ├── prompts/            # Saved prompts from the AI-assisted workflow
│   └── *.json / *.md       # Data-driven test fixtures and supporting docs
├── reports/                # Newman HTML Extra reports (newman/ raw output is gitignored, regenerated per run)
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
