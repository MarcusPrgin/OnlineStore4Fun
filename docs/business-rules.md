# Online Store Business Rules and Completion Contract

This document defines the P0 rules and acceptance evidence for the first release of the online store. A feature is not complete unless it preserves the applicable invariants and produces the required evidence below.

## 1. Business rules and enforcement

| ID | Invariant | Enforcement layer | Required verification |
| --- | --- | --- | --- |
| BR-001 | Every product has a non-empty name and a price greater than or equal to zero. | Zod request schema and SQL `NOT NULL`/`CHECK` constraints. | API validation tests and SQL constraint tests. |
| BR-002 | Product prices and order amounts use integer minor units (for example, cents); floating-point currency values are never stored. | Zod integer validation and SQL integer columns with `CHECK (amount >= 0)`. | API schema test and SQL schema test. |
| BR-003 | A cart belongs to exactly one authenticated customer. | Fastify authentication/authorization and a SQL foreign key. | Unauthenticated and cross-user API tests plus a foreign-key test. |
| BR-004 | A customer may read or modify only their own cart and orders. | Fastify ownership authorization on every customer cart and order route. | Cross-user requests must return `404` without revealing whether the resource exists. |
| BR-005 | Only an administrator may access `/admin/*`. | Fastify role authorization applied to the entire `/admin` route group. | Unauthenticated requests return `401`; authenticated non-admin requests return `403`. |
| BR-006 | Cart quantities are positive whole numbers and may not exceed the configured per-item maximum of 99. | Zod validation and SQL `CHECK (quantity BETWEEN 1 AND 99)`. | Boundary tests for `0`, `1`, `99`, and `100`, plus a SQL constraint test. |
| BR-007 | A cart contains at most one line for a given product; adding the same product changes its quantity instead of creating another line. | SQL unique constraint on `(cart_id, product_id)` and transactional cart update logic. | Duplicate-line SQL test and API add-item test. |
| BR-008 | Product name, price, availability, and inventory are read from the server; values supplied by the browser are never trusted for checkout totals. | Zod rejects unknown checkout pricing fields; checkout service reloads product data inside the transaction. | API test that submits a forged price and confirms it has no effect. |
| BR-009 | Checkout totals equal the sum of the server-side unit price multiplied by quantity for every order item. | Checkout service calculation and SQL transaction. | Deterministic total-calculation test using multiple products and quantities. |
| BR-010 | Inventory may never fall below zero, including during concurrent checkouts. | Atomic SQL update or row locking inside the checkout transaction, plus `CHECK (inventory >= 0)`. | SQL constraint test and concurrent checkout integration test. |
| BR-011 | Creation of an order, its order items, and inventory deductions succeeds or fails as one unit. | SQL transaction. | Forced-failure test proving that all writes roll back. |
| BR-012 | An order has an immutable currency and item price snapshot; later product changes do not change an existing order. | Order-item snapshot columns and application code that prevents updates after creation. | Test that updates a product after checkout and confirms the order is unchanged. |
| BR-013 | A Stripe Checkout Session is associated with exactly one internal order. | SQL unique constraint on the Stripe Checkout Session ID and metadata containing the internal order ID. | Duplicate-session test and Stripe CLI event test. |
| BR-014 | Each Stripe event is processed at most once. | SQL unique constraint on Stripe event ID and transactional webhook handling. | Deliver the same Stripe CLI event twice and confirm only one state transition occurs. |
| BR-015 | A Stripe webhook is processed only when its signature is valid. | Stripe signature verification against the unmodified raw request body before parsing or handling the event. | Valid Stripe CLI event succeeds; missing, altered, or invalid signature returns `400`. |
| BR-016 | Payment status changes are driven by verified Stripe events, not by browser redirects. | Stripe webhook handler and order state-transition rules. | Browser redirect test does not mark an order paid; verified webhook test does. |
| BR-017 | Order status may move only through supported transitions and may not move backward. | Application state-transition function executed in a SQL transaction. | Table-driven tests for every permitted and rejected transition. |
| BR-018 | Secrets, payment details, authorization tokens, and complete webhook payloads are never written to application logs. | Structured logging allowlist and field redaction. | Log-capture test and CloudWatch log inspection. |

Zod is responsible for request shape and local value validation. Fastify authorization is responsible for identity, ownership, and roles. SQL constraints protect rules that must hold regardless of which application code writes to the database. Transactions protect multi-step operations. Stripe signature verification protects webhook authenticity. Application validation does not replace database enforcement where concurrency or multiple writers can violate a rule.

## 2. Completion evidence per subsystem

| Subsystem | Evidence required before completion | Storage/location |
| --- | --- | --- |
| API | Automated tests covering the success path, malformed input, unauthenticated access, unauthorized access, missing resources, and relevant conflicts. Each test records the method, path, expected status, and response assertion. | Version-controlled API test suite and CI output. |
| SQL/database | An automated test for every constraint, unique index, foreign key, and transaction/rollback rule used by the feature. Migration up and down paths must also be tested. | Version-controlled database tests and migration files. |
| Browser/storefront | A passing browser test for the main user path and a screenshot of the final state at the supported desktop viewport. Responsive features also require a mobile viewport screenshot. | Version-controlled browser tests; CI artifacts for screenshots. |
| Admin UI | A passing admin browser test, evidence that a non-admin is rejected, and a screenshot of the completed admin state. | Version-controlled browser tests; CI artifacts for screenshots. |
| Stripe | Stripe CLI event ID, event type, local/test endpoint result, resulting order state, and proof that replaying the event is idempotent. Only Stripe test mode may be used. | Test run notes or CI artifact with secrets and customer data removed. |
| Observability | A CloudWatch log entry showing the request/event correlation ID, outcome, and safe error context. Logs must demonstrate that secrets and sensitive payloads are redacted. | CloudWatch test/staging log group and a linked or captured test artifact. |
| Infrastructure teardown | Date, environment, resources removed, command or workflow used, result, and the person/process performing teardown. A follow-up inventory must show that temporary resources no longer exist. | Deployment run record or CI artifact. |

