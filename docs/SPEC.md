# Library Platform — Specification

Version: 0.3
Status: Decisions locked in §2. Items in §3 are open.

---

## 1. Purpose and scope

A library operations platform sold to individual libraries. Each library is a customer and gets its own Salesforce org.

- **Librarians** (internal users) run daily operations in a Salesforce Lightning console, including books, copies, seats, rooms, reservations, fines, memberships, and library rules.
- **Members** (external users) use a React web app backed by a Node.js API. They browse, borrow requests, reserve, renew, pay fines, manage memberships, and book study seats and rooms.
- **AI agents** run on Agentforce in each library's org: a guest agent, a member agent, and a librarian agent.

**In scope for v1:** catalog, lending, reservations, fines and PayU payments, memberships, study seat and room booking, member authentication, librarian console, library rule configuration, three agents, and tenant provisioning.

**Out of scope for v1:** e-books and digital lending, inter-library loans, multi-currency, native mobile apps, library-to-library integrations, and multi-branch within one library.

---

## 2. Locked decisions

| ID | Decision |
|---|---|
| D1 | One Salesforce org per library. The product is delivered as packages installed in each org. |
| D2 | Member frontend is React. |
| D3 | Member backend is Node.js with TypeScript. |
| D4 | Librarian side is Salesforce (Lightning console and LWC). |
| D5 | AI agents use Agentforce for all three agents. |
| D6 | Fine and membership payments use PayU hosted checkout. |
| D7 | Each library has one branch in v1. A `Branch` reference is kept on resource records for future use. |
| D8 | Librarians configure library rules in the console UI. Every change is validated and audited. |
| D9 | Experience Cloud is not used. Member login and member data access are handled outside Salesforce by the Node API. |
| D10 | Member login uses self-hosted Keycloak (open-source), with one realm per library. Members must use a passkey or TOTP. |
| D11 | The Node API acts as a backend-for-frontend. Browsers hold only an HTTP-only session cookie; access and refresh tokens stay on the server. |
| D12 | Agent actions verify the member's session through a Node API endpoint before doing any work. Revoking a session stops agent access immediately. |
| D13 | Librarians sign in through Salesforce SSO with their library's identity provider, with MFA enforced. |

---

## 3. Open items

| ID | Topic | Current assumption | Owner |
|---|---|---|---|
| T1 | Member authentication | **Closed: self-hosted Keycloak (D10).** Before build, verify Keycloak's current version, passkey support, and breached-password checking options. | Verify |
| T2 | Agentforce for external members | The Node API starts member agent sessions, and each action verifies the session through a Node endpoint (D12). Confirm against current Agentforce documentation: how a session reference is passed, whether agent actions can call external HTTPS endpoints, and licensing for external members. | Verify with Salesforce before build |
| T3 | Loan and fine default values | See §8.2 placeholders | You |
| T4 | Study booking limits | 30-minute slots, 2 hours max per booking, 2 bookings per day | You |
| T5 | Data retention for members who leave | Anonymize after a defined period; period TBD | You |
| T6 | Who pays Salesforce licenses per library | The library pays for its own org and users | Confirm with clients |
| T7 | Each library's PayU account | Each library has its own PayU merchant account, KYC, and settlement | Confirm with clients |
| T8 | Currency | INR | Confirm |
| T9 | Scale per library | Up to 10,000 members, 200,000 copies, 1,000 bookings per day | Confirm |
| T10 | Frontend rendering | Vite single-page app. Move to Next.js only if public catalog SEO needs server rendering. | Decide when catalog launches |
| T11 | Hosting for the Node API and Keycloak | Container hosting in an Indian region, on the lowest-cost plan that supports backups and managed TLS. Verify current free tiers and data location. | You |
| T12 | Auth timing values | Member idle timeout 30 minutes; absolute session limit 12 hours; agent session reference lifetime 15 minutes; one-time linking code lifetime 10 minutes. All proposed. | You |

---

## 4. Users and roles

