# Storefront Route and Access Map

This document defines the browser-facing routes for the first release and the access rule for each route. These are page routes, not the REST API surface documented in [business-rules.md](./business-rules.md). Client-side route guards improve navigation and user experience, but the Fastify API remains the security boundary and must independently enforce authentication, ownership, and administrator access.

## Access rules

| Access rule | Meaning | Required behavior |
| --- | --- | --- |
| Public | Available without authentication. | Render for every visitor. Authenticated visitors may also use the route. |
| Guest only | Intended for visitors who are not authenticated. | Render for guests; redirect an authenticated customer to the validated `returnTo` destination or `/`. |
| Customer | Requires an authenticated customer or administrator session. | Redirect a guest to `/sign-in?returnTo=<encoded-path>`. Preserve only an internal application path. |
| Owner | Requires authentication and ownership of the requested resource. | Load through the authenticated API. If the resource is absent or belongs to another customer, show the same not-found state. Never reveal another customer's resource. |
| Admin | Requires an authenticated identity with the `admin` role. | Redirect a guest to sign-in. Show access denied to an authenticated non-admin; the API must return `403` for the protected request. |

The `returnTo` value must begin with a single `/` and must not begin with `//`, contain a scheme, or point to another origin. Invalid values fall back to `/`. This prevents open redirects.

## Route map

| Browser route | Access | Page responsibility | Data/API dependency | Failure and redirect behavior |
| --- | --- | --- | --- | --- |
| `/` | Public | Storefront home page. Present featured or recent active products and navigation to the catalog, cart, orders, and sign-in as appropriate for the current session. | `GET /products` | Show a retryable error state if products cannot be loaded. The shell and sign-in navigation remain usable. |
| `/products/:slug` | Public | Product detail page for the URL-safe product slug. Display server-provided name, description, image, price, availability, and add-to-cart action. | `GET /products/:slug`; adding an item uses `POST /cart/items` | Show not found when the product is absent or inactive. If a guest selects add to cart, send them to `/sign-in` with this page as `returnTo`; never trust product price from the browser. |
| `/cart` | Customer | Display the authenticated customer's cart, server-calculated subtotal, quantity controls, remove actions, and checkout action. | `GET /cart`, `POST /cart/items`, `PATCH /cart/items/:itemId`, `DELETE /cart/items/:itemId`, `POST /checkout/session` | Guests go to sign-in. Show an empty-cart state for no lines, an inline conflict for unavailable stock, and a retryable error for transient failures. Redirect only to the Stripe Checkout URL returned by the API. |
| `/checkout/result` | Customer | Display the status of the checkout after returning from Stripe. Treat the page as informational; it must not mark an order paid. The page reads the server-confirmed order state and may briefly show `processing` while a webhook is pending. | Read the customer's associated order through `GET /orders/:orderId`; payment status is set only by `POST /webhooks/stripe` | Guests go to sign-in. Missing or invalid result identifiers show a safe error with a link to `/orders`. An order owned by someone else uses the same not-found state as a missing order. |
| `/orders` | Customer | List the authenticated customer's orders, newest first, with status, date, total, and a link to details. | `GET /orders` | Guests go to sign-in. Show a first-order empty state when no orders exist and a retryable error when loading fails. |
| `/orders/:id` | Owner | Show an immutable order summary, item price snapshots, total, and current payment/order status. | `GET /orders/:orderId`, using `:id` as `orderId` | Guests go to sign-in. Missing and cross-customer orders display the identical not-found state. |
| `/sign-in` | Guest only | Authenticate the visitor, display safe validation errors, and resume an interrupted internal route through a validated `returnTo` parameter. | Configured authentication endpoint/session mechanism | Authenticated visitors go to the safe `returnTo` value or `/`. Failed authentication stays on the page and must not reveal whether a particular account exists. |
| `/admin/*` | Admin | Parent boundary for all administration pages. Every child route inherits the admin guard. Initial children cover product management and order inspection. | `/admin/products`, `/admin/products/:productId`, `/admin/orders`, and `/admin/orders/:orderId` API operations | Guests go to sign-in with a safe `returnTo`. Authenticated non-admins see access denied. Unknown admin child paths show not found without bypassing the parent guard. |

## Admin child routes

| Browser route | Access | Purpose |
| --- | --- | --- |
| `/admin` | Admin | Administration landing page with links to products and orders. |
| `/admin/products` | Admin | List active and inactive products and provide a create-product action. |
| `/admin/products/new` | Admin | Create a product using server-validated fields. |
| `/admin/products/:id` | Admin | Edit the allowed fields of an existing product. |
| `/admin/orders` | Admin | List and filter orders for operational support. |
| `/admin/orders/:id` | Admin | Inspect an order without exposing secrets or full Stripe payloads. |

## Route enforcement requirements

1. Resolve the current session before rendering a protected page or show a neutral loading state while it is being resolved. Do not briefly render protected content to a guest.
2. Preserve the complete internal path, query, and fragment when sending a guest to sign-in, but validate the resulting `returnTo` value before navigation.
3. Apply the admin guard to the `/admin/*` parent so a new admin child cannot accidentally be left public.
4. Treat browser guards as user-experience controls only. Fastify must enforce BR-003, BR-004, and BR-005 for every protected API request.
5. Never infer payment success from `/checkout/result`, its query parameters, or a Stripe redirect. Only a verified, idempotently processed Stripe webhook may change payment state under BR-014 through BR-017.
6. Use the same visual not-found result for missing and cross-customer orders. Log only the safe request correlation ID and outcome.
7. A route not listed here is outside the first-release browser surface until this document is updated and its access rule is defined.

## Completion evidence

The route map is complete when automated browser tests demonstrate:

- a guest can open `/` and an active `/products/:slug` page;
- a guest attempting each customer route is sent to sign-in with a safe `returnTo` value;
- successful sign-in returns the customer to that validated route;
- an authenticated customer can use `/cart`, `/orders`, and an owned `/orders/:id`;
- a customer cannot discover or display another customer's order;
- an authenticated non-admin cannot access any `/admin/*` child;
- an administrator can access every listed admin child route;
- an invalid or external `returnTo` value falls back to `/`;
- `/checkout/result` reflects server-confirmed state and never marks an order paid from browser input; and
- unknown product, order, and admin paths show the correct not-found or access-denied state.

Capture desktop and mobile screenshots for the home page, product page, cart, checkout result, order list, order detail, sign-in page, admin product list, and admin order list as required by the completion contract.
