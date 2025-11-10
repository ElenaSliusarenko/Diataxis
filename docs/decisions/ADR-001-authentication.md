# ADR-001 — Authentication architecture: AWS Cognito (Hosted UI)

**Status**: Proposed 
**Date**: November 10, 2025
**Deciders**: Architecture Team, Engineering Leadership  
**Related**: All features, System Architecture  
**Supersedes**: -

---
## Context
We need to implement secure user authentication and authorization for the mobile application. Phase‑0 assessment exists at [/features/auth/assessment.md](/features/auth/assessment.md) and defines the scope (email/password registration with 6‑digit email verification, email login, forgot password, logout, Google and Apple sign‑in, Cognito tokens and refresh) and constraints (use AWS Cognito as primary IdP). This ADR records the architectural decision about client integration approach and related operational choices.

## Decision
We will use AWS Cognito User Pools as the primary identity provider and integrate clients with Cognito using the Cognito **Hosted UI** for OAuth/SSO flows. Mobile clients will use the Authorization Code flow with PKCE when interacting with Cognito. We will **not** introduce a BFF/edge service at this time; clients will interact directly with Cognito (via Hosted UI and SDKs as needed). Password storage and verification remain managed by Cognito.

Key points:
- Use Cognito Hosted UI for sign-in / sign-up / SSO (Google, Apple) to centralize flows and reduce client complexity.
- Enforce Authorization Code + PKCE for all mobile/native clients.
- Do not store client_secret in mobile apps.
- Use Cognito access/id/refresh tokens; refresh tokens stored in secure device storage (Keychain/Keystore).
- BFF/edge layer: deferred (No BFF now). If later needed for claims normalization, auditing, or advanced policies, introduce as separate ADR.

## Rationale
- Hosted UI reduces implementation surface on clients, centralizes security flows, and simplifies SSO provider configuration and branding (faster delivery, fewer client-side bugs).
- PKCE provides strong protection for native/mobile apps where client_secret cannot be safely stored.
- Avoiding a BFF reduces initial scope and operational burden; Cognito provides token management and federation capabilities suitable for MVP increment.
- Keeping passwords and hashing managed by Cognito minimizes security responsibilities and compliance scope for our service.

## Consequences
Positive:
- Faster integration and delivery for SSO and password flows.
- Lower risk of incorrect password handling or hashing logic in our codebase.
- Centralized email/template management through SES + Cognito templates.

Negative / Trade-offs:
- Greater vendor lock‑in to AWS Cognito features and behaviors.
- Less direct control over token formats/claims (can be mitigated with attribute mapping and custom claims where supported).
- If later we require server‑side claims normalization, auditing, or complex session policies, we will need to introduce a BFF and migrate flows — costs and migration path must be planned.

## Alternatives considered
1. Custom auth service (self‑managed passwords with bcrypt/argon2)
   - Pros: Full control over user model and tokens.
   - Cons: Increased security and compliance burden; slower delivery.
2. Cognito Native SDKs with custom UI (no Hosted UI)
   - Pros: Fully branded client UI, more control over UX.
   - Cons: More client complexity and more surface for auth bugs; still requires PKCE for mobile.
3. Introduce BFF/edge now
   - Pros: Centralized claim normalization, single audit point.
   - Cons: Additional service to run and secure; extends scope and delay.

Decision selects Hosted UI + PKCE + no BFF for current increment as best trade‑off for speed, security, and maintainability.

## Implementation Notes (next steps)
- Create / configure Cognito User Pool(s) for dev/stage/prod.
- Create App Clients for mobile platforms and web; enable Authorization Code Grant and PKCE for mobile clients.
- Configure Cognito Hosted UI domain(s); register exact redirect URIs (strict allowlist), bundle IDs and Android SHA fingerprints for mobile.
- Configure SES (or verify SES integration) and email templates for verification and password reset.
- Register and configure IdPs: Google (OIDC) and Apple Sign in (Service ID, Key, Team ID).
- Define token TTLs (access / id / refresh) in collaboration with Security Lead (suggested starting values in assessment).
- Implement secure storage guidance for mobile teams (Keychain/Keystore) and document it in `features/auth/mobile-integration.md`.
- Add audit events for key auth operations (register, verify, login, forgot, logout, revoke).
- Add feature flag `feature.auth.*` and ensure flows are disabled in production by default until rollout.

## Rollout / Feature Flagging
- Use `feature.auth.*` to gate behavioral changes. Default OFF in production.
- Staged rollout: 5% → 25% → 50% → 100% with rollback criteria (error rate, p95 latency, auth failure spike).
- Include metrics and dashboards for auth success rate, error rate, and latency before rollout.

## Acceptance Criteria
- ADR accepted by Lead Dev and Security Lead (sign‑off recorded).
- Cognito configuration (dev) created and validated: test user registration, email verification, sign‑in (email/password), forgot password, and SSO via Google and Apple using Hosted UI + PKCE.
- Mobile PKCE flows verified end‑to‑end; no client_secret persisted on device.
- SES templates tested; verification and reset emails sent/received successfully.
- Feature flag `feature.auth.*` created and honored in environments.
- Documented migration path if BFF/edge is required in the future.

## Implementation Owner / RACI
- Owner: Andrey Bakaev (FS developer) — ADR owner and initial implementation lead
- Security Lead: security review and signoff
- Mobile Lead: mobile integration and PKCE implementation
- Platform/Infra: Cognito, SES, domain configuration
- BA: acceptance criteria & docs

## References
- features/auth/assessment.md
- Project security / OAuth standards (link to canonical standard files per repo)
- Diataxis & Assessment‑First patterns (see project docs / process)