| Role | Where | Access |
|---|---|---|
| Guest | React web app, anonymous | Limited catalog, opening hours, membership info, FAQs, guest agent |
| Member | React web app, authenticated | Own loans, reservations, fines, membership, bookings; full catalog; member agent |
| Librarian | Salesforce org, SSO | Console for all daily operations; librarian agent |
| Library Admin | Salesforce org, SSO | Everything a librarian can do, plus library rules, seats and rooms configuration, and user permissions |
| Platform Operator | Your company, cross-tenant | Tenant provisioning, package deployment, support. No access to member personal data without a logged, approved support ticket. |

---

## 5. Architecture

### 5.1 Components

| Component | Technology | Responsibility |
|---|---|---|
| Member web app | React (Vite), hosted on a static CDN | Member and guest UI; calls Member API only |
| Member API | Node.js, TypeScript, Fastify | Tenant resolution, authentication, session handling, Salesforce gateway, PayU integration, agent gateway, read caching, rate limiting |
| Tenant registry | PostgreSQL | Library records, org references, package version, status |
| Login service | Keycloak (self-hosted, PostgreSQL-backed), one realm per library | Member login, passkeys, TOTP, brute-force protection, password policy, token issuance |
| Session store | PostgreSQL | Server-side sessions for the backend-for-frontend: session ID, member ID, library, timestamps, revoked flag |
| Identity link store | PostgreSQL | Maps the login user to a member record in a library org; no personal details beyond identifiers |
| Payment ledger | PostgreSQL | PayU transaction IDs, status, verification results, reconciliation state |
| Secrets store | Cloud secrets manager | Connected App keys and PayU keys and salts, per library |
| Librarian console | Salesforce Lightning app, LWC | Librarian and Library Admin UI |
| Business logic | Apex services in the `lib-*` packages | All business rules; used by console, agents, and Member API |
| Member API endpoints | Apex REST, versioned under `/services/apexrest/library/v1/` | Scoped endpoints the Member API calls |
| Agents | Agentforce agents with Apex invocable actions and Flows | Guest, member, and librarian assistants |
| Scheduled work | Scheduled and Batch Apex in each org | Overdue checks, fine accrual, hold expiry, membership reminders, reconciliation support |
| Notifications | Salesforce email alerts for librarian-triggered events; Node email for authentication messages | Reminders, holds, confirmations |
| Observability | OpenTelemetry in Node; Apex structured logs in `LibraryLog__c`; Salesforce event monitoring as licensed | Tracing and support |

### 5.2 Request flows

**Member action (for example, reserve a book)**
React → Member API (verify session, resolve library, check rate limit) → Salesforce Apex REST endpoint (integration user via JWT bearer flow; member context passed) → Apex service (validates member scope and business rules) → Salesforce data → response back through Member API → React.

**Member agent conversation**
React chat → Member API (verify session, create an agent session record linked to the login session) → Agentforce agent session started with an opaque session reference → agent calls Apex actions → each action asks the Member API to check the reference before doing work → response back to React.

**Librarian action**
Librarian browser → Lightning console → LWC → Apex service. Same services as above.

**Fine payment**
React → Member API → PayU hosted checkout → PayU redirect and server callback → Member API verifies → Apex service marks fine paid. See §12.

### 5.3 Member API contract

- All Salesforce endpoints are versioned (`/v1/`) and described in an OpenAPI document in `packages/contracts`. The Node API and the Apex endpoints are generated or tested against this contract.
- Endpoints (initial set):

| Method | Path | Purpose |
|---|---|---|
| GET | `/catalog/search` | Search titles (guest: limited fields; member: full) |
| GET | `/catalog/books/{id}` | Book details and copy availability |
| GET | `/members/me` | Membership, status, limits |
| GET | `/loans` | Own loans |
| POST | `/loans/requests` | Borrow request for an available copy |
| POST | `/loans/{id}/renewals` | Renew a loan |
| GET | `/reservations` | Own reservations |
| POST | `/reservations` | Reserve a title |
| DELETE | `/reservations/{id}` | Cancel a reservation |
| GET | `/fines` | Own fines |
| POST | `/fines/{id}/payments` | Start a payment (creates pending record, returns signed PayU request) |
| GET | `/study-slots` | Available slots |
| POST | `/study-bookings` | Book a slot |
| DELETE | `/study-bookings/{id}` | Cancel a booking |
| GET | `/rules` | Public library rules (cached) |

