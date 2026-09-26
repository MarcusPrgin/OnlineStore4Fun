# TinyShop pre–Phase 0 setup checklist

This checklist gets the workstation, accounts, repository, and documentation ready without prematurely building the application. Phase 0 is design work: its purpose is to define the problem and system boundaries before choosing implementation details that are difficult to reverse.

## Current machine audit

Checked on September 26, 2026:

| Item | Current state | Action |
| --- | --- | --- |
| Repository | Cloned; `main` tracks `origin/main` | Ready |
| Git | Installed (`2.50.1`) | Ready |
| Node.js | Installed (`24.13.0`) | Ready for current Vite and Supabase CLI requirements |
| npm | Installed (`11.6.2`) | Ready; optional warning cleanup below |
| Docker CLI | Installed (`28.4.0`) | Start Docker Desktop and verify the engine |
| GitHub CLI | Installed (`2.101.0`) | Re-authenticate; the stored token is invalid |
| Supabase CLI | Not installed | Wait until the database/local-development phase |
| Stripe CLI | Not installed | Wait until the payments/webhook phase |
| AWS CLI | Not installed | Wait until the AWS deployment phase |

Versions are a snapshot, not permanent minimums. The lockfile and CI configuration created later will make the project reproducible.

## 1. Complete before starting Phase 0

### Repository safety

- [x] Create the GitHub repository.
- [x] Clone it locally.
- [ ] Confirm the remote and clean starting state:

  ```bash
  git remote -v
  git status
  ```

- [ ] Re-authenticate GitHub CLI so later pull-request and workflow commands work:

  ```bash
  gh auth login -h github.com
  gh auth status
  ```

- [ ] Decide whether `main` will require pull requests. This is optional for a solo learning repository, but feature branches are still useful.
- [ ] Create a branch for Phase 0 documentation rather than doing all work directly on `main`:

  ```bash
  git switch -c docs/phase-0-boundaries
  ```

- [ ] Make sure macOS, editor backups, or cloud-sync tools are not duplicating the repository while Git is writing files.

### Runtime and terminal

- [x] Confirm Node.js is available:

  ```bash
  node --version
  npm --version
  ```

- [ ] Add a Node version file when the first `package.json` is created. Use the same major version locally and in CI; do not silently switch runtimes between machines.
- [ ] Open Docker Desktop and verify that both the CLI and engine work:

  ```bash
  docker version
  docker run --rm hello-world
  ```

  Docker is not required for the written Phase 0 exercises, but verifying it now prevents a surprise when local Supabase or container work begins.

- [ ] Optional: remove the npm warning about `min-release-age`. npm currently reads that unsupported key from `~/.npmrc`. Preserve any registry authentication entries and remove only the `min-release-age` line, or run:

  ```bash
  npm config delete min-release-age --location=user
  ```

  This is housekeeping, not a blocker. Do not paste the contents of `.npmrc` into an issue, commit, or screenshot because it may contain registry credentials.

### Editor

- [ ] Open the cloned repository as the workspace root—not its parent folder.
- [ ] Confirm the integrated terminal opens inside the repository:

  ```bash
  pwd
  git status
  ```

- [ ] Install or enable these editor extensions if using VS Code:
  - ESLint
  - Prettier
  - Docker
  - Playwright Test for VS Code (can wait until the testing phase)
  - PostgreSQL or Supabase tooling only if it helps inspect data; it is not required
- [ ] Enable format-on-save only after the repository contains an agreed formatter configuration. Avoid editor-global rules that rewrite files differently from CI.

### Accounts—create access, not resources

- [ ] Confirm access to GitHub.
- [ ] Create or confirm a Supabase account. Do not create production data yet.
- [ ] Create or confirm a Stripe account and ensure test mode is available. Do not enter real customer/payment data.
- [ ] Create or confirm an AWS account only if ready to secure it immediately:
  - Enable root-account MFA.
  - Do not create access keys for the root user.
  - Do not provision App Runner, ECR, CloudFront, S3, or retained logs yet.
  - Budget creation comes before billable resources in the AWS phase.

### Phase 0 workspace

- [ ] Keep the TinyShop Project Brief and Kanban roadmap accessible in Obsidian.
- [ ] Create a repository `docs/` folder only when starting Phase 0, then store durable outputs there so architecture decisions are version-controlled.
- [ ] Plan these Phase 0 documents:
  - `docs/problem-and-scope.md`
  - `docs/customer-journey.md`
  - `docs/admin-journey.md`
  - `docs/business-rules.md`
  - `docs/system-boundary.md`
  - `docs/definition-of-done.md`
- [ ] Decide on a simple diagram format. Mermaid inside Markdown is enough; a specialized diagram application is optional.
- [ ] Read the README and restate the project goal in your own words before changing the stack or adding features.

## 2. Do not install application packages yet

Before Phase 0, you do **not** need `node_modules`, a root `package.json`, React, Fastify, Supabase packages, Stripe packages, testing libraries, or AWS SDK packages.

Reasons:

1. Phase 0 has no executable application requirements.
2. Creating the workspace structure is itself part of the learning work in the setup phase.
3. Installing everything at once makes it difficult to understand why a dependency exists.
4. Packages and generated configuration age quickly; install them when they are about to be used.
5. A smaller dependency tree reduces security and troubleshooting noise.

