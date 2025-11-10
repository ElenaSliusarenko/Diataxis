## Auth feature — Implementation Tasks

## Checklist — Core tasks (MVP)

- [ ] 1) Configure Cognito User Pools (dev/stage/prod)  
  - Owner: Platform/Infra (TBD)  
  - Estimate: M (0.5–1d)  
  - Risk: R2  
  - Feature flag: feature.auth.infra  
  - Acceptance Criteria:
    - User Pool(s) created for dev/stage/prod
    - App Clients (iOS/Android/Web) enabled with Authorization Code Grant + PKCE
    - Hosted UI domain configured; redirect URIs registered
    - Password policy and email verification flow configured
    - Document created/updated: [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)

- [ ] 2) Configure SES and email templates (verification / password reset)  
  - Owner: Platform/Infra (TBD)  
  - Estimate: S (0.5d)  
  - Risk: R2  
  - Feature flag: feature.auth.email_templates  
  - Acceptance Criteria:
    - SES domain/identities verified for dev/stage/prod
    - Email templates (verification, reset) created and approved by UX
    - Test emails delivered in dev for registration/forgot flows
    - Delivery logs available for audit

- [ ] 3) Implement Email registration + verification (backend + mobile integration)  
  - Owner: FS Dev + Mobile Dev (TBD)  
  - Estimate: M (1–3d)  
  - Risk: R1  
  - Feature flag: feature.auth.email_registration  
  - Acceptance Criteria:
    - Email registration creates user in Cognito and sends a 6‑digit code
    - Code TTL = 10 minutes (configurable); max attempts = 5
    - After verification, id/access/refresh tokens are issued by Cognito
    - Generic error messages (no PII exposure) and audit logging in place

- [ ] 4) Implement Email login + account lockout / rate limiting  
  - Owner: FS Dev + Mobile Dev  
  - Estimate: M (1–2d)  
  - Risk: R1  
  - Feature flag: feature.auth.email_login  
  - Acceptance Criteria:
    - Successful login returns id/access/refresh tokens
    - Failed logins increment rate‑limit counters; progressive delay/lockout applied
    - Login attempts logged to audit stream

- [ ] 5) Implement Forgot password (reset via email code)  
  - Owner: FS Dev + Mobile Dev  
  - Estimate: S (0.5–1d)  
  - Risk: R1  
  - Feature flag: feature.auth.forgot_password  
  - Acceptance Criteria:
    - Password reset request sends 6‑digit code to verified email
    - Code TTL configurable (suggest 10 minutes)
    - After successful reset, previous refresh tokens are invalidated (revoked)
    - Rate limiting and anti‑abuse rules applied

- [ ] 6) Logout + token revoke / GlobalSignOut flow  
  - Owner: FS Dev + Mobile Dev  
  - Estimate: S (0.5–1d)  
  - Risk: R1  
  - Feature flag: feature.auth.logout  
  - Acceptance Criteria:
    - Logout performs local secure storage wipe and calls Cognito revoke / GlobalSignOut
    - Previously issued refresh tokens cannot be used post‑logout
    - Logout event recorded in audit logs

- [ ] 7) Mobile: PKCE integration & secure storage guidance  
  - Owner: Mobile Lead  
  - Estimate: M (1–2d)  
  - Risk: R1  
  - Feature flag: feature.auth.mobile_pkce  
  - Acceptance Criteria:
    - Authorization Code + PKCE implemented and tested end‑to‑end
    - Developer guidance added for Keychain (iOS) / Keystore (Android): `/features/auth/mobile-integration.md`
    - No client_secret stored in apps; no token leakage in logs

- [ ] 8) Integrate Google IdP in Cognito (dev)  
  - Owner: Platform/Infra + FS Dev  
  - Estimate: S (0.5–1d)  
  - Risk: R2  
  - Feature flag: feature.auth.google_sso  
  - Acceptance Criteria:
    - Google OAuth app registered; IdP configured in Cognito
    - Hosted UI flow with Google works in dev; profile attributes mapped

