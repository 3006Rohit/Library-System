# Security Controls and Review Checklist — Library Platform

Status: v0.1. Every control below needs an owner, an automated test where practical, and sign-off before the first real library goes live.
Source of design: `docs/SPEC.md` §5, §6, §10.2, and §11. Decisions D10–D13.

---

## 1. Assets

| Asset | Where | Why it matters |
|---|---|---|
| Member personal data | Each library's Salesforce org | Privacy law; trust |
| Member login credentials and MFA factors | Keycloak (per-library realm) | Account takeover |
| Session records and tokens | Node API (PostgreSQL), Keycloak | Impersonation |
| PayU merchant keys and salts | Secrets store, per library | Fraudulent payments; payment disputes |
| Connected App private keys | Secrets store, per library | Full API access to a library's org |
| Agent session references and signing keys | Node API and secrets store | Agent acting as a member |
| Payment records | Node API ledger and Salesforce | Financial integrity |

## 2. Threat actors (summary)

- Anonymous internet attackers: credential stuffing, scanning, abuse of the guest agent.
- A registered member trying to reach another member's data (IDOR, prompt injection into the agent).
- A malicious or former librarian.
- Compromised dependency or hosting account.
- A stranger claiming another member's record during account linking.
- Platform operator misuse of support access.

## 3. Controls

Each control has an ID, the requirement, how it's enforced, and the test that proves it.

### 3.1 Browser session (BFF)

| ID | Control | Enforcement | Test |
|---|---|---|---|
| SEC-01 | Session cookie is `__Host-` prefixed, `Secure`, `HttpOnly`, `SameSite=Lax` or stricter | Node API cookie config | Test asserts cookie attributes on login response |
| SEC-02 | No access or refresh tokens in any browser-visible response, storage, or log | Token store is server-side only | Test scans all API responses and client bundle for token patterns |
| SEC-03 | CSRF token required on all state-changing requests | Synchronizer token tied to session | Test: POST without or with wrong token is rejected |
| SEC-04 | Strict Content Security Policy: no inline scripts, `script-src 'self'`, no `eval` | Headers set by Node API and CDN | Test asserts CSP header; browser check in E2E tests |
| SEC-05 | Sessions expire on idle and absolute limits (T12); member can sign out everywhere | Session store with expiry and revoked flag | Tests for idle expiry, absolute expiry, and revocation |

### 3.2 Login (Keycloak)

| ID | Control | Enforcement | Test |
|---|---|---|---|
| SEC-06 | Passkey or TOTP required before personal actions | Keycloak required-action policy per realm | Test: member without second factor cannot reach member routes |
| SEC-07 | Verified email required before account use | Keycloak registration flow | Test: unverified account cannot link or log in to member area |
| SEC-08 | Brute-force detection and lockout enabled | Keycloak realm setting, versioned in `config/keycloak/` | Test: repeated failures trigger lockout |
| SEC-09 | Password policy: minimum length, breached-password check where available | Keycloak policy; breach check via plugin or provider feature (verify) | Test: weak and breached passwords rejected |
| SEC-10 | Password reset does not reveal whether an account exists | Keycloak reset flow with generic message | Test: same response for known and unknown emails |
| SEC-11 | Realm per library; no cross-realm trust | Realm configuration and tenant registry | Test: token from realm A rejected by library B's API |
| SEC-12 | Keycloak runs on supported, patched versions | Patch within 14 days of a security release (proposed) | Checklist item each release |

### 3.3 Tenancy and member scope

| ID | Control | Enforcement | Test |
|---|---|---|---|
| SEC-13 | Tenant resolved only from an allowlist of library domains; unknown hosts rejected | Node API host allowlist; forwarded-host headers ignored unless from the trusted proxy | Test: spoofed Host header rejected |
| SEC-14 | Every Node endpoint loads only the resolved tenant's configuration | Tenant context object required by all handlers | Test per endpoint: library A token on library B host is rejected |
| SEC-15 | Every Apex endpoint checks member scope through one shared function | Shared `MemberScope` check in `lib-core`; endpoints must call it | Test per endpoint: member A cannot read or change member B records |
| SEC-16 | Member identity comes only from the verified session, never from request input | Handler design; schema excludes member ID fields on member routes | Test: member ID in body or query is ignored or rejected |

### 3.4 Salesforce integration

| ID | Control | Enforcement | Test |
|---|---|---|---|
| SEC-17 | Integration user has only the permission set needed by Member API endpoints | Permission set reviewed each release | Review checklist; automated permission report in CI |
| SEC-18 | Connected App uses JWT bearer with a certificate; key stored in the secrets store | Provisioning checklist | Provisioning smoke test |
| SEC-19 | Connected App restricted to Node API outbound IP ranges where possible | Connected App IP restrictions | Provisioning smoke test |
| SEC-20 | Access tokens are short-lived and never persisted outside memory | Token cache in memory only | Test: no token written to database or logs |
| SEC-21 | Apex services run `with sharing` and `WITH USER_MODE` | Code standard in AGENTS.md | Static analysis in CI (PMD or equivalent, verify) |

