# Administrator Journey

## Authentication

1. Administrator signs in through Supabase Auth.
2. React receives a session and access token.
3. React sends the token using `Authorization: Bearer <token>`.
4. Fastify verifies the token.
5. Fastify reads `profiles.role` from PostgreSQL.
6. Fastify rejects the request unless the role is `admin`.

## Create a product

1. React submits `POST /api/v1/admin/products`.
2. Fastify validates the name, slug, integer price, currency, and stock.
3. Fastify checks that the slug is unique.
4. A transaction inserts the `products` row and corresponding `inventory` row.
5. Fastify returns the created product DTO.

## Restock inventory

1. React submits `POST /api/v1/admin/products/:id/inventory-adjustments`.
2. Fastify verifies administrator authorization.
3. Fastify locks the inventory row.
4. Fastify verifies the resulting quantity cannot be negative.
5. Fastify updates inventory and records an audit entry.

## Fulfil an order

1. React requests paid orders from `GET /api/v1/admin/orders`.
2. Administrator selects an order.
3. React submits `PATCH /api/v1/admin/orders/:id/fulfillment`.
4. Fastify validates the requested state transition.
5. Fastify updates the order and inserts an audit-history row.

## Issue a test refund

1. React submits `POST /api/v1/admin/orders/:id/refund`.
2. Fastify verifies the order is refundable.
3. Fastify sends an idempotent refund request to Stripe.
4. Stripe sends a refund webhook.
5. Fastify verifies and processes the webhook.
6. PostgreSQL records the final refund state.
