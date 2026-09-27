# First-Release Wireframes and React Component Map

This document turns the browser routes in [routes.md](./routes.md) into low-fidelity page plans. It defines layout, component responsibilities, responsive behavior, and non-happy-path states before React implementation begins.

These diagrams are structural, not visual designs. Names are provisional React component names in PascalCase. A component should own a meaningful responsibility; ordinary headings, wrappers, and individual text elements do not need their own components.

## Shared application structure

```text
App
├── SessionProvider
├── AppRouter
│   ├── StoreLayout
│   │   ├── StoreHeader
│   │   ├── RouteOutlet
│   │   └── StoreFooter
│   ├── GuestOnlyRoute
│   ├── CustomerRoute
│   ├── OwnerRoute
│   └── AdminRoute
│       └── AdminLayout
│           ├── AdminHeader
│           ├── AdminNavigation
│           └── RouteOutlet
└── NotificationRegion
```

Shared state components:

- `PageLoading`: full-page or primary-content loading state with an accessible label.
- `SectionSkeleton`: preserves a section's layout while its data loads.
- `EmptyState`: explains that a successful request returned no content and offers a useful next action.
- `FieldError`: associates a validation message with one form field.
- `FormErrorSummary`: summarizes form-level validation failures.
- `AccessDenied`: shown to authenticated users without the required role.
- `NotFound`: used for missing public resources and for both missing and non-owned customer resources.
- `ConflictNotice`: explains recoverable state conflicts and supplies a refresh or correction action.
- `ServerError`: gives a safe message, request ID when available, and retry action.
- `StatusBadge`: presents product, payment, and order states using text as well as color.
- `ConfirmDialog`: confirms destructive or consequential actions and restores focus on close.

Global layout rules:

- The store header remains available during page-level loading and retryable failures.
- The desktop content width is constrained and centered. Mobile layouts use a single column with comfortable side padding.
- Visible focus, keyboard operation, semantic headings, field labels, error associations, and sufficient contrast are required from the first implementation.
- Loading placeholders should resemble the final layout to reduce visual movement.
- Error messages never expose stack traces, secrets, raw Stripe payloads, or whether another customer owns a resource.

## `/` — Storefront home

Access: Public.

```text
┌──────────────────────────────────────────────────────────────┐
│ Logo                     Orders  Cart  Sign in / Account     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Store headline                                              │
│  Short value statement                         [Shop products]│
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  Featured products                                           │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐              │
│  │ Image      │  │ Image      │  │ Image      │              │
│  │ Name       │  │ Name       │  │ Name       │              │
│  │ Price      │  │ Price      │  │ Price      │              │
│  │ [View]     │  │ [View]     │  │ [View]     │              │
│  └────────────┘  └────────────┘  └────────────┘              │
├──────────────────────────────────────────────────────────────┤
│ Footer                                                       │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
HomePage
├── HeroSection
└── FeaturedProductSection
    └── ProductGrid
        └── ProductCard × n
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Keep `StoreHeader` and hero visible; show `ProductCardSkeleton` items in the product grid. |
| Empty | Replace the grid with `EmptyState`: no products are currently available. Do not show a broken blank section. |
| Validation | N/A; the route contains no first-release form or user input. |
| Unauthorized | N/A; the route is public. Session-specific navigation may still be loading. |
| Not found | N/A for the route itself; the global router handles unknown paths. |
| Conflict | N/A; the page performs no state-changing operation. |
| Server error | Keep the application shell and hero; replace the grid with `ServerError` and a retry button. |

Mobile: collapse account links into an accessible menu and render product cards in one column; two columns may be used when width permits without shrinking tap targets.

## `/products/:slug` — Product detail

Access: Public; adding to cart requires a customer session.

```text
┌──────────────────────────────────────────────────────────────┐
│ Store header                                                 │
├──────────────────────────────────────────────────────────────┤
│ Home / Product name                                          │
│                                                              │
│ ┌────────────────────────┐  Product name                     │
│ │                        │  Price                            │
│ │     Product image      │  Availability                    │
│ │                        │                                   │
│ └────────────────────────┘  Description                     │
│                             Quantity [ − ][ 1 ][ + ]         │
│                             [Add to cart]                    │
│                             Inline feedback area             │
├──────────────────────────────────────────────────────────────┤
│ Store footer                                                 │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
ProductPage
├── Breadcrumbs
├── ProductMedia
└── ProductPurchasePanel
    ├── ProductPrice
    ├── AvailabilityMessage
    ├── QuantityControl
    ├── AddToCartButton
    └── MutationFeedback
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Show image, title, description, price, and action skeletons in the final two-column footprint. |
| Empty | N/A; a route for a missing product uses the not-found state. An empty description simply omits that optional section. |
| Validation | Mark the quantity control invalid and explain the permitted integer range of 1–99. |
| Unauthorized | The page remains visible. An unauthenticated add-to-cart action navigates to sign-in with this route as `returnTo`. |
| Not found | Replace page content with `NotFound` and a link back to `/`. Use this for absent and inactive products. |
| Conflict | If price, availability, or inventory changed, show `ConflictNotice`, refresh server data, and require confirmation of the current values. |
| Server error | Show `ServerError` with retry when loading fails. A failed add-to-cart action remains inline so the product page is not lost. |

