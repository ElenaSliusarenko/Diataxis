title: Authentication and Authorization — Phase 0 Assessment
status: draft
owner: TBD (BA / Lead Dev / Security Lead)
last_review: YYYY-MM-DD
feature_flag: feature.auth.*
audience: Engineering, Security, Mobile
language: en
Authentication and Authorization — Phase 0 Assessment
Summary
We will implement secure user authentication and authorization for the mobile app using AWS Cognito as the primary identity provider. Initial scope includes: email/password registration with 6‑digit email verification, login for existing users, Google and Apple sign‑in, password recovery, and logout. Tokens: Cognito access/id/refresh with secure storage on device and PKCE for mobile.

In Scope (current increment)

Email registration + 6‑digit email verification
Email login for existing users
Password recovery (forgot password)
Logout (token revoke / global sign out)
OAuth: Google and Apple via Cognito federation
Session renewal via refresh token (secure storage on device)
Out of Scope (future increments)

Phone number registration + SMS verification
Facebook OAuth
Two‑factor authentication (TOTP) and backup codes
Biometric authentication (local unlock of secure storage)
Change email / change password from settings
Architecture and Integration Map

Identity: AWS Cognito User Pool (primary), App Clients (iOS/Android/Web)
Federated IdPs: Google (OIDC), Apple ID (Apple Sign‑in)
Hosted UI: Cognito Hosted UI domain with strict redirect URIs
Email: Amazon SES for verification and password reset templates
Mobile: OAuth 2.0 Authorization Code + PKCE; secure token storage (Keychain/Keystore)
Optional BFF (TBD): claims normalization, auditing, policy enforcement
Observability: auth metrics, error rates, latency dashboards (TBD SLOs)
Environments: dev / stage / prod (domains, Hosted UI config, SES identities)
Token and Session Model (Cognito)

Tokens: access_token (short‑lived), id_token, refresh_token (longer‑lived)
Storage (mobile): refresh_token in secure storage (Keychain/Keystore); access/id kept in memory
Rotation: refresh token rotation supported; revoke on logout/global sign out
Proposed TTLs (TBD with Security/Platform):
access_token: 10–15 minutes
id_token: 10–60 minutes (as needed by client)
refresh_token: 7–30 days
Logout semantics: local token wipe + Cognito revoke/GlobalSignOut
Password hashing: managed by Cognito; no custom bcrypt/argon2 in our code
Security Considerations and Risks

PKCE mandatory for all mobile OAuth flows; no client_secret on device
Strict allowlist of redirect URIs, bundle IDs (iOS) and SHA certs (Android)
Secure token storage; guard against debug builds leaking tokens/logs
Rate limiting for verification/reset attempts; abuse prevention (captcha, throttling)
Email content security: no sensitive data; consistent templates and brand
IdP risks: misconfigured scopes or redirect spoofing → validate state/nonce, enforce exact redirect match
Session hijack: implement refresh revoke and device/session inventory (future)
Audit logging: key auth events (register, verify, login, forgot, logout, revoke)
Compliance/Privacy: PII minimization; secrets management; log redaction
User Flows (high level)

Register with email → receive 6‑digit code → verify → authenticated session
Login with email/password → tokens issued → session persisted via refresh
Forgot password → email code → set new password → login
Google Sign‑In / Apple ID → Cognito federation → tokens issued
Logout → local wipe + revoke/GlobalSignOut
Non‑Functional Requirements (NFRs)

Availability: ≥ 99.9% (leverages Cognito SLA)
Security: OAuth/PKCE compliance; secrets never stored in app code
Performance: p95 end‑to‑end auth < 500 ms (excluding external IdP latency)
Observability: dashboards for error rate, latency, success rates by flow
Accessibility (if UI hosted): follow existing A11y standards for Hosted UI pages
Open Questions (to resolve in ADR/design)

Client approach: Hosted UI vs native SDK flows (per platform) for best UX and security?
Do we need a BFF for claims normalization and centralized audit?
Final token TTLs and forced logout semantics (global sign‑out strategy)?
Hosted UI branding and custom domain requirements?
Exact SES templates (copy, localization) and rate limits?
Feature Flags and Rollout

Feature flag key: feature.auth.*
Environments: OFF in prod by default; ON in dev/stage
Rollout plan: 5% → 25% → 50% → 100% with rollback triggers (error rate > 1–5%, p95 > threshold, auth failures spike)
Evidence: link rollout dashboards and alarms in release PR
Acceptance Criteria (for this Assessment doc)

Integration map completed (Cognito, IdPs, Hosted UI, SES, mobile PKCE)
Risks identified with mitigation notes
Feature flag and staged rollout plan drafted
References block filled and owners assigned
Owners and RACI (initial)

BA: Assessment, requirements consolidation
Lead Dev: Architecture, ADR, final approval
Security Lead: Security review/sign‑off
Mobile Lead (TBD): PKCE implementation and secure storage
Platform/Infra: SES, domains, CI/CD gates
References (canonical and internal)

Standards (canonical): Security, OAuth/SSO, Data Privacy, Observability, Feature Flags — link to central standards repo (TBD)
docs/decisions/adr-00x-authentication.md (to be created)
docs/security/authentication.md (Cognito integration overview; to be created)
reference/integration/cognito-configuration.md (to be created)
reference/api/auth-openapi.yaml (only if BFF/API is introduced; to be created)
runbooks/deployment-auth.md (to be created)
features/auth/requirements.md, features/auth/design.md, features/auth/testing.md (to be created)
Appendix: Environment Checklist (seed)

User Pool created for dev/stage/prod
App Clients (iOS/Android/Web) with PKCE enabled
Hosted UI domain set; redirect URIs registered
SES domain verified; email templates created (verify/reset)
Google and Apple IdPs configured (client IDs, keys, team IDs, bundle IDs, Android SHA)
Logging/monitoring routes configured; dashboards stubbed