Evidence must be reproducible by another developer. A manual statement such as “it worked for me” is not completion evidence. Evidence must not contain credentials, authorization tokens, Stripe secrets, personal data, or full payment/webhook payloads.

## 3. Explicit non-goals

The following are intentionally outside the first release:

- **No Next.js.** The storefront will use the selected client application and will communicate with the Fastify API over HTTP.
- **No Edge Functions.** Business logic, authorization, checkout, and webhook processing run in the Fastify service.
- **No direct browser database writes.** The browser never receives database credentials and performs all reads and writes through the API.
- **No live payments.** Development and acceptance testing use Stripe test mode only. Enabling live mode requires a separate production-readiness review.
- **No tax engine.** Prices and totals do not include automated jurisdictional tax calculation in the first release.
- **No shipping engine.** Carrier rates, label purchasing, address validation, tracking integrations, and fulfillment automation are excluded.
- **No Redis.** The initial release uses the application and SQL database without a separate cache or Redis-backed session/job system.
- **No microservices.** The API is one deployable service with clearly separated internal modules.
- **No Kubernetes.** Deployment uses the simplest approved runtime and does not introduce a Kubernetes cluster.

These exclusions are scope boundaries, not future commitments. Adding one requires a documented need, an updated design, new business rules, and explicit approval.

## 4. Initial REST surface

All request bodies, path parameters, and query parameters are validated with Zod. JSON endpoints use `Content-Type: application/json`, except the Stripe webhook route, which must preserve the raw request body for signature verification. Customer and admin routes use the configured Fastify authentication mechanism. Cross-user resource access returns `404` to avoid disclosing resource existence.

| Method | Path | Access | Success | Purpose and applicable rules |
| --- | --- | --- | --- | --- |
| `GET` | `/health` | Public | `200`; `503` when a required dependency is unavailable | Report service and required dependency health. It must not expose secrets or detailed infrastructure configuration. |
| `GET` | `/products` | Public | `200` | List active products. Supports validated pagination parameters. BR-001, BR-002. |
| `GET` | `/products/:slug` | Public | `200` | Return one active product by its unique public slug; return `404` when it is absent or inactive. BR-001, BR-002. |
| `GET` | `/cart` | Customer | `200` | Return the authenticated customer's cart and server-calculated subtotal. BR-003, BR-004, BR-008, BR-009. |
| `POST` | `/cart/items` | Customer | `201` when created; `200` when an existing line is updated | Add `{ productId, quantity }` to the cart. BR-003, BR-004, BR-006, BR-007. |
| `PATCH` | `/cart/items/:itemId` | Customer | `200` | Replace the line quantity using `{ quantity }`. BR-004, BR-006. |
| `DELETE` | `/cart/items/:itemId` | Customer | `204` | Remove an owned cart line. BR-004. |
| `POST` | `/checkout/session` | Customer | `201` | Validate the cart, create the pending order atomically, and create/reuse its Stripe Checkout Session. BR-008 through BR-013. |
| `GET` | `/orders` | Customer | `200` | List the authenticated customer's orders using validated pagination. BR-004, BR-012. |
| `GET` | `/orders/:orderId` | Customer | `200` | Return an owned order and its immutable item snapshot. BR-004, BR-012. |
| `GET` | `/admin/products` | Admin | `200` | List active and inactive products using validated pagination. BR-005. |
| `POST` | `/admin/products` | Admin | `201` | Create a product. BR-001, BR-002, BR-005. |
| `PATCH` | `/admin/products/:productId` | Admin | `200` | Update allowed product fields. Existing order snapshots remain unchanged. BR-001, BR-002, BR-005, BR-012. |
| `GET` | `/admin/orders` | Admin | `200` | List orders using validated status and pagination filters. BR-005. |
| `GET` | `/admin/orders/:orderId` | Admin | `200` | Return an order for support/operations use. BR-005, BR-012, BR-018. |
| `POST` | `/webhooks/stripe` | Valid Stripe signature | `200` | Verify and process supported Stripe events idempotently. BR-013 through BR-018. |

Standard error responses use this shape:

```json
{
  "error": {
    "code": "MACHINE_READABLE_CODE",
    "message": "Safe human-readable message",
    "requestId": "correlation-id"
  }
}
```

Expected status semantics:

- `400 Bad Request`: invalid syntax, schema validation failure, or invalid Stripe signature.
- `401 Unauthorized`: authentication is missing or invalid.
- `403 Forbidden`: the authenticated identity lacks the required role.
- `404 Not Found`: the resource does not exist, is inactive where applicable, or is owned by another customer.
- `409 Conflict`: the request conflicts with current state, such as insufficient stock or an invalid order transition.
- `500 Internal Server Error`: an unexpected server failure; the response must not expose internal details.
- `503 Service Unavailable`: `/health` detects that a required dependency is unavailable.

Any new endpoint or behavior must update this document with its access policy, applicable business-rule IDs, enforcement layer, and completion evidence before it is considered part of the release scope.
