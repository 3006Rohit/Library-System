# AGENTS.md — Library Platform

This file is the standing instruction set for any AI coding agent working in this repo.
Read `docs/SPEC.md` before implementing a feature. If the spec and this file conflict, stop and ask.

## 1. What this repo contains

A multi-tenant library platform. Each library is a tenant with its own Salesforce org.

- `salesforce/` — Salesforce DX project. Librarian console, business logic (Apex), Agentforce agents, Member API endpoints.
- `apps/member-web/` — React (Vite) single-page app for guests and members.
- `apps/member-api/` — Node.js (TypeScript, Fastify) backend for the member side. Handles tenant routing, member authentication, the Salesforce gateway, PayU, and the agent gateway.
- `packages/contracts/` — OpenAPI document for the Member API and Apex REST endpoints, plus generated TypeScript types. This is the shared contract.
- `docs/` — `SPEC.md` (source of truth), `SECURITY.md` (security review checklist), `RUNBOOK.md`.
- `config/` — environment examples and scratch org definitions. No real secrets.

## 2. Commands

Salesforce (run from `salesforce/`):
- Create scratch org: `sf org create scratch -f config/project-scratch-def.json -a lib-dev -d -y 7`
- Deploy: `sf project deploy start -o lib-dev`
- Apex tests: `sf apex run test -o lib-dev -w 10 -r human -c`
- LWC tests: `npm run test:unit`
- Agent evals: `npm run agents:eval`

Member API (run from `apps/member-api/`):
- Dev server: `npm run dev`
- Unit and integration tests: `npm test`
- Type check and lint: `npm run typecheck && npm run lint`
- Contract tests: `npm run test:contract`

Member web (run from `apps/member-web/`):
- Dev server: `npm run dev`
- Tests: `npm test`
- Lint: `npm run lint`

Run the relevant tests after every meaningful change. Don't report a task as done until they pass.

## 3. Tenancy rules (critical)
- Every Apex service and REST endpoint derives the library and member from the authenticated context. Never trust a library ID or member ID sent by the client.
- Every Member API handler resolves the tenant from the request host and loads only that tenant's configuration and credentials.
- Never write a query that reads across tenants.
- Each Member API endpoint needs a test showing that a member in library A cannot access library B data, and a member cannot access another member's data in the same library.
- Tenant secrets are never returned by any endpoint and never logged.

## 4. Apex standards
- One trigger per object. Triggers contain no logic and delegate to a handler class.
- Layering: REST resource or invocable action → Service → Selector or Domain. Business rules live only in services and domain classes.
- Bulkify everything. No SOQL or DML inside loops. Test with 200+ records where relevant.
- Use `WITH USER_MODE` for queries and DML, and `with sharing` on service classes unless there is a documented reason otherwise.
- No hardcoded IDs, usernames, or URLs.
- Rule values come from `LibraryRule__c` through the rules service, never literals. Loans use their snapshot fields for their own calculations (see SPEC §8.3).
- Custom metadata is for deployment-time configuration only. Librarians edit rules at runtime, so rules never live in custom metadata.
- Throw typed exceptions from `lib-core`. Never swallow exceptions silently. Log through the shared logger.
- Any operation that changes copy availability or slot bookings must guard against double booking. Add a concurrency test that proves it.
- Use `@RestResource` endpoints under `/services/apexrest/library/v1/`. Every endpoint is listed in `packages/contracts` before it's implemented.

## 5. Node API standards (TypeScript)
- `strict` mode on. No `any`. No non-null assertions without a comment that explains why they're safe.
- Validate every input at the boundary with a schema (Zod or equivalent). Types come from the schema.
- Use parameterized queries only. For Postgres, use the project's query builder or ORM; never build SQL by string concatenation.
- Every Salesforce call has a timeout, a retry policy for idempotent operations only, and maps errors to the standard error envelope.
- Use the standard error envelope from `packages/contracts`. Don't expose stack traces or Salesforce internals to clients.
- Logs are structured (pino or equivalent) with request IDs and redaction rules for personal data and secrets.
- PayU code lives in one module. Signing and verification are unit-tested with PayU's documented test vectors. Never implement the hash formula from memory. Copy it from current PayU documentation.
- The integration user's access is limited to the endpoints it needs. Don't use it to access data outside the Member API's scope.
- Sessions are server-side in the Node API (BFF). Never put access or refresh tokens in responses, `localStorage`, `sessionStorage`, or logs. The browser receives only the HTTP-only session cookie.
- Session cookies use `__Host-` prefix, `Secure`, `HttpOnly`, and `SameSite`. State-changing requests require a CSRF token.

