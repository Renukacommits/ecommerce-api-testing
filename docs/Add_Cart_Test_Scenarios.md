# Add Cart – Test Scenarios Reference

**Endpoint:** `POST /carts/add`  
**Project:** E-commerce API Testing Portfolio – DummyJSON

---

## API Notes

- Adding a cart is **simulated** – data is not persisted on the server
- Request body accepts: `userId` (integer) and `products` (array of objects with `id` and `quantity`)
- Stock validation is **not enforced** by DummyJSON – scenarios 31, 32, 37 are N/A for this API

---

## Test Scenarios

| #   | Scenario                                                          | Type       | Notes                                      |
| --- | ----------------------------------------------------------------- | ---------- | ------------------------------------------ |
| 1   | Valid userId, single product, valid quantity                      | Happy Path |                                            |
| 2   | Valid userId, multiple products (5), valid quantities             | Happy Path |                                            |
| 3   | Valid userId, multiple products from different categories         | Happy Path |                                            |
| 4   | Valid userId, quantity = 1 (minimum)                              | Boundary   |                                            |
| 5   | Valid userId, quantity = 20 (assumed maximum)                     | Boundary   | No documented max                          |
| 6   | Valid userId, same product added twice                            | Boundary   |                                            |
| 7   | Valid userId at boundary – userId 208 (last valid user)           | Boundary   |                                            |
| 8   | userId = 0                                                        | Negative   |                                            |
| 9   | userId = -1 (negative)                                            | Negative   |                                            |
| 10  | userId = 9999 (non-existent)                                      | Negative   |                                            |
| 11  | productId = 0                                                     | Negative   |                                            |
| 12  | productId = 9999 (non-existent)                                   | Negative   |                                            |
| 13  | quantity = 0                                                      | Negative   |                                            |
| 14  | quantity = -1 (negative)                                          | Negative   |                                            |
| 15  | Missing userId entirely                                           | Negative   |                                            |
| 16  | Missing products array entirely                                   | Negative   |                                            |
| 17  | Empty products array                                              | Negative   |                                            |
| 18  | Two userIds in body                                               | Negative   |                                            |
| 19  | productId = 195 (simulated product, does not exist on server)     | Negative   |                                            |
| 20  | quantity = string instead of number ("four")                      | Negative   | Wrong data type                            |
| 21  | userId = string instead of number ("abc")                         | Negative   | Wrong data type                            |
| 22  | productId = string instead of number ("abc")                      | Negative   | Wrong data type                            |
| 23  | quantity = decimal number (1.5)                                   | Negative   |                                            |
| 24  | Missing quantity field in product object                          | Negative   |                                            |
| 25  | Missing productId field in product object                         | Negative   |                                            |
| 26  | Empty request body                                                | Negative   |                                            |
| 27  | Valid userId, maximum number of products in one cart              | Boundary   | No documented max                          |
| 28  | userId = null                                                     | Negative   |                                            |
| 29  | productId = null                                                  | Negative   |                                            |
| 30  | quantity = null                                                   | Negative   |                                            |
| 31  | Valid userId, out of stock product                                | Edge       | **N/A – DummyJSON does not enforce stock** |
| 32  | Valid userId, quantity exceeds available stock                    | Edge       | **N/A – DummyJSON does not enforce stock** |
| 33  | Valid userId, duplicate products in same request                  | Edge       |                                            |
| 34  | Very large quantity (999999)                                      | Boundary   |                                            |
| 35  | Very large userId (999999999)                                     | Boundary   |                                            |
| 36  | Valid userId, all products from same category                     | Edge       |                                            |
| 37  | Valid userId, quantity = total stock exactly                      | Edge       | **N/A – DummyJSON does not enforce stock** |
| 38  | Request with extra/unknown fields in body                         | Edge       | API should ignore or reject                |
| 39  | Valid userId, 1 valid product + 1 invalid product in same request | Negative   |                                            |
| 40  | userId = float (1.5)                                              | Negative   | Wrong data type                            |

---

## Data-Driven Testing Candidates

Scenarios selected for JSON data file (Newman `--iteration-data`):

| #   | Scenario                                              | Type       |
| --- | ----------------------------------------------------- | ---------- |
| 1   | Valid userId, single product, valid quantity          | Happy Path |
| 2   | Valid userId, multiple products (5), valid quantities | Happy Path |
| 4   | Valid userId, quantity = 1 (minimum)                  | Boundary   |
| 5   | Valid userId, quantity = 20 (assumed maximum)         | Boundary   |
| 8   | userId = 0                                            | Negative   |
| 9   | userId = -1                                           | Negative   |
| 13  | quantity = 0                                          | Negative   |
| 14  | quantity = -1                                         | Negative   |
| 17  | Empty products array                                  | Negative   |
| 26  | Empty request body                                    | Negative   |

> **Note:** Select final scenarios based on what the API actually returns for each – update expected responses accordingly.

---

## Known API Limitations (DummyJSON)

- Stock validation not enforced
- Simulated POST – data not persisted
- No documented maximum for quantity or number of products per cart