- The Member API never forwards a member ID from the client as trusted input. The member identity comes from the verified session.

---

## 6. Multi-tenancy

- **Isolation boundary:** the Salesforce org. A library's data never shares storage with another library.
- **Node API is multi-tenant.** Every request resolves the tenant from the request host, for example `{library-slug}.{product-domain}`, then loads only that tenant's configuration and credentials. No code path queries across tenants.
- **Tenant credentials** (Connected App private key, integration user, PayU merchant key and salt) live in the secrets store, keyed by library, and are never returned by any API.
- **Provisioning steps (per library):**
  1. Create the library's Salesforce org (or use one the client provides).
  2. Install the packages at a pinned version.
  3. Create the integration user with the restricted API permission set.
  4. Create the Connected App and register the certificate.
  5. Add the tenant to the registry and store its secrets.
  6. Seed default library rules and membership plans.
  7. Configure the PayU merchant reference.
  8. Run the smoke test suite against the tenant.
- **Upgrades:** packages are upgraded per library with a pilot library first, then staged rollout. Each upgrade runs the smoke suite.
- **Tenant-isolation tests:** for each Member API endpoint, a test confirms a member in library A cannot read or change library B data, and a member cannot read another member's records within the same library.

---

## 7. Data model

### 7.1 Salesforce (per library org)

| Object | Purpose | Key fields | Notes |
|---|---|---|---|
| `LibraryProfile__c` | Single record for the library | Name, address, contact, branding, PayU merchant reference | Reference only, not the secret |
| `Book__c` | Title | Title, authors, ISBN, category, language, replacement cost | |
| `BookCopy__c` | Physical copy | Book, barcode (unique), status (Available, OnLoan, Reserved, Lost, Maintenance) | Availability is driven by this object |
| `Member__c` | Member record | Name, contact details, status, plan, membership expiry, `ExternalIdentityId__c` (unique) | Custom object. Login credentials are never stored here. |
| `MembershipPlan__c` | Plan rules | Name, fee, max loans, active flag | Editable by librarians |
| `LibraryRule__c` | Current value of each rule | Rule key (unique per library), value, updated by, updated at | Edited through the console |
| `RuleChangeLog__c` | Audit of rule changes | Rule key, old value, new value, reason (required), changed by, changed at | Cannot be edited or deleted |
| `Loan__c` | Checkout | Member, copy, checkout date, due date, return date, renewal count, status, snapshot fields | Snapshot fields are described in §8.3 |
| `Reservation__c` | Hold request | Member, book, queue position, status, expires at | FIFO queue per book |
| `Fine__c` | Charge | Member, loan, amount, reason, status (Open, Pending Payment, Paid, Waived), waived by | |
| `Payment__c` | Payment reference | Fine, amount, PayU transaction ID (unique), status, PayU reference | No card data |
| `Seat__c` | Study seat | Label, zone, active | |
| `Room__c` | Study room | Name, capacity, active | |
| `StudySlot__c` | Bookable slot | Resource (seat or room), start, end, unique key | Unique key prevents double booking |
| `StudyBooking__c` | Member booking | Member, slot, status, checked-in at | |
| `LibraryLog__c` | Support log | Level, source, message, request ID | No personal data |

### 7.2 Node API (PostgreSQL)

| Table | Purpose | Notes |
|---|---|---|
| `tenants` | Library registry | Slug, org instance URL, org ID, package version, status |
| `identity_links` | Login user to `Member__c` ID | Identifiers only; unique per library |
| `sessions` | Server-side member sessions (BFF) | Session ID hash, member ID, library, created, last seen, expires, revoked |
| `agent_sessions` | Agent session references | Reference hash, linked session, expiry, revoked flag |
| `linking_challenges` | One-time codes for account linking | Code hash, member ID, attempts, expires, used flag |
| `payment_transactions` | PayU ledger | Transaction ID, amount, status, verification result, timestamps, raw response stored encrypted |
| `audit_events` | Security-relevant events | Login, payment, support access |
| `rate_limits` or cache | Request limiting and catalog cache | Use Redis if traffic requires |