Mobile: place product information below the image and keep the primary action full width without fixing it over content.

## `/cart` — Shopping cart

Access: Customer.

```text
┌──────────────────────────────────────────────────────────────┐
│ Store header                                                 │
├──────────────────────────────────────────────────────────────┤
│ Your cart                                                    │
│                                                              │
│ ┌──────────────────────────────────┐ ┌──────────────────────┐ │
│ │ Image  Product       Price       │ │ Order summary        │ │
│ │        Quantity controls         │ │ Subtotal             │ │
│ │        [Remove]                  │ │ Taxes: not calculated│ │
│ ├──────────────────────────────────┤ │                      │ │
│ │ Additional cart lines           │ │ [Proceed to checkout]│ │
│ └──────────────────────────────────┘ └──────────────────────┘ │
│ Inline inventory/conflict notice                             │
├──────────────────────────────────────────────────────────────┤
│ Store footer                                                 │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
CartPage
├── CartItemList
│   └── CartItemRow × n
│       ├── QuantityControl
│       └── RemoveCartItemButton
├── CartSummary
│   └── CheckoutButton
└── ConflictNotice
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | After session resolution, show cart-row and summary skeletons. Disable checkout until server totals are known. |
| Empty | Show `EmptyState` with a link to `/`; omit the inactive order summary and checkout action. |
| Validation | Display quantity errors beside the affected line. Restore the last server-confirmed quantity after invalid input. |
| Unauthorized | Do not render cart content; redirect to sign-in with `/cart` as `returnTo`. |
| Not found | If an item disappears, remove it after refresh and announce the change. The route itself does not use a not-found page. |
| Conflict | Mark affected lines when stock or price changes, display current server values, and disable checkout until reconciled. |
| Server error | Initial-load failure shows a page-level retry. Mutation failure stays on the affected line and retains the last confirmed cart. Checkout creation failure leaves the cart intact. |

Mobile: stack each cart row and place the summary below the items. Keep quantity and remove controls adjacent to the relevant product.

## `/checkout/result` — Checkout result

Access: Customer.

```text
┌──────────────────────────────────────────────────────────────┐
│ Store header                                                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│                 [Status icon]                                │
│                 Checkout status                              │
│                 Clear status explanation                     │
│                                                              │
│                 Order number / safe summary                  │
│                 [View order]  [View all orders]              │
│                                                              │
│                 Processing indicator when pending            │
├──────────────────────────────────────────────────────────────┤
│ Store footer                                                 │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
CheckoutResultPage
├── CheckoutStatusPanel
│   ├── StatusBadge
│   ├── ProcessingIndicator
│   └── CheckoutResultActions
└── SafeOrderSummary
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Show a neutral “Confirming your order” state while retrieving the server-confirmed order. Do not claim payment succeeded. |
| Empty | Missing result context shows a safe explanation and a link to `/orders`; it is not treated as successful payment. |
| Validation | Invalid or malformed result identifiers use the same safe result-context error without echoing raw input. |
| Unauthorized | Redirect to sign-in with the full internal result route as `returnTo`. |
| Not found | Missing and non-owned associated orders use the identical `NotFound` state with a link to `/orders`. |
| Conflict | N/A as a mutation conflict. Pending or cancelled payment is a normal status and receives its own truthful message and next action. |
| Server error | Show `ServerError` with retry and a link to `/orders`. Never convert an inability to confirm into a success message. |

