# Authentication and Authorization — Phase 0 Assessment

Meta
- Status: Draft
- Owner: TBD (BA / Lead Dev / Security Lead)
- Last review: YYYY-MM-DD
- Feature flag: feature.auth.*
- Audience: Engineering, Security, Mobile
- Language: EN (docs), RU (tasks/discussions)

Summary
We will implement secure user authentication and authorization for the mobile app using AWS Cognito as the primary identity provider. Initial scope covers: email/password registration with 6‑digit email verification, login for existing users, Google and Apple sign‑in, password recovery, and logout. Tokens: Cognito access/id/refresh with secure storage on device and PKCE for mobile.

In Scope (current increment)
- Email registration + 6‑digit email verification
- Email login for existing users
- Password recovery (forgot password)
- Logout (token revoke / global sign out)
- OAuth: Google and Apple via Cognito federation
- Session renewal via refresh token (stored securely on device)

Out of Scope (future increments)
- Phone number registration + SMS verification
- Facebook OAuth
- Two‑factor authentication (TOTP) + backup codes
- Biometric authentication (local unlock of secure storage)
- Change email / change password from settings

Architecture and Integration Map
- Identity: AWS Cognito User Pool (primary), App Clients (iOS/Android/Web)
- Federated IdPs: Google (OIDC), Apple ID (Sign in with Apple)
- Hosted UI: Cognito Hosted UI domain with strict redirect URIs
- Email: Amazon SES for verification and password reset templates
- Mobile: OAuth 2.0 Authorization Code + PKCE; secure token storage (Keychain/Keystore)
- Optional BFF (TBD): claims normalization, auditing, policy enforcement
- Observability: auth metrics, error rates, latency dashboards (SLOs TBD)
- Environments: dev / stage / prod (domains, Hosted UI config, SES identities)

Token and Session Model (Cognito)
- Tokens: access_token (short‑lived), id_token, refresh_token (longer‑lived)
- Storage (mobile): refresh_token in secure storage (Keychain/Keystore); access/id kept in memory
- Rotation: refresh token rotation; revoke on logout/global sign out
- Proposed TTLs (TBD with Security/Platform):
  - access_token: 10–15 minutes
  - id_token: 10–60 minutes
  - refresh_token: 7–30 days
- Logout semantics: local token wipe + Cognito revoke/GlobalSignOut
- Password hashing: managed by Cognito (no custom bcrypt/argon2 in our code)

Security Considerations and Risks
- PKCE mandatory for all mobile OAuth flows; no client_secret on device
- Strict allowlist of redirect URIs, iOS bundle IDs and Android SHA certs
- Secure token storage; protect debug builds from leaking tokens/logs
- Rate limiting for verification/reset attempts; abuse prevention (captcha/throttling)
- Validate state/nonce; prevent redirect spoofing; exact redirect matching
- Session hijack: implement refresh revoke and device/session inventory (future)
- Audit logging: register/verify/login/forgot/logout/revoke events
- Compliance/Privacy: PII minimization; secrets management; log redaction

User Flows (high level)
- Register with email → receive 6‑digit code → verify → authenticated session
- Login with email/password → tokens issued → session persisted via refresh
- Forgot password → email code → set new password → login
- Google Sign‑In / Apple ID → Cognito federation → tokens issued
- Logout → local wipe + revoke/GlobalSignOut

Non‑Functional Requirements (NFRs)
- Availability: ≥ 99.9% (leverages Cognito SLA)
- Security: OAuth/PKCE compliance; no secrets in app binaries
- Performance: p95 auth < 500 ms (excluding external IdP latency)
- Observability: dashboards (errors/latency/success rates by flow), alerts
- Accessibility (if Hosted UI used): adhere to org A11y standard

Open Questions (to resolve in ADR/design)
- Client approach: Hosted UI vs native SDK (per platform) for UX and security?
- Need a BFF for claims normalization/audit/policy enforcement?
- Final token TTLs and forced logout semantics (global sign out)?
- Hosted UI branding and custom domain requirements?
- SES templates (copy/localization) and rate limits?

Feature Flags and Rollout
- Feature flag key: feature.auth.*
- Environments: OFF in prod by default; ON in dev/stage
- Rollout plan: 5% → 25% → 50% → 100% with rollback triggers:
  - Error rate > 1–5%
  - p95 latency > threshold
  - Auth failures spike or incident declared
- Evidence: link rollout dashboards and alarms in release PR

Acceptance Criteria (for this Assessment)
- Integration map completed (Cognito, IdPs, Hosted UI, SES, mobile PKCE)
- Key risks identified with mitigations
- Feature flag and staged rollout plan drafted
- References block filled; owners assigned

Owners and RACI (initial)
- BA: Assessment, requirements consolidation
- Lead Dev: Architecture, ADR, final approval
- Security Lead: Security review/sign‑off
- Mobile Lead (TBD): PKCE implementation and secure storage
- Platform/Infra: SES, domains, CI/CD gates

References (canonical and internal)
- Standards (canonical): Security, OAuth/SSO, Data Privacy, Observability, Feature Flags — central standards repo (TBD)
- docs/decisions/adr-00x-authentication.md (to be created)
- docs/security/authentication.md (Cognito integration overview; to be created)
- reference/integration/cognito-configuration.md (to be created)
- reference/api/auth-openapi.yaml (only if BFF/API is introduced; to be created)
- runbooks/deployment-auth.md (to be created)
- features/auth/requirements.md, features/auth/design.md, features/auth/testing.md (to be created)

Appendix: Environment Checklist (seed)
- [ ] User Pool created for dev/stage/prod
- [ ] App Clients (iOS/Android/Web) with PKCE enabled
- [ ] Hosted UI domain set; redirect URIs registered
- [ ] SES domain verified; email templates created (verify/reset)
- [ ] Google and Apple IdPs configured (client IDs, keys, team IDs, bundle IDs, Android SHA)
- [ ] Logging/monitoring configured; dashboards stubbed