**Data rules**
- Personal details live in Salesforce, the system of record. The Node API stores only what it needs for authentication and payment.
- Records use external IDs (barcode, ISBN, identity ID, PayU transaction ID, slot key) so imports and retries are safe.
- Deleting a member is a process: anonymize per T5, keep financial records as required by law.

---

## 8. Library rules

### 8.1 Design

- Rules are per library and stored in `LibraryRule__c`, not in custom metadata, because librarians edit them at runtime.
- Each rule has a definition in Apex: key, label, data type, unit, minimum, maximum, and default. The UI uses these definitions to validate input.
- Every change writes a `RuleChangeLog__c` entry. A reason is required.
- Changing a rule affects new transactions only. Existing loans and fines keep the values they were created with (see §8.3).
- Only Library Admins can change rules. A librarian can view them.
- The Node API reads rules through `GET /rules`. The response is cached for five minutes and cleared when a rule changes, using a platform event.

### 8.2 Rule catalogue (defaults are placeholders until confirmed)

| Key | Meaning | Default | Allowed range |
|---|---|---|---|
| `LOAN_DAYS` | Loan period | 14 | 1–90 |
| `MAX_ACTIVE_LOANS` | Loans per member (overridable per plan) | 5 | 1–50 |
| `MAX_RENEWALS` | Renewals per loan | 2 | 0–5 |
| `HOLD_HOURS` | Time to collect a held copy | 48 | 12–168 |
| `GRACE_DAYS` | Days after due date before fines start | 0 | 0–14 |
| `FINE_PER_DAY` | Fine per overdue day | TBD | ≥ 0 |
| `FINE_CAP_PERCENT` | Fine cap as a percent of replacement cost | 100 | 0–100 |
| `FINE_BLOCK_AMOUNT` | Open fines above this block borrowing and reserving | TBD | ≥ 0 |
| `WAIVER_APPROVAL_AMOUNT` | Waivers above this need a second Library Admin | TBD | ≥ 0 |
| `SLOT_MINUTES` | Study slot length | 30 | 15, 30, or 60 |
| `MAX_BOOKING_MINUTES` | Max length of one booking | 120 | 30–240 |
| `MAX_BOOKINGS_PER_DAY` | Study bookings per member per day | 2 | 1–10 |
| `MEMBERSHIP_REMINDER_DAYS` | Days before expiry to send a reminder | 14 | 0–60 |

### 8.3 Snapshot rule

A loan copies these values when it is created: `LOAN_DAYS`, `MAX_RENEWALS`, `GRACE_DAYS`, `FINE_PER_DAY`, and `FINE_CAP_PERCENT`. Fines calculate from the loan's snapshot. Changing a rule later never changes an existing loan or fine.

---

## 9. Functional requirements

### 9.1 Catalog
- FR-C1: Guests see title, author, category, and general availability only.
- FR-C2: Members see full details and copy availability for the library.
- FR-C3: Search by title, author, ISBN, and category, with pagination.
- FR-C4: Librarians manage books and copies. CSV import validates every row and reports errors per row.

### 9.2 Accounts and authentication
- FR-A1: Members register with a verified email. A passkey or TOTP is required before any personal action.
- FR-A2: Members reset passwords through Keycloak's verified-email flow. Reset messages never reveal whether an account exists.
- FR-A3: Librarians sign in through Salesforce SSO with MFA enforced. Access is controlled by permission sets.
- FR-A4: Member sessions expire after the idle and absolute limits in T12. Members can sign out everywhere, and support staff can revoke sessions.
- FR-A5: A login is linked to a library member record only through a librarian invitation, or a membership number plus a one-time code sent to the contact on file. Conflicting links go to librarian approval. Matching email alone never links.
- FR-A6: A librarian's access is removed on their last day, through SCIM or the documented offboarding step.
- FR-A7: Login and code attempts are rate-limited. Lockouts and repeated failures are logged without personal data.