- [ ] 9) Integrate Apple Sign‑in in Cognito (dev)  
  - Owner: Platform/Infra + Mobile Dev  
  - Estimate: S (0.5–1d)  
  - Risk: R2  
  - Feature flag: feature.auth.apple_sso  
  - Acceptance Criteria:
    - Apple Sign‑in (Service ID, Key, Team ID) configured and tested
    - Hosted UI flow with Apple works on iOS using PKCE; email handling per Apple rules

- [ ] 10) Docs & Diátaxis artifacts (How‑to, Tutorial, Reference, Explanation)  
  - Owner: QA2 / Tech Writer (TBD)  
  - Estimate: S (1–2d)  
  - Risk: R3  
  - Feature flag: feature.auth.docs  
  - Acceptance Criteria:
    - How‑to: register/login/SSO/forgot/logout prepared under `/how-to/...`
    - Reference: Cognito config doc `/reference/integration/cognito-configuration.md` and API docs if BFF exists
    - Explanation: token model and security rationale documented
    - All docs include the required References block: [/features/auth/assessment.md](/features/auth/assessment.md), [/docs/decisions/ADR-001-authentication.md](/docs/decisions/ADR-001-authentication.md)

- [ ] 11) Tests: unit, integration, e2e (critical paths)  
  - Owner: QA / Devs  
  - Estimate: M (1–2d)  
  - Risk: R1  
  - Feature flag: feature.auth.tests  
  - Acceptance Criteria:
    - E2E: register→verify→login green; forgot, logout, SSO e2e passing
    - PKCE replay/expiry tests, refresh rotation & revoke tests included
    - Security checks: SAST, dependency scan, secrets scan pass in CI

- [ ] 12) Observability & SLOs / Dashboards  
  - Owner: SRE / Platform  
  - Estimate: S (0.5–1d)  
  - Risk: R2  
  - Feature flag: feature.auth.observability  
  - Acceptance Criteria:
    - Dashboards: auth success rate, error rate, latency per flow
    - Alerts configured for rollback triggers (error rate threshold, p95 latency)
    - Dashboard links included in PR / release notes

- [ ] 13) Runbook & Rollout plan (staged)  
  - Owner: Lead Dev / SRE  
  - Estimate: S (0.5d)  
  - Risk: R2  
  - Feature flag: feature.auth.rollout  
  - Acceptance Criteria:
    - Runbook: deployment, rollback, incident steps (`/runbooks/deployment-auth.md`)
    - Rollout plan documented: 5% → 25% → 50% → 100% with rollback criteria
    - Feature flag `feature.auth.*` created and toggleable per environment

## Optional / Future (post‑MVP)
- [ ] Phone registration + SMS verification — feature.auth.phone_registration (R2)  
- [ ] Facebook OAuth — feature.auth.facebook_sso (R2)  
- [ ] 2FA (TOTP) + backup codes — feature.auth.2fa (R1)  
- [ ] Biometric local unlock (UX convenience) — feature.auth.biometric (R3)  
- [ ] Account settings: change email / change password — feature.auth.account_settings (R2)

---

## DoR checklist (for each implementation issue)
- [ ] Phase‑0 assessment complete: [/features/auth/assessment.md](/features/auth/assessment.md)  
- [ ] ADR accepted: [/docs/decisions/ADR-001-authentication.md](/docs/decisions/ADR-001-authentication.md)  
- [ ] Owner assigned  
- [ ] Infra ticket created (if required)  
- [ ] Feature flag defined (feature.auth.*)  
- [ ] Test plan drafted

## References (required block)
- [/features/auth/assessment.md](/features/auth/assessment.md)  
- [/docs/decisions/ADR-001-authentication.md](/docs/decisions/ADR-001-authentication.md)  
- Project standards: Security / OAuth/SSO / Observability