### 3.5 Agents

| ID | Control | Enforcement | Test |
|---|---|---|---|
| SEC-22 | Agents receive only an opaque session reference, never a token | Node API design | Test: agent input payload contains no token fields |
| SEC-23 | Each action calls session-check before any work; fails closed on error or timeout | Shared action base class in `lib-agents` | Tests for expired, revoked, forged, and unreachable session-check |
| SEC-24 | Session-check endpoint accepts requests only from the org's known outbound addresses and a valid per-library signature | Node API middleware and secrets store | Tests for missing signature, wrong library key, and wrong source |
| SEC-25 | Agent session references expire (T12) and are single-library | `agent_sessions` table | Test: reference from library A rejected by library B |
| SEC-26 | Model output never selects member or library identity | Actions read identity only from session-check response | Eval scenarios with prompt-injection attempts (`salesforce/agents/evals/`) |
| SEC-27 | Guest agent rate-limited per IP and per session | Node API rate limiter | Load test and rate-limit tests |

### 3.6 Account linking

| ID | Control | Enforcement | Test |
|---|---|---|---|
| SEC-28 | Linking only by librarian invitation, or membership number plus one-time code to the contact on file | Linking service | Tests for each linking path |
| SEC-29 | One-time codes are hashed, expire (T12), allow limited attempts, and are single-use | `linking_challenges` table | Tests for expiry, reuse, and attempt limit |
| SEC-30 | Email match alone never links an account | Linking service rules | Test: registering with an existing member's email does not link |
| SEC-31 | One login per member record; one member record per login per library | Unique constraints in `identity_links` | Test: duplicate link attempts rejected |
| SEC-32 | Conflicts go to librarian approval, with no automatic override | Approval queue in console | Test: conflicting link is queued, not applied |

### 3.7 Librarians and operators

| ID | Control | Enforcement | Test |
|---|---|---|---|
| SEC-33 | Librarian SSO with MFA enforced in each library's Salesforce org | Org login settings | Provisioning checklist |
| SEC-34 | Librarian access removed on last day (SCIM or documented offboarding) | Offboarding runbook; SCIM where available | Quarterly access review |
| SEC-35 | Library Admin permission set required for rules, seats, rooms, and permissions | Permission sets | Tests on Apex and LWC |
| SEC-36 | Platform operator support access requires an approved ticket, is logged, and is time-limited | Support access process; `audit_events` | Quarterly audit of support access |

### 3.8 Secrets, data, and operations

| ID | Control | Enforcement | Test |
|---|---|---|---|
| SEC-37 | Secrets stored only in the secrets store, scoped per library; never in code, images, or logs | Secrets manager and CI secret scanning | CI secret scan on every PR |
| SEC-38 | Secrets rotated on a schedule: certificates and signing keys (TBD), PayU salts when requested | Rotation runbook | Rotation drill before launch |
| SEC-39 | Logs redact personal data, tokens, and secrets | Logger redaction rules | Tests on log output for sample requests |
| SEC-40 | Backups encrypted; restore tested quarterly | Backup configuration | Restore test recorded in runbook |
| SEC-41 | Dependencies pinned, scanned in CI, and updated for critical vulnerabilities within 14 days (proposed) | Lockfiles; dependency scanner in CI | CI scan |
| SEC-42 | Rate limits on login, linking codes, and agent calls | Node API limiter | Rate-limit tests |
| SEC-43 | Payment callbacks verified by signature and server-side status check before any change | PayU module | PayU test vectors and duplicate-callback tests |

## 4. Known gaps and decisions still open

- **Session-check callout from Agentforce.** Confirm during T2 that agent actions can make this call and how the reference is passed. If not, the fallback is a short-lived signed reference with the same checks, and this must be recorded as a risk acceptance.
- **Breached-password checking** depends on the Keycloak version and the plugin or provider chosen. Verify before build.
- **Certificate and signing key rotation periods** (SEC-38) are TBD.
- **Outbound IP ranges** for the Salesforce org (SEC-24) depend on the hosting provider and must be confirmed.

## 5. Release checklist

Before any library goes live:

- [ ] Threat model reviewed and updated for any new feature
- [ ] All SEC controls above have passing tests or a documented exception
- [ ] Tenant-isolation and member-isolation tests pass on every endpoint
- [ ] Agent evaluation set includes permission and prompt-injection scenarios, and passes the thresholds in SPEC §10.7
- [ ] Keycloak realm settings reviewed and exported to `config/keycloak/`
- [ ] Connected App IP restrictions and certificate verified in the library org
- [ ] Secrets rotated or confirmed fresh for the library
- [ ] Backup restore tested within the last quarter
- [ ] External penetration test completed, findings fixed or accepted in writing
- [ ] OWASP ASVS review completed
- [ ] Librarian SSO and MFA verified; offboarding owner named

## 6. Sign-off

| Role | Name | Date |
|---|---|---|
| Security owner | | |
| Engineering lead | | |
| Product owner | | |