### 9.3 Lending
- FR-L1: A member requests a physical copy. The request passes only if BR-01, BR-02, and BR-07 hold.
- FR-L2: A librarian confirms checkout. This creates the loan, sets the due date from the snapshot, and records the copy as on loan.
- FR-L3: A return closes the loan, checks reservations (BR-04), and assesses any fine (BR-06).
- FR-L4: Members renew online when BR-03 holds.

### 9.4 Reservations
- FR-R1: Members reserve titles with no available copy. Reservations join a FIFO queue.
- FR-R2: Members see queue position and cancel reservations.
- FR-R3: Expired holds advance the queue and notify the next member.

### 9.5 Fines and payments
- FR-F1: Members see open fines, reasons, and amounts.
- FR-F2: Members pay through PayU (§12).
- FR-F3: Librarians waive fines. Waivers above `WAIVER_APPROVAL_AMOUNT` need a second Library Admin, and every waiver is logged.
- FR-F4: A daily job accrues fines for overdue loans.

### 9.6 Memberships
- FR-M1: Members view and renew their membership and pay the plan fee through PayU.
- FR-M2: Librarians create, change, and suspend memberships and plans.
- FR-M3: Reminders send `MEMBERSHIP_REMINDER_DAYS` before expiry.

### 9.7 Study seats and rooms
- FR-S1: Members see available slots and book within the rule limits.
- FR-S2: Members cancel bookings before the start time.
- FR-S3: Librarians manage seats and rooms and view the day's schedule.
- FR-S4: Check-in records attendance.

### 9.8 Librarian console
- FR-K1: Member lookup showing loans, fines, reservations, membership, and bookings.
- FR-K2: Checkout, return, renew, and reserve-on-behalf actions.
- FR-K3: Reservation queue management and manual hold release.
- FR-K4: Fine management and waiver approval queue.
- FR-K5: Membership and plan management.
- FR-K6: Seat and room management and daily schedule view.
- FR-K7: Reports: circulation, overdue list, fines collected, seat utilization.
- FR-K8 (new): **Library Rules.** Library Admins view and edit rules inside the console, with validation and a required reason for each change. The audit history is visible to Library Admins.

### 9.9 Agents
See §10.

---

## 10. Agents (Agentforce)

### 10.1 Agent topology

| Agent | Used by | Surface | Context |
|---|---|---|---|
| Guest agent | Anonymous visitors | React app, via Member API | No member context. Read-only. |
| Member agent | Logged-in members | React app, via Member API | Session reference, checked on every action |
| Librarian agent | Librarians and Library Admins | Lightning console | Salesforce user context |

Each agent has its own topics, instructions, and actions. Actions are Apex invocable methods or Flows that call the same services as the UI.

### 10.2 Member context for the member agent

- When a member starts an agent session, the Member API creates an agent session record linked to the member's login session. It passes the agent only an opaque reference, never a token.
- Before doing any work, each Apex action calls the Member API's session-check endpoint with that reference. The endpoint returns the member ID, library, and scope only if the login session is valid and not revoked. Otherwise the action fails closed.
- The session-check call is authenticated with a per-library signing key, uses a timeout, and is accepted only from the Salesforce org's known outbound addresses (see `docs/SECURITY.md`).
- Agent session references expire after the lifetime in T12.
- Actions never take a member ID from the model's output. The member ID comes only from the session-check response.
- Revoking a login session ends agent access immediately.
- How the reference is passed into an Agentforce session, and whether actions can make this callout, must be confirmed against current Agentforce documentation (see T2) before build.

### 10.3 Guest agent capabilities

| Capability | Allowed |
|---|---|
| FAQs, opening hours, membership types, how to join | Yes |
| Catalog search with general availability | Yes |
| Personal data, loans, fines, bookings | No |
| Any writes | No |
| Direct the visitor to sign in for personal actions | Yes |