The pre–Phase 0 readiness test is therefore:

```bash
git status
node --version
npm --version
docker version
gh auth status
```

The repository should be clean after committing these documents, Node/npm should report versions, Docker Desktop should respond, and GitHub authentication should succeed.

## 3. Package installation map for later phases

These are planned groups, not one giant install command. Re-check official documentation and package compatibility when each phase begins.

### Application scaffolding phase

Create the frontend with the React + TypeScript Vite template:

```bash
npm create vite@latest apps/web -- --template react-ts
```

Initialize the API and install only its foundation:

```bash
mkdir -p apps/api
cd apps/api
npm init -y
npm install fastify
npm install --save-dev typescript @types/node tsx
```

Likely repository-wide development tools:

```bash
npm install --save-dev eslint prettier
```

Do not copy these commands blindly into an unexpected directory. Confirm the planned repository structure first.

### Frontend feature phases

Install packages when their first feature is implemented:

```bash
npm install react-router-dom @tanstack/react-query
npm install @mui/material @emotion/react @emotion/styled
npm install react-hook-form zod @hookform/resolvers
```

Potential additions should be justified before installation. For example, do not add Redux if TanStack Query plus local React state already solves the real state problem.

### API foundation phase

Likely Fastify plugins and validation/database tooling:

```bash
npm install @fastify/cors @fastify/helmet @fastify/rate-limit
npm install @fastify/swagger @fastify/swagger-ui
npm install zod
```

Choose the PostgreSQL access layer deliberately during database design. Do not install multiple competing ORMs/query builders. Options might include direct `pg`, a query builder, or an ORM; the schema and transaction requirements should drive that choice.

### Supabase phase

Prefer a project-pinned CLI rather than a machine-global npm install:

```bash
npm install --save-dev supabase
npx supabase --help
npx supabase init
```

Install the browser/server client where it is actually required:

```bash
npm install @supabase/supabase-js
```

Running the full local Supabase stack uses Docker and downloads several images, so confirm disk space and Docker health first. A hosted Supabase development project is an alternative if local-stack complexity would distract from the initial database learning goals.

### Stripe phase

Install the Node library in the API package:

```bash
npm install stripe
```

Install and authenticate the Stripe CLI on macOS when webhook development begins:

```bash
brew install stripe/stripe-cli/stripe
stripe login
stripe --version
```

Keep the secret key and webhook-signing secret out of Git. The browser return URL must never be treated as proof of payment.

### Testing phase

Frontend/unit and component testing:

```bash
npm install --save-dev vitest jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

End-to-end testing:

```bash
npm install --save-dev @playwright/test
npx playwright install
```

Fastify includes request-injection support; begin there before adding extra API-test libraries. Add a package only when a concrete testing gap requires it.

### AWS deployment phase

Install AWS CLI v2 from the official AWS installer when cloud deployment begins, then verify:

```bash
aws --version
```

Before creating any resource:

- Enable MFA and avoid root credentials.
- Create an AWS Budget and billing alerts.
- Choose one region intentionally.
- Decide how local and CI authentication will work without committing long-lived keys.
- Create a resource inventory and teardown checklist.

The application may not require an AWS SDK dependency at all if it does not call AWS APIs at runtime. Infrastructure being hosted on AWS is not, by itself, a reason to add `@aws-sdk/*` packages to the application.

## 4. Package-management rules

- [ ] Use one package manager consistently. This plan assumes npm.
- [ ] Commit the generated lockfile.
- [ ] Never commit `node_modules/`.
- [ ] Review install output instead of ignoring warnings.
- [ ] Run `npm audit` as a signal, then assess whether findings are reachable and relevant rather than applying destructive upgrades blindly.
- [ ] Pin the Node major version in the repository and CI.
- [ ] Install dependencies in the package that uses them; avoid turning every dependency into a root dependency.
- [ ] Remove packages that are no longer imported or justified.
- [ ] Do not install command-line tools globally through npm when the official project recommends a project-local dependency or dedicated installer.

## 5. Secrets checklist

- [ ] Add `.env*` exclusions before creating real environment files.
- [ ] Commit a safe `.env.example`, never `.env`.
- [ ] Use obviously fake placeholder values in documentation.
- [ ] Keep Supabase service-role credentials and Stripe secret keys server-side.
- [ ] Remember that Vite variables exposed to browser code are public, even if their names contain `KEY` or `SECRET`.
- [ ] Do not paste tokens into AI chats, screenshots, issues, README files, shell history, or frontend configuration.
- [ ] If a credential is exposed, revoke and rotate it; deleting the visible line is not enough.

## Ready to begin Phase 0 when

- [ ] The repository opens correctly and `git status` is understood.
- [ ] GitHub CLI authentication succeeds.
- [ ] Node and npm report expected versions.
- [ ] Docker Desktop can run `hello-world` (recommended now, strictly required later).
- [ ] Required accounts are accessible and secured, without unnecessary cloud resources.
- [ ] The README and this checklist are committed.
- [ ] The Obsidian roadmap and project brief are accessible.
- [ ] No application packages have been installed merely to make the repository look busy.
- [ ] You are ready to explain the problem and journeys before writing application code.

At that point, start Phase 0. The first technical deliverable is clarity, not a running server.