Mobile: center the single status panel, keep its text left-aligned when longer, and stack actions vertically.

## `/orders` — Customer order list

Access: Customer.

```text
┌──────────────────────────────────────────────────────────────┐
│ Store header                                                 │
├──────────────────────────────────────────────────────────────┤
│ Your orders                                                  │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ Order #       Date       Status       Total       [View] │ │
│ ├──────────────────────────────────────────────────────────┤ │
│ │ …                                                        │ │
│ └──────────────────────────────────────────────────────────┘ │
│                         [Previous] Page n [Next]              │
├──────────────────────────────────────────────────────────────┤
│ Store footer                                                 │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
OrdersPage
├── OrderList
│   └── OrderListItem × n
│       └── StatusBadge
└── PaginationControls
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Show order-row skeletons while preserving the page title and table/list structure. |
| Empty | Show `EmptyState` with a link to browse products. |
| Validation | Invalid pagination parameters are normalized to safe defaults rather than rendered as form errors. |
| Unauthorized | Redirect to sign-in with `/orders` as `returnTo`. |
| Not found | N/A for the collection route; an out-of-range valid page shows the empty result with navigation back to the first page. |
| Conflict | N/A; the page performs no state-changing action. |
| Server error | Show `ServerError` with retry without removing the store shell. |

Mobile: render each order as a labeled card instead of forcing the desktop table to scroll horizontally.

## `/orders/:id` — Customer order detail

Access: Owner.

```text
┌──────────────────────────────────────────────────────────────┐
│ Store header                                                 │
├──────────────────────────────────────────────────────────────┤
│ [Back to orders]        Order #…        [Status]              │
│ Date                                                         │
│ ┌────────────────────────────────────┐ ┌────────────────────┐ │
│ │ Items                              │ │ Summary            │ │
│ │ Product snapshot   Qty   Price     │ │ Subtotal           │ │
│ │ …                                  │ │ Total              │ │
│ └────────────────────────────────────┘ └────────────────────┘ │
│ Payment/order status explanation                             │
├──────────────────────────────────────────────────────────────┤
│ Store footer                                                 │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
OrderDetailPage
├── OrderHeader
│   └── StatusBadge
├── OrderItemList
│   └── OrderItemRow × n
├── OrderSummary
└── OrderStatusExplanation
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Show header, item-row, and summary skeletons after session resolution. |
| Empty | An order with no items violates the documented invariant and is shown as a safe server error, not a normal empty order. |
| Validation | A malformed route ID goes directly to the common not-found presentation. |
| Unauthorized | Guests go to sign-in. A different customer receives the same result as a missing order. |
| Not found | Show `NotFound` for absent and non-owned orders, with no ownership details and a link to `/orders`. |
| Conflict | N/A; customer order detail is read-only. Status changes arriving during viewing should refresh without showing a mutation conflict. |
| Server error | Show `ServerError` with retry and a link back to the order list. |

Mobile: stack the summary below the item list and display item fields with labels rather than a compressed table.

## `/sign-in` — Sign in

Access: Guest only.