### 10.4 Member agent capabilities

| Capability | Allowed | Confirmation needed |
|---|---|---|
| Everything in the guest agent, with full catalog details | Yes | No |
| Show own loans, reservations, fines, membership, bookings | Yes | No |
| Request a borrow of an available copy | Yes | Yes |
| Reserve a title | Yes | Yes |
| Renew a loan | Yes | Yes |
| Cancel a reservation or booking | Yes | Yes |
| Book a seat or room | Yes | Yes |
| Start a fine or membership payment | Yes | Yes (payment happens on PayU's page) |
| Act on another member's records | No | Not allowed |

### 10.5 Librarian agent capabilities

| Capability | Allowed | Confirmation | Permission |
|---|---|---|---|
| Look up members, books, copies, bookings | Yes | No | Librarian |
| Checkout, return, renew, reserve on behalf | Yes | Yes | Librarian |
| Release a hold or reorder a queue | Yes | Yes | Librarian |
| Waive a fine below `WAIVER_APPROVAL_AMOUNT` | Yes | Yes | Librarian |
| Waive a fine above `WAIVER_APPROVAL_AMOUNT` | Queued for approval | Yes | Librarian, then Library Admin approval |
| Create or change memberships and plans | Yes | Yes | Librarian |
| Change library rules | Yes | Yes, with reason | Library Admin |
| Create or deactivate seats and rooms | Yes | Yes | Library Admin |
| Standard reports | Yes | No | Librarian |

### 10.6 Agent rules common to all agents
- Every write asks for confirmation and shows exactly what will change.
- The agent states clearly when it can't do something and offers the next step the user can take.
- Agents never display or request card data, passwords, or identity provider tokens.
- Agent transcripts are retained according to T5 and contain no more personal data than necessary.

### 10.7 Quality and cost
- Each agent has an evaluation set in `salesforce/agents/evals/` with at least 50 scenarios, including permission boundaries: for example, a guest asking for another member's loans, or a member asking to waive a fine.
- Release thresholds (set before launch): 100% pass on permission and security scenarios; at least 90% on task scenarios (TBD).
- Cost budget per agent conversation and per member per month is set before launch (TBD), after checking current Agentforce pricing.
- Rate limits apply in the Member API for the guest agent to protect cost.

---

## 11. Non-functional requirements

| Area | Requirement |
|---|---|
| Security | Least-privilege integration user; field-level and sharing rules reviewed each release; secrets never in code or logs; OWASP ASVS review before launch; controls and tests listed in `docs/SECURITY.md` |
| Privacy | Personal data minimized; retention per T5; data subject requests supported |
| Performance | Catalog search p95 under 1.5 s; member pages p95 under 2 s; first agent response under 4 s (targets TBD) |
| Scalability | Node API stateless and horizontally scalable; catalog cached; Salesforce work done in bulk-safe code; Salesforce API usage monitored against the org's limits per library |
| Reliability | Scheduled jobs are idempotent; external calls have timeouts and retries; clear messages when PayU or Salesforce is unavailable |
| Maintainability | Monorepo for member apps; separate Salesforce DX project; contracts shared through OpenAPI; test coverage targets; no hardcoded values |
| Observability | Tracing across React, Node, Salesforce, and agent calls; alerts on payment failures, error rates, and API limits |
| Accessibility | WCAG 2.1 AA for the React app and the console |
| Compliance | Local data protection law applies (T5); audit trail for rule changes, waivers, and permission changes |
| Availability | Member API and React app targets TBD; Salesforce availability follows the platform's service commitments |

---

## 12. PayU payments

### 12.1 Flow

1. The member chooses a fine or membership payment in the React app.
2. The Member API asks Salesforce to create a `Payment__c` in Pending status, with a unique transaction ID and the verified amount from the fine or plan.
3. The Member API builds the signed PayU request using the library's merchant key and salt. The member is sent to PayU's hosted checkout. **Card details are entered only on PayU's page.**
4. PayU redirects the member to a success or failure URL on the Member API.
5. The Member API verifies the response signature and calls PayU's server-side verification. It does not trust the redirect alone.
6. PayU's server callback is processed the same way. Processing is idempotent, keyed by transaction ID.
7. Only after verification does the Member API tell Salesforce to mark the payment and fine or membership as paid. Salesforce records the PayU reference.
8. A daily reconciliation job compares PayU's transaction reports with `Payment__c` and flags any mismatch for a Library Admin.

### 12.2 Rules
- The exact hash formula, field names, and endpoints must be copied from PayU's current integration documentation. Test against PayU's test environment before any live merchant is enabled.
- Each library's merchant key and salt are stored only in the secrets store.
- Refunds are separate records. Refund steps are defined before launch.
- A payment amount must match the fine or plan amount on the server. The client's amount is never trusted.

---

## 13. Integrations

| System | Purpose | Method | Failure behavior |
|---|---|---|---|
| Salesforce (per library org) | System of record and business logic | REST via Connected App, JWT bearer flow for the integration user | Retry with backoff; member sees a clear message; no partial writes |
| Agentforce | Agent runtime | Agent API from Member API and console | Agent unavailable message; form-based fallback for core actions |
| Keycloak (D10) | Member login, passkeys, TOTP, sessions | OIDC authorization code flow; Node API is a confidential client | Login unavailable message; no bypass; sessions fail closed |
| Librarian identity provider (D13) | Librarian login | Salesforce SSO (SAML or OIDC) | Clear message; no shared accounts |
| PayU | Payments | Hosted checkout, signed requests, server verification, callbacks | Payment stays pending; member can retry; reconciliation covers gaps |
| Email provider | Authentication messages from Node | SMTP or API | Retry; failure logged |

---

## 14. Environments and release

| Environment | Purpose | Source |
|---|---|---|
| Local and scratch orgs | Development and unit tests | Feature branches |
| Shared dev org | Integration tests | Merge to `develop` |
| Pilot library org | First real customer, staged rollout | Release candidate |
| Client orgs | Production per library | Pinned package versions after pilot |

- CI runs on every pull request: lint, Apex tests, LWC tests, Node unit and integration tests, React tests, contract tests against the OpenAPI document, and a scratch org deploy.
- Production package upgrades require manual approval and a rollback plan.

---

## 15. Acceptance criteria by epic

| Epic | Done when |
|---|---|
| Tenant provisioning | A new library can be provisioned from the checklist in §6 and passes the smoke suite |
| Tenant isolation | Tests show no cross-library or cross-member data access on every Member API endpoint |
| Catalog | Guests see only limited fields; members see full details; search meets its target |
| Lending | BR-01 to BR-04 are covered by tests; checkout and return work end to end |
| Reservations | FIFO order holds under concurrent requests; expired holds advance the queue |
| Rules | Librarian and Admin permissions are enforced; every change is logged; loans keep their snapshot |
| Fines and payments | PayU flow passes in the test environment; duplicate callbacks are safe; reconciliation flags mismatches |
| Memberships | Expiry blocks new loans and bookings; reminders send on schedule |
| Facilities | No double booking under concurrent requests; check-in recorded |
| Authentication | Passkey or TOTP enforced for members; linking rules (FR-A5) pass tests; SSO works for librarians; session revocation takes effect immediately; session-check rejects expired, revoked, and forged references |
| Console | Librarians complete all FR-K items in UAT without developer help |
| Agents | Evaluation set passes the thresholds in §10.7; all permission scenarios pass |
| Release | Security review signed off; coverage targets met; runbook written |

---

## 16. Glossary

- **Tenant:** one library, with its own Salesforce org and configuration.
- **Copy:** a physical item identified by barcode.
- **Hold:** a copy reserved for a member waiting to collect it.
- **Snapshot:** the rule values stored on a loan when it's created.
- **Session reference:** an opaque, short-lived ID an agent passes to the session-check endpoint. The endpoint confirms which member it belongs to.
- **Backend-for-frontend (BFF):** the Node API that holds the login tokens and gives the browser only a session cookie.
