# Customer Journey

| Step | Initiator | Request or event | System action | Result/source of truth |
| --- | --- | --- | --- | --- |
| 1 | Customer | Opens `/` | React requests `GET /api/v1/products` | PostgreSQL products/inventory |
| 2 | Customer | Submits sign-in form | React calls Supabase Auth | Supabase returns a session/JWT |
| 3 | React | Adds bearer token | Calls `POST /api/v1/cart/items` | Fastify verifies identity |
| 4 | Fastify | Validates cart addition | Checks product and inventory, then writes cart item | PostgreSQL cart |
| 5 | Customer | Clicks checkout | Calls `POST /api/v1/checkout/session` | Fastify revalidates cart |
| 6 | Fastify | Creates pending order | Locks inventory and snapshots prices | PostgreSQL order |
| 7 | Fastify | Creates Checkout Session | Sends server-calculated items to Stripe | Stripe returns checkout URL |
| 8 | Browser | Opens checkout URL | Customer uses Stripe test card | Stripe processes payment |
| 9 | Stripe | Sends signed event | `POST /api/v1/webhooks/stripe` | Fastify verifies signature |
| 10 | Fastify | Processes event once | Marks order paid and updates inventory | PostgreSQL order state |
| 11 | React | Polls/refetches order | `GET /api/v1/orders/:id` | UI displays processed state |
