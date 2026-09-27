# TinyShop Authority Map

| Concern | Source of truth | Who may change it? | Notes |
| --- | --- | --- | --- |
| User identity | Supabase Auth | Supabase Auth | Determines who signed in |
| Application role | PostgreSQL `profiles.role` | Authorized backend/admin process | Never trust a role sent by React |
| Products and prices | PostgreSQL `products` | Fastify admin endpoint | React only displays returned data |
| Inventory | PostgreSQL `inventory` | Fastify transactions | Lock rows during checkout |
| Cart | PostgreSQL `carts` and `cart_items` | Authenticated Fastify endpoints | Customer can only access their cart |
| External payment outcome | Stripe | Stripe | Confirmed through a signed webhook |
| Internal order state | PostgreSQL `orders` | Fastify webhook/admin logic | Updated after verified Stripe events |
| Fulfilment state | PostgreSQL | Authorized administrator endpoint | Must follow allowed transitions |
