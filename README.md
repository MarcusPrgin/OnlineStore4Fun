# ONLINE STORE FULLSTACK APP

TinyShop is a deliberately scoped full-stack e-commerce application built to strengthen practical understanding of frontend development, backend APIs, relational data modelling, authentication, payments, webhooks, testing, containers, cloud deployment, networking, and operations.

This is primarily a learning project. The goal is not to create the largest possible store; it is to understand and be able to explain the complete lifecycle of a reliable transaction—from a browser request to an authenticated API call, database transaction, Stripe webhook, deployed service, and production log.

## Project status

**Current stage:** Phase 0 — establish the problem and boundaries.

The repository has been created and cloned. Application packages have intentionally not been installed yet. The product journeys, business rules, system boundaries, and evidence of completion will be defined before implementation begins.

See [PRE_PHASE_0_SETUP.md](PRE_PHASE_0_SETUP.md) for the environment readiness checklist.

## Canonical customer journey

The finished application should allow a user to:

1. Register and sign in.
2. Browse visible, in-stock products.
3. Add products to a server-validated cart.
4. Begin checkout using prices and inventory recalculated by the backend.
5. Pay with Stripe Checkout in test mode.
6. Have the order marked paid only after a verified, idempotently processed Stripe webhook.
7. Return later and view an accurate historical order.

An administrator should be able to manage products and inventory, inspect paid orders, and move orders through valid fulfilment states.

## Learning objectives

- Build a typed React frontend without hiding networking and state-management decisions behind a large framework.
- Design a Node.js API with intentional endpoints, validation, authentication, authorization, logging, and error handling.
- Model relational data and enforce invariants with PostgreSQL constraints and transactions.
- Understand browser-to-API networking, HTTP, CORS, tokens, request lifecycles, retries, and failure states.
- Integrate Stripe Checkout and treat webhooks—not browser redirects—as the source of payment truth.
- Test business rules, API boundaries, database behaviour, webhook idempotency, and one end-to-end customer journey.
- Package the API with Docker and deploy the system to AWS with cost controls, logs, health checks, and a teardown plan.

## Planned architecture

```text
Browser
  └── React + TypeScript SPA
        ├── Supabase Auth (sign-in/session)
        └── HTTPS requests with access token
              └── Node.js + Fastify API
                    ├── PostgreSQL on Supabase
                    ├── Stripe Checkout API
                    └── Stripe webhook receiver

Production hosting
  ├── Frontend: S3 + CloudFront
  ├── API image: ECR
  ├── API runtime: App Runner
  ├── Secrets: AWS-managed secret storage
  └── Logs/operational evidence: CloudWatch and service logs
```

The browser may handle presentation and session UX, but it must not be trusted to choose prices, payment state, inventory truth, ownership, or administrator permissions.

## Planned technology stack

| Area | Technology | Purpose |
| --- | --- | --- |
| Frontend | React, TypeScript, Vite | Typed single-page application |
| UI and routing | Material UI, React Router | Accessible components and navigation |
| Server state | TanStack Query | Fetching, caching, retries, and invalidation |
| Forms and validation | React Hook Form, Zod | Form state and shared validation concepts |
| Backend | Node.js, Fastify, TypeScript | HTTP API and business rules |
| Database and auth | Supabase PostgreSQL and Auth | Relational persistence and user identity |
| Payments | Stripe Checkout and webhooks | Test-mode checkout and asynchronous payment confirmation |
| Testing | Vitest, Testing Library, Fastify tests, Playwright | Unit, integration, component, and end-to-end coverage |
| Packaging | Docker | Reproducible API runtime |
| Cloud | AWS ECR, App Runner, S3, CloudFront, CloudWatch | Image storage, API hosting, static hosting, CDN, and operations |
| Automation | GitHub Actions | Type-checking, tests, and production builds |

The stack is planned, not yet installed. Dependencies will be added only when the corresponding phase begins and their role is understood.

## Scope

### In scope

- Customer registration, sign-in, and session restoration
- Public catalogue and product details
- Authenticated cart
- Server-calculated totals and inventory checks
- Stripe test checkout
- Verified and idempotent webhook processing
- Customer order history and details
- Administrator product, inventory, fulfilment, and test-refund workflows
- Tests, CI, Docker packaging, AWS deployment, monitoring evidence, and teardown

### Deliberately out of scope initially

- Marketplace or multiple sellers
- Real-money production payments
- Shipping-carrier integrations
- Tax engine
- Promotions, recommendations, wish lists, and reviews
- Multiple currencies or locales
- Native mobile application
- Microservices, Kubernetes, or event-streaming infrastructure

Optional features should not be added until the canonical journey and definition of done are complete.

## Planned repository structure

The exact structure will be decided during the setup phase. The expected direction is:

```text
tinyshop/
├── apps/
│   ├── web/             # React application
│   └── api/             # Fastify API
├── packages/
│   └── contracts/       # Optional shared schemas/types
├── supabase/
│   ├── migrations/
│   └── seed.sql
├── docs/                # Architecture, journeys, diagrams, runbooks
├── .github/workflows/   # CI
├── README.md
└── PRE_PHASE_0_SETUP.md
```

This is a target, not a commitment. Phase 0 should clarify the system before folders are scaffolded.

## Roadmap

1. Define the problem, journeys, boundaries, and evidence of completion.
2. Design screens, states, navigation, and order transitions.
3. Scaffold the frontend, API, tooling, and first browser-to-server request.
4. Model PostgreSQL data, constraints, migrations, seeds, and transactions.
5. Build the API foundation, catalogue, authentication, cart, and orders.
6. Integrate Stripe Checkout and idempotent webhooks.
7. Add administrator workflows and harden the user experience.
8. Add unit, integration, database, webhook, component, and end-to-end tests.
9. Containerize the API and deploy the system to AWS.
10. Perform security, reliability, observability, documentation, and teardown exercises.

The detailed working roadmap lives in the TinyShop Obsidian Kanban board.

## Local development

Local development commands will be added when the applications are scaffolded. Eventually, a new contributor should be able to run something similar to:

```bash
npm install
npm run dev
npm run typecheck
npm test
npm run build
```

These commands do not exist yet and should not be documented as working until they are implemented and verified.

## Environment variables

Real credentials must never be committed. A future `.env.example` will document names and safe placeholders for values such as:

- Frontend API URL and public Supabase configuration
- API database/Supabase configuration
- Stripe secret key and webhook-signing secret
- Allowed frontend origin
- Runtime environment and port

Secrets that grant backend or infrastructure access must never be included in frontend bundles.

## Definition of done

TinyShop is complete when:

- A new user can register, authenticate, and buy an in-stock product with a Stripe test card.
- The order becomes paid only after a verified webhook is processed safely, including duplicate delivery.
- The customer can later view the historical purchase.
- An administrator can manage inventory and fulfil the order.
- Important business rules are protected by meaningful tests.
- The system has run locally and in AWS.
- Costs, secrets, logs, documentation, operational failure cases, and cloud teardown have been reviewed.
- The complete request, checkout, and webhook lifecycles can be explained without reading the implementation.

## Security and cost principles

- Never commit secrets or copy them into screenshots, logs, container layers, or frontend assets.
- Treat browser input, route guards, return URLs, and webhook payloads as untrusted until verified.
- Use database constraints and transactions for invariants that must survive concurrent requests.
- Use test-mode Stripe data only.
- Create AWS budgets before provisioning billable resources.
- Keep a resource inventory and teardown checklist from the first cloud deployment.

## License

No license has been selected yet. Until one is added, the source remains under the repository owner's default copyright.