```text
┌──────────────────────────────────────────────────────────────┐
│ Store header                                                 │
├──────────────────────────────────────────────────────────────┤
│                   ┌──────────────────────────┐               │
│                   │ Sign in                  │               │
│                   │ Email                    │               │
│                   │ [                      ] │               │
│                   │ Password                 │               │
│                   │ [                      ] │               │
│                   │ Form error summary       │               │
│                   │ [Sign in]                │               │
│                   └──────────────────────────┘               │
├──────────────────────────────────────────────────────────────┤
│ Store footer                                                 │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
SignInPage
└── SignInPanel
    └── SignInForm
        ├── FormErrorSummary
        ├── EmailField
        ├── PasswordField
        └── SubmitButton
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | While resolving the session, show a neutral panel skeleton. During submission, retain fields, disable duplicate submission, and label progress. |
| Empty | N/A; required fields are handled as validation errors rather than an empty page state. |
| Validation | Show field-level messages for missing or malformed input and focus the first invalid field after submission. |
| Unauthorized | Invalid credentials show one generic form-level message that does not reveal whether the account exists. |
| Not found | N/A; invalid `returnTo` values fall back to `/` rather than producing a not-found state. |
| Conflict | If a valid session appears while this page is open, redirect to the validated `returnTo` destination or `/`. |
| Server error | Preserve the entered email, clear or retain the password according to the authentication library's safe behavior, and show a retryable generic error. |

Mobile: the form fills the available width with no horizontal scrolling; input zoom and password-manager support must remain intact.

## `/admin/*` — Shared administration layout and guard

Access: Admin.

```text
┌──────────────────────────────────────────────────────────────┐
│ Admin header                         Storefront  Account      │
├──────────────────┬───────────────────────────────────────────┤
│ Dashboard        │                                           │
│ Products         │              Child route                  │
│ Orders           │              content                      │
│                  │                                           │
└──────────────────┴───────────────────────────────────────────┘
```

Proposed component tree:

```text
AdminRoute
└── AdminLayout
    ├── AdminHeader
    ├── AdminNavigation
    └── AdminRouteOutlet
```

Shared state treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Show a neutral session/role loading state before any admin data or navigation is rendered. |
| Empty | Defined by the active child route. |
| Validation | Defined by the active child route. |
| Unauthorized | Guests go to sign-in. Authenticated non-admins see `AccessDenied`; no admin child request should be made first. |
| Not found | An unknown child under `/admin/*` shows the admin not-found state inside the guarded layout. |
| Conflict | Defined by the active child route. |
| Server error | Failure to establish authorization shows a safe retryable error; it must never default to allowing access. |

Mobile: replace the persistent sidebar with a labeled menu or drawer that is keyboard accessible and closes after navigation.

## `/admin` — Administration dashboard

Access: Admin.

```text
┌──────────────────────────────────────────────────────────────┐
│ Admin layout                                                 │
│  Dashboard                                                   │
│  ┌───────────────────────┐  ┌─────────────────────────────┐  │
│  │ Products              │  │ Orders                      │  │
│  │ Manage catalog        │  │ Inspect customer orders     │  │
│  │ [Open products]       │  │ [Open orders]               │  │
│  └───────────────────────┘  └─────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
AdminDashboardPage
└── AdminSectionGrid
    └── AdminSectionCard × 2
```

State treatment: session loading, unauthorized, unknown-route, and server failure use the shared admin states. Empty, validation, and conflict are N/A because this dashboard is navigation-only.

## `/admin/products` — Product administration

Access: Admin.

```text
┌──────────────────────────────────────────────────────────────┐
│ Admin layout                                                 │
│  Products                                  [Create product]  │
│  Search [____________]  Status [All ▼]                       │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Product      Slug       Price      Inventory   Status   │ │
│  │ …                                               [Edit]  │ │
│  └─────────────────────────────────────────────────────────┘ │
│                           [Previous] Page n [Next]            │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
AdminProductsPage
├── AdminPageHeader
├── ProductFilters
├── AdminProductTable
│   └── AdminProductRow × n
└── PaginationControls
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Show filter controls and product-row skeletons. |
| Empty | With no products, show create-product guidance. With filters applied, explain that no products match and offer clear-filters. |
| Validation | Normalize invalid pagination; reject unsupported filter values without sending them to the API. |
| Unauthorized | Use the shared admin guard state. |
| Not found | N/A for the unfiltered collection route. |
| Conflict | N/A because the list does not change product state in the first release. |
| Server error | Replace the table area with `ServerError` and retry while retaining safe filter input. |

Mobile: render product rows as cards with name, price, inventory, status, and edit action; filters stack above the results.

## `/admin/products/new` — Create product

Access: Admin.

```text
┌──────────────────────────────────────────────────────────────┐
│ Admin layout                                                 │
│  [Back to products]  Create product                          │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Name        [________________________________________]  │ │
│  │ Slug        [________________________________________]  │ │
│  │ Description [________________________________________]  │ │
│  │ Price       [____________]  Inventory [____________]    │ │
│  │ Status      [Active toggle]                             │ │
│  │ Form errors                                             │ │
│  │                                  [Cancel] [Create]      │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
CreateProductPage
└── ProductForm
    ├── FormErrorSummary
    ├── ProductIdentityFields
    ├── ProductDescriptionField
    ├── ProductCommerceFields
    ├── ProductStatusField
    └── FormActions
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Session loading uses the admin guard. Submission disables duplicate create requests and announces progress. |
| Empty | N/A; empty required inputs become validation errors. |
| Validation | Display Zod/client feedback beside fields and merge safe server validation errors into the same presentation. |
| Unauthorized | Use the shared admin guard state. |
| Not found | N/A for creation. |
| Conflict | A duplicate slug or concurrently created equivalent product highlights the conflicting field and preserves other input. |
| Server error | Preserve safe form values, show the request-level error, and allow retry without duplicating a successful request. |

Mobile: stack all fields and actions; keep Cancel visually secondary and avoid placing it where it is easy to tap accidentally.

## `/admin/products/:id` — Edit product

Access: Admin.

```text
┌──────────────────────────────────────────────────────────────┐
│ Admin layout                                                 │
│  [Back to products]  Edit product                            │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Same fields as create, populated from server data       │ │
│  │                                                         │ │
│  │ Unsaved-change / conflict message region                │ │
│  │                                  [Cancel] [Save]        │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
EditProductPage
├── ProductFormSkeleton | ProductForm
│   └── shared create-form children
└── UnsavedChangesPrompt
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Show a form-shaped skeleton until the existing product is loaded. Submission shows progress without discarding values. |
| Empty | N/A; no product data uses not found rather than an empty form. |
| Validation | Use the shared `ProductForm` validation presentation. |
| Unauthorized | Use the shared admin guard state. |
| Not found | Show admin `NotFound` with a link to `/admin/products`. |
| Conflict | If the product changed since it was loaded or the slug collides, show current server information and require review before retry. |
| Server error | Initial failure shows retry/back actions. Save failure retains edits and allows a safe retry. |

Mobile: use the create-product form layout and ensure the unsaved-changes prompt works with touch and browser back navigation.

## `/admin/orders` — Order administration

Access: Admin.

```text
┌──────────────────────────────────────────────────────────────┐
│ Admin layout                                                 │
│  Orders                                                      │
│  Status [All ▼]  From [date]  To [date]  [Apply] [Clear]     │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Order #   Date   Customer ref   Status   Total   [View] │ │
│  │ …                                                       │ │
│  └─────────────────────────────────────────────────────────┘ │
│                           [Previous] Page n [Next]            │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
AdminOrdersPage
├── AdminPageHeader
├── OrderFilters
├── AdminOrderTable
│   └── AdminOrderRow × n
│       └── StatusBadge
└── PaginationControls
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Keep filters visible and show order-row skeletons. |
| Empty | Distinguish no orders at all from no matches for the current filters; filtered empty state offers clear-filters. |
| Validation | Invalid date ranges and unsupported statuses are explained next to the relevant controls before a request is sent. |
| Unauthorized | Use the shared admin guard state. |
| Not found | N/A for the collection route. |
| Conflict | N/A because the list is read-only in the first release. |
| Server error | Replace results with `ServerError` and retry while preserving valid filters. |

Mobile: use order cards and a collapsible filter panel; never hide status text behind color alone.

## `/admin/orders/:id` — Admin order detail

Access: Admin.

```text
┌──────────────────────────────────────────────────────────────┐
│ Admin layout                                                 │
│  [Back to orders]  Order #…                    [Status]      │
│  Created date · Safe customer reference                      │
│  ┌────────────────────────────────────┐ ┌────────────────────┐│
│  │ Item snapshots                     │ │ Totals             ││
│  │ Product   Quantity   Unit price    │ │ Subtotal / Total   ││
│  └────────────────────────────────────┘ └────────────────────┘│
│  Payment and event summary (no secrets/raw payloads)         │
└──────────────────────────────────────────────────────────────┘
```

Proposed component tree:

```text
AdminOrderDetailPage
├── AdminOrderHeader
│   └── StatusBadge
├── OrderItemList
├── OrderSummary
└── PaymentEventSummary
```

State treatment:

| State | Planned behavior |
| --- | --- |
| Loading | Show header, item, summary, and event-summary skeletons. |
| Empty | Missing event history may show “No events recorded”; an order with no items is a safe server error, not a normal empty order. |
| Validation | A malformed ID uses the not-found presentation. |
| Unauthorized | Use the shared admin guard state. |
| Not found | Show admin `NotFound` with a link to `/admin/orders`. |
| Conflict | N/A while this route remains inspection-only. Refresh server status if it changes. |
| Server error | Show retry/back actions and never display raw internal or Stripe error content. |

Mobile: stack item, totals, and event sections; labels replace compressed table headings.

## Route-state coverage matrix

This matrix verifies that every required state was considered. `N/A` means the state cannot occur in the defined first-release behavior, not that it was forgotten.

| Route | Loading | Empty | Validation | Unauthorized | Not found | Conflict | Server error |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `/` | Product skeletons | No active products | N/A | N/A | Global router | N/A | Product-section retry |
| `/products/:slug` | Detail skeleton | N/A | Quantity | Sign-in on cart action | Missing/inactive product | Inventory/price change | Page or inline retry |
| `/cart` | Cart skeleton | Empty cart | Quantity | Sign-in redirect | Removed line handling | Inventory/price change | Page/line retry |
| `/checkout/result` | Confirming status | Missing context | Invalid context | Sign-in redirect | Missing/non-owned order | N/A; pending is normal | Retry plus orders link |
| `/orders` | Row skeletons | No orders/results | Safe pagination defaults | Sign-in redirect | N/A | N/A | List retry |
| `/orders/:id` | Detail skeleton | Invalid empty order | Malformed ID to not found | Sign-in or concealed ownership | Missing/non-owned order | N/A | Retry/back |
| `/sign-in` | Session/submit progress | N/A | Field errors | Generic credential failure | N/A | Existing session redirect | Preserve safe input/retry |
| `/admin/*` | Session/role loading | Child-defined | Child-defined | Redirect/access denied | Unknown child | Child-defined | Fail closed/retry |
| `/admin` | Guard loading | N/A | N/A | Shared admin state | N/A | N/A | Shared admin state |
| `/admin/products` | Row skeletons | No products/matches | Filter normalization | Shared admin state | N/A | N/A | Results retry |
| `/admin/products/new` | Submit progress | N/A | Product fields | Shared admin state | N/A | Duplicate slug | Preserve values/retry |
| `/admin/products/:id` | Form skeleton | N/A | Product fields | Shared admin state | Missing product | Stale edit/slug | Preserve edits/retry |
| `/admin/orders` | Row skeletons | No orders/matches | Filter/date range | Shared admin state | N/A | N/A | Results retry |
| `/admin/orders/:id` | Detail skeleton | Optional event history only | Malformed ID to not found | Shared admin state | Missing order | N/A | Retry/back |

## Suggested implementation order

1. Build `AppRouter`, `SessionProvider`, the route guards, and the two shared layouts without page-specific business logic.
2. Build the shared state components and verify their keyboard, focus, and screen-reader behavior.
3. Implement public read-only pages: `/` and `/products/:slug`.
4. Implement sign-in and verify safe `returnTo` handling.
5. Implement customer pages: `/cart`, `/orders`, `/orders/:id`, then `/checkout/result`.
6. Implement the shared admin boundary and navigation.
7. Implement admin list pages, then product forms and detail pages.
8. Add browser tests for every row in the route-state coverage matrix that is not `N/A`.
9. Capture the desktop and mobile evidence required by [business-rules.md](./business-rules.md) and [routes.md](./routes.md).

The component names may change during implementation when real responsibilities become clearer. Any change must preserve the route access rules, state coverage, and business-rule enforcement boundaries.