## 6. React standards
- Functional components and hooks only. Server state through a data-fetching library, not hand-rolled fetch logic in components.
- Types come from `packages/contracts`. Don't redefine API shapes in the frontend.
- Show errors in plain language. Never show internal IDs or stack traces to members.
- Guest pages show only the fields defined for guests. Never fetch member-only data for guest views.
- Each page has tests covering loading, empty, error, and success states.
- Accessibility: keyboard navigation, labels on all inputs, and WCAG 2.1 AA contrast.
- Payments: the React app never handles card data. It only redirects to the URL the Member API returns.

## 7. Lightning Web Components standards
- LWC calls Apex through the wire service or imperative calls in `lib-console`. It never queries objects directly in JavaScript.
- Each component has Jest tests for its states.
- Librarians see only what their permission set allows. Hide actions they can't perform and enforce the same check in Apex.

## 8. Agentforce standards
- Agents never perform writes directly. Every write is an Apex invocable action or Flow that calls the same service the UI uses.
- Every write asks for confirmation and states exactly what will change.
- Guest agent: read-only, public information only, no member context, no record IDs in responses.
- Member agent: each action calls the session-check endpoint before doing any work and acts only on the member it returns. Fail closed if the check fails or times out. Never accept a member ID from model output or agent input.
- Librarian agent: all actions require the librarian permission set. Waivers above `WAIVER_APPROVAL_AMOUNT` and rule changes require Library Admin approval.
- Agent instructions and topics are versioned in the repo. Changes require the evaluation suite to pass the thresholds in SPEC §10.7.
- Never put secrets, keys, or personal data in agent instructions or prompts.
- Verify current Agentforce APIs and pricing before implementing the agent gateway (SPEC T2).

## 9. Security and privacy
- Never commit credentials, keys, salts, tokens, `.env` files, or org-specific secrets. Use `.env.example` with placeholder values.
- Card data is never stored, logged, or displayed by any component. Payment references only.
- Personal data stays in Salesforce. The Member API stores only identity links and payment identifiers.
- Mask or omit personal data in logs, agent transcripts, and error messages.
- Login is Keycloak. Realm configuration (password policy, MFA, session lifetimes, brute-force settings) is versioned in `config/keycloak/`. Changes need review and a note in `docs/SECURITY.md`.
- Session-check and linking endpoints are security-critical. Changes to them need a security review and tests for expired, revoked, forged, and replayed inputs.
- Changes to sharing, field-level security, permission sets, or the integration user require a security checklist update in `docs/SECURITY.md`.

## 10. Testing standards
- Unit tests for every service, domain method, and Node module, covering the happy path, each business rule, and each failure path.
- Integration tests for Member API endpoints against a Salesforce scratch org or recorded fixtures.
- Contract tests confirm the Member API and Apex REST endpoints match the OpenAPI document.
- Tenant-isolation tests on every endpoint (see §3).
- Apex: use `@TestSetup` with data factories from `lib-core`. Never use `SeeAllData=true`. Coverage target 90%+ on `lib-lending`, `lib-fines`, and `lib-facilities`; 85% minimum overall for deploy.
- Tests assert outcomes, not just that code ran.
- Agents: evaluation sets in `salesforce/agents/evals/` must pass before release.

## 11. Git workflow
- Branch per feature. Commit after each working step with a clear message, e.g. `feat(lending): add FIFO reservation queue`.
- Every change goes through a pull request, even when working alone. The PR links the SPEC section it implements.
- Never force-push to `main`.
- Never commit secrets. If one is committed by accident, stop and tell the user immediately. Don't just delete it in a new commit.

## 12. Ask before you act
Stop and ask the user before:
- deleting or renaming any Salesforce object, field, or metadata type
- changing a permission set, sharing rule, or integration user permissions
- changing the OpenAPI contract in a breaking way
- adding a dependency, managed package, paid service, or external integration
- changing a default rule value or any open item (T2–T12) in SPEC §3
- changing Keycloak realm settings, session lifetimes, or the access rules of the session-check endpoint
- changing how payments, card handling, or personal data are processed
- deploying to any client org, pilot org, or production environment

## 13. Definition of done
- Behavior matches the acceptance criteria in SPEC §15.
- All relevant tests pass, coverage targets are met, and the deploy or build succeeds.
- Contract and tenant-isolation tests pass for any endpoint that changed.
- SPEC is updated if behavior changed. Code and spec must not drift.
- `docs/SECURITY.md` is updated if access or data handling changed.
